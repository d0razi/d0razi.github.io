---
title: "ejs.render() prototype pollution 취약점 분석"
author: d0razi
date: 2024-01-20 11:43
categories: [InfoSec, Web]
tags: [research]
image: /assets/img/media/banner/Hacker.jpg
---

```js
// Setup app
const express = require("express");
const app  = express();
const port = 3000;

// Select ejs templating library
app.set('view engine', 'ejs');

// Routes
app.get("/", (req, res) => {
	res.render("index");
})

app.get("/vuln", (req, res) => {
	// simulate SSPP vulnerability
	var a = req.query.a;
	var b = req.query.b;
	var c = req.query.c;

	var obj = {};
	obj[a][b] = c; // Hack!!

	res.send("OK!");
})

// Start app
app.listen(port, () => {
	console.log(`App listening on port ${port}`)
})
```

위와 같이 prototype pollution이 가능하고 render() 함수가 호출되는 상황에 발생하는 취약점입니다.



어떤 prototype을 name에 입력해야 RCE를 할 수 있을지 찾던 중 멘토님이 아래 코드에 render를 분석해보라고 알려주셔서 render를 분석해봤습니다.

```js
app.get("/", (req, res) => {
	res.render("index");
})
```


`node_modules/ejs/ejs.js` 파일에 render 함수 코드입니다.
```js
exports.render = function (template, d, o) {
	var data = d || utils.createNullProtoObjWherePossible();
	var opts = o || utils.createNullProtoObjWherePossible();

	// No options object -- if there are optiony names
	// in the data, copy them to options
	if (arguments.length == 2) {
		utils.shallowCopyFromList(opts, data, _OPTS_PASSABLE_WITH_DATA);
	}

	return handleCache(opts, template)(data);
};
```

각 함수들 코드를 오디팅 하던 중 handleCache() 함수에서 client와 escapeFuntion이 사용되는 Template을 호출하는 compile을 찾았습니다.
```js
function handleCache(options, template) {
	var func;
	var filename = options.filename;
	var hasTemplate = arguments.length > 1;

	if (options.cache) {
		if (!filename) {
			throw new Error('cache option requires a filename');
		}
		func = exports.cache.get(filename);
		if (func) {
			return func;
		}
		if (!hasTemplate) {
			template = fileLoader(filename).toString().replace(_BOM, '');
		}
	}
	else if (!hasTemplate) {
		// istanbul ignore if: should not happen at all
		if (!filename) {
		    throw new Error('Internal EJS error: no file name or template ' + 'provided');
		}
		template = fileLoader(filename).toString().replace(_BOM, '');
	}
	func = exports.compile(template, options);
	if (options.cache) {
		exports.cache.set(filename, func);
	}
	return func;
}
```

```js
exports.compile = function compile(template, opts) {

	var templ;

	// v1 compat
	// 'scope' is 'context'
	// FIXME: Remove this in a future version
	if (opts && opts.scope) {
		if (!scopeOptionWarned){
			console.warn('`scope` option is deprecated and will be removed in EJS 3');
			scopeOptionWarned = true;
		}
		if (!opts.context) {
		opts.context = opts.scope;
		}
		delete opts.scope;
	}
	templ = new Template(template, opts);
	return templ.compile();
};
```

