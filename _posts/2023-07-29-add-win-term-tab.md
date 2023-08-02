---
title: "윈도우 터미널에 Ubuntu 탭 추가하기"
author: d0razi
date: 2023-07-29 19:00
categories: [OS, Window]
tags: [window]
toc: false
image: /assets/img/media/window.jpg
---

uuidgen 명령어 입력 후 결과값을 복사합니다.

![Untitled](/assets/img/media/post_img/add-tab/Untitled.png)

그리고 설정에서 json 파일을 열어줍니다.

![Untitled](/assets/img/media/post_img/add-tab/Untitled%201.png)

![Untitled](/assets/img/media/post_img/add-tab/Untitled%202.png)

그리고 열린 json 파일에서 profiles 마지막에 ,를 찍어주고 아래 코드를 입력해줍니다.

위 YOUR_GUID에는 아까 나온 값을 넣어주면 됩니다.

```
{
		"font": 
		{
		    "face": "D2Coding"
		},
		"commandline":"wsl.exe -d Ubuntu",
		"guid": "{f9204698-d381-4969-8fa5-122b24fb0857}",
		"hidden": false,
		"name":"Ubuntu",
		"startingDirectory":"~"
}
```

![Untitled](/assets/img/media/post_img/add-tab/Untitled%203.png)

추가가 잘 된 모습입니다.