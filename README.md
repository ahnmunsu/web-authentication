# 웹 서버 인증(authentication)

## 목차
1.  **[Session Cookie 방식](#Session-Cookie-방식)**
2.  **[JWT(Json Web Token)](#JWTJson-Web-Token)**
3.  **[JWT Refresh Token](#JWT-Refresh-Token)**
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

### 구현 예 (Node.js, Express, Redis)
https://github.com/ahnmunsu/nodejs-express-session-redis
---
**[⬆ 목차](#목차)**

## JWT(Json Web Token)
JSON Web Token(JWT) 은 웹표준(RFC 7519)으로써 두 개체에서 JSON 객체를 사용하여 가볍고 자가수용적인(self-contained) 방식으로 정보를 안전성 있게 전달해 준다.
### JWT 사용 목적
* 회원 인증
  * 로그인 시 유저 정보 기반으로 토큰 발급
  * 유저는 요청 시 마다 토큰을 포함하여 전달
  * 서버는 토큰 유효성 검증 및 권한 확인하여 처리
* 정보 교류
  * 정보가 sign 되어 있기 때문에 정보를 보낸이와 내용이 조작되지 않았는지 검증 가능
### JWT 구조
![jwt](./images/jwt.png)
#### 헤더(Header)
* Header는 두 가지 정보를 저장하고 있다.
  * typ: 토큰 타입을 지정한다. JWT로 지정한다.
  * alg: hash 알고리즘을 지정한다. HMAC SHA256, RSA가 사용된다. signature에서 이 알고리즘을 사용한다.
#### Header 인코딩 구현
```js
const header = {
  "typ": "JWT",
  "alg": "HS256"
};

// encode to base64
const headerStr = JSON.stringify(header);
const encodedHeader = Buffer.alloc(headerStr.length, headerStr)
                            .toString('base64')
                            .replace('=', ''); /* Remove base64 pad characters */
                            
console.log('header: ',encodedHeader); // header:  eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```
#### 정보(Payload)
* Payload 부분에는 토큰에 저장할 정보가 들어 있다.
* 토큰에 저장하는 정보를 클레임(claim)이라고 하며, name-value 쌍으로 되어있다.
* 클레임의 종류
  * 등록된(registered) 클레임
  * 공개(public) 클레임
  * 비공개(private) 클레임

#### 등록된(registered) 클레임
* 토큰에 대한 정보로써 미리 정해진 클레임이다.
* 등록된 클레임은 선택적으로 사용할 수 있다.
* 클레임 종류
  * iss: 토큰 발급자(issuer)
  * sub: 토큰 제목(subject)
  * aud: 토큰 대상자(audience)
  * exp: 토큰 만료 시간(expiration). NumericDate 형식(예: 1583033110)
  * nbf: 토큰 활성 시점(Not Before). NumericDate 형식
  * iat: 토큰 발급 시간(issued at)
  * jti: JWT 고유 식별자. 중복 처리를 방지하기 위해 사용.
#### 공개(public) 클레임
* 충돌이 방지된(collision-resistant) 이름을 가지고 있어야 한다.
* 클레임 이름을 URI 형식으로 짓는다.
```js
{
  "https://github.com/ahnmunsu/is_admin": true
}
```
#### 비공개(private) 클레임
* 서버-클라이언트 간에 협의하여 정의된 클레임이다.
```js
{
  "username": "ahnmunsu"
}
```
#### Payload 인코딩 구현
```js
const payload = {
    "iss": "github.com/ahnmunsu",
    "exp": "1583033110",
    "https://github.com/ahnmunsu/is_admin": true,
    "userId": "10000",
    "username": "ahnmunsu"
};

// encode to base64
const payloadStr = JSON.stringify(payload);
const encodedPayload = Buffer.alloc(payloadStr.length, payloadStr)
                            .toString('base64')
                            .replace('=', ''); /* Remove base64 pad characters */

console.log('payload: ',encodedPayload); 
/*
payload:  eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiZXhwIjoiMTU4MzAzMzExMCIsImh0dHBzOi8vZ2l0aHViLmNvbS9haG5tdW5zdS9pc19hZG1pbiI6dHJ1ZSwidXNlcklkIjoiMTAwMDAiLCJ1c2VybmFtZSI6ImFobm11bnN1In0
*/
```
#### 서명(signature)
* 서명은 Header의 인코딩 값과 Payload의 인코딩 값을 합친 후 주어진 비밀키로 Hash하여 생성한다.
* 서명을 생성하는 과정의 pseudo code는 아래와 같다.
```js
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```
#### 서명 해싱 및 인코딩 구현
```js
const crypto = require('crypto');

const encodedHeader = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9";
const encodedPayload = "eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiZXhwIjoiMTU4MzAzMzExMCIsImh0dHBzOi8vZ2l0aHViLmNvbS9haG5tdW5zdS9pc19hZG1pbiI6dHJ1ZSwidXNlcklkIjoiMTAwMDAiLCJ1c2VybmFtZSI6ImFobm11bnN1In0";

const signature = crypto.createHmac('sha256', 'secret')
             .update(encodedHeader + '.' + encodedPayload)
             .digest('base64')
             .replace('=', '');

console.log('signature: ',signature); // signature:  KZoSy2cyh6LlliJ1tJ+YOo4dgVoxHLowbmJdUQB5oNA
```
### JWT 검증
* 위 구현 예에서 생성한 JWT
`eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9`.`eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiZXhwIjoiMTU4MzAzMzExMCIsImh0dHBzOi8vZ2l0aHViLmNvbS9haG5tdW5zdS9pc19hZG1pbiI6dHJ1ZSwidXNlcklkIjoiMTAwMDAiLCJ1c2VybmFtZSI6ImFobm11bnN1In0`.`KZoSy2cyh6LlliJ1tJ+YOo4dgVoxHLowbmJdUQB5oNA`
* 위 값을 https://jwt.io 의 디버거에 붙여 넣고 `your-256-bit-secret` 부분에 비밀키인 `secret`을 입력한다.
![jwt_verification](./images/jwt_verification.png)
### JWT 인증 과정
![jwt2](./images/jwt2.png)
1. 사용자가 로그인을 한다.  
```
POST http://192.168.0.38:4000/users/authenticate HTTP/1.1
Content-Type: application/json
User-Agent: PostmanRuntime/7.20.1
Accept: */*
Cache-Control: no-cache
Postman-Token: 5da98e59-dc63-4e4c-8a09-a75b2913c543
Host: 192.168.0.38:4000
Accept-Encoding: gzip, deflate
Content-Length: 44
Connection: keep-alive

{
	"username": "test",
	"password": "test"
}
```
2. DB에서 계정 정보를 읽어 사용자를 확인 한다.  
3. Payload에 유효 기간 등 필요한 registered claim을 설정하고 토큰을 발급한다.
4. 기타 정보와 함께 토큰을 사용자에게 전달한다.
```
HTTP/1.1 200 OK
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
Content-Length: 250
ETag: W/"fa-oAxmn5mxiHn9VACYSF59wopSHpk"
Date: Sun, 01 Mar 2020 10:45:02 GMT
Connection: keep-alive

{ 
  "id":1,
  "username":"test",
  "firstName":"Test",
  "lastName":"User",
  "token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiYXVkIjoxLCJpYXQiOjE1ODMwNTk1MDIsImV4cCI6MTU4MzA2MzEwMn0.0u4pe4nRFgxTddhMqbexylj9uOJuLNHAErr9PrBDsDM"
}
```
5. 사용자는 받은 토큰을 저장해 두고 요청을 보낼 때마다 헤더에 실어서 보낸다.  
```
GET http://192.168.0.38:4000/users/ HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiYXVkIjoxLCJpYXQiOjE1ODMwNTk1MDIsImV4cCI6MTU4MzA2MzEwMn0.0u4pe4nRFgxTddhMqbexylj9uOJuLNHAErr9PrBDsDM
User-Agent: PostmanRuntime/7.20.1
Accept: */*
Cache-Control: no-cache
Postman-Token: 451e8f4e-05c0-40bd-81a0-c4e6ab62d030
Host: 192.168.0.38:4000
Accept-Encoding: gzip, deflate
Connection: keep-alive
```
6. 서버는 해당 토큰의 signature를 비밀키로 복호화하여 조작 여부, 유효 기간을 확인한다.  
7. 인증에 이상이 없으면 Payload를 디코딩하여 사용자의 ID에 맞는 응답을 전달한다.  
```
HTTP/1.1 200 OK
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
Content-Length: 65
ETag: W/"41-qKX2x0QmEu2twz7f6vncJbVnIvo"
Date: Sun, 01 Mar 2020 10:54:21 GMT
Connection: keep-alive

[{"id":1,"username":"test","firstName":"Test","lastName":"User"}]
```

### JWT 장점
* Session Cookie 방식은 저장소 관리가 필요하지만 JWT 방식은 발급 후 검증만 하기 때문에 저장소가 필요없다.
* 확장성이 뛰어나 토큰 기반으로 하는 다른 인증 시스템에 접근이 가능하다. 예) 페이스북, 구글 로그인 등

### JWT 단점
* JWT는 한번 발급되면 유효 기간이 완료 될 때까지 계속 사용이 가능하다. 그렇기 때문에 악의적인 사용자에게도 한번 발급된 토큰을 취소할 수 없다.
	* 해결책으로는 access token의 기간을 짧게 하고 refresh token을 함께 발급한다.
* Payload가 암호화 되지 않기 때문에 중요한 정보를 담을 수 없다.
* Session Cookie 방식보다 JWT의 길이가 길어서 요청이 많을수록 자원 낭비가 심해진다.
### 구현 예 (Node.js + Express + jsonwebtoken)
https://github.com/ahnmunsu/node-jwt-authentication-api
---
**[⬆ 목차](#목차)**

## JWT Refresh Token
![refresh_token](./images/refresh_token.png)
1. 사용자가 ID, PW를 통해 로그인한다.  
```
POST http://192.168.0.38:4000/users/authenticate HTTP/1.1
Content-Type: application/json
User-Agent: PostmanRuntime/7.22.0
Accept: */*
Cache-Control: no-cache
Postman-Token: 890b0d83-dd13-4268-aa94-0f86d5f6a8da
Host: 192.168.0.38:4000
Accept-Encoding: gzip, deflate, br
Content-Length: 44
Connection: keep-alive

{
	"username": "test",
	"password": "test"
}
```
2. 서버에서는 회원 DB에서 값을 비교한다.  
3. 인증이 완료된 회원은 Access Token과 Refresh Token을 발급하고 Refresh Token은 서버에 저장해둔다.
4. 사용자에게 발급한 토큰을 전달한다.
```
HTTP/1.1 200 OK
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
Content-Length: 530
ETag: W/"212-KiXuDVQcC1TFJcdeIeDn36FuYu0"
Date: Tue, 03 Mar 2020 13:43:19 GMT
Connection: keep-alive

{
	"id":1,
	"username":"test",
	"firstName":"Test",
	"lastName":"User",
	"accessToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiYXVkIjoxLCJpYXQiOjE1ODMyNDI5OTksImV4cCI6MTU4MzI0MzA1OX0._mx1A0WVnY5OCZ2O9bL46x299-_toSUxcidJayAJvOs",
	"refreshToken":"txkacOBP51yJjdjDQum8g07MjmqtHeUhR58AONRqnAJIx2xQj1o4uE23pYv8kAMb2uMYEhxK1oGCigOAtM9kdYjsbVt9W7JPm4IunxjMdnfgEdFmE7o6FWTvvGs0zCXOuPRn29cHREy2zlhPpCBxvsdlviiK05M9yTkiykxlFbkC43tpo4LdE6Oe2Nw59hxenXBEiGomBxWOkrdOFW3l5WgaZlnNf5bLLyx2OPVGO7Bx5Qs7P7OPFkXddTSSW0gL"
}
```
5. 사용자는 Refresh Token은 저장소에 저장 후 Access Token을 헤더에 실어 요청을 보낸다.  
```
GET http://192.168.0.38:4000/users/ HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiYXVkIjoxLCJpYXQiOjE1ODMyNDMzMDIsImV4cCI6MTU4MzI0MzM2Mn0.M-S6RoNUyl0y1LXNZmjUT4Jn5ZR1EbmzsPvN0dObRHw
User-Agent: PostmanRuntime/7.22.0
Accept: */*
Cache-Control: no-cache
Postman-Token: fe530e4b-3b5e-4ba3-a586-d211e1d68cea
Host: 192.168.0.38:4000
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```
6. 서버는 Access Token를 검증한다.  
7. 검증에 이상이 없으면 요청한 데이터를 보낸다.  
```
HTTP/1.1 200 OK
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
Content-Length: 65
ETag: W/"41-qKX2x0QmEu2twz7f6vncJbVnIvo"
Date: Tue, 03 Mar 2020 13:48:29 GMT
Connection: keep-alive

[
	{
		"id":1,
		"username":"test",
		"firstName":"Test",
		"lastName":"User"
	}
]
```
8. 시간이 지나 Access Token이 만료되었다.  
9. 사용자는 이전과 동일하게 Access Token을 헤더에 실어 요청을 보낸다.  
```
GET http://192.168.0.38:4000/users/ HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiYXVkIjoxLCJpYXQiOjE1ODMyNDMzMDIsImV4cCI6MTU4MzI0MzM2Mn0.M-S6RoNUyl0y1LXNZmjUT4Jn5ZR1EbmzsPvN0dObRHw
User-Agent: PostmanRuntime/7.22.0
Accept: */*
Cache-Control: no-cache
Postman-Token: a2fcb648-a471-4498-9cf7-6c0a56f1f2ab
Host: 192.168.0.38:4000
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```
10. 서버는 Acess Token이 만료됐음을 확인한다.  
11. Access Token이 만료되었음을 응답을 통해 알린다.  
```
HTTP/1.1 401 Unauthorized
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
Content-Length: 27
ETag: W/"1b-Ji+DCy0zEZBDY9hox7El2YNCqZU"
Date: Tue, 03 Mar 2020 13:51:24 GMT
Connection: keep-alive

{
	"message":"Expired Token"
}
```
12. 사용자는 Refresh Token과 Acess Token을 함께 서버로 보낸다.  
```
POST http://192.168.0.38:4000/users/refresh HTTP/1.1
Content-Type: application/json
User-Agent: PostmanRuntime/7.22.0
Accept: */*
Cache-Control: no-cache
Postman-Token: 5c6d5f9f-ddb2-4de7-b133-a9293111df82
Host: 192.168.0.38:4000
Accept-Encoding: gzip, deflate, br
Content-Length: 489
Connection: keep-alive

{
	"userId": 1,
	"accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiYXVkIjoxLCJpYXQiOjE1ODMyNDQwOTgsImV4cCI6MTU4MzI0NDE1OH0.ISZ_dXa6xTJQV0xzIb-4CP0VRiYpyu6uDYS3el1vmaE",
	"refreshToken": "LBDW5qApn39Wd3fHaJFhKav0w12o2jqlDTY5vzj0Wsimm4g7p53H1I66nJ8QwL6FLBXrlWuMMT78HaPTutNSe0bdo7eBqwURc6tWrkp2A5tl0kJJpzrZEahopxUApPnIEeugdgfhvP4vctIKcYP69Biwh13Jk21s5GBZFnuAPMFjcgJM1SBh7iFka2mTgQeOk9bonn0aHX7s1eXdabSXe6a1GNPvqUIuzcmDk4cM52awheLMGgfusjaKEyp3XzYC"
}
```
13. 서버는 받은 Acess Token을 검증한 후, 받은 Refresh Token과 저장된 Refresh Token을 비교하여 이상 없을 시 Access Token을 새로 발급한다.  
```
HTTP/1.1 200 OK
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
Content-Length: 194
ETag: W/"c2-yu3bB7SZVZ1RIvPDYcGefcm+TNM"
Date: Tue, 03 Mar 2020 14:01:56 GMT
Connection: keep-alive

{
	"accessToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tL2Fobm11bnN1IiwiYXVkIjoxLCJpYXQiOjE1ODMyNDQxMTYsImV4cCI6MTU4MzI0NDE3Nn0.OQGSAl1lr34KeEdf5RbXwd1MzZvjqodSGvNk5-ImrCc"
}
```
14. 서버는 새로운 Acess Token을 헤더에 실어 다시 요청을 진행한다.  

---
**[⬆ 목차](#목차)**

## OAuth
---
**[⬆ 목차](#목차)**

## References
* https://tansfil.tistory.com/58
* https://tansfil.tistory.com/59
* https://tansfil.tistory.com/60
* https://velopert.com/2389
* https://solidgeargroup.com/refresh-token-with-jwt-authentication-node-js/