```js
function Template(text, opts) {
	opts = opts || utils.createNullProtoObjWherePossible();
	var options = utils.createNullProtoObjWherePossible();
	this.templateText = text;
	/** @type {string | null} */
	this.mode = null;
	this.truncate = false;
	this.currentLine = 1;
	this.source = '';
	options.client = opts.client || false;
	options.escapeFunction = opts.escape || opts.escapeFunction || utils.escapeXML;
	options.compileDebug = opts.compileDebug !== false;
	options.debug = !!opts.debug;
	options.filename = opts.filename;
	options.openDelimiter = opts.openDelimiter || exports.openDelimiter || _DEFAULT_OPEN_DELIMITER;
	options.closeDelimiter = opts.closeDelimiter || exports.closeDelimiter || _DEFAULT_CLOSE_DELIMITER;
	options.delimiter = opts.delimiter || exports.delimiter || _DEFAULT_DELIMITER;
	options.strict = opts.strict || false;
	options.context = opts.context;
	options.cache = opts.cache || false;
	options.rmWhitespace = opts.rmWhitespace;
	options.root = opts.root;
	options.includer = opts.includer;
	options.outputFunctionName = opts.outputFunctionName;
	options.localsName = opts.localsName || exports.localsName || _DEFAULT_LOCALS_NAME;
	options.views = opts.views;
	options.async = opts.async;
	options.destructuredLocals = opts.destructuredLocals;
	options.legacyInclude = typeof opts.legacyInclude != 'undefined' ? !!opts.legacyInclude : true;

    if (options.strict) {
    	options._with = false;
    }
    else {
    	options._with = typeof opts._with != 'undefined' ? opts._with : true;
    }

    this.opts = options;

    this.regex = this.createRegex();
}
```

위 함수를 분석해봤는데 딱히 client와 escapeFuntion prototype을 조작한다고 해서 RCE를 발생시킬 수 있는 부분을 찾지 못했습니다.

