---
title: (34) Dreamhack crawling 문제 풀이
date: 2024-02-11 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명


## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/34/1.png)
사이트에 처음 들어오면 이런 페이지가 나옵니다.  
  
구글을 입력했더니
![](https://kyuyeop.github.io/assets/img/post/34/2.png)
이렇게 html을 가져옵니다.  
  
일단은 여기까지 밖에 할 수 있는게 없으니 코드를 보겠습니다.
```python
import socket
import requests
import ipaddress
from urllib.parse import urlparse
from flask import Flask, request, render_template

app = Flask(__name__)
app.flag = '__FLAG__'

def lookup(url):
    try:
        return socket.gethostbyname(url)
    except:
        return False

def check_global(ip):
    try:
        return (ipaddress.ip_address(ip)).is_global
    except:
        return False

def check_get(url):
    ip = lookup(urlparse(url).netloc)
    if ip == False or ip =='0.0.0.0':
        return "Not a valid URL."
    res=requests.get(url)
    if check_global(ip) == False:
        return "Can you access my admin page~?"
    for i in res.text.split('>'):
        if 'referer' in i:
            ref_host = urlparse(res.headers.get('refer')).netloc
            if ref_host == 'localhost':
                return False
            if ref_host == '127.0.0.1':
                return False 
    res=requests.get(url)
    return res.text

@app.route('/admin')
def admin_page():
    if request.remote_addr != '127.0.0.1':
        return "This is local page!"
    return app.flag

@app.route('/validation')
def validation():
    url = request.args.get('url', '')
    ip = lookup(urlparse(url).netloc)
    res = check_get(url)
    return render_template('validation.html', url=url, ip=ip, res=res)

@app.route('/')
def index():
    return render_template('index.html')

if __name__=='__main__':
    app.run(host='0.0.0.0', port=3333)
```
`127.0.0.1`인 아이피로 `/admin`에 접속하면 flag를 준다고 합니다. 하지만 `check_global` 단계에서 내부 아이피를 차단하고, 추가로 referer를 이용해서 `localhost, 127.0.0.1`을 차단합니다.  
  
해결 방법은 굉장히 단순합니다.    
`http://127.0.0.1:3333/admin`을 어떤 방식을 이용해서든 리다이렉트로 이동되도록 만들면 끝입니다. 이렇게 되면 외부아이피를 통해서 접근이 되고, referer도 해당 외부 아이피로 잡힙니다.
  
저는~~(스미싱에 자주 쓰는)~~ tinyurl을 이용했습니다. url을 단축해서 입력하면
![](https://kyuyeop.github.io/assets/img/post/34/3.png)
이렇게 flag를 얻을 수 있습니다.
{% endraw %}