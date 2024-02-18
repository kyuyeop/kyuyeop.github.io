---
title: (7) Dreamhack command-injection-chatgpt 문제 풀이
date: 2024-01-12 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
특정 Host에 ping 패킷을 보내는 서비스입니다.  
Command Injection을 통해 플래그를 획득하세요. 플래그는 flag.py에 있습니다.  
chatGPT와 함께 풀어보세요!  

## 문제 풀이
코드 부터 보겠습니다.
```python
#!/usr/bin/env python3
import subprocess

from flask import Flask, request, render_template, redirect

from flag import FLAG

APP = Flask(__name__)


@APP.route('/')
def index():
    return render_template('index.html')


@APP.route('/ping', methods=['GET', 'POST'])
def ping():
    if request.method == 'POST':
        host = request.form.get('host')
        cmd = f'ping -c 3 {host}'
        try:
            output = subprocess.check_output(['/bin/sh', '-c', cmd], timeout=5)
            return render_template('ping_result.html', data=output.decode('utf-8'))
        except subprocess.TimeoutExpired:
            return render_template('ping_result.html', data='Timeout !')
        except subprocess.CalledProcessError:
            return render_template('ping_result.html', data=f'an error occurred while executing the command. -> {cmd}')

    return render_template('ping.html')


if __name__ == '__main__':
    APP.run(host='0.0.0.0', port=8000)
```
22번째 줄에서 그냥 /bin/sh로 ping 커맨드의 결과를 받아오는게 보일 겁니다. 다른 필터링도 안보이고요.
![](https://kyuyeop.github.io/assets/img/post/7/1.png)
이렇게 정상적인 아이피를 쳐주면
![](https://kyuyeop.github.io/assets/img/post/7/2.png)
그냥 커맨드 실행 결과를 그대로 뱉어주네요.  
다른 명령어를 실행하고 싶으면 ;나 &&하고 그냥 다음 명령어를 입력하기만 하면 됩니다.  
  
그냥 flag.py를 읽도록 이렇게 입력해 보겠습니다.
```
8.8.8.8; cat flag.py
```
![](https://kyuyeop.github.io/assets/img/post/7/3.png)
flag를 얻었습니다.
{% endraw %}