그래서 쭉 코드 오디팅을 하다가 `Template.prototype` 오브젝트를 찾았습니다.
```js
Template.prototype = {
    createRegex: function () {
    var str = _REGEX_STRING;
    var delim = utils.escapeRegExpChars(this.opts.delimiter);
    var open = utils.escapeRegExpChars(this.opts.openDelimiter);
    var close = utils.escapeRegExpChars(this.opts.closeDelimiter);
    str = str.replace(/%/g, delim)
		.replace(/</g, open)
		.replace(/>/g, close);
    return new RegExp(str);
    },

    compile: function () {
    /** @type {string} */
    var src;
    /** @type {ClientFunction} */
    var fn;
    var opts = this.opts;
    var prepended = '';
    var appended = '';
    /** @type {EscapeCallback} */
    var escapeFn = opts.escapeFunction;
    /** @type {FunctionConstructor} */
    var ctor;
    /** @type {string} */
    var sanitizedFilename = opts.filename ? JSON.stringify(opts.filename) : 'undefined';

    if (!this.source) {
        this.generateSource();
        prepended +=
        '  var __output = "";\n' +
        '  function __append(s) { if (s !== undefined && s !== null) __output += s }\n';
        if (opts.outputFunctionName) {
			if (!_JS_IDENTIFIER.test(opts.outputFunctionName)) {
				throw new Error('outputFunctionName is not a valid JS identifier.');
			}
			prepended += '  var ' + opts.outputFunctionName + ' = __append;' + '\n';
        }
        if (opts.localsName && !_JS_IDENTIFIER.test(opts.localsName)) {
        	throw new Error('localsName is not a valid JS identifier.');
        }
        if (opts.destructuredLocals && opts.destructuredLocals.length) {
			var destructuring = '  var __locals = (' + opts.localsName + ' || {}),\n';
			for (var i = 0; i < opts.destructuredLocals.length; i++) {
				var name = opts.destructuredLocals[i];
				if (!_JS_IDENTIFIER.test(name)) {
					throw new Error('destructuredLocals[' + i + '] is not a valid JS identifier.');
				}
				if (i > 0) {
					destructuring += ',\n  ';
				}
				destructuring += name + ' = __locals.' + name;
			}
			prepended += destructuring + ';\n';
        }
        if (opts._with !== false) {
			prepended +=  '  with (' + opts.localsName + ' || {}) {' + '\n';
			appended += '  }' + '\n';
        }
        appended += '  return __output;' + '\n';
        this.source = prepended + this.source + appended;
    }

    if (opts.compileDebug) {
        src = 'var __line = 1' + '\n'
        + '  , __lines = ' + JSON.stringify(this.templateText) + '\n'
        + '  , __filename = ' + sanitizedFilename + ';' + '\n'
        + 'try {' + '\n'
        + this.source
        + '} catch (e) {' + '\n'
        + '  rethrow(e, __lines, __filename, __line, escapeFn);' + '\n'
        + '}' + '\n';
    }
    else {
        src = this.source;
    }

    if (opts.client) {
        src = 'escapeFn = escapeFn || ' + escapeFn.toString() + ';' + '\n' + src;
        if (opts.compileDebug) {
        	src = 'rethrow = rethrow || ' + rethrow.toString() + ';' + '\n' + src;
        }
    }

    if (opts.strict) {
        src = '"use strict";\n' + src;
    }
    if (opts.debug) {
        console.log(src);
    }
    if (opts.compileDebug && opts.filename) {
        src = src + '\n'
        + '//# sourceURL=' + sanitizedFilename + '\n';
    }

    try {
        if (opts.async) {
			// Have to use generated function for this, since in envs without support,
			// it breaks in parsing
			try {
				ctor = (new Function('return (async function(){}).constructor;'))();
			}
			catch(e) {
				if (e instanceof SyntaxError) {
					throw new Error('This environment does not support async/await');
				}
				else {
					throw e;
				}
			}
        }
        else {
        	ctor = Function;
        }
        fn = new ctor(opts.localsName + ', escapeFn, include, rethrow', src);
    }
    catch(e) {
        // istanbul ignore else
        if (e instanceof SyntaxError) {
			if (opts.filename) {
				e.message += ' in ' + opts.filename;
			}
			e.message += ' while compiling ejs\n\n';
			e.message += 'If the above error is not helpful, you may want to try EJS-Lint:\n';
			e.message += 'https://github.com/RyanZim/EJS-Lint';
			if (!opts.async) {
				e.message += '\n';
				e.message += 'Or, if you meant to create an async function, pass `async: true` as an option.';
			}
        }
        throw e;
    }

    // Return a callable function which will execute the function
    // created by the source-code, with the passed data as locals
    // Adds a local `include` function which allows full recursive include
    var returnedFn = opts.client ? fn : function anonymous(data) {
        var include = function (path, includeData) {
			var d = utils.shallowCopy(utils.createNullProtoObjWherePossible(), data);
			if (includeData) {
				d = utils.shallowCopy(d, includeData);
			}
			return includeFile(path, opts)(d);
        };
        return fn.apply(opts.context,
        [data || utils.createNullProtoObjWherePossible(), escapeFn, include, rethrow]);
    };
    if (opts.filename && typeof Object.defineProperty === 'function') {
        var filename = opts.filename;
        var basename = path.basename(filename, path.extname(filename));
        try {
			Object.defineProperty(returnedFn, 'name', {
				value: basename,
				writable: false,
				enumerable: false,
				configurable: true
			});
        } catch (e) {/* ignore */}
    }
    return returnedFn;
    },

    generateSource: function () {
    var opts = this.opts;

    if (opts.rmWhitespace) {
        // Have to use two separate replace here as `^` and `$` operators don't
        // work well with `\r` and empty lines don't work well with the `m` flag.
        this.templateText =
        this.templateText.replace(/[\r\n]+/g, '\n').replace(/^\s+|\s+$/gm, '');
    }

    // Slurp spaces and tabs before <%_ and after _%>
    this.templateText =
        this.templateText.replace(/[ \t]*<%_/gm, '<%_').replace(/_%>[ \t]*/gm, '_%>');

    var self = this;
    var matches = this.parseTemplateText();
    var d = this.opts.delimiter;
    var o = this.opts.openDelimiter;
    var c = this.opts.closeDelimiter;

    if (matches && matches.length) {
        matches.forEach(function (line, index) {
        var closing;
        // If this is an opening tag, check for closing tags
        // FIXME: May end up with some false positives here
        // Better to store modes as k/v with openDelimiter + delimiter as key
        // Then this can simply check against the map
        if ( line.indexOf(o + d) === 0          // If it is a tag
            && line.indexOf(o + d + d) !== 0) { // and is not escaped
            closing = matches[index + 2];
            if (!(closing == d + c || closing == '-' + d + c || closing == '_' + d + c)) {
            	throw new Error('Could not find matching close tag for "' + line + '".');
            }
        }
        self.scanLine(line);
        });
    }

    },

    parseTemplateText: function () {
    var str = this.templateText;
    var pat = this.regex;
    var result = pat.exec(str);
    var arr = [];
    var firstPos;

    while (result) {
        firstPos = result.index;

        if (firstPos !== 0) {
        arr.push(str.substring(0, firstPos));
        str = str.slice(firstPos);
        }

        arr.push(result[0]);
        str = str.slice(result[0].length);
        result = pat.exec(str);
    }

    if (str) {
        arr.push(str);
    }

    return arr;
    },

    _addOutput: function (line) {
    if (this.truncate) {
        // Only replace single leading linebreak in the line after
        // -%> tag -- this is the single, trailing linebreak
        // after the tag that the truncation mode replaces
        // Handle Win / Unix / old Mac linebreaks -- do the \r\n
        // combo first in the regex-or
        line = line.replace(/^(?:\r\n|\r|\n)/, '');
        this.truncate = false;
    }
    if (!line) {
        return line;
    }

    // Preserve literal slashes
    line = line.replace(/\\/g, '\\\\');

    // Convert linebreaks
    line = line.replace(/\n/g, '\\n');
    line = line.replace(/\r/g, '\\r');

    // Escape double-quotes
    // - this will be the delimiter during execution
    line = line.replace(/"/g, '\\"');
    this.source += '      ; __append("' + line + '")' + '\n';
    },

    scanLine: function (line) {
		var self = this;
		var d = this.opts.delimiter;
		var o = this.opts.openDelimiter;
		var c = this.opts.closeDelimiter;
		var newLineCount = 0;
	
		newLineCount = (line.split('\n').length - 1);
	
		switch (line) {
		case o + d:
		case o + d + '_':
			this.mode = Template.modes.EVAL;
			break;
		case o + d + '=':
			this.mode = Template.modes.ESCAPED;
			break;
		case o + d + '-':
			this.mode = Template.modes.RAW;
			break;
		case o + d + '#':
			this.mode = Template.modes.COMMENT;
			break;
		case o + d + d:
			this.mode = Template.modes.LITERAL;
			this.source += '      ; __append("' + line.replace(o + d + d, o + d) + '")' + '\n';
			break;
		case d + d + c:
			this.mode = Template.modes.LITERAL;
			this.source += '      ; __append("' + line.replace(d + d + c, d + c) + '")' + '\n';
			break;
		case d + c:
		case '-' + d + c:
		case '_' + d + c:
			if (this.mode == Template.modes.LITERAL) {
				this._addOutput(line);
			}
	
			this.mode = null;
			this.truncate = line.indexOf('-') === 0 || line.indexOf('_') === 0;
			break;
		default:
			// In script mode, depends on type of tag
			if (this.mode) {
				// If '//' is found without a line break, add a line break.
				switch (this.mode) {
					case Template.modes.EVAL:
					case Template.modes.ESCAPED:
					case Template.modes.RAW:
					if (line.lastIndexOf('//') > line.lastIndexOf('\n')) {
						line += '\n';
					}
				}
				switch (this.mode) {
					// Just executing code
					case Template.modes.EVAL:
						this.source += '      ; ' + line + '\n';
						break;
						// Exec, esc, and output
					case Template.modes.ESCAPED:
						this.source += '      ; __append(escapeFn(' + stripSemi(line) + '))' + '\n';
						break;
						// Exec and output
					case Template.modes.RAW:
						this.source += '      ; __append(' + stripSemi(line) + ')' + '\n';
						break;
					case Template.modes.COMMENT:
						// Do nothing
						break;
						// Literal <%% mode, append as raw output
					case Template.modes.LITERAL:
						this._addOutput(line);
						break;
				}
			}
			// In string mode, just add the output
			else {
				this._addOutput(line);
			}
		}
	
		if (self.opts.compileDebug && newLineCount) {
			this.currentLine += newLineCount;
			this.source += '      ; __line = ' + this.currentLine + '\n';
		}
    }
};
```

