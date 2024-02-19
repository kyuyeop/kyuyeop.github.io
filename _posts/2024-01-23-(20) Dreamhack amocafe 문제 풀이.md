---
title: (20) Dreamhack amocafe 문제 풀이
date: 2024-01-23 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
아모카페에 오신 것을 환영합니다!  
메뉴 번호를 입력하여 주문할 수 있는 웹 서비스가 작동하고 있습니다. 아모의 최애 메뉴를 대신 주문해 주면 아모가 플래그를 준다고 합니다. 첨부파일로 주어지는 웹 서비스의 코드를 분석하여 메뉴 번호를 알아 내세요! 플래그는 `flag.txt` 파일과 `FLAG` 변수에 있습니다.
문제에서 주어진 `flag.txt` 파일에 적혀있는 플래그는 sample 플래그입니다.  
플래그 형식은 DH{...} 입니다.

## 문제 풀이
코드부터 확인해봅시다.
```python
#!/usr/bin/env python3
from flask import Flask, request, render_template

app = Flask(__name__)

try:
    FLAG = open("./flag.txt", "r").read()       # flag is here!
except:
    FLAG = "[**FLAG**]"

@app.route('/', methods=['GET', 'POST'])
def index():
    menu_str = ''
    org = FLAG[10:29]
    org = int(org)
    st = ['' for i in range(16)]

    for i in range (0, 16):
        res = (org >> (4 * i)) & 0xf
        if 0 < res < 12:
            if ~res & 0xf == 0x4:
                st[16-i-1] = '_'
            else:
                st[16-i-1] = str(res)
        else:
            st[16-i-1] = format(res, 'x')
    menu_str = menu_str.join(st)

    # POST
    if request.method == "POST":
        input_str =  request.form.get("menu_input", "")
        if input_str == str(org):
            return render_template('index.html', menu=menu_str, flag=FLAG)
        return render_template('index.html', menu=menu_str, flag='try again...')
    # GET
    return render_template('index.html', menu=menu_str)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```
일련의 과정에따라 `menu_str`이 만들어 집니다.  
  
그래서 사이트에 들어가보면
![](https://kyuyeop.github.io/assets/img/post/20/1.png)
위와 같이 코드에 따라 `menu_str`이 보여집니다. 계산된 `menu_str`은 `1_c_3_c_0__ff_3e`입니다.  
  
flag는 인풋의 입력값과 `str(int(FLAG[10:29]))`의 값이 같으면 얻을 수 있습니다.  
  
아마 이번 문제는 `menu_str`의 생성과정을 역으로 수행하여 `1_c_3_c_0__ff_3e`가 어떤 값을 통해 만들어진건지 알아내야 하는듯 합니다.  
  
그럼 생성 코드만 떼어내서 자세히 살펴보겠습니다.
```python
menu_str = ''
org = FLAG[10:29]
org = int(org)
st = ['' for i in range(16)]

for i in range (0, 16):
    res = (org >> (4 * i)) & 0xf
    if 0 < res < 12:
        if ~res & 0xf == 0x4:
            st[16-i-1] = '_'
        else:
            st[16-i-1] = str(res)
    else:
        st[16-i-1] = format(res, 'x')
menu_str = menu_str.join(st)
```
알기 쉽게 코드를 정리하면 다음과 같이 진행됩니다.
```
1.FLAG[10:29]를 int로 변환해서 org 변수에 넣습니다.
2.org를 왼쪽으로 4*i만큼(4비트씩) 비트쉬프트하여 마지막 자리의 16진수값을 가져와 res에 넣습니다.(0xf와 and연산을 하므로 한자리만 남음)
3.이렇게 구한 res가 0(0x0)보다 크고 12(0xc)보다 작고, res에 not 연산후 마지막 자리의 16진수값이 4(0x4)이면 menu_str의 앞에 _를 추가
4.res가 0(0x0)보다 크고 12(0xc)보다 작고, res에 not 연산후 마지막 자리의 16진수값이 4(0x4)가 아니면 res값(int)을 menu_Str의 앞에 추가
5.res가 0(0x0)이거나 12(0xc)과 같거나 크면 res(hex)값을 menu_str의 앞에 추가합니다.
```
이제 본격적으로 역 변환을 해보겠습니다. 그냥 하나씩 수동으로 해도 되지만, 폼이 죽으므로 코드를 짜보겠습니다.
```python
menu_str = '1_c_3_c_0__ff_3e'
originalHex = ''

for i in menu_str:
  if i == '_':
    originalHex += 'b'
    continue
  if 0 < int(i, 16) < 12:
    originalHex += str(int(i, 16))
    continue
  originalHex += i
print(f'hex = 0x{originalHex}')
print(f'int = {int(originalHex, 16)}')
```
각 조건별로 코드를 짰습니다. 그대로 원본코드에 사실 문제가 있습니다. 바로 10과 11은 그대로 출력되버려서 역연산 할때 1,0인지 10인지 구별이 안됩니다. 다행히 준 곳엔 2자리 자연수가 등장하지 않아서 다행이죠.  
  
위 코드르 실행하면
```
hex = 0x1bcb3bcb0bbffb3e
int = 2002760202557848382
```
가 나옵니다.

`FLAG[10:29]`의 값이 `2002760202557848382`인가 봅니다.  
입력해주면
![](https://kyuyeop.github.io/assets/img/post/20/2.png)
![](https://kyuyeop.github.io/assets/img/post/20/3.png)
flag가 나옵니다.
{% endraw %}