# Hello Spring Security

 스프링 시큐리티는 표준 서블릿 필터로 서블릿 컨테이너와 통합됨. 서블릿 컨테이너에서 실행하는 모든 어플리케이션에서 동작함. 

[스프링 시큐리티 레퍼런스](https://github.com/spring-projects/spring-security/tree/5.3.2.RELEASE/samples/boot/helloworld)

---

### 8.1. Updating Dependencies

##### Gradle

```xml
dependencies {
    compile "org.springframework.boot:spring-boot-starter-security"
}
```



### 8.3. Spring Boot Auto Configuration

- 스프링 시큐리티의 디폴트 설정을 활성화해서 `springSecurityFilterChain`이라는 이름의 서블릿 `Filter` 빈을 생성. 이 빈이 어플리케이션 내의 모든 보안 처리를 담당 (어플리케이션 URL 보호, 제출한 사용자 이름과 비밀번호 검증, 로그인 폼으로 리다이렉트 등)
- `user` 라는 사용자 이름과 콘솔에도 출력되는 랜덤 생성한 비밀번호를 가지고 있는 `UserDetailsService` 빈을 만듦.
- 서블릿 컨테이너에 `springSecurityFilterChain`이란 이름의 `Filter` 빈을 등록해 모든 요청에 적용

##### 기능 요약

- 어플리케이션의 모든 상호작용에 사용자 인증 요구
- 디폴트 로그인 폼 생성
- `user`라는 이름과 콘솔에 출력한 비밀번호를 사용한 폼 기반 인증 지원 
- BCrypt로 저장할 비밀번호 보호
- 사용자 로그아웃 지원
- CSRF 공격 방어
- Session Fixation 방어
- 보안 헤더 통합
- 서블릿 API 메소드 통합