위 코드에서 중요한 부분은 `compile: function` 부분입니다.
```js
    compile: function () {
    /** @type {string} */
    var src;
    /** @type {ClientFunction} */
    var fn;
    var opts = this.opts;
    var prepended = '';
    var appended = '';
    /** @type {EscapeCallback} */
    var escapeFn = opts.escapeFunction;
    /** @type {FunctionConstructor} */
    var ctor;
    /** @type {string} */
    var sanitizedFilename = opts.filename ? JSON.stringify(opts.filename) : 'undefined';

    if (!this.source) {
        this.generateSource();
        prepended +=
        '  var __output = "";\n' +
        '  function __append(s) { if (s !== undefined && s !== null) __output += s }\n';
        if (opts.outputFunctionName) {
			if (!_JS_IDENTIFIER.test(opts.outputFunctionName)) {
				throw new Error('outputFunctionName is not a valid JS identifier.');
			}
			prepended += '  var ' + opts.outputFunctionName + ' = __append;' + '\n';
        }
        if (opts.localsName && !_JS_IDENTIFIER.test(opts.localsName)) {
        	throw new Error('localsName is not a valid JS identifier.');
        }
        if (opts.destructuredLocals && opts.destructuredLocals.length) {
			var destructuring = '  var __locals = (' + opts.localsName + ' || {}),\n';
			for (var i = 0; i < opts.destructuredLocals.length; i++) {
				var name = opts.destructuredLocals[i];
				if (!_JS_IDENTIFIER.test(name)) {
					throw new Error('destructuredLocals[' + i + '] is not a valid JS identifier.');
				}
				if (i > 0) {
					destructuring += ',\n  ';
				}
				destructuring += name + ' = __locals.' + name;
			}
			prepended += destructuring + ';\n';
        }
        if (opts._with !== false) {
			prepended +=  '  with (' + opts.localsName + ' || {}) {' + '\n';
			appended += '  }' + '\n';
        }
        appended += '  return __output;' + '\n';
        this.source = prepended + this.source + appended;
    }

    if (opts.compileDebug) {
        src = 'var __line = 1' + '\n'
        + '  , __lines = ' + JSON.stringify(this.templateText) + '\n'
        + '  , __filename = ' + sanitizedFilename + ';' + '\n'
        + 'try {' + '\n'
        + this.source
        + '} catch (e) {' + '\n'
        + '  rethrow(e, __lines, __filename, __line, escapeFn);' + '\n'
        + '}' + '\n';
    }
    else {
        src = this.source;
    }

    if (opts.client) {
        src = 'escapeFn = escapeFn || ' + escapeFn.toString() + ';' + '\n' + src;
        if (opts.compileDebug) {
        	src = 'rethrow = rethrow || ' + rethrow.toString() + ';' + '\n' + src;
        }
    }

    if (opts.strict) {
        src = '"use strict";\n' + src;
    }
    if (opts.debug) {
        console.log(src);
    }
    if (opts.compileDebug && opts.filename) {
        src = src + '\n'
        + '//# sourceURL=' + sanitizedFilename + '\n';
    }

    try {
        if (opts.async) {
			// Have to use generated function for this, since in envs without support,
			// it breaks in parsing
			try {
				ctor = (new Function('return (async function(){}).constructor;'))();
			}
			catch(e) {
				if (e instanceof SyntaxError) {
					throw new Error('This environment does not support async/await');
				}
				else {
					throw e;
				}
			}
        }
        else {
        	ctor = Function;
        }
        fn = new ctor(opts.localsName + ', escapeFn, include, rethrow', src);
    }
    catch(e) {
        // istanbul ignore else
        if (e instanceof SyntaxError) {
			if (opts.filename) {
				e.message += ' in ' + opts.filename;
			}
			e.message += ' while compiling ejs\n\n';
			e.message += 'If the above error is not helpful, you may want to try EJS-Lint:\n';
			e.message += 'https://github.com/RyanZim/EJS-Lint';
			if (!opts.async) {
				e.message += '\n';
				e.message += 'Or, if you meant to create an async function, pass `async: true` as an option.';
			}
        }
        throw e;
    }

    // Return a callable function which will execute the function
    // created by the source-code, with the passed data as locals
    // Adds a local `include` function which allows full recursive include
    var returnedFn = opts.client ? fn : function anonymous(data) {
        var include = function (path, includeData) {
			var d = utils.shallowCopy(utils.createNullProtoObjWherePossible(), data);
			if (includeData) {
				d = utils.shallowCopy(d, includeData);
			}
			return includeFile(path, opts)(d);
        };
        return fn.apply(opts.context,
        [data || utils.createNullProtoObjWherePossible(), escapeFn, include, rethrow]);
    };
    if (opts.filename && typeof Object.defineProperty === 'function') {
        var filename = opts.filename;
        var basename = path.basename(filename, path.extname(filename));
        try {
			Object.defineProperty(returnedFn, 'name', {
				value: basename,
				writable: false,
				enumerable: false,
				configurable: true
			});
        } catch (e) {/* ignore */}
    }
    return returnedFn;
    },
```

