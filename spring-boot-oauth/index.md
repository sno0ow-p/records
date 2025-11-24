---
title: 스프링 시큐리티 와 OAuth2 와 JWT 의 콜라보
date: 2021-04-27
pin: false
tags:
- Spring
- OAuth 2.0
- JWT
---

# Spring Security 와 OAuth 2.0 와 JWT 의 콜라보

스프링 부트를 토대로, 스프링 시큐리티의 Filter Chain과 OAuth 2.0 로그인 연동을 JWT 토큰 기반으로 구현했던 기록.

## Spring Security

스프링 시큐리티에서 애플리케이션 보안을 구성하는 두 가지 영역에 대해 간단하게 설명해보려 한다. 인증 (Authentication) 과 인가 (Authorization). 두 영역은 사실상 스프링 시큐리티의 핵심이라고 할 수 있다.

### Spring Security Structure

스프링 시큐리티는 주로 서블릿 필터와 이들로 구성된 필터 체인을 사용한다. 우선은 소셜 로그인이 아닌 기본적인 폼 로그인을 할 경우의 구조를 살펴보자.

![](images/security-aritchtecture.png)

1. 사용자가 로그인 정보와 함께 인증 요청 (`HttpRequest`)
2. `AuthenticationFilter`가 요청을 가로챔. 이때 가로챈 정보를 통해 `UsernamePasswordAuthenticationToken` 객체 (사용자가 입력한 데이터를 기반으로 생성, 즉 현 상태는 미검증 Authentication) 생성
3. `ProviderManager` (`AuthenticationManager` 구현체) 에게 `UsernamePasswordAuthenticationToken` 객체를 전달
4. `AuthenticationProvider`에 `UsernamePasswordAuthenticationToken` 객체를 전달
5. 실제 DB로 부터 사용자 인증 정보를 가져오는 `UserDetailsService`에 사용자 정보를 넘겨줌
6. 넘겨받은 정보를 통해 DB에서 찾은 사용자 정보인 `UserDetails` 객체를 생성
7. `AuthenticationProvider`는 `UserDetails`를 넘겨받고 사용자 정보를 비교
8. 인증이 완료되면, 사용자 정보를 담은 `Authentication` 객체를 반환
9. 최초의 `AuthenticationFilter`에 `Authentication` 객체가 반환됨
10. `Authentication` 객체를 `SecurityContext`에 저장

여기서 주의깊게 살펴봐야 할 부분은 `UserDetailsService`와 `UserDetails` 이다. 실질적인 인증 과정은 사용자가 입력한 데이터 (ID, PW 등) 와 `UserDetailsService`의 `loadUserByUsername()` 메서드가 반환하는 `UserDetails` 객체를 비교함으로써 동작한다. 따라서 `UserDetailsService`와 `UserDetails` 구현을 어떻게 하느냐에 따라서 인증의 세부 과정이 달라진다.

아래는 스프링 시큐리티에서 동작하는 기본 필터들의 목록 및 순서다. 만약 OAuth 2.0 로그인을 사용한다면, `UsernamePasswordAuthenticationFilter` 대신 `OAuth2LoginAuthenticationFilter` 가 호출된다. 두 필터의 상위 클래스는 `AbstractAuthenticationProcessingFilter`이다. 사실 스프링 시큐리티는 `AbstractAuthenticationProcessingFilter`를 호출하고, 로그인 방식에 따라 구현체인 `UsernamePasswordAuthenticationFilter` 와 `OAuth2LoginAuthenticationFilter` 가 동작하는 방식이다.

![](iamges/security-filters.png)

조금 더 상세히 메서드 및 부가적인 과정을 표현한 그림은 아래와 같다. 이 포스팅에서는 `AuthenticationSuccessHandler`, `AuthenticationFailureHandler`, `UserDetailsService`, `UserDetails`, `AuthenticationEntryPoint`, `AccessDeniedHandler` 정도를 살펴보려 한다.

![](images/security-filter-invocation.png)



