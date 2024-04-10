---
title: (54) Dreamhack what-is-my-ip 문제 풀이
date: 2024-04-10 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
지난주에 1개 밖에 못올리기도 했고, 선거라 마침 쉬니까 오늘은 2문제 올려봅니다.

## 문제 설명
How are they aware of us even behind the wall?

## 문제 풀이
사이트에 처음 들어온 모습입니다.
![](https://kyuyeop.github.io/assets/img/post/54/1.png)

별다른건 없고 제 아이피만 보입니다. 바로 코드를 확인해봅시다.
```python
#!/usr/bin/python3
import os
from subprocess import run, TimeoutExpired
from flask import Flask, request, render_template

app = Flask(__name__)
app.secret_key = os.urandom(64)


@app.route('/')
def flag():
    user_ip = request.access_route[0] if request.access_route else request.remote_addr
    try:
        result = run(
            ["/bin/bash", "-c", f"echo {user_ip}"],
            capture_output=True,
            text=True,
            timeout=3,
        )
        return render_template("ip.html", result=result.stdout)

    except TimeoutExpired:
        return render_template("ip.html", result="Timeout!")


app.run(host='0.0.0.0', port=3000)

```
보아하니 `user_ip`를 이용해 command injection을 하는 문제 같습니다.

`request.access_route`가 핵심인듯 해서 이게 뭐하는 녀석인가 살펴보겠습니다.
If a forwarded header exists this is a list of all ip addresses from the client ip to the last proxy server.
공식 사이트에 이렇게 쓰여있습니다. 여기서 말하는 헤더는 `X-Forwarded-For`헤더를 말하는것 같습니다.  
이 헤더가 존재하면 헤더의 값이 `request.access_route`리스트 앞에 붙게 됩니다. 

그렇다면 `X-Forwarded-For`헤더를 이용해 `user_ip`를 바꿀 수 있습니다. 우리는 `/flag`파일을 읽어야 하므로 `X-Forwarded-For: ;cat /flag`를 포함해서 요청을 보내게 되면 `request.access_route[0]`에 조작된 헤더의 값이 들어가고 이는 `user_ip`가 됩니다.

실제로 헤더를 추가해서 요청을 보내보면
![](https://kyuyeop.github.io/assets/img/post/54/2.png)

위와 같이 flag가 나옵니다.
{% endraw %}