```js
    compile: function () {
    /** @type {string} */
    var src;
    /** @type {ClientFunction} */
    var fn;
    var opts = this.opts;
    var prepended = '';
    var appended = '';
    /** @type {EscapeCallback} */
    ••• var escapeFn = opts.escapeFunction; •••
    /** @type {FunctionConstructor} */
    var ctor;
    /** @type {string} */
    var sanitizedFilename = opts.filename ? JSON.stringify(opts.filename) : 'undefined';

    if (!this.source) {
        this.generateSource();
        prepended +=
        '  var __output = "";\n' +
        '  function __append(s) { if (s !== undefined && s !== null) __output += s }\n';
        if (opts.outputFunctionName) {
        if (!_JS_IDENTIFIER.test(opts.outputFunctionName)) {
            throw new Error('outputFunctionName is not a valid JS identifier.');
        }
        prepended += '  var ' + opts.outputFunctionName + ' = __append;' + '\n';
        }
        if (opts.localsName && !_JS_IDENTIFIER.test(opts.localsName)) {
        throw new Error('localsName is not a valid JS identifier.');
        }
        if (opts.destructuredLocals && opts.destructuredLocals.length) {
        var destructuring = '  var __locals = (' + opts.localsName + ' || {}),\n';
        for (var i = 0; i < opts.destructuredLocals.length; i++) {
            var name = opts.destructuredLocals[i];
            if (!_JS_IDENTIFIER.test(name)) {
            throw new Error('destructuredLocals[' + i + '] is not a valid JS identifier.');
            }
            if (i > 0) {
            destructuring += ',\n  ';
            }
            destructuring += name + ' = __locals.' + name;
        }
        prepended += destructuring + ';\n';
        }
        if (opts._with !== false) {
        prepended +=  '  with (' + opts.localsName + ' || {}) {' + '\n';
        appended += '  }' + '\n';
        }
        appended += '  return __output;' + '\n';
        this.source = prepended + this.source + appended;
    }

    if (opts.compileDebug) {
        src = 'var __line = 1' + '\n'
        + '  , __lines = ' + JSON.stringify(this.templateText) + '\n'
        + '  , __filename = ' + sanitizedFilename + ';' + '\n'
        + 'try {' + '\n'
        + this.source
        + '} catch (e) {' + '\n'
        + '  rethrow(e, __lines, __filename, __line, escapeFn);' + '\n'
        + '}' + '\n';
    }
    else {
        src = this.source;
    }

    •••
    if (opts.client) {
		src = 'escapeFn = escapeFn || ' + escapeFn.toString() + ';' + '\n' + src;
        if (opts.compileDebug) {
        src = 'rethrow = rethrow || ' + rethrow.toString() + ';' + '\n' + src;
        }
    }
    //- `escapeFn`이 존재하면 그대로 사용하고, 없으면 `escapeFn.toString()`의 결과를 문자열로 변환하여 사용합니다. 그리고 이를 `src` 문자열 앞에 추가합니다.
    •••

    if (opts.strict) {
        src = '"use strict";\n' + src;
    }
    if (opts.debug) {
        console.log(src);
    }
    if (opts.compileDebug && opts.filename) {
        src = src + '\n'
        + '//# sourceURL=' + sanitizedFilename + '\n';
    }

    try {
        if (opts.async) {
        // Have to use generated function for this, since in envs without support,
        // it breaks in parsing
        try {
            ctor = (new Function('return (async function(){}).constructor;'))();
        }
        catch(e) {
            if (e instanceof SyntaxError) {
            throw new Error('This environment does not support async/await');
            }
            else {
            throw e;
            }
        }
        }
        else {
        ctor = Function;
        }
        •••
        fn = new ctor(opts.localsName + ', escapeFn, include, rethrow', src);
        //`opts.localsName + ', escapeFn, include, rethrow'` 부분이 생성되는 함수의 인수 이름을 나타내며, `src`는 함수 본문을 나타냅니다. 이렇게 생성된 함수 객체는 `fn` 변수에 할당됩니다.
        •••
    }
    catch(e) {
        // istanbul ignore else
        if (e instanceof SyntaxError) {
        if (opts.filename) {
            e.message += ' in ' + opts.filename;
        }
        e.message += ' while compiling ejs\n\n';
        e.message += 'If the above error is not helpful, you may want to try EJS-Lint:\n';
        e.message += 'https://github.com/RyanZim/EJS-Lint';
        if (!opts.async) {
            e.message += '\n';
            e.message += 'Or, if you meant to create an async function, pass `async: true` as an option.';
        }
        }
        throw e;
    }

    // Return a callable function which will execute the function
    // created by the source-code, with the passed data as locals
    // Adds a local `include` function which allows full recursive include
    ••• // fn 함수를 returnedFn에 할당합니다.
    var returnedFn = opts.client ? fn : function anonymous(data) {
        var include = function (path, includeData) {
        var d = utils.shallowCopy(utils.createNullProtoObjWherePossible(), data);
        if (includeData) {
            d = utils.shallowCopy(d, includeData);
        }
        return includeFile(path, opts)(d);
        };
        return fn.apply(opts.context,
        [data || utils.createNullProtoObjWherePossible(), escapeFn, include, rethrow]);
    };
    if (opts.filename && typeof Object.defineProperty === 'function') {
        var filename = opts.filename;
        var basename = path.basename(filename, path.extname(filename));
        try {
        Object.defineProperty(returnedFn, 'name', {
            value: basename,
            writable: false,
            enumerable: false,
            configurable: true
        });
        } catch (e) {/* ignore */}
    }
    return returnedFn; // 할당된 함수를 return
    •••
    },
```

마지막 return 부분에서 입력할 value 값을 return 하는 것을 확인하고 즉시 실행하는 함수 표현식을 이용해서 RCE가 가능하다는 것을 알았습니다.