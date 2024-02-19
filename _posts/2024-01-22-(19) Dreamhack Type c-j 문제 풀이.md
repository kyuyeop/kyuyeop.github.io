---
title: (19) Dreamhack Type c-j 문제 풀이
date: 2024-01-22 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
php로 작성된 페이지입니다.  
알맞은 Id과 Password를 입력하여 플래그를 획득하세요.  
플래그의 형식은 DH{...} 입니다.

## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/19/1.png)
들어오면 이런 사이트가 나옵니다.  
바로 코드로 넘어가겠습니다.  
  
`index.php`는 form으로 check.php로 id와 pw만 넘겨주고, `flag.php`는 단순히 flag만 변수로 가지고 있어서 생략합니다.  

`check.php`
```php
<html>
<head>
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
<title>Type c-j</title>
</head>
<body>
    <!-- Fixed navbar -->
    <nav class="navbar navbar-default navbar-fixed-top">
      <div class="container">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Type c-j</a>
        </div>
        <div id="navbar">
          <ul class="nav navbar-nav">
            <li><a href="/">Index page</a></li>
          </ul>
        </div><!--/.nav-collapse -->
      </div>
    </nav><br/><br/><br/>
    <div class="container">
    <?php
    function getRandStr($length = 10) {
        $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
        $charactersLength = strlen($characters);
        $randomString = '';
    
        for ($i = 0; $i < $length; $i++) {
            $randomString .= $characters[mt_rand(0, $charactersLength - 1)];
        }
        return $randomString;

    }
    require_once('flag.php');
    error_reporting(0);
    $id = getRandStr();
    $pw = sha1("1");
    // POST request
    if ($_SERVER["REQUEST_METHOD"] == "POST") {
      $input_id = $_POST["input1"] ? $_POST["input1"] : "";
      $input_pw = $_POST["input2"] ? $_POST["input2"] : "";
      sleep(1);

      if((int)$input_id == $id && strlen($input_id) === 10){
        echo '<h4>ID pass.</h4><br>';
        if((int)$input_pw == $pw && strlen($input_pw) === 8){
            echo "<pre>FLAG\n";
            echo $flag;
            echo "</pre>";
          }
        } else{
          echo '<h4>Try again.</h4><br>';
        }
      }else {
      echo '<h3>Fail...</h3>';
     }
    ?> 
    </div> 
</body>
</html>
```
flag를 얻으려면 `if((int)$input_id == $id && strlen($input_id) === 10)` 조건문과 `if((int)$input_pw == $pw && strlen($input_pw) === 8)` 조건문을 통과해야 합니다.

여기서 중요한 것은 `(int)$input_id == $id` 조건을 어떻게 맞추는지 입니다.  
이걸 알려면 다음 두 가지를 먼저 알아야 합니다.  

### string -> int 형변환
이건 말로 설명하는것 보다 어떻게 작동하는지 직접 보는게 이해가 될겁니다.
```
int('1234asdf') -> 1234
('asdf1234') -> 0
int('a1s2d3f4') -> 0
```
이처럼 가장 처음에 있는 숫자로 이루어진 부분만 가져오고, 문자로 시작하면 `0`을 반환합니다.  
  
### 서로 다른 자료형의 비교
`if(1234 == '1234asdf')` 처럼 조건문을 사용하는 경우 앞서 나온 `int` 형식에 맞추어 비교를 위해 뒤에 오는 문자열을 `int`로 형변환 해서 비교합니다.  
따라서 위 조건문은 참이 되는 거죠.  
<br>
  
먼저 아이디 부터 알아내 봅시다.  
아이디는 `genRandStr`함수에 의해 `0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ` 문자 중에서 랜덤으로 10글자를 골라서 정해집니다. 이때 높은 확률로 아이디는 문자로 시작할 겁니다. 그렇다면 `$id`를 `int`로 형변환 한 값이 `0`이 될겁니다.
  
아이디의 길이는 10자 이므로, 아이디에 임의의 문자 10개를 입력하거나 `0000000000`을 입력하면 아이디는 해결했습니다.  
만약 아이디가 숫자로 시작하는 경우 실패할 수는 있지만, 어차피 매 실행마다 `id`의 값이 바뀌기 때문에 계속해서 같은 요청을 보내면 해결될 겁니다.
  
다음은 비밀번호 입니다.  
`$pw = sha1("1");`에 의해 `$pw`는 `356a192b7913b04c54574d18c28d46e6395428ab` 라는 값을 가집니다.  
이때 마찬가지로 형변환을 하면 356이므로 8자리를 맞춰서 `00000356`이나 `356a192b`, 아니면 356하고 임의의 문자 5개를 입력하면 됩니다.  
  
앞서 설명한 것과 같이 입력하면
![](https://kyuyeop.github.io/assets/img/post/19/2.png)
![](https://kyuyeop.github.io/assets/img/post/19/3.png)
flag가 나옵니다.
{% endraw %}