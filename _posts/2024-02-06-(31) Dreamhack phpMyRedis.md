---
title: (31) Dreamhack phpMyRedis
date: 2024-02-06 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
php로 redis를 관리하는 서비스 입니다.
취약점을 찾고 flag를 획득하세요!

## 문제 풀이
이건 처음 접속 화면입니다.
![](https://kyuyeop.github.io/assets/img/post/31/1.png)
`index.php`의 코드는 다음과 같습니다.
```php
<?php
    include_once "./core.php";
?>
<html>
    <head></head>
    <link rel="stylesheet" href="/static/bulma.min.css" />
    <body>
        <div class="container card">
            <div class="card-content">
                <div class="columns">
                    <div class="column is-10">
                        <h1 class="title">phpMyRedis</h1>
                    </div>
                    <div>
                        <div class="column is-2"><a href="/config.php" class="card-footer-item">Config</a></div>
                    </div>
                </div>
                <form method="post">
                    <div class="field">
                        <label class="label">Command</label>
                        <div class="control">
                            <textarea class="textarea" name="cmd"><?=isset($_POST['cmd'])?$_POST['cmd']:'return 1;'?></textarea>
                        </div>
                        <label class="checkbox">
                            <input type="checkbox" name="save">Save
                        </label>
                    </div>
                    <div class="control">
                        <input class="button is-success" type="submit" value="submit">
                    </div>
                </form>
                <?php 
                    if(isset($_POST['cmd'])){
                        $redis = new Redis();
                        $redis->connect($REDIS_HOST);
                        $ret = json_encode($redis->eval($_POST['cmd']));
                        echo '<h1 class="subtitle">Result</h1>';
                        echo "<pre>$ret</pre>";
                        if (!array_key_exists('history_cnt', $_SESSION)) {
                            $_SESSION['history_cnt'] = 0;
                        }
                        $_SESSION['history_'.$_SESSION['history_cnt']] = $_POST['cmd'];
                        $_SESSION['history_cnt'] += 1;

                        if(isset($_POST['save'])){
                            $path = './data/'. md5(session_id());
                            $data = '> ' . $_POST['cmd'] . PHP_EOL . str_repeat('-',50) . PHP_EOL . $ret;
                            file_put_contents($path, $data);
                            echo "saved at : <a target='_blank' href='$path'>$path</a>";
                        }
                    }
                ?>
            </div>
        </div>
        <br/>
        <div class="container card">
            <div class="card-content">
                <div class="columns">
                    <div class="column is-10">
                        <h1 class="title">Command History</h1>
                    </div>
                    <div class="column is-2"><a href="/reset.php" class="card-footer-item">Reset</a></div>
                </div>
                <div class="content">
                    <ul>
                    <?php
                        for($i=0; $i<$_SESSION['history_cnt']; $i++){
                            echo "<li>".$_SESSION['history_'.$i]."</li>";
                        }
                    ?>
                    </ul>
                </div>
            </div>
        </div>
    </body>
</html>
```
위 코드를 보면 post로 `command`를 입력받고 `$redis->eval`을 통해 lua 스크립트를 실행합니다. 아래는 히스토리 저장 코드 이고요.  
![](https://kyuyeop.github.io/assets/img/post/31/2.png)
config 페이지 입니다. get, set으로 redis의 설정을 조회, 변경 할 수 있습니다. `config.php`의 내용은 다음과 같습니다.
```php
<?php
    include_once "./core.php";
?>
<html>
    <head></head>
    <link rel="stylesheet" href="/static/bulma.min.css" />
    <body>
        <div class="container card">
            <div class="card-content">
                <div class="columns">
                    <div class="column is-10">
                        <h1 class="title">phpMyRedis</h1>
                    </div>
                    <div>
                        <div class="column is-2"><a href="/" class="card-footer-item">Command</a></div>
                    </div>
                </div>
                <form method="post">
                    <label class="label">Config</label>
                    <div class="field">
                        <div class="control">
                            <div class="select">
                                <select name="option">
                                    <option>GET</option>
                                    <option>SET</option>
                                </select>
                            </div>
                        </div>
                    </div>
                    <div class="field">
                        <label class="label">Key</label>
                        <div class="control">
                            <input class="input" type="text" name="key">
                        </div>
                    </div>
                    <div class="field">
                        <label class="label">Value</label>
                        <div class="control">
                            <input class="input" type="text" name="value">
                        </div>
                    </div>
                    <div class="control">
                        <input class="button is-success" type="submit" value="submit">
                    </div>
                </form>
                <?php 
                    if(isset($_POST['option'])){
                        $redis = new Redis();
                        $redis->connect($REDIS_HOST);
                        if($_POST['option'] == 'GET'){
                            $ret = json_encode($redis->config($_POST['option'], $_POST['key']));
                        }elseif($_POST['option'] == 'SET'){
                            $ret = $redis->config($_POST['option'], $_POST['key'], $_POST['value']);
                        }else{
                            die('error !');
                        }                        
                        echo '<h1 class="subtitle">Result</h1>';
                        echo "<pre>$ret</pre>";
                    }
                ?>
            </div>
        </div>
    </body>
</html>
```
sql은 redis가 알아서 관리하기 때문에 sqli같은 취약점은 없습니다.  
  
일단 코드에는 딱히 취약점이 없고, redis 시스템을 이용해야 할듯 합니다.  
일단 config 페이지에서 redis의 설정을 조회, 변경할 수 있으니 일단 이 페이지를 이용해 봅시다.
![](https://kyuyeop.github.io/assets/img/post/31/3.png)
key에 `dbfilename`을 넣어보면 `dump.rdb`가 나옵니다. 이 `dbfilename`은 데이터가 저장될 파일의 경로입니다.
![](https://kyuyeop.github.io/assets/img/post/31/4.png)
`save`를 넣으면 위와 같은 결과가 나옵니다. `save`의 설정에 공식 사이트의 설명은 다음과 같습니다.
```
# Save the DB to disk.
#
# save <seconds> <changes> [<seconds> <changes> ...]
#
# Redis will save the DB if the given number of seconds elapsed and it
# surpassed the given number of write operations against the DB.
#
# Snapshotting can be completely disabled with a single empty string argument
# as in following example:
#
# save ""
#
# Unless specified otherwise, by default Redis will save the DB:
#   * After 3600 seconds (an hour) if at least 1 change was performed
#   * After 300 seconds (5 minutes) if at least 100 changes were performed
#   * After 60 seconds if at least 10000 changes were performed
#
# You can set these explicitly by uncommenting the following line.
#
# save 3600 1 300 100 60 10000
```
![](https://kyuyeop.github.io/assets/img/post/31/5.png)
위와 같이 db 파일 이름을 shell.php로 바꾸고,
![](https://kyuyeop.github.io/assets/img/post/31/6.png)
그리고 저장 간격을 10초마다 1개 값에 대한 요청이 있을때 저장하도록 바꿨습니다.  
  
이렇게 설정되면 뭔가 변경된 내용이 있으면 원래 있던 dump.rdp대신 shell.php에 변경값이 들어갈 겁니다.
![](https://kyuyeop.github.io/assets/img/post/31/7.png)
그리고 위와 같은 명령어로 shell 이라는 key에 php 웹쉘 코드를 저장해버렸습니다. 저 파일에 접속한다면 다른 기본적인 형태를 구성하는 바이너리와 함께 `<?php ?>`는 php코드대로 해석되어 보여질 겁니다.    
  
그리고 shell.php?cmd=명령어로 접속하면 명령어 실행이 가능합니다.
![](https://kyuyeop.github.io/assets/img/post/31/8.png)
이렇게 ls를 입력해봤습니다. 잘 되는것 같아 보이네요.  
명령어에 /flag를 입력해보니
![](https://kyuyeop.github.io/assets/img/post/31/9.png)
flag가 나옵니다.
{% endraw %}