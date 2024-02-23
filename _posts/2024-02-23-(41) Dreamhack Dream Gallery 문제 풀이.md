---
title: (41) Dreamhack Dream Gallery 문제 풀이
date: 2024-02-23 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
드림이는 갤러리 사이트를 구축했습니다.  
그런데 외부로 요청하는 기능이 안전한 건지 모르겠다고 하네요...  
갤러리 사이트에서 취약점을 찾고 flag를 획득하세요!  
flag는 `/flag.txt`에 있습니다.

## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/41/1.png)
처음 사이트에 들어가면 선글라스들이 있습니다.  

![](https://kyuyeop.github.io/assets/img/post/41/2.png)
외부로 요청하는 기능에 취약점이 있다고 했으니 이 페이지가 우리가 flag를 얻는데 필요한 페이지인듯 합니다.  
  
코드를 살펴봅시다.
```python
from flask import Flask, request, render_template, url_for, redirect
from urllib.request import urlopen
import base64, os

app = Flask(__name__)
app.secret_key = os.urandom(32)

mini_database = []


@app.route('/')
def index():
    return redirect(url_for('view'))


@app.route('/request')
def url_request():
    url = request.args.get('url', '').lower()
    title = request.args.get('title', '')
    if url == '' or url.startswith("file://") or "flag" in url or title == '':
        return render_template('request.html')

    try:
        data = urlopen(url).read()
        mini_database.append({title: base64.b64encode(data).decode('utf-8')})
        return redirect(url_for('view'))
    except:
        return render_template("request.html")


@app.route('/view')
def view():
    return render_template('view.html', img_list=mini_database)


@app.route('/upload', methods=['GET', 'POST'])
def upload():
    if request.method == 'POST':
        f = request.files['file']
        title = request.form.get('title', '')
        if not f or title == '':
            return render_template('upload.html')

        en_data = base64.b64encode(f.read()).decode('utf-8')
        mini_database.append({title: en_data})
        return redirect(url_for('view'))
    else:
        return render_template('upload.html')


if __name__ == "__main__":
    img_list = [
        {'초록색 선글라스': "static/assetA#03.jpg"}, 
        {'분홍색 선글라스': "static/assetB#03.jpg"},
        {'보라색 선글라스': "static/assetC#03.jpg"}, 
        {'파란색 선글라스': "static/assetD#03.jpg"}
    ]
    for img in img_list:
        for k, v in img.items():
            data = open(v, 'rb').read()
            mini_database.append({k: base64.b64encode(data).decode('utf-8')})
    
    app.run(host="0.0.0.0", port=80, debug=False)
```
취약점을 찾아야 할 곳은 `/request`이니 해당 부분만 중점적으로 보겠습니다.  
이 페이지는 다음과 같이 작동합니다.  
1. url은 소문자로, title은 그대로 가져와 변수에 저장.  
2. url, title이 비어 있거나, url이 `file://`으로 시작하거나, `flag`라는 키워드가 url에 포함되어 있다면 해당 페이지를 다시 렌더하고 종료.  
3. 위 조건을 만족하면 입력한 url을 읽고, base64로 인코딩하여 딕셔너리 형태로 mini_database에 저장. 완료 후 `/view`로 리다이렉트.  
4. `3`을 실행하면서 에러가 있다면 해당 페이지를 다시 렌더.  
  
flag가 `/flag.txt`에 있다고 했으므로 우리는 `file:///flag.txt`과 같은 역할을 수행할 수 있는 url을 넣어줘야 합니다.  
  
그렇기 위해서는 필터링을 우회해야 합니다.
### file:// 우회
1. 공백을 이용해 우회 ` file:///flag.txt`
2. `<` `>`를 이용해 우회 `<file:///flag.txt>`
3. `/`을 하나만 이용해 우회 `file:/flag`

### flag 우회
URL Double Encoding을 이용해 우회 `file:///fla%67.txt`(form에서 넘겨질 때 :`file:///fla%2567.txt`)  
변수에 저장될 때 한번, urlopen 과정에서 한번, 총 두번 디코딩이 되기 때문에 더블 인코딩을 통해 우회가 가능합니다.  
<br>
  
위와 같은 방법으로 우회하게 된다면 결과적으로 아래와 같은 url을 입력해야 합니다.
```
 file:///fla%67.txt
<file:///fla%67.txt>
file:/fla%67.txt
```
이번에는 가장 아래의 페이로드를 이용해 보겠습니다.  

![](https://kyuyeop.github.io/assets/img/post/41/3.png)
위와 같이 적고 요청을 누르면 정상적으로 처리가 되고 `/view`로 이동됩니다.  

![](https://kyuyeop.github.io/assets/img/post/41/4.png)
가장 아래쪽을 보면 방금 만들었던 `flag`라는 이름이 보입니다.  

![](https://kyuyeop.github.io/assets/img/post/41/5.png)
개발자 도구를 열고 이미지 부분의 코드를 확인하면 위와 같이 base64 인코딩된 값을 얻을 수 있습니다.  
이렇게 얻은 `REh7YjIwMzdhMDI2YjQwY2M5ODgwNGU5MWI1YTJhMDdmNTR9`을 base64 디코딩 툴을 이용해 변환하면 `DH{b2037a026b40cc98804e91b5a2a07f54}`라는 flag를 얻을 수 있습니다.
{% endraw %}