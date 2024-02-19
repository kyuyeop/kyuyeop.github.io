---
title: (15) Dreamhack NoSQL-CouchDB 문제 풀이
date: 2024-01-18 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 풀이
app.js
```javascript
var createError = require('http-errors');
var express = require('express');
var path = require('path');
var cookieParser = require('cookie-parser');

const nano = require('nano')(`http://${process.env.COUCHDB_USER}:${process.env.COUCHDB_PASSWORD}@couchdb:5984`);
const users = nano.db.use('users');
var app = express();

// view engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'ejs');

app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, 'public')));

/* GET home page. */
app.get('/', function(req, res, next) {
  res.render('index');
});

/* POST auth */
app.post('/auth', function(req, res) {
    users.get(req.body.uid, function(err, result) {
        if (err) {
            console.log(err);
            res.send('error');
            return;
        }
        if (result.upw === req.body.upw) {
            res.send(`FLAG: ${process.env.FLAG}`);
        } else {
            res.send('fail');
        }
    });
});

// catch 404 and forward to error handler
app.use(function(req, res, next) {
  next(createError(404));
});

// error handler
app.use(function(err, req, res, next) {
  // set locals, only providing error in development
  res.locals.message = err.message;
  res.locals.error = req.app.get('env') === 'development' ? err : {};

  // render the error page
  res.status(err.status || 500);
  res.render('error');
});

module.exports = app;
```
  
views/index.ejs
```html
<html>
    <head>
        <link rel="stylesheet" href="/css/bulma.min.css" />
    </head>
    <body>
        <div class="container card">
            <div class="card-content">
                <h1 class="title">Login</h1>
                <form id="form">
                <div class="field">
                    <label class="label">uid</label>
                    <input class="input" type="text" placeholder="uid" name="uid" required>
                </div>

                <div class="field">
                    <label class="label">upw</label>
                    <input class="input" type="password" placeholder="upw" name="upw" required>
                </div>

                <div class="field is-grouped">
                    <div class="control">
                    <input class="button is-success" type="submit" value="Login"/>
                    </div>
                    <div class="control">
                    <input class="button" type="reset" value="Cancel"/>
                    </div>
                </div>
                </form>
            </div>
        </div>
        <div id="modal-div" class="modal">
            <div class="modal-background"></div>
            <div class="modal-content">
              <div class="box">
                <p id="modal-text"></p>
              </div>
            </div>
            <button class="modal-close is-large" aria-label="close"></button>
        </div>
        <script src="/js/jquery-3.6.0.min.js"></script>
        <script>
            $.fn.serializeObject = function () {
                'use strict';
                var result = {};
                var extend = function (i, element) {
                    var node = result[element.name];
                    if ('undefined' !== typeof node && node !== null) {
                    if ($.isArray(node)) {
                        node.push(element.value);
                    } else {
                        result[element.name] = [node, element.value];
                    }
                    } else {
                    result[element.name] = element.value;
                    }
                };

                $.each(this.serializeArray(), extend);
                return result;
            };
            $("#form").submit(function( event ) {
                event.preventDefault();
                $("#form").serializeObject()
                $.ajax({
                    type:"POST",
                    data: JSON.stringify($("#form").serializeObject()),
                    dataType:"json",
                    url: "/auth",
                    contentType:"application/json",
                }).always(function(e){
                    const $target = document.getElementById('modal-div');
                    document.getElementById('modal-text').innerText = e.responseText;
                    openModal($target);
                });
            });
            /* modal */
            // Functions to open and close a modal
            function openModal($el) {
                $el.classList.add('is-active');
            }

            function closeModal($el) {
                $el.classList.remove('is-active');
            }

            function closeAllModals() {
                (document.querySelectorAll('.modal') || []).forEach(($modal) => {
                closeModal($modal);
                });
            }

            // Add a click event on various child elements to close the parent modal
            (document.querySelectorAll('.modal-background, .modal-close, .modal-card-head .delete, .modal-card-foot .button') || []).forEach(($close) => {
                const $target = $close.closest('.modal');
                $close.addEventListener('click', () => {
                closeModal($target);
                });
            });

            // Add a keyboard event to close all modals
            document.addEventListener('keydown', (event) => {
                const e = event || window.event;
                if (e.keyCode === 27) { // Escape key
                closeAllModals();
                }
            });
        </script>
    </body>
</html>
```
app.js 32번째 줄의 `result.upw === req.body.upw` 조건을 만족시켜야 합니다.  
  
작동 방식은 form 입력값을 json형태로 /auth 페이지로 보내고, /auth에서 db에서 입력한 uid해당하는 유저의 upw가져와 입력된 upw와 비교해서 맞다면 flag를 보여주는 방식입니다.  
  
CouchDB의 특징은 [여기](https://dreamhack.io/lecture/courses/293)서 확인할 수 있습니다.  
  
`result.upw === req.body.upw` 조건을 맍혹하는 upw brute force로 찾는것도 불가능하진 않겠으나, 당연히 그러라고 만든 문제는 아니겠죠.

그렇다면 양쪽 값을 모두 undefined로 바꾼다면 어떨까요?  
백엔드에서 입력값을 검사하지 않기 때문에 req.body.uid를 undefined로 만들기는 어렵지 않습니다. 문제는 db의 결과 입니다. 결과에는 upw값이 포함되어 있으면 안됩니다.  
  
그래서 \_all_docs를 사용합니다. \_all_docs를 입력으로 넣으면 반환으로 모든 도큐먼트가 나옵니다. 따라서 upw인 키를 찾을 수 없게 됩니다. upw값이 없으니 undefined가 될거고, 결과적으로 `result.upw === req.body.upw` 조건이 참이 됩니다.  
  
실제로 어떻게 공격하는지 알아봅시다. 저는 burp suite를 이용했습니다.  
intercept on을 해주고 uid에는 \_all_docs, upw에는 아무 값이나 넣고 login을 누릅니다.
![](https://kyuyeop.github.io/assets/img/post/15/1.png)
그러면 아래와 같이 보내려던 POST 요청을 잡을 수 있습니다.
![](https://kyuyeop.github.io/assets/img/post/15/2.png)
아래와 같이 데이터 중에서 upw의 값을 날려버립니다. 즉, upw가 undefined가 되도록 하는거죠.
![](https://kyuyeop.github.io/assets/img/post/15/3.png)
그리고 intercept off해주면
![](https://kyuyeop.github.io/assets/img/post/15/34.png)
flag가 나타납니다.
{% endraw %}