## OAuth 2.0

### 기본 개념

기본적으로 OAuth (OpenID Authentication) 란, 타사의 사이트에 대한 접근 권한을 얻고 그 권한을 이용하여 개발할 수 있도록 도와주는 프레임워크다. 구글, 카카오, 네이버 등과 같은 사이트에서 로그인을 하면 직접 구현한 사이트에서도 로그인 인증을 받을 수 있도록 되는 구조다.

물론 구글에서 로그인을 했다고 해서, 개발한 웹 사이트에 구글 ID와 PW를 그대로 전달해주면 안되므로, Access Token을 발급 받고, 그 토큰을 기반으로 원하는 기능을 구현해야 한다.

Access Token은 로그인을 하지 않고 인증을 할 수 있도록 해주는 인증 토큰 정도의 개념입니다. 유저 `A`가 직접 개발한 웹 사이트 `X`에서 자신의 구글 캘린더에 대한 접근을 허용해 준다면, Access Token을 통해 해당 정보 권한을 받아올 수 있어서 그 정보를 토대로 캘린더에 글을 작성하고 삭제하는 등의 작업을 할 수 있게 된다.

### 주요 용어

여기서 Access Token을 발급 받기 위한 일련의 과정들을 인터페이스로 정의해둔 것이 바로 OAuth 다. OAuth에서 중요한 용어는 크게 세 가지다.

- `Resource Owner`: 개인 정보의 소유자를 가리킨다. 유저 `A`가 이에 해당한다.
- `Client`: 제 3의 서비스로부터 인증을 받고자 하는 서버다. 직접 개발한 웹 사이트 `X`가 이에 해당한다.
- `Resource Server`: 개인 정보를 저장하고 있는 서버를 의미한다. 구글이 이에 해당한다.

유저 `A`가 구글에서 제공해주는 서비스를 이용하는 셈이므로 타 사의 서비스를 이용하기 위해서는 신청을 해야 한다. 신청 방법은 구글, 카카오, 네이버, 페이스북 등 각각 모두 방식이 다르지만, 반드시 필요로 하는 내용은 ID, PW, 본인 인증 방법 이렇게 세 가지 정도다. 각 사이트의 개발자 Docs를 참고하면 쉽게 등록하고 발급받을 수 있다.

- `Client ID`: `Resource Server`에서 발급해주는 ID. 웹 사이트 `X`에 구글이 할당한 ID를 알려주는 것이다.
- `Client Secret`: `Resource Server`에서 발급해주는 PW. 웹 사이트 `X`에 구글이 할당한 PW를 알려주는 것이다.
- `Authorized Redirect Uri`: Client 측에서 등록하는 Url. 만약 이 Uri로부터 인증을 요구하는 것이 아니라면, `Resource Server`는 해당 요청을 무시한다.

### 예시 동작 설명

Client, 즉 직접 개발한 웹 사이트 `X`가 `Resource Server`에 등록이 완료되었다면, 이제 Access Token을 발급 받을 수 있다. 구글을 예시로 계속 설명하자면, 유저 `A`가 웹 사이트 `X`에서  특정 기능을 이용하기 위해서 로그인이 요구되고, 구글 인증 Access Token을 받기 위해 구글 로그인 링크로 연결된다.

예시 링크(`https://accounts.google.com/?client_id=123&scope=profile,email&redirect_uri=http://localhost`)의 쿼리 스트링을 살펴보면, `client_id`는 `123`, `scope`는 `profile`과 `email`, `redirect_uri`는 `http://localhost`임을 알 수 있다.

유저 `A`가 구글 로그인을 정상적으로 했다면, 구글은 이전에 등록되었던 `client_id=123`인 서버의 `redirect_uri`와 동일한지 확인한다.

일치하는 경우, 유저 `A`에게 `scope=profile,email` 기능을 넘겨줄 것인지에 대한 승인 여부를 불어보고, 동의한다면 구글은 이에 해당하는 `authorization_code`라는 임시 PW를 발급한다.

