---
title: "OAuth 취약점 알아보기"
date: 2022-10-24
draft: false
description: "OAuth 취약점 알아보기"
tags: ["web","oauth"]
categories: ["security"]
---


<img src=featured.png width=50% alt="thumbnail" style="display: block;margin: auto"/>


PortSwigger의 OAuth 2.0 authentication vulnerabilities를 번역/정리하였습니다 😊
[OAuth 2.0 authentication vulnerabilities]([https://portswigger.net/web-security/oauth](https://portswigger.net/web-security/oauth))

[지난 글 \(OAuth 2.0 알아보기\)]({{< ref "posts/20220904-oauth_concept/index.md" >}})을 통해서 OAuth에 대해서 알아보았습니다. 이번 글에서는 OAuth의 취약점을 살펴보고자 합니다.

> OAuth가 사용되는 것이 식별되면 아래 페이지에 접속하여 주요 정보가 출력되는지 확인해야 합니다.
- /.well-known/oauth-authorization-server
- /.well-known/openid-configuration
> 

# 📌 00. 취약점 분류

OAuth에서 발생할 수 있는 취약점으로는 아래와 같이 크게 5가지 정도가 있습니다.

- 클라이언트 어플리케이션의 취약점
    - 💥 **암시적 부여 유형 취약점**
        - OAuth 암시적 흐름을 통한 인증 우회
    - 💥 **미흡한 CSRF 보호**
        - 강제 OAuth 프로필 연결
- OAuth 서비스의 취약점
    - 💥 **인증 코드 및 액세스 토큰 누출**
        - redirect_uri를 통한 OAuth 계정 도용
        - redirect_uri 검증 미흡
        - 프록시 페이지를 통해 OAuth 액세스 토큰 훔치기
    - 💥 **미흡한 범위 유효성 검사**
    - 💥 **확인되지 않은 사용자 등록**

# 📌 01. 암시적 부여 유형

암시적 부여 유형으로 OAuth 구현 시 입력 값 검증 및 인증이 미흡하여 발생합니다.

암시적 부여 유형은 보통 단일 페이지에서 사용되나 편리하다는 이점으로 인해 서버-클라이언트 어플리케이션에서 사용되기도 합니다. 

이때 인증 세션을 유지하기 위해 사용자 정보를 서버로 보내어 쿠키(세션)을 받게 되는데, 전송되는 사용자 정보를 변조하여 타 사용자로 로그인을 시도해볼 수 있습니다.

### 🔒 대응방안

여러 페이지를 가지는 클라이언트-서버 어플리케이션에서는 가급적 **승인코드 부여 방식**으로 OAuth를 구현해야 하며 사용자의 입력 값을 검증하여 부적절한 인증 관련 정보를 필터링해야 합니다.

# 📌 02. 미흡한 CSRF 보호

클라이언트 어플리케이션에서 CSRF 보호가 미흡한 경우, CSRF 취약점을 이용해 사용자에게 공격자의 프로필을 강제로 연결시킬 수 있습니다.

**/oauth-linking?code=[...]** 와 같은 페이지에 사용자가 강제로 접속하도록 하면 공격자의 소셜 프로필이 사용자의 계정에 강제로 연결될 수 있습니다.

### 🔒 대응방안

CSRF 토큰과 같은 역할을 하는 `state` 파라미터를 이용하여 CSRF 취약점에 대응합니다. 또한, OAuth만을 이용하여 로그인하도록 구현할 경우에도 해당 취약점에 대응할 수 있습니다.

# 📌 03. 인증 코드 및 액세스 토큰 누출

## 03-01. redirect_uri를 통한 OAuth 계정 도용

인증 유형에 따라 승인 코드 또는 토큰을 탈취할 수 있는 취약점입니다. 

/auth 페이지 요청 시 `redirect_uri` 파라미터를 함께 보내며, 사용자는 승인코드 또는 토큰과 함께 redirect_uri로 리다이렉션 됩니다.

이를 이용하여 공격자는 `redirect_uri`를 공격자 서버 주소로 변경하여 사용자가 승인코드 또는 토큰과 함께 변조된 `redirect_uri`에 리다이렉션 되도록 만들 수 있습니다.

공격 절차는 아래와 같습니다.

1. `redirect_uri`가 포함된 /auth 페이지 접속 요청을 Intercept 하여 `redirect_uri` 파라미터가 변조 가능한지 테스트합니다. (Repeater로 변조 값 전송 등)
2. `redirect_uri`를 공격자 서버 주소로 변조하여 사용자가 승인 코드 또는 토큰과 함께 공격자 서버로 리다이렉션 되도록 만듭니다.
3. 변조된 /auth URL을 배포하여 공격자 서버에 기록된 승인 코드 또는 토큰을 확인합니다.
4. 공격자는 /oauth-callback URL에 승인 코드 또는 토큰을 파라미터로 포함시켜 접속, 사용자 계정에 무단 접속할 수 있습니다.

## 03-02. redirect_uri 검증 미흡

인증 서버 측에서 `redirect_uri` 검증이 미흡할 경우 아래와 같은 우회 패턴이 유효할 수 있습니다.

- URI 구문 분석을 이용하여 허용된 문자열로 시작하여 공격자 서버 주소 삽입
    - [https://default-host.com](https://default-host.com/) [&@foo.evil-user.net](mailto:&@foo.evil-user.net)#@bar.evil-user.net/
- 중복된 `redirect_uri` 파라미터를 포함
    - [https://oauth-authorization-server.com/?client_id=123&redirect_uri=client-app.com/callback&redirect_uri=evil-user.net](https://oauth-authorization-server.com/?client_id=123&redirect_uri=client-app.com/callback&redirect_uri=evil-user.net)
- [localhost](http://localhost) 와 같이 개발 시에 이용될 수 있는 도메인 사용
    - localhost.evil-user.net
- response_mode 등 파라미터를 일부 변경하여 파라미터 파싱 방식이 변경되는지 확인

## 03-03. 프록시 페이지를 통해 OAuth 액세스 토큰 훔치기

다음과 같은 패턴을 이용하여 `redirect_uri` 검증을 우회할 수 있습니다.

- 상위 디렉토리 탐색 문자 (`..`) 이용
    - [https://client-app.com/oauth/callback/../../example/path](https://client-app.com/example/path)
    - 위 uri는 [https://client-app.com/example/path](https://client-app.com/example/path) 로 인식됩니다.

우회 URI를 찾았으면, 해당 페이지에서 추가적인 취약점이 나오는지 확인하여 악용해야 합니다. 예를 들어 Open Redirect, XSS 등이 있습니다.

### 🔒 대응방안

- state, nonce 와 같은 파라미터는 완벽한 대응책이 되지 못합니다.
클라이언트 어플리케이션으로 승인코드 전송 시에 인증서버 측에서 `redirect_uri`가 유효한지 검증해야 합니다. 가능하다면 화이트리스트로 검증할 것을 권장합니다.
클라이언트 어플리케이션과 인증 서버 사이의 절차이므로 브라우저 측인 사용자(공격자)는 개입(변조)할 수 없어 검증 자체가 변조될 위험이 적습니다.
- 특수문자 등 위회 패턴을 발생 시킬 수 있는 문자를 필터링합니다.
- 상세한 `redirect_uri` 필터링, Open Redirect, XSS, HTML injection 등의 취약점도 조치를 해야합니다.

# 📌 04. **미흡한 권한 범위 유효성 검사**

## 04-01. 승인 코드 부여 방식

악성 클라이언트 어플리케이션을 인증 서버에 등록하여 사용자의 승인 코드를 탈취할 수 있으며 획득한 코드를 이용해 새로운 권한 범위 파라미터를 설정하여 토큰을 요청할 수 있습니다.

초기 권한 범위와 이후 요청 권한 범위에 대해 검증이 미흡한 경우, 공격자가 요청한 권한 범위의 토큰이 악성 클라이언트 어플리케이션으로 전송됩니다.

## 04-02. 암시적 부여 방식

액세스 토큰이 사용자 브라우저를 통해 전송되므로 공격자가 토큰을 탈취하여 직접 사용 가능합니다.

탈취한 액세스 토큰을 이용해 /userinfo 등에 요청을 보내며 새 범위 파라미터를 설정하면 새 권한 범위의 정보가 전송됩니다.

### 🔒 대응방안

인증 서버에서는 권한 범위 요청이 오면, 해당 어플리케이션이나 사용자에게 유효(적당)한 권한인지 검증해야 합니다.

# 📌 **05. 확인되지 않은 사용자 등록**

몇몇 OAuth 서비스 제공자는 가입자의 상세한 정보를 확인하지 않아 악의적인 사용자가 피해 사용자의 정보를 이용해 계정을 생성할 수 있으며, 생성된 계정을 이용해 인증 요청을 시도할 수 있습니다.

### 🔒 대응방안

기존 계정에 타 OAuth 서비스 계정을 연동시킬 때에는 기존 계정에서 본인인증 등을 수행하여 사칭 계정이 어플리케이션 계정에 무단 연동되는 것을 막아야합니다.

# 📖 References
\[1\] [OAuth 2.0 authentication vulnerabilities](https://portswigger.net/web-security/oauth)