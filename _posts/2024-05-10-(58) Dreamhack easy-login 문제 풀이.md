---
title: (58) Dreamhack easy-login 문제 풀이
date: 2024-05-10 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
관리자로 로그인하여 플래그를 획득하세요!

플래그 형식은 `DH{...}` 입니다.

## 문제 풀이
처음 사이트에 접속하면
![](https://kyuyeop.github.io/assets/img/post/58/1.png)

이런 로그인 창이 하나 있습니다. 계정도 모르고 otp도 알 수 없으니 일단 코드 부터 확인해 봅시다.

먼저 `index.php`입니다.
```php
<?php

function generatePassword($length) {
    $characters = '0123456789abcdef';
    $charactersLength = strlen($characters);
    $pw = '';
    for ($i = 0; $i < $length; $i++) {
        $pw .= $characters[random_int(0, $charactersLength - 1)];
    }
    return $pw;
}

function generateOTP() {
    return 'P' . str_pad(strval(random_int(0, 999999)), 6, "0", STR_PAD_LEFT);
}

$admin_pw = generatePassword(32);
$otp = generateOTP();

function login() {
    if (!isset($_POST['cred'])) {
        echo "Please login...";
        return;
    }

    if (!($cred = base64_decode($_POST['cred']))) {
        echo "Cred error";
        return;
    }

    if (!($cred = json_decode($cred, true))) {
        echo "Cred error";
        return;
    }

    if (!(isset($cred['id']) && isset($cred['pw']) && isset($cred['otp']))) {
        echo "Cred error";
        return;
    }

    if ($cred['id'] != 'admin') {
        echo "Hello," . $cred['id'];
        return;
    }
    
    if ($cred['otp'] != $GLOBALS['otp']) {
        echo "OTP fail";
        return;
    }

    if (!strcmp($cred['pw'], $GLOBALS['admin_pw'])) {
        require_once('flag.php');
        echo "Hello, admin! get the flag: " . $flag;
        return;
    }

    echo "Password fail";
    return;
}

?>

<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet" type="text/css" href="style.css">
    <title>Easy Login</title>
</head>
<body>
    <div class="login-container">
        <h2>Login as admin to get flag<h2>
        <form action="login.php" method="post">
            <div class="form-group">
                <label for="id">ID</label>
                <input type="text" name="id"></br>
            </div>
            <div class="form-group">
                <label for="pw">PW</label>
                <input type="text" name="pw"></br>
            </div>
            <div class="form-group">
                <label for="otp">OTP</label>
                <input type="text" name="otp"></br>
            </div>
            <button type="submit" class="button">Login</button>
        </form>
        <div class="message">
            <?php login(); ?>
        </div>
    </div>
</body>
</html>

```

그 다음 `login.php`입니다.
```php
<form id="redir" action="index.php" method="post">
    <?php
    $a = array();
    foreach ($_POST as $k => $v) {
        $a[$k] = $v;
    }

    $j = json_encode($a);
    echo '<input type="hidden" name="cred" value="' . base64_encode($j) . '">';
    ?>
</form>

<script type="text/javascript">
    document.getElementById('redir').submit();
</script>
```

`flag.php`는 단순히 flag만 들어있는 파일이라 생략하겠습니다.  
일단 `login.php`는 단순히 들어온 데이터를 json -> base64 순으로 인코딩을 거쳐서 hidden형태의 input으로 돌려줍니다. `index.php`와 데이터를 주고 받기 위한 방식입니다.

중요한 데이터 처리 부분은 `index.php`에서 처리되니 자세히 보도록 하겠습니다.  
password 부분부터 보면 랜덤으로 16진수 형태의 password를 생성하도록 되어 있네요.  
otp도 마찬가지로 'P' + 6자리 랜덤 숫자를 사용합니다.

매 실행마다 위의 두 값이 바뀌기 때문에 이걸 알아내는건 어렵습니다. 대신 우회를 하는 방법을 사용해 보도록 합시다.

```php
if ($cred['otp'] != $GLOBALS['otp']) {
    echo "OTP fail";
    return;
}
```
이렇게 되어 있습니다. `$GLOBALS['otp']`의 값은 P로 시작하는 7자리 랜덤 문자열입니다. 등호 비교에서 `true == '임의의 문자열'`는 항상 참입니다. `$cred`가 index.php를 거쳐서 작동하기 때문에 이 값을 변경할 수 있습니다. 딱히 암호화 되어 있는것도 아니니 수정하는게 어렵지 않죠.

```php
if (!strcmp($cred['pw'], $GLOBALS['admin_pw'])) {
    require_once('flag.php');
    echo "Hello, admin! get the flag: " . $flag;
    return;
}
```
이것만 우회하면 이제 flag를 얻을 수 있습니다. 이건 `strcmp`함수의 취약점을 이용해야 하는 문제입니다.  
특정 버전 이하의 php에서는 배열과 문자열을 `strcmp`에 인자로 집어 넣으면 대소 비교를 못하고 0을 반환하도록 되어 있습니다. 따라서 배열을 넘겨 줌으로서 우회가 가능한 거죠.

burp suite를 켜고 아무값을 입력해서 전송을 한 다음 forward를 해보면
![](https://kyuyeop.github.io/assets/img/post/58/2.png)

이렇게 cred이라는 값이 전달되고 있습니다. 그리고 오른쪽에 디코딩 된 값을 보면 입력한 값이 json형태로 보여집니다. 이 값을 수정해보도록 합시다.  
![](https://kyuyeop.github.io/assets/img/post/58/3.png)

앞서 설명한대로, otp 에는 true를, pw에는 빈 배열을 넣고 요청을 날려보면
![](https://kyuyeop.github.io/assets/img/post/58/4.png)

이렇게 flag가 출력됩니다.
{% endraw %}