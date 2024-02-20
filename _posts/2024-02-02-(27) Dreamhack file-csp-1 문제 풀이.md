---
title: (27) Dreamhack file-csp-1 문제 풀이
date: 2024-02-02 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
문제에서 요구하는 조건에 맞게 CSP를 작성하면 플래그를 획득할 수 있습니다.

## 문제 풀이
사이트에는 아래 두 페이지가 구현되어 있습니다.
![](https://kyuyeop.github.io/assets/img/post/27/1.png)
![](https://kyuyeop.github.io/assets/img/post/27/2.png)
위의 `/test`페이지는 굳이 쓸 필요가 없어서 보진 않겠습니다.  
  
코드를 살펴봅시다.
```python
#!/usr/bin/env python3
import os
import shutil
from time import sleep
from urllib.parse import quote

from flask import Flask, request, render_template, redirect, make_response
from selenium import webdriver

from flag import FLAG

APP = Flask(__name__)


@APP.route('/')
def index():
    return render_template('index.html')


@APP.route('/test', methods=['GET', 'POST'])
def test_csp():
    return render_template('test.html')


@APP.route('/verify', methods=['GET', 'POST'])
def verify_csp():
    global CSP
    if request.method == 'POST':
        csp = request.form.get('csp')
        try:
            options = webdriver.ChromeOptions()
            for _ in ['headless', 'window-size=1920x1080', 'disable-gpu', 'no-sandbox', 'disable-dev-shm-usage']:
                options.add_argument(_)
            driver = webdriver.Chrome('/chromedriver', options=options)
            driver.implicitly_wait(3)
            driver.set_page_load_timeout(3)
            driver.get(f'http://localhost:8000/live?csp={quote(csp)}')
            try:
                a = driver.execute_script('return a()');
            except:
                a = 'error'
            try:
                b = driver.execute_script('return b()');
            except:
                b = 'error'
            try:
                c = driver.execute_script('return c()');
            except Exception as e:
                c = 'error'
                c = e
            try:
                d = driver.execute_script('return $(document)');
            except:
                d = 'error'

            if a == 'error' and b == 'error' and c == 'c' and d != 'error':
                return FLAG

            return f'Try again!, {a}, {b}, {c}, {d}'
        except Exception as e:
            return f'An error occured!, {e}'

    return render_template('verify.html')


@APP.route('/live', methods=['GET'])
def live_csp():
    csp = request.args.get('csp', '')
    resp = make_response(render_template('csp.html'))
    resp.headers.set('Content-Security-Policy', csp)
    return resp


if __name__ == '__main__':
    APP.run(host='0.0.0.0', port=8000, threaded=True)
```
일단 `/verify`페이지는 인풋으로 csp 문자열을 받고, chrome driver를 이용해서 `/live`페이지에 입력한 csp를 적용하여 불러와서 함수 `a, b, c` 그리고 `$(document)`를 리턴하도록 하고 있네요.  
  
`a, b`함수는 `error`, `c`함수는 문자`c`를, `d`함수는 `error`가 아니면 flag를 준다고 합니다.  
  
일단 아무것도 안넣고(공백만 두고) 들어가보겠습니다.
![](https://kyuyeop.github.io/assets/img/post/27/3.png)
이렇게 나옵니다.  
  
그럼 placeholder에 있는걸 그대로 입력해보면
![](https://kyuyeop.github.io/assets/img/post/27/4.png)
마지막 메세지가 error로바뀌었네요.  
  
렌더링되는 csp.html을 보면
```html
<!doctype html>
<html>
  <head>
  <!-- block me -->
  <script>
    function a() { return 'a'; }
    document.write('a: block me!<br>');
  </script>
  <!-- block me -->
   <script nonce="i_am_super_random">
    function b() { return 'b'; }
    document.write('b: block me!<br>');
  </script>
  <!-- allow me -->
  <script
  src="https://code.jquery.com/jquery-3.4.1.slim.min.js"
  integrity="sha256-pasqAKBDmFT4eHoN2ndd6lN370kFiGUFyTiUHWhU7k8="
  crossorigin="anonymous"></script>
  <!-- allow me -->
   <script nonce="i_am_super_random">
    function c() { return 'c'; }
    document.write('c: allow me!<br>');
    try { $(document); document.write('jquery: allow me!<br>'); } catch (e) {  }
  </script>
  </head>
</html>
```
jquery에서는 `$요소`형태를 사용하기 때문에 반환이 있고, 바닐라 js에선 그런게 없기 때문에 'error'가 납니다. `error`가 있다는건 다른 말로는 jquery를 불러오지 않고 있다는 뜻이죠.  
  
이쯤에서 csp가 뭔지 알아봐야 겠습니다.
### CSP
CSP는 XSS같은 같은 보안 취약점을 막아줄 수 있는 추가 보안 계층입니다. CSP는 http 헤더에 포함됩니다.  
CSP설정에는 여러가지가 있는데 이번에 필요한건 script-src라는 스크립트 관련 설정입니다.  
src 설정에는 허용 도메인, 현재 도메인, 소스코드 내 js/css만, nonce, 해쉬값 등이 있습니다.  
앞서 사용한 script-src 'unsafe-inline' 을 하면 소스코드 내 존재하는 js만 실행합니다.  
즉, jquery는 실행이 안되는거죠.  
<br>
  
이 문제는 jquery와 c 함수만 실행 가능하도록 csp를 설정해주면 되는 문제입니다.  
앞서 언급했듯, 해쉬값을 지정해서 하는 방법이 있습니다. `Content-Security-Policy: script-src '해쉬값'` 형태로 하면 특정 해쉬값을 가지는 스크립트만 실행할 수 있습니다.  
  
jquery의 해쉬는 이미 무결성 검증용으로 `<script>`에서 복사해와서 사용하면 됩니다. 그럼 마지막 부분의 해쉬를 알아내기만 하면 되겠죠.  
해쉬는 공백을 포함해서 `<script></script>`사이의 모든것을 해쉬합니다. 따라서 아래 영역의 해쉬를 구해주면 됩니다.
![](https://kyuyeop.github.io/assets/img/post/27/5.png)
[여기](https://report-uri.com/home/hash)에서 해쉬를 구해보면 `sha256-l1OSKODPRVBa1/91J7WfPisrJ6WCxCRnKFzXaOkpsY4=`가 나옵니다.  
  
이제 실행하려는 두 스크립트의 해쉬를 알았으므로, 아래와 같은 값을 넣어주면
```
script-src 'sha256-pasqAKBDmFT4eHoN2ndd6lN370kFiGUFyTiUHWhU7k8=' 'sha256-l1OSKODPRVBa1/91J7WfPisrJ6WCxCRnKFzXaOkpsY4='
```
![](https://kyuyeop.github.io/assets/img/post/27/6.png)
flag가 나옵니다.
{% endraw %}