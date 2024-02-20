---
title: (38) Dreamhack filestorage 문제 풀이
date: 2024-02-15 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
파일을 관리할 수 있는 구현이 덜 된 홈페이지입니다.

## 문제 풀이
일단 사이트에 들어가면
![](https://kyuyeop.github.io/assets/img/post/38/1.png)
이런 사이트가 나옵니다. 파일을 만들어서 업로드하는 것으로 보입니다. 여기서 뭔가 될것 같진 않으니 코드를 보면서 다른 페이지가 있는지 확인해 봅시다.
```javascript
const express=require('express');
const bodyParser=require('body-parser');
const ejs=require('ejs');
const hash=require('crypto-js/sha256');
const fs = require('fs');
const app=express();


var file={};
var read={};
function isObject(obj) {
  return obj !== null && typeof obj === 'object';
}
function setValue(obj, key, value) {
  const keylist = key.split('.');
  const e = keylist.shift();
  if (keylist.length > 0) {
    if (!isObject(obj[e])) obj[e] = {};
    setValue(obj[e], keylist.join('.'), value);
  } else {
    obj[key] = value;
    return obj;
  }
}

app.use(bodyParser.urlencoded({ extended: false }));
app.set('view engine','ejs');


app.get('/',function(req,resp){
  read['filename']='fake';
  resp.render(__dirname+"/ejs/index.ejs");

})

app.post('/mkfile',function(req,resp){
  let {filename,content}=req.body;
  filename=hash(filename).toString();
  fs.writeFile(__dirname+"/storage/"+filename,content,function(err){
    if(err==null){
      file[filename]=filename;
      resp.send('your file name is '+filename);
    }else{
      resp.write("<script>alert('error')</script>");
      resp.write("<script>window.location='/'</script>");
    }
  })

})

app.get('/readfile',function(req,resp){
  let filename=file[req.query.filename];
  if(filename==null){
    fs.readFile(__dirname+'/storage/'+read['filename'],'UTF-8',function(err,data){
      resp.send(data);
    })
  }else{
    read[filename]=filename.replaceAll('.','');
    fs.readFile(__dirname+'/storage/'+read[filename],'UTF-8',function(err,data){
      if(err==null){
        resp.send(data);
      }else{
        resp.send('file is not existed');
      }
    })
  }

})

app.get('/test',function(req,resp){
  let {func,filename,rename}=req.query;
  if(func==null){
    resp.send("this page hasn't been made yet");
  }else if(func=='rename'){
    setValue(file,filename,rename)
    resp.send('rename');
  }else if(func=='reset'){
    read={};
    resp.send("file reset");
  }
})


app.listen(8000);
```
역시 `/mkfile, /readfile, /test` 3개의 페이지가 더 존재하는군요.  
  
각 페이지의 역할을 정리하면  
  
__mkfile__  
요청으로 부터 `filename`과 `content`를 받아서 `서버경로/storage/hash(sha-256)`된 `filename` 로 저장. `file[filename]`을 `filename`으로 설정.  

__readfile__  
요청으로 부터 `filename`을 받아서 `filename` 변수를 `file[req.query.filename]`으로 지정. 만약 `filename`이 `null`이면 `서버경로/storage/read['filename']`(※ '있음. 문자열)을, `null`이 아니면 `서버경로/storage/read[filename]`(※ 변수 filename)을 읽어서 반환. 존재하지 않는 경우 메세지 표시.  

__test__  
요청으로 부터 `func`를 받아서 `null`이면 개발중이라는 문구를 보여주고, `rename`이면 `file[filename]`을 `rename`으로 변경하고 `rename`출력, `reset`이면 `read`라는 `obj`를 초기화하고 `file reset`출력.  
  
또한 도커 파일을 보면
```
FROM node

RUN mkdir -p /app
RUN echo 'BISC{fake flag}' > /flag

WORKDIR /app

COPY ./ /app

RUN npm install

EXPOSE 8000

CMD ["npm","start"]
```
로 되어 있으므로 `/flag`에 flag가 있음을 알 수 있습니다. 따라서 `/flag`를 읽을 수 있도록 만드는게 문제의 목표입니다.  
  
문제는 이 페이지가 일반적으로는 생성한 파일만 읽을 수 있도록 되어 있어서 `readfile`에 `../../../../../../../flag`를 넣어도 flag를 읽어주지 않는다는 겁니다.  
그래서 flag에 바로 접근하는대신 `read['filname']`의 값을 flag의 경로로 바꿀 겁니다. 이때 사용할 공격은 js의 프로토타입 오염 공격입니다.
### Prototype Pollution(프로토타입 오염)
node.js에서 어떤 객체의 `__proto__`는 `Object.prototype`을 나타냅니다. 따라서 `file.__prototype__.filename`의 값을 수정하면 `file`과 같이 객체인 `read`의 `filename`도 변경되게 됩니다.  
<br>
  
test페이지를 보면 이 공격이 가능하도록 `setValue`함수가 구현되어 있습니다. `file`을 변경하도록 되어 있으나, `__proto__`를 이용해 모든 객체의 속성을 변경하으므로 `read`의 속성도 바뀌게 됩니다. 따라서
```
/test?func=rename&filename=file.__proto__.filename&rename=../../../../../../../flag
```
로 접속하게 된다면 객체의 filename 속성이 flag로 변경되었을 겁니다.  
  
여기서 한 가지를 더 해줘야 하는데
```
/test?func=reset
```
위 주소로도 접속을 해줘야 합니다. read 객체가 원래 filename을 가지고 있었으므로 filename을 가져오면 우리가 지정한 속성이 아닌 처음에 정의된대로 fake를 가져올 겁니다. 객체를 비워줌으로서 기본값을 가져올 수 있도록 해주는 겁니다.  
  
이를 모두 입력하고 /readfile페이지 아무 파라미터 없이 접속하면 
![](https://kyuyeop.github.io/assets/img/post/38/2.png)
flag를 구할 수 있습니다.
{% endraw %}