이후 `http://localhost/?authorization_code=2`으로 리다이렉트되며, 웹 사이트 `X`의 서버는 이 `authorization_code`를 가지고 구글에게 Access Token을 요청한다. 그리고 유저 `A`의 인증이 필요할 때마다 Access Token을 이용하여 접근한다.



## JWT 도입

다음은 스프링 부트에서 OAuth 2.0 로그인을 할 때 사용하는 여러 방법 중, JWT를 사용하게 된 이유이다.

### 서버 기반 인증 방식

먼저 서버 기반 인증 방식에 대해 알아야 한다. 서버 기반 인증 방식의 핵심은 서버 측에 사용자 정보를 저장하는 것이다. Spring Security에서는 별도의 설정이 없다면 세션을 이용하여 처리한다. 즉, 사용자가 로그인을 하면 서버는 해당 사용자의 세션을 만들고, 서버의 메모리와 데이터베이스에 저장한다. 만약 마이크로 서비스 개발을 진행하거나 서버를 확장하게 된다면, 모든 서버에게 세션의 정보를 공유해야 하므로 이를 위한 별도의 중앙 세션 관리 서버를 두곤 한다.

### Access Token과 Refresh Token

Access Token은 리소스 (사용자 정보) 에 직접 접근할 수 있도록 해주는 정보만을 가지고 있다. Refresh Token에 비해 짧은 만료 기간을 가지며, 주로 세션에 담아 관리한다.

Refresh Token은 새로운 Access Token을 발급받기 위한 정보를 담고 있다. 클라이언트가 Access Token이 없거나 만료된 상태라면, Refresh Token을 통해 Auth Server에 요청하여 새로운 Access Token을 발급 받을 수 있다. Refresh Token은 외부에 노출되지 않도록 하기 위해 보통은 DB에 저장하곤 한다.

### OAuth 2.0

위에서 설명했듯이 구글, 카카오 등에서 제공하는 Authorization Server를 통해 회원 정보를 인증하고 Access Token을 발급 받는다. 그리고 발급 받은 Access Token을 이용해 직접 개발한 서버의 API 서비스를 이용 및 호출한다. 마이크로 서비스이거나 서버 간의 통신이 잦은 경우, Access Token을 자주 주고 받을 수 밖에 없다.

그리고 각 서버는 API 호출 요청에 대해서 전달 받은 Access Token이 유효한 지를 확인해야 한다. 이는 서버에서 클라이언트의 상태 (Access Token의 유효성) 를 관리하게끔 하며, 또 API를 호출할 때마다 Access Token이 유효한지 매번 DB에서 조회하고 새로 갱신 시 업데이트 작업을 해주어야 한다. 즉, 클라이언트 상태를 관리 및 공유할 추가적인 저장 공간과 매번 요청마다 Access Token의 검증 및 업데이트를 위한 DB 호출이 발생하는 구조다.

마이크로 서비스 개발처럼, 서버의 수가 많은 경우에는 각각의 서버가 Access Token의 유효성 및 권한 확인을 Auth Server에 요청하기 때문에 병목 현상 등이 발생해, 서버의 부하로 이어질 수 있다. 이 문제점을 해소하기 위해 JWT 기반 인증을 도입한다.

### JWT

JWT는 Claim 기반 방식을 사용한다. 여기서 Claim이란 사용자에 대한 속성 값들을 가리킨다. 즉, JWT은 의미있는 토큰 (사용자의 상태를 포함) 으로 구성되어 있기 때문에, Auth Server에 검증 요청을 보내야만 했던 과정을 생략하고 각 서버에서 수행할 수 있게 되어 비용 절감 및 Stateless 아키텍처를 구성할 수 있다.

