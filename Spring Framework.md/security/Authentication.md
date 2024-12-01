### Spring Security 인증 개념
**인증(Authentication)**은 사용자가 애플리케이션에 접근하려고 할 때 그 사용자가 진짜 그 사람인지 확인하는 과정입니다. 일반적으로 사용자는 **사용자 이름**과 **비밀번호**와 같은 로그인 자격 증명을 입력하게 됩니다. 사용자가 입력한 정보가 시스템에 저장된 것과 일치하면, 시스템은 그 사용자가 신원 확인된 유효한 사용자임을 인식하고 접근을 허용하게 됩니다.

Spring Security는 이러한 인증 과정에서 많은 기본적인 기능을 제공해 줍니다. **Spring Security Starter**를 프로젝트에 추가하면, 자동으로 인증 과정이 설정되며, 기본 사용자 계정을 하나 생성해 줍니다. 그러나 일반적으로는 여러 사용자가 애플리케이션에 접근할 수 있어야 하기 때문에, 직접 여러 사용자를 설정하고 싶을 때가 많습니다.

여기에서는 **Spring Security 6.1.0** 버전을 기준으로 하여, 몇 명의 **하드코딩된 인메모리 사용자**를 추가하는 방법을 설명합니다.

### 사용자 저장소 (User Storage) 설정하기
Spring Security에서 인증을 설정하기 위해 **사용자 저장소**를 정의해야 합니다. 이를 위해서는 사용자 정보를 관리하는 **Spring 빈(Bean)**을 제공하는 **설정 클래스(Configuration Class)**를 만들어야 합니다.

#### 자바 예제 코드
```java
import org.springframework.context.annotation.Configuration;

@Configuration
public class SecurityConfig {
    // 설정 클래스 정의 (비어있는 상태)
}
```

### 하드코딩된 사용자 정의하기
Spring Security에서는 사용자 정보를 관리하기 위해 **UserDetailsService** 빈을 정의할 수 있습니다. 이때 우리는 Spring Security에서 제공하는 **User 클래스**를 활용할 수 있는데, 이 클래스는 사용자 이름, 비밀번호와 같은 사용자 정보를 저장하는 데 사용됩니다. 그 다음 **InMemoryUserDetailsManager**를 사용하여 사용자를 메모리에 저장합니다.

#### 자바 예제 코드
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@Configuration
public class SecurityConfig {

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user1 = User.withUsername("user1")
                .password("pass1")
                .roles()  // 역할을 정의할 수 있지만 여기서는 사용하지 않음
                .build();
        UserDetails user2 = User.withUsername("user2")
                .password("pass2")
                .roles()
                .build();

        return new InMemoryUserDetailsManager(user1, user2);
    }
}
```
위 코드에서는 `user1`과 `user2`라는 두 명의 사용자를 정의하고, 비밀번호는 각각 `"pass1"`, `"pass2"`입니다.

### 비밀번호 인코더 설정하기
위의 설정만으로는 아직 **에러**가 발생하게 됩니다. Spring Security는 보안상의 이유로 비밀번호를 반드시 **인코딩(암호화)**하도록 요구하기 때문입니다. 만약 비밀번호를 인코딩하지 않고 평문 그대로 저장할 경우, 데이터베이스에 접근할 수 있는 누군가가 비밀번호를 쉽게 알 수 있게 됩니다. 

Spring Security에서 비밀번호를 인코딩하기 위해 **PasswordEncoder** 인터페이스를 사용합니다. 이 인터페이스에는 두 가지 주요 메서드가 있습니다:
- **encode()**: 평문 비밀번호를 받아서 인코딩된 비밀번호를 반환합니다. 비밀번호를 저장하기 전에 이 메서드를 사용합니다.
- **matches()**: 입력된 평문 비밀번호와 저장된 인코딩된 비밀번호가 일치하는지 확인합니다. Spring Security가 인증 과정에서 이 메서드를 사용하여 사용자가 입력한 비밀번호가 맞는지 확인합니다.

다음은 **PasswordEncoder**를 설정하는 방법입니다.

#### 자바 예제 코드
```java
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;

