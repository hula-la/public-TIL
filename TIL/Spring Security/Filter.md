> https://godekdls.github.io/Spring%20Security/servletsecuritythebigpicture/

## Authentication을 제공하는 필터

- UsernamePasswordAuthenticationFilter : 폼 로그인을 처리해준다.
- OAuth2LoginAuthenticationFilter : 소셜 로그인을 처리해준다.
- RememberMeAuthenticationFilter : remember-me 쿠키 로그인을 처리해준다.
- AnonymousAuthenticationFilter : 로그인하지 않았다는 것을 인증해준다.
- SecurityContextPersistenceFilter : 기존 로그인을 유지해준다. (기본적으로 session 을 이용함)
- BearerTokenAuthenticationFilter : JWT 로그인을 처리해준다.
- BasicAuthenticationFilter : ajax 로그인 (Session이 있는 경우에 사용)
- OpenIDAuthenticationFilter : OpenID 로그인을 처리해준다.