- 클라이언트 (사용자) 는 Auth Server에 로그인을 한다.
- Auth Server에서 인증을 완료한 사용자는 JWT 토큰을 전달 받는다.
- 클라이언트는 특정 애플리케이션 서버에 리소스 (서비스에 필요한 데이터) 를 요청할 때, 앞서 전달 받은 JWT 토큰을 Authorization Header에 넣어 전달한다.
- 애플리케이션 서버는 전달 받은 JWT 토큰의 유효성을 직접 검사하여 사용자 인증을 할 수 있다.

JWT 방식은 확장성에 큰 강점을 가진다. 만약 세션을 사용하는 경우, 서버를 확장할 때마다 각 서버에 세션 정보를 저장하게 된다. 이렇게 될 경우, 특정 서버에서 로그인 인증을 받을 때 다른 서버에서는 로그인을 했는지 알 수 없다는 단점이 있다. 하지만 JWT는 서버의 수와는 상관없이 토큰을 인증하는 방식을 알고 있다면 인증 과정에 문제가 없다. 더불어, 웹과 앱 간의 쿠키 세션 처리에도 유용하다. 브라우저와 앱에서의 쿠키 처리 방법은 각기 다를 수 있기 떄문에 JWT를 이용하는 것이 다양한 디바이스 차원에서 좋다.

고려해야 할 점은, 사용자 인증 정보가 필요한 요청을 보낼 때 헤더에 JWT 토큰 값을 넣어 보내야 하기 때문에 데이터가 증가하여 네트워크 부하가 늘어날 수 있다. 또한 토큰 자체에 사용자 정보를 담고 있기 때문에 JWT가 만료되기 전에 탈취당하면 서버에서 처리할 수 있는 일이 없다. JWT 방식은 한 번 만들어 클라이언트에게 전달하면 제어가 불가능하기 때문에 만료 시간을 필수적으로 넣어 주어야 한다.

그래서 이번 포스팅에서는 짧은 만료 기간을 갖는 JWT 형식의 Access Token과 긴 만료 기간을 갖는 JWT 형식의 Refresh Token, 두 가지 토큰을 사용하려 한다. 이렇게 구성한다면 아무래도 Refresh Token을 DB에 저장하고 새로운 Access Token을 발급하기 위해 거쳐야 하는 추가적인 과정이 생기게 된다.

만약 Access Token만을 이용해 사용자 인증을 관리하려 할 때, 두 가지 경우가 가능하다. 첫번째는 Access Token의 만료 기간을 짧게 설정하여 사용자가 자주 로그인을 해야하는 경우, 그리고 두번째는 Access Token의 만료 기간을 길게 설정하여 사용자가 로그인하는 빈도는 줄어들지만 토큰이 탈취된다면 긴 만료 기간만큼 위험에 노출되는 경우다.

다른 대안들을 찾아보았지만, 결국 보안성과 편의성 (혹은 성능) 관점에서 Trade Off 가 존재한다. 그래서 적절한 협의점인, Refresh Token을 관리하는 방법을 택한 것이다.



## 시작하기

### 스프링 부트와 OAuth 2.0 설정

스프링 부트 2.x에서 OAuth2 연동 방법이 크게 변경되었다. 기존에 사용한 `spring-security-oauth` 대신, `spring-security-oauth2-client` 라이브러리를 사용해서 진행한다.

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-oauth2-client -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

`pom.xml`에 소셜 로그인 등 클라이언트 입장에서 소셜 기능 구현 시 필요한 의존성을 추가한다. 해당 의존성은 `spring-boot-starter-oauth2-client`와 `spring-security-oauth2-jose`를 기본으로 관리한다.

이후 `Resource Server`로부터 직접 개발한 웹 사이트 등록을 마쳤다는 가정 하에 설명을 이어서 한다.

```yml
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/authservice
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        format_sql: true

  security:
    oauth2:
      client:
        registration:
          google:
            client-id: # 발급 받은 client-id #
            client-secret: # 발급 받은 client-secret #
            scope: # 필요한 권한 #
          kakao:
            client-id: # 발급 받은 client-id #
            client-secret: # 발급 받은 client-secret #
            scope: # 필요한 권한 #
            redirect-uri: "{baseUrl}/{action}/oauth2/code/{registrationId}"
            authorization-grant-type: authorization_code
            client-name: kakao
            client-authentication-method: POST

        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id

```

