### Spring Security의 인증과 인가 개념
먼저 알아둘 점은 **인가(Authorization)**는 **인증(Authentication)**이 완료된 후에 이루어진다는 것입니다. 사용자가 애플리케이션에 접근할 때 **인증**을 통해 사용자가 누구인지를 확인하고 나면, 시스템은 그 사용자가 요청한 리소스에 접근할 권한이 있는지 결정합니다. 이렇게 **인가**를 통해 특정 리소스에 대한 접근을 허용하거나 제한하게 됩니다.

이 주제에서는 **Spring Security 6.1.0**을 기준으로, 여러 엔드포인트를 가진 애플리케이션을 만들고, 접근 규칙을 설정하는 방법을 다룰 것입니다. 최종적으로는 **Postman**을 사용하여 프로그램을 테스트하는 방법도 설명합니다.

### 역할과 권한 (Roles and Authorities)
**인가의 주요 목표**는 특정 리소스에 대해 **일부 사용자는 접근을 허용하고, 다른 사용자는 접근을 제한**하는 것입니다. 이를 위해 사용자를 구분할 수 있는 메커니즘이 필요합니다. Spring Security는 이러한 작업을 돕기 위해 **역할(Role)**과 **권한(Authority)**이라는 개념을 제공합니다.

- **권한(Authority)**은 사용자가 애플리케이션 내에서 수행할 수 있는 **행동**을 나타냅니다. 예를 들어, 제시카(Jessica)는 특정 엔드포인트에 대해 읽기(READ)와 쓰기(WRITE)만 가능하고, 조셉(Joseph)은 읽기, 쓰기, 삭제(DELETE), 업데이트(UPDATE)를 모두 할 수 있다고 할 수 있습니다. 권한은 단순한 **문자열**로 표현되며, 애플리케이션 개발 중에 권한 이름은 자유롭게 정할 수 있습니다.

- **역할(Role)**은 여러 권한의 집합으로, 특정 사용자 유형을 정의할 수 있습니다. 예를 들어, 애플리케이션에서 데이터의 읽기와 쓰기만 가능한 사용자와, 읽기, 쓰기, 업데이트, 삭제를 모두 할 수 있는 사용자 유형을 나눌 수 있습니다. 이를 위해 권한을 네 개 정의하는 대신, **ROLE_USER**와 **ROLE_ADMIN** 두 개의 역할로 정의하는 것이 더 효율적입니다.

**Spring Security**에서는 역할과 권한이 내부적으로 **비슷하게 동작**합니다. 역할은 기본적으로 "ROLE_"이라는 접두어를 가진 **권한**으로 처리됩니다.

### 엔드포인트 설정
프로그램은 여러 엔드포인트를 가지며, 일부는 인증 없이 접근할 수 있고, 나머지는 **인증** 및 **특정 역할**이나 **권한**이 필요합니다:
- **GET /**, **GET /public** - 인증 없이 접근 가능
- **/secured** - 인증된 사용자만 접근 가능
- **/user** - USER 또는 ADMIN 역할이 있는 인증된 사용자만 접근 가능
- **/admin** - ADMIN 역할이 있는 인증된 사용자만 접근 가능
- **POST /public** - WRITE 권한이 있는 인증된 사용자만 접근 가능

### 초기 설정
먼저 **Spring Boot 프로젝트**를 생성하고, **web**과 **security starters**를 포함했다고 가정합니다. 각 엔드포인트를 처리할 수 있는 **컨트롤러**를 추가해보겠습니다.

#### 자바 예제 코드 (REST 컨트롤러)
```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DemoController {

    @PostMapping(path = "/public")
    public String postPublic() {
        return "Access to 'POST /public' granted";
    }

    @GetMapping(path = "/public")
    public String getPublic() {
        return "Access to 'GET /public' granted";
    }

    @GetMapping(path = "/secured")
    public String secured() {
        return "Access to '/secured' granted";
    }

    @GetMapping(path = "/user")
    public String user() {
        return "Access to '/user' granted";
    }

    @GetMapping(path = "/admin")
    public String admin() {
        return "Access to '/admin' granted";
    }
}
```
### 사용자와 역할 정의하기
이제 사용자를 생성하고 **역할**과 **권한**을 할당하겠습니다. 총 세 명의 사용자가 필요합니다. 각각 다음과 같은 권한과 역할을 갖게 될 것입니다:
1. WRITE 권한만 있는 사용자
2. USER 역할을 가진 사용자
3. ADMIN 역할과 WRITE 권한을 가진 사용자

#### 자바 예제 코드 (사용자 정의)
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@Configuration
public class SecurityConfig {

