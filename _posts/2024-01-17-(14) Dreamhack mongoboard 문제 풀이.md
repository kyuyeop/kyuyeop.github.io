---
title: (14) Dreamhack mongoboard 문제 풀이
date: 2024-01-17 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
node와 mongodb로 구성된 게시판입니다.  
비밀 게시글을 읽어 FLAG를 획득하세요.  
* MongoDB < 4.0.0

## 문제 풀이
먼저 사이트에 들어가 보겠습니다.
![](https://kyuyeop.github.io/assets/img/post/14/1.png)
Write에는 아래와 같이 글을 작성할 수 있는 기능이 있네요.
![](https://kyuyeop.github.io/assets/img/post/14/2.png)
![](https://kyuyeop.github.io/assets/img/post/14/3.png)
들어가보면 이렇게 나옵니다.
![](https://kyuyeop.github.io/assets/img/post/14/4.png)
우리가 알아야 하는건 FLAG페이지의 내용일 겁니다. 들어가면 당연히 secret이라고 뜹니다.
![](https://kyuyeop.github.io/assets/img/post/14/5.png)
그럼 본격적으로 시작해 보겠습니다.
```javascript
module.exports = function(app, MongoBoard){
    app.get('/api/board', function(req,res){
        MongoBoard.find(function(err, board){
            if(err) return res.status(500).send({error: 'database failure'});
            res.json(board.map(data => {
                return {
                    _id: data.secret?null:data._id,
                    title: data.title,
                    author: data.author,
                    secret: data.secret,
                    publish_date: data.publish_date
                }
            }));
        })
    });

    app.get('/api/board/:board_id', function(req, res){
        MongoBoard.findOne({_id: req.params.board_id}, function(err, board){
            if(err) return res.status(500).json({error: err});
            if(!board) return res.status(404).json({error: 'board not found'});
            res.json(board);
        })
    });

    app.put('/api/board', function(req, res){
        var board = new MongoBoard();
        board.title = req.body.title;
        board.author = req.body.author;
        board.body = req.body.body;
        board.secret = req.body.secret || false;

        board.save(function(err){
            if(err){
                console.error(err);
                res.json({result: false});
                return;
            }
            res.json({result: true});

        });
    });
}
```
보드 저장과 불러오는 기능을 api로 구현해 두었습니다. 메인페이지 구현에 대한 코드는 제공하고있지 않네요.  
  
그럼 좀 더 분석해 봅시다. 먼저 개발자도구의 네트워크 페이지를 열고 첫번째 게시글을 들어가 보았습니다.
![](https://kyuyeop.github.io/assets/img/post/14/6.png)
GET으로 /api/board/{게시글의 no값}으로 요청을 보내고 json형식으로 글의 내용 데이터를 받는걸 확인했습니다.  
  
여기서 우리가 봐야하는건 MongoDB의 \_id입니다. 기본적으로 MongoDB는 ObjectsId라는 고유한 값을 모든 도큐먼트에 부여합니다. 따라서 이를 이용해 어떤 도큐먼트인지 구별해 낼 수 있습니다.
  
그리고 이 값은 특정한 구조를 가집니다.
### ObjectsId
첫 4바이트->타임스탬프
MongoDB의 타임스탬프는 1970.1.1부터의 시간을 초 단위로 저장하는데, 이 값을 알면 이 도큐먼트가 언제 생성된건지 알 수 있다.
  
다음 5바이트->랜덤
서로 다른 시스템에서 생성되는 값이 충돌하지 않도록 랜덤 값을 부여한다. 같은 프로세스라면 이 5바이트가 유지된다.
  
마지막 3바이트->카운터
단순히 1씩 증가하는 카운터이다. 1초내에 동일 프로세스의 유일성을 보장한다. 초당256^3개 까지 생성가능하다.
  
즉, FLAG 게시물의 no값 중 앞의 8자리 글자(4바이트)를 알 수 있다면 api에 필터링 과정이 없으므로 내용에 접근할 수 있을 겁니다. 다행히 publish_date에 밀리초 단위의 타임스탬프를 제공하고 있어서 이를 다시 16진수로 바꾸기만 하면 됩니다.
  
앞선 게시물과 1초가 차이나므로 타임스탬프 바이트에 1, 카운터에 1을 더한 값이 구하고자 하는 FLAG의 \_id가 될겁니다.
  
앞선 게시물의 \_id가 `65a78a70 9f527a618c 282f9e` 이므로 FLAG 게시물의 \_id는  `65a78a71 9f527a618c 282f9f`일 겁니다.  
  
구한 \_id의 값을 이용해 `/api/board/65a78a719f527a618c282f9f` 접속해보면..?
![](https://kyuyeop.github.io/assets/img/post/14/7.png)
이렇게 body에 flag가 들어 있는 json이 돌아옵니다.
{% endraw %}