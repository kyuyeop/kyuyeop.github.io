---
title: (29) Dreamhack Tomcat Manager
date: 2024-02-04 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
드림이가 톰캣 서버로 개발을 시작하였습니다.
서비스의 취약점을 찾아 플래그를 획득하세요.
플래그는 `/flag` 경로에 있습니다.

## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/29/1.png)
일단 들어오면 이것 밖에 없습니다.  
뭐라도 해야하니까 문제 파일을 열어보겠습니다.  
![](https://kyuyeop.github.io/assets/img/post/29/2.png)  
이 3가지 밖에 없습니다. 소스 코드는 없는걸까요?  
  
ROOT.war 파일이 소스코드들이 모여있는 파일입니다. 반디집 같은 프로그램으로 열어보면  
![](https://kyuyeop.github.io/assets/img/post/29/3.png)  
이런 파일들이 있습니다. 코드들을 확인해봅시다.  
  
먼저 `index.jsp`입니다.
```jsp
<html>
<body>
    <center>
        <h2>Under Construction</h2>
        <p>Coming Soon...</p>
        <img src="./image.jsp?file=working.png"/>
    </center>
</body>
</html>
```
딱 html코드만 있네요. 그리고 `image.jsp?file=working.png`를 불러와서 띄워주고 있습니다. 그러면 `image.jsp`가 뭐하는 놈인지 알아봅시다.
```jsp
<%@ page trimDirectiveWhitespaces="true" %>
<%
String filepath = getServletContext().getRealPath("resources") + "/";
String _file = request.getParameter("file");

response.setContentType("image/jpeg");
try{
    java.io.FileInputStream fileInputStream = new java.io.FileInputStream(filepath + _file);
    int i;   
    while ((i = fileInputStream.read()) != -1) {  
        out.write(i);
    }   
    fileInputStream.close();
}catch(Exception e){
    response.sendError(404, "Not Found !" );
}
%>
```
get으로 `file`을 받아서 해당 위치의 파일을 돌려주는 형식입니다.  
  
여기서 눈여겨 봐야하는 것은 그냥 파일을 읽어서 돌려주는게 전부라는 겁니다. 즉, 이미지일 필요가 없다는 뜻이죠. 경로만 잘 입력하면 원하는 파일을 읽을 수 있습니다.
### 트래버셜(traversal) 취약점
트래버셜 취약점은 `../`를 이용해 지정된 루트 폴더의 상위폴더로 이동할 수 있는 취약점입니다.  
예를 들어 `/a/b/c/image.png`가 있다고 할때 웹서버에서 `c`를 메인 디렉토리로 지정했다고 합시다.  
일반적인 접근이라면 `www.hacking.com/image.png`로 접근하게 되지만 `www.hacking.com/../../flag`에 접근하게 된다면, 그리고 권한이 있다면 이 파일을 읽을 수 있게 됩니다.
### 로컬 파일 인클루전(LFI) 취약점
  
로컬에 존재하는 파일을 불러오는 코드가 있을 때 트래버셜 취약점과 폴더 경로를 조합하여 로컬에 있는 임의의 파일을 읽을 수 있는 취약점입니다.  
<br>  

그럼 어떤 파일을 읽어야 할까요? 이 문제의 이름에 힌트가 있습니다. 바로 관리자 페이지 입니다. 톰캣의 경우 관리자 페이지는 `/manager/html` 로 접속하면 나옵니다.
![](https://kyuyeop.github.io/assets/img/post/29/4.png)
물론 위와 같이 로그인이 필요합니다.  
  
여기에 접속할 수 있는 유저 정보는 `[톰캣경로]/conf/tomcat-users.xml`에 존재합니다.
  
이를 읽을수 있으면 로그인 할 수 있겠죠.  
  
폴더구조를 완벽히 알 순 없기 때문에 `conf/tomcat-users.xml`를 두고 `../`를 추가하며 찾았습니다.  
  
`../../../conf/tomcat-users.xml`를 읽어 봅시다. 일반적으로 접속하면 이미지로 처리되어 읽을 수가 없으니 burp suite를 쓰거나 페이지를 저장해서 보시면 됩니다.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">

    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <role rolename="manager-jmx"/>
    <role rolename="manager-status"/>
    <role rolename="admin-gui"/>
    <role rolename="admin-script"/>
    <user username="tomcat" password="P2assw0rd_4_t0mC2tM2nag3r31337" roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script" />  
</tomcat-users>
```
돌아온 응답을 보면 위와 같습니다. 어드민 계정은 `tomcat / P2assw0rd_4_t0mC2tM2nag3r31337` 이네요. 이제 접속하면
![](https://kyuyeop.github.io/assets/img/post/29/5.png)
이 페이지를 볼 수 있겠습니다.  
  
내려보면
![](https://kyuyeop.github.io/assets/img/post/29/6.png)
이렇게 war파일을 올릴 수 있는 곳이 있습니다. 여기에 웹쉘을 업로드 해봅시다.  
  
[여기](https://github.com/BustedSec/webshell/blob/master/webshell.war)에 올라와 있는 웹쉘을 다운받아서 deploy 해줬습니다.
![](https://kyuyeop.github.io/assets/img/post/29/7.png)
이렇게 webshell 폴더가 생겼네요.  
이제 `/webshell`로 접속해서
![](https://kyuyeop.github.io/assets/img/post/29/8.png)
`/flag`를 입력하게 된다면
![](https://kyuyeop.github.io/assets/img/post/29/9.png)
flag를 얻을 수 있습니다.
{% endraw %}