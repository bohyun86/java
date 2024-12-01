# Spring Security를 사용하여 REST API POST 요청으로 로그인 설정 가이드

이 가이드에서는 Spring Security를 활용하여 로그인 폼이 아닌 REST API POST 요청을 통해 인증을 구현하는 방법을 단계별로 설명합니다. 제공된 코드를 기반으로 차근차근 따라오시면 됩니다.

## 목차

1. [프로젝트 설정](#1-프로젝트-설정)
2. [의존성 추가](#2-의존성-추가)
3. [CustomAuthenticationFilter 생성](#3-customauthenticationfilter-생성)
4. [SecurityConfig 설정](#4-securityconfig-설정)
5. [CustomUserDetailsService 구현](#5-customuserdetailsservice-구현)
6. [SecurityUser 클래스 생성](#6-securityuser-클래스-생성)
7. [REST API 엔드포인트 설정](#7-rest-api-엔드포인트-설정)
8. [테스트](#8-테스트)
9. [정리](#9-정리)

---

## 1. 프로젝트 설정

Spring Boot를 사용하여 새로운 프로젝트를 생성합니다. 이 프로젝트는 REST API를 제공하며, Spring Security를 사용하여 인증을 처리합니다.

## 2. 의존성 추가

`build.gradle` 또는 `pom.xml` 파일에 필요한 의존성을 추가합니다.

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Spring Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Lombok (선택 사항) -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>

<!-- JPA 및 데이터베이스 관련 의존성 (필요에 따라 추가) -->
```

## 3. CustomAuthenticationFilter 생성

`CustomAuthenticationFilter`는 REST API로부터 받은 로그인 요청을 처리하는 필터입니다.

```java
package com.example.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@Slf4j
public class CustomAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    private final AuthenticationManager authenticationManager;

    public CustomAuthenticationFilter(AuthenticationManager authenticationManager) {
        super("/user/loginPro"); // 인증 요청을 처리할 경로 설정
        this.authenticationManager = authenticationManager;
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException, IOException {
        log.info("인증 시도 중...");

        if (request.getInputStream().available() == 0) {
            log.error("요청 본문이 비어 있습니다.");
            throw new IllegalArgumentException("요청 본문이 비어 있습니다.");
        }

        ObjectMapper objectMapper = new ObjectMapper();
        Map<String, String> authRequest = objectMapper.readValue(request.getInputStream(), HashMap.class);

        String username = authRequest.get("name");
        String password = authRequest.get("password");

        log.info("사용자명: {}, 비밀번호: {}", username, password);

        if (username == null || password == null) {
            throw new IllegalArgumentException("사용자명 또는 비밀번호가 없습니다.");
        }

        UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(username, password);
        return authenticationManager.authenticate(authToken);
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                            FilterChain chain, Authentication authResult) throws IOException, ServletException {
        // 인증 정보를 SecurityContext에 저장
        SecurityContextHolder.getContext().setAuthentication(authResult);

        // 세션에 인증 정보 저장
        request.getSession(true).setAttribute("SPRING_SECURITY_CONTEXT", SecurityContextHolder.getContext());

        log.info("인증된 사용자 권한: {}", authResult.getAuthorities());
        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().write("인증 성공!");
    }

    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                              AuthenticationException failed) throws IOException, ServletException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.getWriter().write("인증 실패: " + failed.getMessage());
    }
}
```

### 설명

- **`attemptAuthentication`**: 클라이언트로부터 받은 로그인 요청을 처리합니다. JSON 형태의 사용자명과 비밀번호를 읽어들여 인증을 시도합니다.
- **`successfulAuthentication`**: 인증 성공 시 호출되며, 인증 정보를 세션과 `SecurityContextHolder`에 저장합니다.
- **`unsuccessfulAuthentication`**: 인증 실패 시 호출되어 적절한 응답을 반환합니다.

## 4. SecurityConfig 설정

Spring Security의 설정을 담당하는 `SecurityConfig` 클래스를 생성합니다.

```java
package com.example.config;

import org.springframework.boot.autoconfigure.security.servlet.PathRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityCustomizer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class SecurityConfig {

    private final CustomUserDetailsService customUserDetailsService;

    public SecurityConfig(CustomUserDetailsService customUserDetailsService) {
        this.customUserDetailsService = customUserDetailsService;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        // 기본 패스워드 인코더 설정
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }

    @Bean
    public WebSecurityCustomizer webSecurityCustomizer() {
        // 정적 리소스에 대한 보안 설정 무시
        return web -> web.ignoring().requestMatchers(PathRequest.toStaticResources().atCommonLocations());
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http, AuthenticationConfiguration authConfig) throws Exception {
        AuthenticationManager authenticationManager = authConfig.getAuthenticationManager();

        http
                .authorizeHttpRequests(matcherRegistry -> matcherRegistry
                        .requestMatchers("/user/register", "/user/index").permitAll()
                        .requestMatchers("/user/userList").hasRole("ADMIN")
                        .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
                        .anyRequest().permitAll()
                )
                .userDetailsService(customUserDetailsService) // 사용자 정보 서비스를 설정
                .addFilterBefore(new CustomAuthenticationFilter(authenticationManager), UsernamePasswordAuthenticationFilter.class);

        http.csrf().disable(); // CSRF 보호 비활성화 (필요에 따라 조정)

        // 권한 없는 페이지 접근 시 처리
        http.exceptionHandling()
                .accessDeniedHandler((request, response, accessDeniedException) -> {
                    response.sendRedirect("/");
                });

        // 세션 관리 설정 (필요에 따라 조정)
        http.sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED);

        return http.build();
    }
}
```

### 설명

- **`PasswordEncoder`**: 비밀번호를 암호화하기 위한 인코더를 설정합니다.
- **`WebSecurityCustomizer`**: 정적 리소스에 대한 보안 설정을 무시합니다.
- **`SecurityFilterChain`**: HTTP 보안 설정을 정의합니다.
  - 인증이 필요한 경로와 권한을 설정합니다.
  - `CustomAuthenticationFilter`를 `UsernamePasswordAuthenticationFilter` 이전에 추가합니다.
  - CSRF 설정, 예외 처리, 세션 관리 등을 설정합니다.

## 5. CustomUserDetailsService 구현

사용자 정보를 로드하는 `UserDetailsService`를 구현합니다.

```java
package com.example.config;

import com.example.dto.Users;
import com.example.repository.UsersRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.util.Optional;

@Service
@RequiredArgsConstructor
@Slf4j
public class CustomUserDetailsService implements UserDetailsService {

    private final UsersRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String name) throws UsernameNotFoundException {
        Optional<Users> user = userRepository.findByName(name);

        if (user.isEmpty()) {
            throw new UsernameNotFoundException("사용자를 찾을 수 없습니다: " + name);
        }

        log.info("사용자 정보: {}", user.get());

        return new SecurityUser(user.get());
    }
}
```

### 설명

- 데이터베이스 또는 저장소에서 사용자 정보를 로드합니다.
- 사용자명이 존재하지 않으면 `UsernameNotFoundException`을 발생시킵니다.
- 사용자 정보를 `SecurityUser` 객체로 반환합니다.

## 6. SecurityUser 클래스 생성

`UserDetails`를 구현하는 `SecurityUser` 클래스를 생성하여 사용자 인증 정보를 제공합니다.

```java
package com.example.config;

import com.example.dto.Authority;
import com.example.dto.Users;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.stream.Collectors;

@RequiredArgsConstructor
public class SecurityUser implements UserDetails {

    private final Users user;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getAuthorities().stream()
                .map(Authority::getAuthority)
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getName();
    }

    public String getEmail() {
        return user.getEmail();
    }

    // 계정 상태에 대한 추가 메서드 구현
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

### 설명

- 사용자 권한, 비밀번호, 사용자명 등을 제공하는 `UserDetails` 구현체입니다.
- 계정의 상태 (잠김, 만료 등)를 결정하는 메서드를 제공합니다.

## 7. REST API 엔드포인트 설정

컨트롤러를 생성하여 REST API 엔드포인트를 설정합니다.

```java
package com.example.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @GetMapping("/user/main")
    public String userMain() {
        return "사용자 메인 페이지";
    }

    @GetMapping("/user/userList")
    public String userList() {
        return "사용자 목록 (관리자 전용)";
    }
}
```

### 설명

- `/user/main`: 인증된 사용자들이 접근할 수 있는 엔드포인트입니다.
- `/user/userList`: 관리자 권한을 가진 사용자만 접근할 수 있는 엔드포인트입니다.

## 8. 테스트

### 8.1 로그인 요청

프론트엔드 또는 REST 클라이언트를 사용하여 로그인 요청을 보냅니다.

```javascript
// 예시 JavaScript 코드
const requestBody = {
    name: "사용자명",
    password: "비밀번호"
};

fetch('/user/loginPro', {
    method: 'POST',
    body: JSON.stringify(requestBody),
    headers: {
        'Content-Type': 'application/json'
    }
}).then(response => {
    if (response.ok) {
        return response.text();
    } else {
        throw new Error("인증 실패");
    }
}).then(data => {
    console.log("로그인 성공:", data);
    // 인증 후 필요한 작업 수행
}).catch(error => {
    console.error("로그인 실패:", error.message);
});
```

### 8.2 인증된 요청

로그인 후 세션이 유지되므로, 인증이 필요한 엔드포인트에 접근할 수 있습니다.

```javascript
fetch('/user/main', {
    method: 'GET',
    credentials: 'include' // 세션 쿠키를 포함하여 요청
}).then(response => {
    if (response.ok) {
        return response.text();
    } else {
        throw new Error("접근 권한 없음");
    }
}).then(data => {
    console.log("응답:", data);
}).catch(error => {
    console.error("오류:", error.message);
});
```

### 8.3 관리자 권한 요청

관리자 권한이 필요한 엔드포인트에 접근하려면 해당 권한을 가진 계정으로 로그인해야 합니다.

```javascript
fetch('/user/userList', {
    method: 'GET',
    credentials: 'include'
}).then(response => {
    if (response.ok) {
        return response.text();
    } else {
        throw new Error("접근 권한 없음");
    }
}).then(data => {
    console.log("응답:", data);
}).catch(error => {
    console.error("오류:", error.message);
});
```

## 9. 정리

이 가이드에서는 Spring Security를 사용하여 REST API를 통한 로그인 인증을 구현하는 방법을 설명하였습니다.

- `CustomAuthenticationFilter`를 통해 로그인 요청을 처리하고 인증합니다.
- `SecurityConfig`에서 필요한 보안 설정과 필터를 등록합니다.
- `CustomUserDetailsService`와 `SecurityUser`를 통해 사용자 정보를 로드하고 인증에 사용합니다.
- 세션을 사용하여 인증 상태를 유지하며, 이후의 요청에서 인증 정보를 활용합니다.

이러한 구성을 통해 로그인 폼이 아닌 REST API 요청으로도 Spring Security의 인증과 권한 부여 기능을 활용할 수 있습니다.

---

**참고사항**

- 실제 구현 시에는 예외 처리, 입력 검증, 보안 설정 등을 더욱 세밀하게 구성해야 합니다.
- CSRF 보호를 비활성화하였으므로, 필요한 경우 CSRF 토큰을 활용하는 방법을 고려해야 합니다.
- 패스워드 인코딩 및 저장 시 적절한 보안 알고리즘을 사용해야 합니다.
- HTTPS를 사용하여 통신 보안을 강화하는 것이 좋습니다.