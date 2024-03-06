---
title: (46) Dreamhack development-env 문제 풀이
date: 2024-03-06 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
초보 개발자 민수는 실수로 development 환경을 사용해 배포해버리고 말았습니다..

## 문제 풀이
사이트에 들어온 모습입니다.
![](https://kyuyeop.github.io/assets/img/post/46/1.png)

소스코드를 보면 `guest/guestPW`라는 계정이 있습니다. 일단 로그인해보면 admin이 아니라고 합니다.
![](https://kyuyeop.github.io/assets/img/post/46/2.png)

그럼 제공되는 소스코드를 살펴보겠습니다.
```javascript
const express = require("express");
const cryptolib = require("./libs/customcrypto");
var cookieParser = require("cookie-parser");
var parsetrace = require("parsetrace");

const isDevelopmentEnv = true;

const app = express();
const port = 3000;

const flag = "DH{FAKE_FLAG}";
app.set("view engine", "ejs");

app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

let database = {
  guest: "guestPW",
  admin: cryptolib.generateRandomString(15),
}; //don't try to guess admin password

app.get("/", async (req, res) => {
  try {
    let token = req.cookies.auth || "";
    const payloadData = await cryptolib.readJWT(token, "FAKE_KEY");
    if (payloadData) {
      userflag = payloadData["uid"] == "admin" ? flag : "You are not admin";
      res.render("main", { username: payloadData["uid"], flag: userflag });
    } else {
      res.render("login");
    }
  } catch (e) {
    if (isDevelopmentEnv) {
      res.json(JSON.parse(parsetrace(e, { sources: true }).json()));
    } else {
      res.json({ message: "error" });
    }
  }
});

app.post("/validate", async (req, res) => {
  try {
    let contentType = req.header("Content-Type").split(";")[0];
    if (
      ["multipart/form-data", "application/x-www-form-urlencoded"].indexOf(
        contentType
      ) === -1
    ) {
      throw new Error("content type not supported");
    } else {
      let bodyKeys = Object.keys(req.body);
      if (bodyKeys.indexOf("id") === -1 || bodyKeys.indexOf("pw") === -1) {
        throw new Error("missing required parameter");
      } else {
        if (
          typeof database[req.body["id"]] !== "undefined" &&
          database[req.body["id"]] === req.body["pw"]
        ) {
          if (
            req.get("User-Agent").indexOf("MSIE") > -1 ||
            req.get("User-Agent").indexOf("Trident") > -1
          )
            throw new Error("IE is not supported");
          jwt = await cryptolib.generateJWT(req.body["id"], "FAKE_KEY");
          res
            .cookie("auth", jwt, {
              maxAge: 30000,
            })
            .send(
              "<script>alert('success');document.location.href='/'</script>"
            );
        } else {
          res.json({ message: "error", detail: "invalid id or password" });
        }
      }
    }
  } catch (e) {
    if (isDevelopmentEnv) {
      res.status(500).json({
        message: "devError",
        detail: JSON.parse(parsetrace(e, { sources: true }).json()),
      });
    } else {
      res.json({ message: "error", detail: e });
    }
  }
});

app.listen(port);
```
JWT를 이용해서 유저를 구분하고 있습니다.
### JWT
JWT는 세 파트로 나누어지며, 각 파트는 `.`으로 구분됩니다. Header, Payload, Sinature로 이루어지며 `aaaaa.bbbbb.ccccc`형태입니다. 이때 각 요소는 Base64url 인코딩됩니다.  
Header에는 토큰의 타입과 해시 암호화 알고리즘이 담겨있습니다.  
Payload json형태의 데이터가 들어갑니다.  
마지막으로 Signature는 secret key를 포함하여 암호화되어 있습니다.  
<br>

Signature파트에서 암호화에 사용된 secret key를 알아낼 수 있다면 JWT를 위조해서 admin으로 접속할 수 있습니다.  
  
이때 우리는 개발버전이라는 점을 이용해서 이 키를 알아낼 겁니다.  
`parsetrace(e, { sources: true }).json()`형태로 사용한다면 개발에 도움을 주기 위해 에러가 발생한 라인과, 그 주변 라인의 정보를 담아서 반환합니다. 따라서 적절한 위치에서 에러를 유발할 수 있으면 소스코드를 확인할 수 있습니다.  
에러가 난 줄의 위 아래로 3줄의 정보를 알려주기 때문에 throw와 "FAKE_KEY"가 가까이 있는 에러를 이용해야 합니다.  
  
찾아보면
```javascript
          if (
            req.get("User-Agent").indexOf("MSIE") > -1 ||
            req.get("User-Agent").indexOf("Trident") > -1
          )
            throw new Error("IE is not supported");
          jwt = await cryptolib.generateJWT(req.body["id"], "FAKE_KEY");
```
위와 같은 부분이 있습니다.  
`req`의 header의 `User-Agent`값 중에서 `MSIE`나 `Trident`가 들어있으면 `IE is not supported`에러를 throw 하도록 되어 있습니다.  
이 에러를 유발시켜봅시다.
![](https://kyuyeop.github.io/assets/img/post/46/3.png)

`validate`페이지에 접속하는 패킷을 리피터로 보내고 `User-Agent`에 `MSIE`를 추가하여 send 했습니다.
![](https://kyuyeop.github.io/assets/img/post/46/4.png)

위와 같이 소스코드의 일부가 반환되는것을 볼 수 있고 찾아보면 `kitvP5j71fwycLz`라는 key가 노출되고 있습니다.  
  
이제 구한 키를 가지고 ![여기](https://jwt.io/)에서 JWT를 생성해봅시다.
![](https://kyuyeop.github.io/assets/img/post/46/5.png)
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwidWlkIjoiYWRtaW4ifQ.wQt9Fxeq7YtS5exzZ8x2hytp2zbE79YR5elSu5E1DiA
```
이걸 쿠키로 해서 접속해봅시다.
![](https://kyuyeop.github.io/assets/img/post/46/6.png)

이렇게 쿠키를 수정해주고, 새로고침을 해보면
![](https://kyuyeop.github.io/assets/img/post/46/7.png)
flag가 출력됩니다.
{% endraw %}