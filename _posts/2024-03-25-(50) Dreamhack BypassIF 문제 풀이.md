---
title: (50) Dreamhack BypassIF 문제 풀이
date: 2024-03-25 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
Admin의 `KEY`가 필요합니다! 알맞은 `KEY`값을 입력하여 플래그를 획득하세요.  
플래그 형식은 DH{...} 입니다.

## 문제 풀이
처음 사이트에 접속한 모습입니다.
![](https://kyuyeop.github.io/assets/img/post/50/1.png)

입력하고 제출하는 버튼만 보일뿐 별다른건 보이지 않네요. 바로 코드를 확인해보겠습니다.
```python
#!/usr/bin/env python3
import subprocess
from flask import Flask, request, render_template, redirect, url_for
import string
import os
import hashlib

app = Flask(__name__)

try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"

KEY = hashlib.md5(FLAG.encode()).hexdigest()
guest_key = hashlib.md5(b"guest").hexdigest()

# filtering
def filter_cmd(cmd):
    alphabet = list(string.ascii_lowercase)
    alphabet.extend([' '])
    num = '0123456789'
    alphabet.extend(num)
    command_list = ['flag','cat','chmod','head','tail','less','awk','more','grep']

    for c in command_list:
        if c in cmd:
            return True
    for c in cmd:
        if c not in alphabet:
            return True

@app.route('/', methods=['GET', 'POST'])
def index():
    # GET request
    return render_template('index.html')



@app.route('/flag', methods=['POST'])
def flag():
     # POST request
    if request.method == 'POST':
        key = request.form.get('key', '')
        cmd = request.form.get('cmd_input', '')
        if cmd == '' and key == KEY:
            return render_template('flag.html', txt=FLAG)
        elif cmd == '' and key == guest_key:
            return render_template('guest.html', txt=f"guest key: {guest_key}")
        if cmd != '' or key == KEY:
            if not filter_cmd(cmd):
                try:
                    output = subprocess.check_output(['/bin/sh', '-c', cmd], timeout=5)
                    return render_template('flag.html', txt=output.decode('utf-8'))
                except subprocess.TimeoutExpired:
                    return render_template('flag.html', txt=f'Timeout! Your key: {KEY}')
                except subprocess.CalledProcessError:
                    return render_template('flag.html', txt="Error!")
            return render_template('flag.html')
        else:
            return redirect('/')
    else: 
        return render_template('flag.html')


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000, debug=True)
```
flag를 어디서 얻을 수 있는가 하니, post로 받아온 `cmd_input`가 빈값이고 `key`가 `KEY` 즉, `hashlib.md5(FLAG.encode()).hexdigest()`이면 얻을 수 있도록 되어 있습니다.  
일단 `KEY`값을 알아내야 하는데 `cmd_input`가 빈 값이 아닐때 입력값을 `subprocess`로 실행할때 `subprocess.TimeoutExpired`예외가 발생하면 KEY를 얻을 수 있습니다.  
`timeout=5`으로 설정되어 있으므로 5초 이상의 무언가를 실행하면 됩니다. 간단하게는 `sleep 5`를 이용하면 5초 동안 기다리도록 하면 됩니다.
![](https://kyuyeop.github.io/assets/img/post/50/2.png)

burpsuite를 이용해서 위와 같이 `cmd_input=sleep 5`를 추가하고 보내줬습니다.
![](https://kyuyeop.github.io/assets/img/post/50/3.png)

요청을 보내고 5초를 기다리면 아래쪽에 `409ac0d96943d3da52f176ae9ff2b974`라는 `KEY`가 출력됩니다.  
  
메인페이지로 돌아가서 얻은 키를 입력하면
![](https://kyuyeop.github.io/assets/img/post/50/3.png)

flag를 얻을 수 있습니다.
{% endraw %}