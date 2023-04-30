## 로그인 구현

##### 용어 정리

- `redirect URI`
  - 서비스에서 파라미터로 인증 정보가 주었을 때, 인증이 성공하면 구글에서 리다이렉트할 URL
  - spring boot 2 security에서는 기본적으로 `{domain}/login/oauth2/code/{socialId}`
  - 별도로 redirect URL 을 지원하는 Controller을 만들 필요 없음
- `user_name_attribute`
  - 회원 식별자를 담는 키


##### 1. Role (Enum)

- 권한 코드에 항상 `ROLE_`이 앞에 있어야 함.

```java
@Getter
@RequiredArgsConstructor
public enum Role {
    
    GUEST("ROLE_GUEST", "손님"),
    USER("ROLE_USER", "일반 사용자");
    
    private final String key;
    private final String title;
}

```



##### 2. User(Account) 생성

- 레퍼런스에선 update 메서드를 둠으로써, 소셜 로그인을 할 때마다 최신의 이름과, 이미지가 업데이트되도록 했음.
- 이번 프로젝트에선 소셜로그인에서 이름과 이미지를 가져오지 않기 때문에 생략

```java
...
        public User update(String name, String picture) {
        this.name = name;
        this.picture = picture;

        return this;
    }

    public String getRoleKey() {
        return this.role.getKey();
    }
...
```



##### 3. UserRepository (AccountRepository) 생성

- 예시에선 Email로 했음
- 카카오와 네이버 등 소셜에서 사용하는 email이 겹치는 경우에 대비해서 커스텀 socialId를 만들어서 저장할 예정이므로, `findBySocialId`로 수정

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email); // findBySocialId
}

```



##### 4. OAuthAttribute

- `OAuth2UserService` 에서 사용할 `OAuthAttribute` 클래스 생성
- 원래는 name, email 과 같은 사용자 정보들이 더 있었지만, 다 생략
- 전체적인 메커니즘을 봤을 때 OAuthAttribute 가 크게 필요 없어 보여서 생략

```java
@Getter
public class OAuthAttributes {
    private Map<String, Object> attributes;
    private String nameAttributeKey;
    // private String name;


    @Builder
    public OAuthAttributes(Map<String, Object> attributes, String nameAttributeKey) {
        this.attributes = attributes;
        this.nameAttributeKey = nameAttributeKey;
    }

    public  static OAuthAttributes of(String registrationId, String userNameAttributeName, Map<String, Object> attributes) {
        return ofGoogle(userNameAttributeName, attributes);
    }

    private static OAuthAttributes ofGoogle(String userNameAttributeName, Map<String, Object> attributes) {
        return OAuthAttributes.builder()
                //.name((String) attributes.get("name"))
                .attributes(attributes)
                .nameAttributeKey(userNameAttributeName)
                .build();
    }

    public User toEntity() {
        return User.builder()
                //.name(name)
                .role(Role.GUEST)
                .build();
    }
}
```



##### 5. CustomOAuth2UserService

```java
@RequiredArgsConstructor
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {
    private final UserRepository userRepository;
    private final HttpSession httpSession;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2UserService<OAuth2UserRequest, OAuth2User> delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);

        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint().getUserNameAttributeName();

        OAuthAttributes attributes = OAuthAttributes.of(registrationId, userNameAttributeName, oAuth2User.getAttributes());

        User user = saveOrUpdate(attributes);

        httpSession.setAttribute("user", new SessionUser(user));

        return new DefaultOAuth2User(Collections.singleton(new SimpleGrantedAuthority(user.getRoleKey())),
                attributes.getAttributes(),
                attributes.getNameAttributeKey());
        }

        private User saveOrUpdate(OAuthAttributes attributes) {
            User user = userRepository.findByEmail(attributes.getEmail())
                    .map(entity -> entity.update(attributes.getName(), attributes.getPicture()))
                    .orElse(attributes.toEntity());

            return userRepository.save(user);
        }
}
```

- 다른 레퍼런스 찾아봄

  ```java
  @Service
  public class CustomOAuth2UserService extends DefaultOAuth2UserService {
  
      private final UserRepository userRepository;
  
      public CustomOAuth2UserService(UserRepository userRepository) {
          this.userRepository = userRepository;
      }
  
      @Override
      public OAuth2User loadUser(OAuth2UserRequest oAuth2UserRequest) {
          Map<String, Object> attributes = super.loadUser(oAuth2UserRequest).getAttributes();
          Provider provider = Provider.from(oAuth2UserRequest.getClientRegistration().getRegistrationId());
  
          validateAttributes(attributes);
  
          registerIfNewUser(attributes, provider);
          
          return null;
      }
  
      private void validateAttributes(Map<String, Object> userInfoAttributes) {
          if (!userInfoAttributes.containsKey("email")) {
              throw new IllegalArgumentException("서드파티의 응답에 email이 존재하지 않습니다!!!");
          }
      }
  
      private void registerIfNewUser(Map<String, Object> userInfoAttributes, Provider provider) {
          String email = (String) userInfoAttributes.get("email");
  
          Optional<User> optionalUser = userRepository.findByEmailAndProvider(email, provider);
  
          if (optionalUser.isPresent()) {
              return;
          }
          User user = new User(null, email, "대충 아무 텍스트", provider);
          userRepository.save(user);
      }
  }
  ```

  



##### 6. SecurityConfig

```java
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    private final CustomOAuth2UserService customOAuth2UserService;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .csrf().disable()
                .headers().frameOptions().disable()
                .and()
                    .authorizeRequests()
                    .antMatchers("/api/v1/**").hasRole(Role.USER.name())
                    .anyRequest().authenticated()
                .and()
                    .logout()
                        .logoutSuccessUrl("/")
                .and()
                    .oauth2Login()
                        .userInfoEndpoint() // 로그인 성공 이후 사용자 정보 가져올 때 설정
                    .userService(customOAuth2UserService);
    }
}
```

