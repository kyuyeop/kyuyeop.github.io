---
title: (56) Dreamhack [wargame.kr] dun worry about the vase 문제 풀이
date: 2024-04-18 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
중간이 코앞이라 공부할 시간이 부족하네요..
## 문제 설명
Do you know about "padding oracle vulnerability" ?

## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/56/1.png)

처음에 들어가면 다른건 아무것도 없고 로그인 창이 하나 있습니다. 일단 입력되어있는 값을 그대로 두고 로그인을 해보면
![](https://kyuyeop.github.io/assets/img/post/56/2.png)

guest로 로그인이 되긴 하는데 admin 세션을 얻는게 미션이라고 합니다. 주어진 코드도 없고 더 알아낼 수 있는게 없으니 일단 웹 요청을 어떻게 보냈는지 확인해봅시다.

![](https://kyuyeop.github.io/assets/img/post/56/3.png)

burp suite로 확인해보니 쿠키로 `LOg1n=l8dGC9G%2Bnic%3DKtrY4XTvO8U%3D`라는 수상한 값이 있습니다. url decode를 해보면`l8dGC9G+nic=KtrY4XTvO8U=`라는 값입니다.

`=`이 들어있는걸 보니 `l8dGC9G+nic=`와 `KtrY4XTvO8U=`가 각각 뭔가에 base64인코딩을 한 것 같습니다.  
복호화 해보면 둘 다 8바이트의 값을 가지고 있습니다.

문제 설명에서 나온 `padding oracle vulnerability`를 생각해보면 앞서 나온 값 두개는 각각 iv, 암호문 부분이지 싶습니다.

그리고 예측 해볼만한 값이 guest밖에 없기 때문에 이 값들은 이용하면 guest가 나온다고 가정해 봅시다.  
이때 암호화 되는 데이터는 길이가 5이므로 8바이트 블럭에서 guest를 채우 남은 부분은 `\x03`으로 채워집니다. admin도 마찬가지로 길이가 5이므로 `\x03`으로 패딩됩니다.

```python
import base64

iv = base64.b64decode('l8dGC9G+nic=')

expect = b'guest\x03\x03\x03'
target = b'admin\x03\x03\x03'

x = [ iv[i] ^ expect[i] for i in range(8) ]
changed_iv = [ x[i] ^ target[i] for i in range(8) ]

print(base64.b64encode(bytes(changed_iv)))
```
iv와 예상값을 xor하고 다시 원하는 값을 xor해서 iv를 변조하도록 코드를 만들었습니다.
제 경우 실행결과는 `kdZOEcu+nic=`입니다.

![](https://kyuyeop.github.io/assets/img/post/56/4.png)
이렇게 쿠키를 다시 수정해서 보내주면 flag를 얻을 수 있습니다.(이게 적는 동안 서버가 닫혀서 플래그가 출력된 사진이 없는점 양해부탁드립니다.)
{% endraw %}