`application.yml`에 `Resource Server`에 등록한 정보를 연동한다. 구글 혹은 페이스북 등은 `CommonOAuth2Provider` 클래스에서 기본적인 정보를 제공하지만, 카카오 혹은 네이버 등은 그런 게 없으므로 필요한 정보들을 추가로 넣어야 한다. 추가로, H2 DB와 JPA를 설정한 이유는 추후에 유저가 OAuh2 로그인을 통해 회원 가입을 했을 때 데이터를 기록하기 위함이다.

### Member Entity

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString(of = {"id", "oAuth2Id", "email", "nickname", "introduction", "role"})
public class Member extends BaseTimeEntity {

    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    @Column(unique = true, nullable = false)
    private String oAuth2Id;

    @Column(unique = true)
    private String email;

    @Column(unique = true, nullable = false)
    private String nickname;

    private String profileImageUrl;

    private String introduction;

    @Column(nullable = false)
    @Enumerated(value = EnumType.STRING)
    private Role role;

}
```

```java
@Getter
@RequiredArgsConstructor
public enum Role {

    USER("ROLE_USER"), ADMIN("ROLE_ADMIN");

    private final String key;

}
```

- `Member` 엔티티에 `oAuth2Id`가 있는 이유는 다음과 같다. 근래 들어서 개인 정보에 대한 보안이 강화되면서 소셜 로그인을 통해 받아올 수 있는 정보가 갈수록 제한되고 있어서, 심지어는 이메일 주소도 선택적으로 받아올 수 밖에 없는 경우도 있다. 이럴 경우 우선은 소셜 로그인 한 사용자를 식별할 수 있어야 하므로 `Resource Server`에서 넘겨주는 식별자 (ID) 를 저장한다. 물론, 여러 `Resource Server`를 지원할 경우 알맞게 수정해서 저장해야 한다.

### Spring Security Configuration

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.addFilterAfter(jwtAuthenticationFilter, LogoutFilter.class);

        http.authorizeRequests()
                .mvcMatchers("/", "/signUp", "/access-denied", "/exception/**").permitAll()
                .mvcMatchers("/dashboard").hasRole("USER")
                .mvcMatchers("/admin").hasRole("ADMIN")
                .anyRequest().authenticated()
                .expressionHandler(configExpressionHandler());

        http.exceptionHandling()
                .authenticationEntryPoint(configAuthenticationEntryPoint())
                .accessDeniedHandler(configAccessDeniedHandler());

        http.oauth2Login()
                .userInfoEndpoint().userService(customOAuth2UserService)
                .and()
                .successHandler(configSuccessHandler())
                .failureHandler(configFailureHandler())
                .permitAll();

        http.httpBasic();

        http.logout()
                .deleteCookies("JSESSIONID");
    }

    private SecurityExpressionHandler<FilterInvocation> configExpressionHandler() {
        /* ... */
    }

    private CustomAuthenticationEntryPoint configAuthenticationEntryPoint() {
        /* ... */
    }
    
    private CustomAccessDeniedHandler configAccessDeniedHandler() {
        /* ... */
    }

    private CustomAuthenticationSuccessHandler configSuccessHandler() {
        /* ... */
    }
    
    private CustomAuthenticationFailureHandler configFailureHandler() {
        /* ... */
    }
    
    /* ... */

}
```

- `@EnableWebSecurity`는 스프링 시큐리티 설정들을 활성화 시켜준다.
- `http.authorizeRequests()`로 URL별 권한 관리를 설정할 수 있다.
  - `antMatchers()`는 권한 관리 대상을 지정한다. `permitAll()`, `hasRole()`등으로 권한 설정 가능하다.
  - `anyRequest()`는 설정된 값들 이외 나머지 Url들을 나타낸다. `authenticated()`로 인증된 사용자만 접근 가능하도록 설정 가능하다.
  - `expressionHandler()`는 사용자 권한의 계층 구조를 정하는 클래스를 지정한다.
