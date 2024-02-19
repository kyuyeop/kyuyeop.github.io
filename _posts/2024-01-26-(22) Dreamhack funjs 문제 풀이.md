---
title: (22) Dreamhack funjs 문제 풀이
date: 2024-01-26 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
입력 폼에 데이터를 입력하여 맞으면 플래그, 틀리면 `NOP !`을 출력하는 HTML 페이지입니다.
main 함수를 분석하여 올바른 입력 값을 찾아보세요 !

## 문제 풀이
파일은 `index.html` 하나만 있습니다. 서버도 제공되지 않습니다.
```html
<html>
    <head>
    <style>*{margin: 0;}</style>
    <script>
    var box;
    window.onload = init;
    function init() {
      box = document.getElementById("formbox");
      setInterval(moveBox,1000);
    }
    function moveBox() {
        box.posX = Math.random() * (window.innerWidth - 64); 
        box.posY = Math.random() * (document.documentElement.scrollHeight - 64); 
        box.style.marginLeft = box.posX + "px";
        box.style.marginTop  = box.posY + "px";
        debugger;
    }

    function text2img(text){
        var imglist = box.getElementsByTagName('img');
        while(imglist.length > 0) {imglist[0].remove();}
        var canvas = document.createElement("canvas");
        canvas.width = 620;
        canvas.height = 80;
        var ctx = canvas.getContext('2d');
        ctx.font = "30px Arial";
        var text = text;
        ctx.fillText(text,10,50);
        var img = document.createElement("img");
        img.src = canvas.toDataURL();
        box.append(img);
    };

    function main(){
        var _0x1046=['2XStRDS','1388249ruyIdZ','length','23461saqTxt','9966Ahatiq','1824773xMtSgK','1918853csBQfH','175TzWLTY','flag','getElementById','94hQzdTH','NOP\x20!','11sVVyAj','37594TRDRWW','charCodeAt','296569AQCpHt','fromCharCode','1aqTvAU'];
        var _0x376c = function(_0xed94a5, _0xba8f0f) {
            _0xed94a5 = _0xed94a5 - 0x175;
            var _0x1046bc = _0x1046[_0xed94a5];
            return _0x1046bc;
        };
        var _0x374fd6 = _0x376c;
        (function(_0x24638d, _0x413a92) {
            var _0x138062 = _0x376c;
            while (!![]) {
                try {
                    var _0x41a76b = -parseInt(_0x138062(0x17f)) + parseInt(_0x138062(0x180)) * -parseInt(_0x138062(0x179)) + -parseInt(_0x138062(0x181)) * -parseInt(_0x138062(0x17e)) + -parseInt(_0x138062(0x17b)) + -parseInt(_0x138062(0x177)) * -parseInt(_0x138062(0x17a)) + -parseInt(_0x138062(0x17d)) * -parseInt(_0x138062(0x186)) + -parseInt(_0x138062(0x175)) * -parseInt(_0x138062(0x184));
                    if (_0x41a76b === _0x413a92) break;
                    else _0x24638d['push'](_0x24638d['shift']());
                } catch (_0x114389) {
                    _0x24638d['push'](_0x24638d['shift']());
                }
            }
        }(_0x1046, 0xf3764));
        var flag = document[_0x374fd6(0x183)](_0x374fd6(0x182))['value'],
            _0x4949 = [0x20, 0x5e, 0x7b, 0xd2, 0x59, 0xb1, 0x34, 0x72, 0x1b, 0x69, 0x61, 0x3c, 0x11, 0x35, 0x65, 0x80, 0x9, 0x9d, 0x9, 0x3d, 0x22, 0x7b, 0x1, 0x9d, 0x59, 0xaa, 0x2, 0x6a, 0x53, 0xa7, 0xb, 0xcd, 0x25, 0xdf, 0x1, 0x9c],
            _0x42931 = [0x24, 0x16, 0x1, 0xb1, 0xd, 0x4d, 0x1, 0x13, 0x1c, 0x32, 0x1, 0xc, 0x20, 0x2, 0x1, 0xe1, 0x2d, 0x6c, 0x6, 0x59, 0x11, 0x17, 0x35, 0xfe, 0xa, 0x7a, 0x32, 0xe, 0x13, 0x6f, 0x5, 0xae, 0xc, 0x7a, 0x61, 0xe1],
            operator = [(_0x3a6862, _0x4b2b8f) => {
                return _0x3a6862 + _0x4b2b8f;
            }, (_0xa50264, _0x1fa25c) => {
                return _0xa50264 - _0x1fa25c;
            }, (_0x3d7732, _0x48e1e0) => {
                return _0x3d7732 * _0x48e1e0;
            }, (_0x32aa3b, _0x53e3ec) => {
                return _0x32aa3b ^ _0x53e3ec;
            }],
            getchar = String[_0x374fd6(0x178)];
        if (flag[_0x374fd6(0x17c)] != 0x24) {
            text2img(_0x374fd6(0x185));
            return;
        }
        for (var i = 0x0; i < flag[_0x374fd6(0x17c)]; i++) {
            if (flag[_0x374fd6(0x176)](i) == operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i])) {} else {
                text2img(_0x374fd6(0x185));
                return;
            }
        }
        text2img(flag);
    }
    </script>
    </head>
    <body>
        <div id='formbox'>
            <h2>Find FLAG !</h2>
            <input type='flag' id='flag' value=''>
            <input type='button' value='submit' onclick='main()'>
        </div>
    </body>
</html>
```
일단 코드를 살펴보면 form을 움직이는 코드와 텍스트 입력값을 이미지로 출력하는 함수가 구현되어 있습니다.  
해당 부분은 딱히 중요한건 없어 보이고, main 함수를 분석해야 하는 것 같으니 해당 부분만 자세히 보겠습니다.
```javascript
function main(){
    var _0x1046=['2XStRDS','1388249ruyIdZ','length','23461saqTxt','9966Ahatiq','1824773xMtSgK','1918853csBQfH','175TzWLTY','flag','getElementById','94hQzdTH','NOP\x20!','11sVVyAj','37594TRDRWW','charCodeAt','296569AQCpHt','fromCharCode','1aqTvAU'];
    var _0x376c = function(_0xed94a5, _0xba8f0f) {
        _0xed94a5 = _0xed94a5 - 0x175;
        var _0x1046bc = _0x1046[_0xed94a5];
        return _0x1046bc;
    };
    var _0x374fd6 = _0x376c;
    (function(_0x24638d, _0x413a92) {
        var _0x138062 = _0x376c;
        while (!![]) {
            try {
                var _0x41a76b = -parseInt(_0x138062(0x17f)) + parseInt(_0x138062(0x180)) * -parseInt(_0x138062(0x179)) + -parseInt(_0x138062(0x181)) * -parseInt(_0x138062(0x17e)) + -parseInt(_0x138062(0x17b)) + -parseInt(_0x138062(0x177)) * -parseInt(_0x138062(0x17a)) + -parseInt(_0x138062(0x17d)) * -parseInt(_0x138062(0x186)) + -parseInt(_0x138062(0x175)) * -parseInt(_0x138062(0x184));
                if (_0x41a76b === _0x413a92) break;
                else _0x24638d['push'](_0x24638d['shift']());
            } catch (_0x114389) {
                _0x24638d['push'](_0x24638d['shift']());
            }
        }
    }(_0x1046, 0xf3764));
    var flag = document[_0x374fd6(0x183)](_0x374fd6(0x182))['value'],
        _0x4949 = [0x20, 0x5e, 0x7b, 0xd2, 0x59, 0xb1, 0x34, 0x72, 0x1b, 0x69, 0x61, 0x3c, 0x11, 0x35, 0x65, 0x80, 0x9, 0x9d, 0x9, 0x3d, 0x22, 0x7b, 0x1, 0x9d, 0x59, 0xaa, 0x2, 0x6a, 0x53, 0xa7, 0xb, 0xcd, 0x25, 0xdf, 0x1, 0x9c],
        _0x42931 = [0x24, 0x16, 0x1, 0xb1, 0xd, 0x4d, 0x1, 0x13, 0x1c, 0x32, 0x1, 0xc, 0x20, 0x2, 0x1, 0xe1, 0x2d, 0x6c, 0x6, 0x59, 0x11, 0x17, 0x35, 0xfe, 0xa, 0x7a, 0x32, 0xe, 0x13, 0x6f, 0x5, 0xae, 0xc, 0x7a, 0x61, 0xe1],
        operator = [(_0x3a6862, _0x4b2b8f) => {
            return _0x3a6862 + _0x4b2b8f;
        }, (_0xa50264, _0x1fa25c) => {
            return _0xa50264 - _0x1fa25c;
        }, (_0x3d7732, _0x48e1e0) => {
            return _0x3d7732 * _0x48e1e0;
        }, (_0x32aa3b, _0x53e3ec) => {
            return _0x32aa3b ^ _0x53e3ec;
        }],
        getchar = String[_0x374fd6(0x178)];
    if (flag[_0x374fd6(0x17c)] != 0x24) {
        text2img(_0x374fd6(0x185));
        return;
    }
    for (var i = 0x0; i < flag[_0x374fd6(0x17c)]; i++) {
        if (flag[_0x374fd6(0x176)](i) == operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i])) {} else {
            text2img(_0x374fd6(0x185));
            return;
        }
    }
    text2img(flag);
}
```
먼저 `_0x374fd6`과 `_0x138062` 즉, `_0x376c`로 정의된 함수에 대해 알아봅시다. 이 함수는 인자 두개를 받을 수 있지만, 사용하지는 않습니다. 이 함수는 첫번째 인자로 들어온 값에서 `0x175`을 빼고 `_0x1046`리스트에서 해당 인덱스의 값을 반환하는 함수입니다.

