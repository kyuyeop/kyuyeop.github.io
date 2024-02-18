---
title: (9) Dreamhack [wargame.kr] login filtering 문제 풀이
date: 2024-01-13 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
I have accounts. but, it's blocked.  
can you login bypass filtering?

## 문제 풀이
일단 들어오면
![](https://kyuyeop.github.io/assets/img/post/9/1.png)
이런 사이트가 나옵니다. get source를 누르면 아래 코드가 나옵니다.
```php
<?php

if (isset($_GET['view-source'])) {
    show_source(__FILE__);
    exit();
}

/*
create table user(
 idx int auto_increment primary key,
 id char(32),
 ps char(32)
);
*/

 if(isset($_POST['id']) && isset($_POST['ps'])){
  include("./lib.php"); # include for $FLAG, $DB_username, $DB_password.

  $conn = mysqli_connect("localhost", $DB_username, $DB_password, "login_filtering");
  mysqli_query($conn, "set names utf8");

  $id = mysqli_real_escape_string($conn, trim($_POST['id']));
  $ps = mysqli_real_escape_string($conn, trim($_POST['ps']));

  $row=mysqli_fetch_array(mysqli_query($conn, "select * from user where id='$id' and ps=md5('$ps')"));
  if(isset($row['id'])){
   if($id=='guest' || $id=='blueh4g'){
    echo "your account is blocked";
   }else{
    echo "login ok"."<br />";
    echo "FLAG : ".$FLAG;
   }
  }else{
   echo "wrong..";
  }
 }
?>
<!DOCTYPE html>
<style>
 * {margin:0; padding:0;}
 body {background-color:#ddd;}
 #mdiv {width:200px; text-align:center; margin:50px auto;}
 input[type=text],input[type=[password] {width:100px;}
 td {text-align:center;}
</style>
<body>
<form method="post" action="./">
<div id="mdiv">
<table>
<tr><td>ID</td><td><input type="text" name="id" /></td></tr>
<tr><td>PW</td><td><input type="password" name="ps" /></td></tr>
<tr><td colspan="2"><input type="submit" value="login" /></td></tr>
</table>
 <div><a href='?view-source'>get source</a></div>
</form>
</div>
</body>
<!--

you have blocked accounts.

guest / guest
blueh4g / blueh4g1234ps

-->
```
코드를 보니 guest, blueh4g가 아닌 계정으로 로그인에 성공하면 flag를 준답니다.  
  
이 문제는 mysql을 쓰고 있는데, mysql은 대소문자를 가리지 않습니다.  
가장 아래 있는 주석을 보면 아이디와 비밀번호로 추정되는게 있습니다.
따라서 그냥 GUEST / guest로 로그인하면 id가 소문자 guest가 아니니 로그인이 됩니다.
![](https://kyuyeop.github.io/assets/img/post/9/2.png)
flag가 출력됩니다.  
  
쉽습니다.
{% endraw %}