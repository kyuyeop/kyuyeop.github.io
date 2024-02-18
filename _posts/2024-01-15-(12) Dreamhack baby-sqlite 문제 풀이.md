---
title: (12) Dreamhack baby-sqlite 문제 풀이
date: 2024-01-15 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
로그인 시 계정의 정보가 출력되는 웹 서비스입니다.  
SQL INJECTION 취약점을 통해 플래그를 획득하세요. 문제에서 주어진 `init.sql` 파일의 테이블명과 컬럼명은 실제 이름과 다릅니다.  
플래그 형식은 `DH{...}` 입니다.

## 문제 풀이
먼저 사이트에서 로그인 페이지에 들어온 모습입니다.
![](https://kyuyeop.github.io/assets/img/post/12/1.png)
이번 문제는 app.py 하나만 제공합니다.
```python
#!/usr/bin/env python3
from flask import Flask, request, render_template, make_response, redirect, url_for, session, g
import urllib
import os
import sqlite3

app = Flask(__name__)
app.secret_key = os.urandom(32)
from flask import _app_ctx_stack

DATABASE = 'users.db'

def get_db():
    top = _app_ctx_stack.top
    if not hasattr(top, 'sqlite_db'):
        top.sqlite_db = sqlite3.connect(DATABASE)
    return top.sqlite_db


try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'


@app.route('/')
def index():
    return render_template('index.html')


@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')

    uid = request.form.get('uid', '').lower()
    upw = request.form.get('upw', '').lower()
    level = request.form.get('level', '9').lower()

    sqli_filter = ['[', ']', ',', 'admin', 'select', '\'', '"', '\t', '\n', '\r', '\x08', '\x09', '\x00', '\x0b', '\x0d', ' ']
    for x in sqli_filter:
        if uid.find(x) != -1:
            return 'No Hack!'
        if upw.find(x) != -1:
            return 'No Hack!'
        if level.find(x) != -1:
            return 'No Hack!'

    
    with app.app_context():
        conn = get_db()
        query = f"SELECT uid FROM users WHERE uid='{uid}' and upw='{upw}' and level={level};"
        try:
            req = conn.execute(query)
            result = req.fetchone()

            if result is not None:
                uid = result[0]
                if uid == 'admin':
                    return FLAG
        except:
            return 'Error!'
    return 'Good!'


@app.teardown_appcontext
def close_connection(exception):
    top = _app_ctx_stack.top
    if hasattr(top, 'sqlite_db'):
        top.sqlite_db.close()


if __name__ == '__main__':
    os.system('rm -rf %s' % DATABASE)
    with app.app_context():
        conn = get_db()
        conn.execute('CREATE TABLE users (uid text, upw text, level integer);')
        conn.execute("INSERT INTO users VALUES ('dream','cometrue', 9);")
        conn.commit()

    app.run(host='0.0.0.0', port=8001)
```
이 문제는 좀 특이합니다. 조건이 서버에서 실행한 sql문에 admin이 반환되면 되는건데, 보통은 admin이라는 유저의 패스워드 비교를 우회하지만, 이 문제에서는 admin이라는 유저가 존재하지 않습니다.  
  
또한 sql 인젝션에 주로 사용되는 싱글쿼터 라던가 탭이나 스페이스 같은 것도 전부 필터링 되고 있습니다. 때문에 uid나 upw에서 sqli을 수행하긴 어려워 보이네요.  
  
다만, level 은 싱글쿼터를 사용하지 않고 있어서 level의 입력값을 변경하는게 출제 의도로 보입니다.  
  
먼저 필터링을 우회하는 방법에 대해 알아보도록 하겠습니다.

### 필터링 우회
#### 공백 우회
먼저 아주 빈번하게 쓰이는 공백(스페이스) 문자부터 알아보겠습니다.  
공백 문자는 `/\*\*/`로 대체가 가능합니다.
#### 문자열 우회
문자열은 values 함수, char함수, ||를 이용해서 가능합니다.  
char(값) 은 값에 해당하는 ascii 문자를 반환합니다.  
a라고 하면 10진수 97번인데, `char(97)`로 나타낼 수 있습니다.  
다음 문자는 ||로 연결할 수 있습니다. 그 다음 values함수로 이 결과를 감싸게 되면 하나의 값으로 인식시킬 수 있습니다.

이걸 이용해 admin을 나타내면  
```sql
values(char(97)||char(100)||char(109)||char(105)||char(110))
```  
로 나타낼 수 있습니다.

### UNION
union은 앞 sql문과 뒤의 sql문의 결과를 합쳐주는 문법입니다. 이 특징 때문에 sqli에 빈번하게 쓰입니다.  
주의할 게, union 사용시에는 두 sql 구문의 반환의 column 개수가 같아야 합니다. 맞추지 않으면 에러가 납니다.  
만약 앞선 sql에서 \*을 사용했다면 null을 사용하거나 임의의 값으로 column을 맞춰줘야 합니다.  
  

그럼 level에 입력해야하는걸 알아봅시다.
```sql
1/**/union/**/values(char(97)||char(100)||char(109)||char(105)||char(110))
```
먼저 아무 값이나 입력해주고, 앞서 알아본 우회 기법을 이용해 union으로 admin을 전달해 줬습니다.  
그냥 admin만 전달하는 이유는 요청하는 sql문이 uid 값만 가져오기 때문에 column 수를 맞추기 위함입니다.
```sql
SELECT uid FROM users WHERE uid='{uid}' and upw='{upw}' and level={level};
```
당연히 uid, upw, level에는 실제로 존재하는 유저의 값을 넣으면 안됩니다.  
왜냐하면 result[0]을 가져오기 때문에 첫번째로 나온 결과가 사용되기 때문이죠.
  
뭘 입력해야 할지 구상이 되었으니 실제로 입력만 하면 되겠죠.  
근데 level을 이용하려고 했더니, level 입력란이 보이질 않습니다.
  
이럴때 방법은 두가지 입니다.
  
첫번째 방법은 어차피 프론트 엔드니까, 그냥 form에 input을 추가해버리는 겁니다.
![](https://kyuyeop.github.io/assets/img/post/12/2.png)
![](https://kyuyeop.github.io/assets/img/post/12/3.png)
이렇게 level 입력란을 만들어 버릴 수 있습니다.
  
두번째 방법은 burp suite 같은 프록시 프로그램을 이용해서, post 요청을 보낼때 요청에 level값을 끼워넣는 겁니다.

먼저 브라우저를 열고 문제 주소로 접속한 뒤 Intercept is on 으로 바꿔 줍니다.
![](https://kyuyeop.github.io/assets/img/post/12/4.png)
그리고 브라우저에서 없는 값을 아무거나 입력하고 제출을 누르면
![](https://kyuyeop.github.io/assets/img/post/12/5.png)
burp suite 창에 아래처럼 보내려는 요청이 뜹니다.
![](https://kyuyeop.github.io/assets/img/post/12/6.png)
16번째 줄을 보면 아까 우리가 입력한 uid, upw값이 들어있죠. 여기에 level을 끼워서 보내면 됩니다.
![](https://kyuyeop.github.io/assets/img/post/12/7.png)
이렇게 입력하고 Intercept is on 버튼을 다시 눌러서 전송하게 되면?
![](https://kyuyeop.github.io/assets/img/post/12/8.png)
flag를 얻을 수 있습니다.
{% endraw %}