- `http.exceptionHandling()`로 예외 처리를 설정할 수 있다.
  - `authenticationEntryPoint()`는 인증되지 않은 사용자가 인증이 필요한 URL에 접근할 경우 발생하는 예외 처리 클래스를 지정한다.
  - `accessDeniedHandler()`는 인증한 사용자가 추가 권한이 필요한 URL에 접근할 경우 발생하는 예외 처리 클래스를 지정한다.
- `http.oauth2Login()`로 OAuth2 로그인 관련 처리를 설정할 수 있다.
  - `userService()`는 OAuth2 인증 과정에서 `Authentication`을 생성에 필요한 `OAuth2User`를 반환하는 클래스를 지정한다.
  - `successHandler()`는 인증을 성공적으로 마친 경우 처리할 클래스를 지정한다.
  - `failureHandler()`는 인증을 실패한 경우 처리할 클래스를 지정한다.

### OAuth2 인증 과정

차례차례 OAuth2 인증 과정을 살펴보자.

1. 사용자가 소셜 로그인을 정상적으로 완료

2. `AbstractAuthenticationProcessingFilter`에서 OAuth2 로그인 과정을 호출

3. `Resource Server`에서 넘겨주는 정보를 토대로 `OAuth2LoginAuthenticationFilter`의 `attemptAuthentication()`에서 인증 과정을 수행

4. `attemptAuthentication()` 처리 과정에서 `OAuth2AuthenticationToken`을 생성하기 위해 `OAuth2LoginAuthenticationProvider`의 `authenticate()`를 호출

5. `authenticate()` 처리 과정에서 `OAuth2User`를 생성하기 위해 `OAuth2UserService`의 `loadUser()`를 호출

   `OAuth2UserService`의 기본 구현체는 `DefaultOAuth2UserService`이지만, 커스텀한 `OAuth2User`를 반환하도록 구현하고 싶었으므로 직접 구현한 `CustomOAuth2UserService`의 `loadUser()`가 호출됨

   ```java
   @Service
   public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {
   
       @Override
       public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
           Assert.notNull(userRequest, "userRequest cannot be null");
   
           UserInfoEndpoint userInfoEndpoint = userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint();
   
           String userInfoUri = userInfoEndpoint.getUri();
           validateUserInfoUri(userRequest, userInfoUri);
   
           String nameAttributeKey = userInfoEndpoint.getUserNameAttributeName();
           validateUserNameAttributeName(userRequest, nameAttributeKey);
   
           Map<String, Object> attributes = getAttributes(userRequest);
           Set<GrantedAuthority> authorities = getAuthorities(userRequest, attributes);
   
           return KakaoOAuth2User.builder()
                   .authorities(authorities)
                   .attributes(attributes)
                   .nameAttributeKey(nameAttributeKey)
                   .build();
       }
       
       /* ... */
   
   }
   ```
   - `registrationId`: `Resource Server`의 ID. `kakao`가 이에 해당한다.
   - `nameAttributeKey`: OAuth2 로그인 진행 시 키가 되는 필드를 가리킨다. 구글의 경우 기본적으로 'sub'로 지원하지만, 네이버와 카카오는 기본 지원하지 않는다.
6. `loadUser()` 처리 과정에서 `CustomOAuth2User`를 반환

   `CustomOAuth2User`를 따로 만든 이유는 각 `Resource Server`마다 제공하는 정보가 다르므로 `OAuth2User` 인터페이스로 묶어버리기에는 한계가 있다고 생각했다. 이후 동작에 필요한 필드 혹은 메서드를 추가하기 위해 커스텀 `OAuth2User`를 만들었다.

   ```java
   public interface CustomOAuth2User extends OAuth2User {
   
       String getOAuth2Id();
   
       String getEmail();
   
       String getNickname();
   
       String getNameAttributeKey();
   
   }
   ```

   ```java
   public class KakaoOAuth2User implements CustomOAuth2User {
       /* ... */
   }
   ```

