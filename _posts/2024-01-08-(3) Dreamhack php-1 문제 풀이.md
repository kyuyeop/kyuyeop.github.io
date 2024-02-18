---
title: (3) Dreamhack php-1 문제 풀이
date: 2024-01-08 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
php로 작성된 Back Office 서비스입니다.  
LFI 취약점을 이용해 플래그를 획득하세요. 플래그는 /var/www/uploads/flag.php에 있습니다.

## LFI 취약점
LFI 취약점은 Local File Inclusion의 약자로, 로컬(서버)에 있는 파일을 사용하는 취약점을 뜻합니다.  
주로 php로 만들어진 사이트에서 include나 require같은 함수를 필터링 없이 사용하면 발생합니다.  

이 취약점을 이용하면 원래는 접근하지 못하던 파일에 접근하거나 wrapper를 이용해 시스템 명령을 실행하거나, 쉘을 따낼 수도 있습니다.

## 문제 풀이
사이트에 들어오면
![](https://kyuyeop.github.io/assets/img/post/3/1.png)
![](https://kyuyeop.github.io/assets/img/post/3/2.png)
![](https://kyuyeop.github.io/assets/img/post/3/3.png)
이 3개의 페이지를 보여주네요.  

List 페이지에서 flag.php에 들어가 내용을 보도록 하겠습니다.
![](https://kyuyeop.github.io/assets/img/post/3/4.png)
그냥 flag를 보여주면 좋으련만, 그렇게 만들리가 없죠. Permission denied가 뜹니다.  
이게 왜 뜨나 view.php를 보겠습니다.  

아, 참고로 view.php인 이유는 index.php 즉 메인 페이지에
```php
<?php
    include $_GET['page']?$_GET['page'].'.php':'main.php';
?>
```
이런 구문이 있기 때문에 주소창에서 page 파라미터에 해당하는 view.php가 보여지고 있다는 것을 알 수 있습니다.  

다시 본론으로 돌아와서, view.php는 이렇게 작성되어 있습니다.
```php
<h2>View</h2>
<pre><?php
    $file = $_GET['file']?$_GET['file']:'';
    if(preg_match('/flag|:/i', $file)){
        exit('Permission denied');
    }
    echo file_get_contents($file);
?>
</pre>
```
file 파라미터로 넘어오는 값에 필터링을 수행하는 부분이 있네요. 이 파일을 이용하긴 힘들어 보입니다.  

그럼 어떻게 해야 할까요?  

index.php 에서도 다른 php 파일을 열어 보여주는 기능이 있었죠? 그렇다면 이걸 활용해 봅시다.

flag.php에 해당하는 경로를 page로 넘겨 봅시다. 이때 page 내용 뒤에 .php를 붙이니 page에는 .php를 제외하고 입력해야 합니다.
![](https://kyuyeop.github.io/assets/img/post/3/5.png)
flag.php가 보이기는 합니다...만 flag는 없고 can you see $flag만 나오고 있네요. 아마도 flag.php에 flag는 프론트엔드로 넘기지 않나 봅니다. 그러면 전체 파일을 읽어서 flag를 가져와야 겠습니다.  
  
이때 사용하는 게 바로 php wrapper 입니다. 그냥 입력된 경로를 include해서 파일을 보여주도록 만들어져 있어서 가능한 일입니다.  

다양한 wrapper가 있지만, 이번에 사용할 것은  
php://filter/convert.base64-encode/resource=경로  
입니다.  

이 wrapper를 사용하면 파일 내용을 base64로 인코딩하여 가져올 수 있습니다. 이렇게 가져오면 <?php ?>가 인식되지 않고 모든 소스를 읽어들일 수 있습니다.  

그래서 page로 php://filter/convert.base64-encode/resource=/var/www/uploads/flag를 넘겨준다면..!
![](https://kyuyeop.github.io/assets/img/post/3/6.png)
이렇게(잘리긴 했지만) base64 인코딩 된 문자열을 얻을 수 있습니다.  

이걸 다시 디코딩 하면
```php
<?php
  $flag = 'DH{bb9db1f303cacf0f3c91e0abca1221ff}';
?>
can you see $flag?
```
이런 결과를 얻을 수 있습니다.  
  
php에서 사용가능한 wrapper는 [여기](https://www.php.net/manual/en/wrappers.php)에서 확인 가능합니다.
{% endraw %}