그 아래로는 셔플을 하고 값을 추가 하는 과정이 있습니다. 굳이 설명할 필요는 없을것 같고, 그냥 콘솔에서 실행해보면 바뀐 리스트의 내용을 알 수 있습니다. 실제로 실행해보면 아래와 같습니다.
```javascript
_0x1046 = ["37594TRDRWW", "charCodeAt", "296569AQCpHt", "fromCharCode", "1aqTvAU", "2XStRDS", "1388249ruyIdZ", "length", "23461saqTxt", "9966Ahatiq", "1824773xMtSgK", "1918853csBQfH", "175TzWLTY", "flag", "getElementById", "94hQzdTH", "NOP !", "11sVVyAj"]
```
그럼 이제 아래와 같은 코드를 해석할 수 있습니다.
```javascript
var flag = document[_0x374fd6(0x183)](_0x374fd6(0x182))['value'],
```
`_0x374fd6(0x183)`는 `_0x1046[0x183-0x175]`, 즉 `getElementById` 입니다. 뒤에 있는 `_0x374fd6(0x182)`도 해석하면 flag가 나오고요.  
따라서 이 문장은
```javascript
var flag = document['getElementById']('flag')['value']
```
으로 바꿔 쓸 수 있고, 프론트엔드에서 흔히 쓰는 형태로 보자면
```javascript
var flag = document.getElementById('flag').value
```
id가 flag인 태그의 값을 가져온다는 뜻입니다. 여기서는 form의 input입니다.  
  
