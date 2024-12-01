### Spring Boot Security 개요 및 시작하기
Spring Security는 Spring 프레임워크의 모듈로서 **인증(Authentication)**과 **권한 부여(Authorization)** 기능을 제공하여 웹 애플리케이션을 보호하는 역할을 합니다. 이를 통해 애플리케이션에서 요청에 대한 접근 권한을 제어하고 사용자 데이터를 안전하게 보호할 수 있습니다. Spring Security는 **의존성 추가**만으로도 기본적인 보안 기능을 활성화할 수 있는 간편함을 제공합니다.

#### 1. **시작하기**
Spring Boot 프로젝트에 Spring Security를 적용하기 위해서는 보안 스타터 의존성을 추가해야 합니다. 이 작업은 매우 간단하며, `pom.xml` 파일에 다음과 같은 의존성을 추가하는 방식으로 시작됩니다:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

이 의존성을 추가하면 Spring Boot가 기본적으로 제공하는 보안 설정이 자동으로 적용되어 **HTTP 기본 인증**, **폼 기반 인증**, **CSRF 보호** 등의 기능이 활성화됩니다.

#### 2. **기본적인 Spring Security 동작**
의존성을 추가하고 애플리케이션을 실행하게 되면, 기본적으로 **로그인 페이지**가 자동 생성되고 **기본 사용자 계정**이 만들어집니다. 사용자는 **"user"**라는 기본 사용자명과, 서버 실행 시 **콘솔**에 출력된 자동 생성 비밀번호를 사용하여 로그인할 수 있습니다.

- **기본 로그인 페이지**: `http://localhost:8080/login`에서 확인할 수 있으며, 이 페이지를 통해 사용자 인증이 이루어집니다.
- **기본 비밀번호**는 매번 애플리케이션 실행 시마다 새롭게 생성되며 콘솔에서 확인할 수 있습니다.

**application.properties** 파일을 통해 직접 사용자명과 비밀번호를 설정할 수도 있습니다:
```properties
spring.security.user.name=someone
spring.security.user.password=123
```
이렇게 설정하면 기본 비밀번호가 더 이상 생성되지 않으며, 설정한 값으로 로그인할 수 있습니다.

### Spring Security의 구성 요소들
Spring Security는 크게 다음과 같은 주요 구성 요소들로 나누어집니다.

#### 1. **Filter Chain (필터 체인)**
Spring Security의 **필터 체인**은 애플리케이션으로 들어오는 HTTP 요청을 **가로채서 보안 처리**를 수행합니다. 각 필터는 **인증**이나 **권한 부여**와 같은 보안 제어를 시행하며, 필터들은 특정한 순서대로 실행됩니다. 중요한 필터들은 다음과 같습니다:

- **UsernamePasswordAuthenticationFilter**: 사용자명과 비밀번호를 받아 인증을 수행합니다.
- **BasicAuthenticationFilter**: 요청의 "Authorization" 헤더에서 인증 정보를 추출하여 인증을 수행합니다.
- **SecurityContextPersistenceFilter**: 세션을 기반으로 **SecurityContext**를 유지하거나 새로 생성합니다.
- **DefaultLoginPageGeneratingFilter**와 **DefaultLogoutPageGeneratingFilter**: 기본 Spring 로그인 및 로그아웃 페이지를 생성합니다.
- **LogoutFilter**: 사용자를 로그아웃 처리하고 세션을 무효화합니다.

필터 체인은 보안의 첫 단계로서, 모든 HTTP 요청을 분석하고 필요한 인증 및 권한 부여 작업을 시행합니다.

#### 2. **AuthenticationManager (인증 관리자)**
**AuthenticationManager**는 **인증**을 담당하는 핵심 인터페이스입니다. 필터에서 받은 사용자 정보를 기반으로 인증을 수행하며, 기본적으로 **ProviderManager**라는 구현체를 사용합니다.

- **ProviderManager**는 여러 개의 **AuthenticationProvider(인증 제공자)**를 가지고 있으며, 각 제공자를 순차적으로 호출하여 인증을 시도합니다.

#### 3. **AuthenticationProvider (인증 제공자)**
**AuthenticationProvider**는 실제 인증 로직을 처리하는 역할을 담당합니다. 다양한 인증 제공자가 있으며, 각 인증 제공자는 자체적인 인증 방식으로 사용자 자격 증명을 처리합니다. 예를 들면:

- **DaoAuthenticationProvider**: **UserDetailsService**를 사용하여 사용자 정보를 데이터베이스에서 조회하고, 사용자명과 비밀번호를 검증합니다.
- **JaasAuthenticationProvider**: JAAS를 사용하여 인증합니다.
- **OpenIDAuthenticationProvider**와 **OAuth2AuthenticationProvider**: 각각 OpenID와 OAuth 2.0을 통해 인증을 지원합니다.

#### 4. **UserDetailsService**
**UserDetailsService**는 **DaoAuthenticationProvider**가 사용자 정보를 조회할 때 사용하는 인터페이스로, 데이터베이스나 메모리 등에서 사용자 정보(사용자명, 비밀번호, 권한)를 가져옵니다.

Spring Security는 다양한 **UserDetailsService** 구현체를 제공합니다:
- **InMemoryUserDetailsManager**: 모든 사용자 정보를 메모리에 저장합니다.
- **JdbcUserDetailsManager**: **JDBC**를 사용하여 데이터베이스에서 사용자 정보를 관리합니다.

#### 5. **Security Context와 Principal**
인증이 완료된 후에는 해당 사용자 정보를 요청이 처리되는 동안 사용할 수 있어야 합니다. 이를 위해 Spring Security는 **SecurityContextHolder**라는 객체를 사용합니다. **SecurityContextHolder**는 기본적으로 **ThreadLocal**에 사용자 정보를 저장하여 현재 요청의 모든 처리 과정에서 해당 정보에 접근할 수 있도록 합니다.

- **Principal**: 인증된 사용자(또는 시스템/서비스)를 나타내며, 사용자가 시스템 내에서 수행할 수 있는 작업을 결정하는 권한 정보를 포함합니다.
- **GrantedAuthority**: 사용자가 수행할 수 있는 작업(권한)을 나타내며, 역할(role)과 관련된 권한을 부여합니다.

### Spring Security 인증 흐름 요약
1. **사용자 요청**이 들어오면 **필터 체인(Filter Chain)**이 이를 가로채고, 적절한 **필터**를 통해 요청을 인증합니다. 예를 들어, **UsernamePasswordAuthenticationFilter**는 요청의 사용자명과 비밀번호를 기반으로 인증을 시도합니다.
2. 필터는 **AuthenticationManager**를 호출하여 인증을 처리합니다. 기본적으로 **ProviderManager**는 다양한 **AuthenticationProvider**를 사용하여 인증을 시도합니다.
3. 인증이 **성공**하면 인증된 사용자 정보를 **SecurityContextHolder**에 저장하여 요청이 처리되는 동안 **Security Context**를 통해 언제든지 해당 정보에 접근할 수 있게 합니다.

### Spring Security의 커스터마이징
기본 Spring Security 설정은 빠르고 간단하게 애플리케이션에 보안을 추가할 수 있도록 설계되었지만, 실제 애플리케이션에서는 더 세밀한 보안 요구 사항을 필요로 할 수 있습니다. 이러한 경우, 개발자는 보안 설정을 **커스터마이징**하여 사용자 정의 인증 및 권한 부여 흐름을 구성할 수 있습니다.

#### 사용자 정의 설정 예시
- **Form Login 커스터마이징**: 기본 로그인 페이지 대신 사용자가 직접 만든 로그인 페이지를 사용할 수 있습니다.
- **다중 사용자 지원**: 특정 URL에 대한 접근 권한을 사용자의 역할(Role)에 따라 제어할 수 있습니다.
- **비밀번호 인코딩**: **BCryptPasswordEncoder**와 같은 비밀번호 인코더를 사용하여 비밀번호를 안전하게 저장할 수 있습니다.

### 정리
Spring Boot Security는 간단한 의존성 추가만으로 강력한 보안 기능을 제공합니다. Spring Security의 기본 동작을 통해 애플리케이션을 쉽게 보호할 수 있지만, 복잡한 보안 요구 사항을 만족하기 위해서는 더 깊은 이해와 사용자 정의 설정이 필요합니다. 주요 구성 요소들을 이해하고, **필터 체인(Filter Chain)**, **AuthenticationManager**, **AuthenticationProvider**, 그리고 **UserDetailsService**의 역할과 상호작용을 잘 활용하면 더욱 안전하고 신뢰할 수 있는 애플리케이션을 구축할 수 있습니다.