    @Bean
    public UserDetailsService userDetailsService() {
        var user1 = User.withUsername("user1")
                .password(passwordEncoder().encode("pass1"))
                .authorities("WRITE") // WRITE 권한만 있는 사용자
                .build();
        var user2 = User.withUsername("user2")
                .password(passwordEncoder().encode("pass2"))
                .roles("USER") // USER 역할을 가진 사용자
                .build();
        var user3 = User.withUsername("user3")
                .password(passwordEncoder().encode("pass3"))
                .authorities("ROLE_ADMIN", "WRITE") // ADMIN 역할과 WRITE 권한을 가진 사용자
                .build();

        return new InMemoryUserDetailsManager(user1, user2, user3);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```
**roles** 메서드를 사용할 때는 **ROLE_** 접두어를 추가하지 않아도 됩니다. Spring Security가 자동으로 **ROLE_**을 붙여줍니다. 하지만 **authorities** 메서드를 사용할 때는 **ROLE_** 접두어를 직접 추가해야 합니다. 예를 들어, `roles("ADMIN")`은 `authorities("ROLE_ADMIN")`과 동일하게 처리됩니다.

### HttpSecurity를 이용한 접근 제어 설정
이제 각 엔드포인트에 대한 접근 권한을 설정할 차례입니다. 이를 위해 Spring Security의 **HttpSecurity** 객체를 사용하여 **SecurityFilterChain** 빈을 생성하고, `authorizeHttpRequests()` 메서드를 사용해 접근을 설정할 수 있습니다.

#### 자바 예제 코드 (SecurityFilterChain 설정)
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
            .authorizeHttpRequests(matcherRegistry -> matcherRegistry
                    .requestMatchers("/user").hasAnyRole("USER", "ADMIN") // USER 또는 ADMIN 역할 필요
                    .requestMatchers("/admin").hasRole("ADMIN") // ADMIN 역할 필요
                    .requestMatchers(HttpMethod.POST, "/public").hasAuthority("WRITE") // WRITE 권한 필요
                    .requestMatchers("/secured").authenticated() // 인증된 사용자만 접근 가능
                    .requestMatchers(HttpMethod.GET, "/*").permitAll() // 인증 여부와 상관없이 접근 가능
                    .anyRequest().denyAll() // 그 외 모든 요청은 접근 불가
            )
            .httpBasic(Customizer.withDefaults()) // 기본 HTTP 인증 사용
            .csrf(csrfConfigurer -> csrfConfigurer.disable()) // CSRF 보호 비활성화 (POST 테스트용)
            .build();
}
```
#### 각 줄의 설명:
1. **`requestMatchers("/user").hasAnyRole("USER", "ADMIN")`**: `/user` 엔드포인트는 USER 또는 ADMIN 역할을 가진 사용자만 접근할 수 있도록 설정합니다.
2. **`requestMatchers("/admin").hasRole("ADMIN")`**: `/admin` 엔드포인트는 ADMIN 역할을 가진 사용자만 접근 가능합니다.
3. **`requestMatchers(HttpMethod.POST, "/public").hasAuthority("WRITE")`**: `/public` 엔드포인트에 POST 요청을 하려면 WRITE 권한이 필요합니다.
4. **`requestMatchers("/secured").authenticated()`**: `/secured` 엔드포인트는 인증된 사용자만 접근할 수 있습니다.
5. **`requestMatchers(HttpMethod.GET, "/*").permitAll()`**: 모든 사용자에게 GET 요청에 대해 접근을 허용합니다.
6. **`anyRequest().denyAll()`**: 그 외의 모든 요청은 접근을 차단합니다.

여기서 **매처의 순서**가 중요합니다. 여러 매처가 동일한 URL 패턴을 가리킬 경우 **첫 번째 매처가 적용**되므로, 설정 순서를 주의해야 합니다. 예를 들어, `/secured` 엔드포인트가 "/*" 와일드카드에 포함될 수 있기 때문에 `/secured` 매처를 먼저 설정해야 합니다.

### Postman을 사용한 테스트
프로그램을 실행한 후 **Postman**을 사용해 각 엔드포인트에 접근할 수 있습니다. 예를 들어, **루트 URL**과 **/public**은 인증 없이도 접근이 가능합니다. 하지만 **/secured**나 **/user**, **/admin**과 같은 엔드포인트는 각각 인증된 사용자와 특정 역할을 가진 사용자만 접근할 수 있습니다.

인증이 필요한 엔드포인트에 인증 없이 접근하려고 하면, **401 Unauthorized** 상태 코드가 반환됩니다. 이때 올바른 사용자 이름과 비밀번호를 입력하면 인증이 완료되어 접근할 수 있습니다. 

또한, 역할이 없는 사용자가 `/admin`에 접근하려고 하면 **403 Forbidden** 상태 코드가 반환됩니다. 이는 인증은 되었지만 해당 엔드포인트에 대한 **접근 권한이 없기** 때문입니다.

### 결론
이번 주제에서는 **Spring Security**를 사용하여 인가를 설정하는 방법을 배웠습니다. **authorizeHttpRequests** 메서드를 사용하여 특정 엔드포인트에 대한 접근 규칙을 설정하고, 각 엔드포인트에 대해 **역할**이나 **권한**을 설정하는 방법을 배웠습니다. 또한, **HTTP 기본 인증**을 활성화하고 **CSRF 보호**를 비활성화하여 Postman으로 POST 요청을 테스트하는 방법도 설명했습니다.

이러한 원칙을 따라 Spring 애플리케이션에서 **민감한 리소스에 대한 접근을 제한**하고 사용자에게 **안전한 환경**을 제공할 수 있습니다.