나머지도 같은 방식으로 전부 해석해보겠습니다.
```javascript
var flag = document['getElementById']('flag')['value'],
    _0x4949 = [0x20, 0x5e, 0x7b, 0xd2, 0x59, 0xb1, 0x34, 0x72, 0x1b, 0x69, 0x61, 0x3c, 0x11, 0x35, 0x65, 0x80, 0x9, 0x9d, 0x9, 0x3d, 0x22, 0x7b, 0x1, 0x9d, 0x59, 0xaa, 0x2, 0x6a, 0x53, 0xa7, 0xb, 0xcd, 0x25, 0xdf, 0x1, 0x9c],
    _0x42931 = [0x24, 0x16, 0x1, 0xb1, 0xd, 0x4d, 0x1, 0x13, 0x1c, 0x32, 0x1, 0xc, 0x20, 0x2, 0x1, 0xe1, 0x2d, 0x6c, 0x6, 0x59, 0x11, 0x17, 0x35, 0xfe, 0xa, 0x7a, 0x32, 0xe, 0x13, 0x6f, 0x5, 0xae, 0xc, 0x7a, 0x61, 0xe1],
    operator = [(a, b) => {
        return a + b;
    }, (a, b) => {
        return a - b;
    }, (a, b) => {
        return a * b;
    }, (a, b) => {
        return a ^ b;
    }],
    getchar = String['fromCharCode'];
if (flag['length'] != 0x24) {
    text2img('NOP !');
    return;
}
for (var i = 0x0; i < flag['length']; i++) {
    if (flag['charCodeAt'](i) == operator[i % operator['length']](_0x4949[i], _0x42931[i])) {} else {
        text2img('NOP !');
        return;
    }
}
text2img(flag);
```
이제 좀 보기가 좋군요.  
일단 `NOP !`이 반환되는 조건은 flag의 길이가 `0x24` 즉 `36`자리가 아니면 반환되고요. 입력값과 어떤 연산된 값이 일치하지 않으면 반환되네요. 이 조건을 모두 만족하는 값, 그게 flag일 겂니다.  
  
그럼 flag를 구해 보겠습니다. 저는 아래와 같은 자바스크립트 코드를 이용했습니다.
```javascript
var _0x4949 = [0x20, 0x5e, 0x7b, 0xd2, 0x59, 0xb1, 0x34, 0x72, 0x1b, 0x69, 0x61, 0x3c, 0x11, 0x35, 0x65, 0x80, 0x9, 0x9d, 0x9, 0x3d, 0x22, 0x7b, 0x1, 0x9d, 0x59, 0xaa, 0x2, 0x6a, 0x53, 0xa7, 0xb, 0xcd, 0x25, 0xdf, 0x1, 0x9c],
    _0x42931 = [0x24, 0x16, 0x1, 0xb1, 0xd, 0x4d, 0x1, 0x13, 0x1c, 0x32, 0x1, 0xc, 0x20, 0x2, 0x1, 0xe1, 0x2d, 0x6c, 0x6, 0x59, 0x11, 0x17, 0x35, 0xfe, 0xa, 0x7a, 0x32, 0xe, 0x13, 0x6f, 0x5, 0xae, 0xc, 0x7a, 0x61, 0xe1],
    operator = [(a, b) => {
        return a + b;
    }, (a, b) => {
        return a - b;
    }, (a, b) => {
        return a * b;
    }, (a, b) => {
        return a ^ b;
    }],
getchar = String['fromCharCode'];

var flag = '';

for (var i = 0x0; i < 0x24; i++) {
    flag += getchar(operator[i % operator['length']](_0x4949[i], _0x42931[i]))
}

flag
```
flag의 길이를 알고 있으므로 간소화하여 만들었습니다.  
이걸 실행하면
```
DH{cfd4a77a013ea616d3d5cc0ddf87c1ea}
```
flag가 나옵니다.
{% endraw %}