7. 정상적으로 6번까지의 과정이 끝났다면 `AbstractAuthenticationProcessingFilter`에서 `successHandler`의 `onAuthenticationSuccess()`을 호출

   `successHandler`의 기본 구현체는 `SavedRequestAwareAuthenticationSuccessHandler`이지만, 추가적인 과정을 위해 `CustomAuthenticationSuccessHandler`을 구현했다. 원래는 `onAuthenticationSuccess()`는 리다이렉션하는 역할 뿐이었지만, OAuth2 로그인을 성공적으로 마치고 해당 사용자가 처음 로그인한 경우 DB에 정보를 저장하는 역할도 추가했다.

   ```java
   @Component
   @RequiredArgsConstructor
   public class CustomAuthenticationSuccessHandler extends SavedRequestAwareAuthenticationSuccessHandler {
   
       @Override
       public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws ServletException, IOException {
           OAuth2AuthenticationToken oAuth2AuthenticationToken = checkInvalidAuthenticationToken(authentication);
           CustomOAuth2User principal = checkInvalidPrincipal(oAuth2AuthenticationToken.getPrincipal());
   
           joinNewMember(oAuth2AuthenticationToken.getAuthorizedClientRegistrationId(), principal);
   
           clearAuthenticationAttributes(request);
           getRedirectStrategy().sendRedirect(request, response, "/auth/success");
       }
       
       /* ... */
   
   }
   ```

8. 6번까지의 과정이 정상적으로 끝나지 않았다면 `AbstractAuthenticationProcessingFilter`에서 `failureHandler`에서 `onAuthenticationFailure()`을 호출

   `failureHandler`의 기본 구현체는 `SimpleUrlAuthenticationFailureHandler`이지만, 커스텀한 예외 처리를 위해 `CustomAuthenticationFailureHandler`를 구현했다.

여기까지가 올바른 요청일 경우의 처리 과정이다. 이외의 경우는 크게 두 가지가 있다.

- 인증되지 않은 사용자가 인증이 필요한 URL에 접근하려 한다면 `authenticationEntryPoint`에서 예외 처리

  기본 구현체인 `LoginUrlAuthenticationEntryPoint`가 잘 구현되어 있지만, 커스텀한 예외 처리 동작을 위해 `CustomAuthenticationEntryPoint`를 구현했다.

- 인증된 사용자가 권한이 부족한 URL에 접근하려 한다면 `accessDeniedHandler`에서 예외 처리

  기본 구현체인 `AccessDeniedHandlerImpl`가 있지만, 역시 커스텀 예외 처리 동작을 위해 `CustomAccessDeniedHandler`를 구현했다.

이렇게 OAuth2 인증 과정이 마무리 된다. 이어서는 인증된 정보를 토대로 JWT를 발급하는 과정을 살펴보자.

### JWT 기반 토큰 인증

위에서 설명했듯이, 마이크로 서비스에서는 기존 인증 방식으로는 서버 부하가 걸릴 수 있기 때문에 JWT 방식을 사용한다. 즉 이는 클라이언트 (사용자) 가 JWT 토큰을 갖고 있다면 각 마이크로 서버에서는 해당 토큰을 검증하기만 하면 인증 과정을 마칠 수 있다는 얘기다.

따라서 구현해야 하는 것들은 토큰 인증을 처리할 Security Filter 와 실질적인 토큰 생성 및 검증을 할 Utility 정도다.