@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```
`createDelegatingPasswordEncoder()` 메서드는 **DelegatingPasswordEncoder** 인스턴스를 생성하는데, 이 인코더는 여러 종류의 인코더(bcrypt, SHA-256, noop 등)를 사용할 수 있도록 도와줍니다.

#### 비밀번호 인코딩하여 사용자 추가하기
사용자를 추가할 때는 반드시 비밀번호를 인코딩해야 합니다. 예를 들어, `BCryptPasswordEncoder`를 사용하여 첫 번째 사용자의 비밀번호를 인코딩할 수 있습니다.

```java
UserDetails user1 = User.withUsername("user1")
        .password(this.passwordEncoder().encode("pass1"))
        .roles()
        .build();
```
`withDefaultPasswordEncoder()`를 사용하여 비밀번호를 인코딩할 수도 있지만, 이 메서드는 보안상 안전하지 않아 **Deprecated(사용 자제)** 처리되어 있습니다.

### 전체 코드 예제
다음은 모든 설정을 적용한 전체 코드입니다.

#### 자바 예제 코드
```java
package org.hyperskill.hsspringtopics;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@Configuration
public class SecurityConfig {

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user1 = User.withUsername("user1")
                .password(this.passwordEncoder().encode("pass1"))
                .roles()
                .build();
        UserDetails user2 = User.withDefaultPasswordEncoder()
                .username("user2")
                .password("pass2")
                .roles()
                .build();

        return new InMemoryUserDetailsManager(user1, user2);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```
이 코드에서는 `user1`과 `user2`를 메모리에 저장하고 각각 인코딩된 비밀번호를 사용합니다.

### HttpSecurity 설정
사용자가 애플리케이션에 접근할 때 어떤 인증 방법을 사용할 것인지 정의하려면 **SecurityFilterChain** 빈을 사용하여 설정을 추가할 수 있습니다. 이 빈을 정의할 때 **HttpSecurity** 빌더를 사용하여 **DefaultSecurityFilterChain**의 인스턴스를 구성합니다.

#### 자바 예제 코드
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
            .authorizeHttpRequests(matcherRegistry -> matcherRegistry
                    .anyRequest().authenticated()  // 모든 요청에 대해 인증 필요
            )
            .formLogin(Customizer.withDefaults())  // 폼 기반 로그인 활성화
            .httpBasic(Customizer.withDefaults())  // HTTP 기본 인증 활성화
            .build();
}
```
- **anyRequest().authenticated()**: 애플리케이션의 모든 URL에 접근하려면 인증이 필요함을 의미합니다.
- **formLogin()**: 폼 기반 인증을 활성화합니다.
- **httpBasic()**: 기본 HTTP 인증을 활성화합니다.

이 설정을 통해 Spring Security가 기본적으로 적용하는 인증 방법을 명시적으로 활성화 또는 비활성화할 수 있습니다.

### 결론
이 주제에서는 **하드코딩된 인메모리 사용자**를 생성하여 Spring Security에서 인증을 구성하는 방법에 대해 배웠습니다. Spring Security는 기본적으로 강력한 보안 설정을 제공하며, 이 설정을 재정의하여 애플리케이션의 요구에 맞는 인증 구성을 추가할 수 있습니다.

기억해야 할 점은 이번에 다룬 하드코딩된 사용자는 주로 테스트 목적이거나 예제로 사용되며, 실제 애플리케이션에서는 사용자 정보를 **데이터베이스**에 저장하고 등록 API 등을 통해 관리하는 것이 일반적입니다. 또한, **비밀번호는 절대로 평문으로 저장하지 않고 인코딩**된 상태로 저장하여 보안을 강화해야 합니다.

Spring Security는 인증 외에도 **인가(Authorization)**, **입력 유효성 검사(Input Validation)**, **오류 처리(Error Handling)** 등 다양한 보안 기능을 제공하므로, 이러한 부분도 함께 고려하여 애플리케이션의 보안을 강화하는 것이 중요합니다.