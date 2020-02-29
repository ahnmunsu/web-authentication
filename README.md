# 웹 서버 인증(authentication)

## 목차
1.  **[Session Cookie 방식](#Session-Cookie-방식)**
2.  **[JWT(Json Web Token)](#JWTJson-Web-Token)**
3.  **[OAuth](#OAuth)**


---

## Session Cookie 방식
![session_cookie](./images/session_cookie.png)
1. 사용자가 로그인을 한다.
```
POST http://192.168.0.38:3000/login HTTP/1.1
Host: 192.168.0.38:3000
Connection: keep-alive
Content-Length: 36
Accept: */*
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.122 Safari/537.36
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Origin: http://192.168.0.38:3000
Referer: http://192.168.0.38:3000/
Accept-Encoding: gzip, deflate
Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7

email=ahnmunsu%40gmail.com&pass=1234
```
2. 서버에서는 계정 정보를 읽어 사용자를 확인한 후, 사용자의 고유한 ID 값을 부여하여 세션 저장소에 저장한 후, 이와 연결되는 세션 ID를 발행한다.  
![redis_session_id](./images/redis_session_id.png)
```
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 4
ETag: W/"4-5f2c/g6AOREdVLWI53srsMrUHDo"
Set-Cookie: connect.sid=s%3AYYnqfZSfR0TT74rPhpK6bR2TvX9-aVLo.zH6USMrZO5bsz5sV%2BkS3rpEgxh6ad1gTYloDSKfm44k; Path=/; HttpOnly
Date: Sat, 29 Feb 2020 15:24:59 GMT
Connection: keep-alive

done
```
3. 사용자는 서버에서 해당 세션 ID를 받아 쿠키에 저장한 후, 인증이 필요한 요청마다 쿠키를 헤더에 담아 보낸다.
![cookie](./images/cookie.png)
```
GET http://192.168.0.38:3000/admin HTTP/1.1
Host: 192.168.0.38:3000
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.122 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://192.168.0.38:3000/
Accept-Encoding: gzip, deflate
Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7
Cookie: connect.sid=s%3AYYnqfZSfR0TT74rPhpK6bR2TvX9-aVLo.zH6USMrZO5bsz5sV%2BkS3rpEgxh6ad1gTYloDSKfm44k

```
4. 서버에서는 쿠키를 받아 세션 저장소에서 대조를 한후 대응되는 정보를 가져 온다.  
5. 인증이 완료되고 서버는 사용자에 맞는 데이터를 보내 준다.  

### Session Cookie 방식 장점
* 쿠키가 담긴 HTTP 요청이 유출 되더라도 쿠키 자체(세션 ID)는 ID나 PW와 같은 중요 계정 정보를 담고 있지 않기 때문에 안전하다.
* 서버는 사용자마다 발급한 고유한 쿠키 값으로 어떤 사용자인지 확인이 가능하다.

### Session Cookie 방식 단점
* 쿠키가 담긴 HTTP 요청이 유출되면 유출된 쿠키를 사용해서 서버에 접근이 가능하다.
  * 해결책
    * HTTPS를 사용해서 요청을 암호화  
    * 세션의 유효 시간을 제한  
* 서버에 세션을 저장하기 때문에 추가적인 저장 공간이 필요하고 서버 부하가 생긴다.
---
**[⬆ 목차](#목차)**

## JWT(Json Web Token)
---
**[⬆ 목차](#목차)**

## OAuth
---
**[⬆ 목차](#목차)**

## References
*  https://tansfil.tistory.com/58
*  https://tansfil.tistory.com/59
*  https://tansfil.tistory.com/60
