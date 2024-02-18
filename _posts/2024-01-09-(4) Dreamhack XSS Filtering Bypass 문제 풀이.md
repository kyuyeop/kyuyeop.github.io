---
title: (4) Dreamhack XSS Filtering Bypass 문제 풀이
date: 2024-01-09 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
먼저 들어오면 이런 모습입니다.
![](https://kyuyeop.github.io/assets/img/post/4/1.png)
![](https://kyuyeop.github.io/assets/img/post/4/2.png)
vuln 페이지에는 뭐가 없네요. 개발자 도구로 코드를 뜯어보니 param을 그대로 반영한 듯 싶습니다. xss 취약점이 있을 수 있겠네요.
![](https://kyuyeop.github.io/assets/img/post/4/3.png)
memo는 들어갈 때 마다 hello를 띄워주네요.
![](https://kyuyeop.github.io/assets/img/post/4/4.png)
flag는 이렇게 되어 있습니다. POST 요청을 날려주네요.
  
그럼 이제 코드를 확인해 봅시다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
import urllib
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)

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

def xss_filter(text):
    _filter = ["script", "on", "javascript:"]
    for f in _filter:
        if f in text.lower():
            text = text.replace(f, "")
    return text

@app.route("/")
def index():
    return render_template("index.html")


@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    param = xss_filter(param)
    return param


@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'


memo_text = ""


@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text)


app.run(host="0.0.0.0", port=8000)
```
여기서 주의깊게 봐야할 부분은
```python
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

def xss_filter(text):
    _filter = ["script", "on", "javascript:"]
    for f in _filter:
        if f in text.lower():
            text = text.replace(f, "")
    return text

@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    param = xss_filter(param)
    return param

@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'
```
이 3개의 함수와 vuln, flag 페이지 입니다. FLAG는 오직 /flag에 POST요청을 날렸을때 cookie로만 사용되고 있습니다. 저걸 반드시 이용해야 한다는 뜻이죠. 이어서 check_xss를 거쳐 vuln 페이지가 read_url 함수를 통해 읽어집니다.  
  
그래서 쿠키가 어디까지 가는가 하니, read_url 함수가 webdriver를 통해 vuln 페이지를 접속하는데, 이때 쿠키가 같이 넘어가는 걸 알 수 있습니다.  
그리고 vuln 페이지는 xss_filter 함수를 통해 아주 간단한 문자열 검사를 수행하고 있네요.  
  
즉, flag에서 input에 무언가를 입력하면 flag가 담긴 쿠키와 입력값을 가지고 서버가 vuln 페이지에 한번 들러줍니다.  
  
그럼 서버측에 있는 쿠키를 빼내야 flag를 얻을 수 있다는 것이겠죠? 일단 쿠키를 빼내는 기본적인 방법은
```javascript
document.cookie
```
라는 자바스크립트를 사요하는 방법입니다. 로컬이라면 그냥 alert를 이용해 띄워주면 되겠지만, 이 서버는 쿠키를 서버측에서 가지고 있다가 끝나버립니다.  
  
그래서 memo 페이지를 이용할 겁니다. memo 페이지는 memo 파라미터로 넘어온 모든걸 기록해주는 역할을 하고 있습니다. 그래서 서버에서 쿠키를 빼내 memo 페이지에 저장하는 자바스크립트를 실행할 수만 있다면 해결됩니다. 이를 자바스크립트로 나타내면 다음과 같습니다.
```javascript
location.href="/memo?memo="+document.cookie;
```
서버(read_url 과정)에서 이 코드를 수행하면 memo 파라미터로 쿠키를 넘기고, memo 페이지로 이동하게 할 수 있습니다.  
  
그러면 vuln 페이지로 이렇게만 보내주면 될까요?
```html
<script>location.href="/memo?memo="+document.cookie;</script>
```
vuln 페이지에는 xss_filter 함수로 필터링을 한다고 했었죠? 그래서 이걸 그대로 보낸다면
```html
<>locatin.href="/memo?memo="+document.cookie;</>
```
이렇게 되버릴 겁니다.  
  
따라서 필터링을 우회해야 합니다.  
  
코드를 보면 단순히 입력을 소문자로 바꿔서 script, on, javascript: 가 있다면 지워버리고(''로 치환하고) 결과를 반환합니다. 이 작업은 최초 한번만 작동하기에 치환되는 문자열 일부에 그 문자열을 한번 더 쓴다면 의도한 대로 결과를 얻을 수 있습니다.  
즉, 원하는 페이로드를 보내려면
```html
<sscriptcript>locatioonn.href="/memo?memo="+document.cookie;</sscriptcript>
```
이런식으로 글자 사이에 한번더 치환 문자열을 넣어서 치환을 거쳐 원하는 문자열을 완성시키도록 해야 합니다.  

이렇게 완성된 페이로드를 xss 페이지 input에 넣고
![](https://kyuyeop.github.io/assets/img/post/4/5.png)
제출하게 된다면...
![](https://kyuyeop.github.io/assets/img/post/4/6.png)
good이 떴습니다!
공격에 성공했다는 뜻이죠. 이제 memo 페이지를 확인해 보면...
![](https://kyuyeop.github.io/assets/img/post/4/7.png)
이렇게 flag가 출력되었습니다.  
  
이번엔 \<script\>를 이용했지만 대표적인 xss 페이로드는 스크립트 태그를 사용하는 방법 이외에, img, iframe 을 이용하는 방법도 있습니다.
  
이런종류의 xss를 막으려면 한번만 필터링 할게 아니라, 반복문을 통해 치환 전과 치환 후가 같아질때 까지 필터링을 해줘야 합니다.
{% endraw %}