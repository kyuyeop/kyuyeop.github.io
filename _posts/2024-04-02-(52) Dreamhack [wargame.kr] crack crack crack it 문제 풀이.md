---
title: (52) Dreamhack [wargame.kr] crack crack crack it 문제 풀이
date: 2024-04-02 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
.htaccess crack!  
  
can you local bruteforce attack?

## 문제 풀이
![](https://kyuyeop.github.io/assets/img/post/52/1.png)

사이트에 들어가면 위와 같습니다. 별건 없고 `.htpasswd`를 다운받을 수 있도록 되어 있습니다.  
아마 패스워드를 찾아내서 입력하면 flag를 주는 문제인가 봅니다.

### .htpasswd
`.htpasswd`는 HTTP인증을 이용하여 사이트를 보호할때 사용하는 파일로, 파일의 각 라인에는 사용자명과 암호가 들어있습니다.
그 규칙은 다음과 같습니다.
```
[사용자명]:$[암호화방법]$[salt]$[해시]
```  
  

그럼 일단 `.htpasswd`에 뭐가 들었는지 확인해 보겠습니다.
```
blueh4g:$1$J/RUeWXk$S9KADgOh4EsXyh9U83JOi.
```
제 경우에는 이런 내용이 들어있습니다. 아마 이 글을 보시는 분은 저랑은 ip가 다를거고, `salt`도 랜덤으로 들어가는듯 하니, 아마 다른 문자열이 들어있을 겁니다.  
`blueh4g`는 사용자 이름이고, `1`은 `md5_crypt`를 뜻합니다. `salt`로는 `J/RUeWXk`가 사용되고 패스워드를 암호화하면 `J/RUeWXk$S9KADgOh4EsXyh9U83JOi.`가 됩니다.  
  
이제 이걸 만족하는 패스워드를 무차별 대입을 이용해서 찾아내야 합니다.  
저는 파이썬으로 간단하게 만들어 보았습니다.
```python
import string
import itertools
from tqdm import tqdm
from passlib.hash import md5_crypt

known_passwd = 'G4HeulB'

for i in range(1, 9):
    for j in tqdm(list(itertools.product(string.ascii_lowercase + string.digits, repeat=i))):
        passwd = known_passwd + ''.join(j)
        if md5_crypt.verify(passwd, '$1$J/RUeWXk$S9KADgOh4EsXyh9U83JOi.'):
            print(passwd)
            exit()
```
패스워드의 길이를 모르니 일단 최대 8자리까지 넣을 수는 있도록 했는데 5자리가 넘어가면 너무 오래 걸리기에 그 전에 끝날겁니다.(실제로도 4자리에서 끝)  
  
3분 정도 걸려서 `G4HeulBz130`라는 값을 찾았습니다. 이 값을 페이지의 입력란에 넣어주면
![](https://kyuyeop.github.io/assets/img/post/52/2.png)

이렇게 flag를 얻을 수 있습니다.
{% endraw %}