- JWT 토큰 인증 과정을 처리하는 `JwtAuthenticationFilter`

  스프링 시큐리티에서는 기본적으로 토큰 처리를 위한 필터가 없으므로 구현해서 Filter Chain에 추가해야 한다.

  ```java
  @Component
  @RequiredArgsConstructor
  public class JwtAuthenticationFilter extends OncePerRequestFilter {
  
      private final JwtUtil jwtUtil;
  
      @Override
      protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
          if (isAppropriateRequestForFilter(request)) {
              try {
                  String token = jwtUtil.resolveToken(request);
                  Authentication authentication = jwtUtil.getAuthentication(token);
                  SecurityContextHolder.getContext().setAuthentication(authentication);
              } catch (JWTVerificationException e) {
                  /* ... */
              }
          }
          filterChain.doFilter(request, response);
      }
      
      /* ... */
  
  }
  ```
  - JWT 토큰 검증이 필요한 경우에만 동작하도록 조건 처리를 한다.
  - 클라이언트에서는 Authorization Header에 토큰을 담아서 보내므로, `HttpServletRequest`에서 토큰을 추출한 후, 검증하여 `Authentication`을 `SecurityContext`에 저장한다.

  ```java
  @Configuration
  @EnableWebSecurity
  @RequiredArgsConstructor
  public class SecurityConfig extends WebSecurityConfigurerAdapter {
  
      private final JwtAuthenticationFilter jwtAuthenticationFilter;
  
      @Override
      protected void configure(HttpSecurity http) throws Exception {
          /* ... */
          
          http.addFilterAfter(jwtAuthenticationFilter, LogoutFilter.class);
      }
      
      /* ... */
  
  }
  ```

  - 구현한 `JwtAuthenticationFilter`를 Filter Chain의 알맞은 위치에 추가한다.

    실질적인 인증 과정이 일어나기 직전 위치인 `LogoutFilter` 직후 위치에 추가한다.

- 토큰 검증 및 생성 과정을 처리할 `JwtUtil`

  ```java
  @Component
  public class JwtUtil {
  
      public String generateRefreshToken(CustomOAuth2User customOAuth2User) {
          /* ... */
      }
  
      public String generateAccessToken(String refreshToken) {
          /* ... */
      }
  
      public String resolveToken(HttpServletRequest request) {
          /* ... */
      }
  
      public Authentication getAuthentication(String accessToken) {
          /* ... */
      }
      
      /* ... */
  
  }
  ```

### 클라이언트에게 토큰 전달

OAuth2 로그인을 정상적으로 마친 사용자는 토큰 없는 상태이므로, 특정 URL로 리다이렉션하여 Controller에서 JWT 기반 Access Token과 Refresh Token을 발급 및 Response로 전달한다. 클라이언트는 발급받은 Access Token과 Refresh Token을 안전한 공간에 보관한다. Access Token은 매 요청마다 Authorization Header에 추가해 보낼 것이고, Refresh Token은 Access Token이 만료되었다는 응답을 받았을 경우 Access Token 재발급을 위해 사용한다.

Access Token 재발급을 위해 클라이언트가 Refresh Token을 Authorization Header에 추가해 보낸 경우, Auth Server는 DB에 기록되어 있는 사용자의 Refresh Token과 동일한지 검증한 후, 문제가 없다면 해당 요청을 보낸 사용자가 본인임을 의미하므로 새로운 Access Token을 발급해 Response로 전달한다.



## 참고 자료

### [참고 블로그 01](https://ozofweird.tistory.com/m/582)

### [참고 블로그 02](https://webfirewood.tistory.com/115)

### [참고 블로그 03](https://velog.io/@ehdrms2034/Spring-Security-JWT-Redis%EB%A5%BC-%ED%86%B5%ED%95%9C-%ED%9A%8C%EC%9B%90%EC%9D%B8%EC%A6%9D%ED%97%88%EA%B0%80-%EA%B5%AC%ED%98%84)

### [참고 Github 01](https://github.com/SilverNine/spring-boot-jwt-tutorial)

### [참고 Github 02](https://github.com/ehdrms2034/SpringBootWithJava/blob/master/Spring_React_Login/backend/src/src/main/java/com/donggeun/springSecurity/service/JwtUtil.java)

### [참고 Github 03](https://github.com/boyd-dev/SimpleSpringBoot/blob/main/src/main/java/com/example/demo/utils/JwtUtils.java)