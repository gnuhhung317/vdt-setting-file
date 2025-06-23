# Giải pháp Authentication & Authorization cho API Service

## 1. Yêu cầu
- Một số URL của API service phải xác thực qua cookie, basic auth, hoặc token, nếu không sẽ trả về HTTP 403.
- Phân quyền 2 loại người dùng:
  - **user**: chỉ được GET (200), POST/DELETE trả về 403.
  - **admin**: GET (200), POST/DELETE (2xx).

## 2. Giải pháp sử dụng
- Sử dụng **Spring Security** để cấu hình xác thực và phân quyền.
- Demo sử dụng **Basic Auth** với user/password lưu trong bộ nhớ (InMemoryUserDetailsManager).
- Có thể mở rộng sang cookie hoặc token (JWT) nếu cần.

### Cấu hình chính trong `SecurityConfig.java`
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(authz -> authz
            .requestMatchers("/api/todos/**").authenticated()
            .requestMatchers(HttpMethod.GET, "/api/todos/**").hasAnyRole("USER", "ADMIN")
            .requestMatchers(HttpMethod.POST, "/api/todos/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.PUT, "/api/todos/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.DELETE, "/api/todos/**").hasRole("ADMIN")
            .anyRequest().permitAll())
        .httpBasic()
        .exceptionHandling(e -> e
            .accessDeniedHandler((request, response, accessDeniedException) -> response.sendError(403))
            .authenticationEntryPoint((request, response, authException) -> response.sendError(403))
        )
        .formLogin().disable();
    return http.build();
}

@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.withDefaultPasswordEncoder()
        .username("user")
        .password("user123")
        .roles("USER")
        .build();
    UserDetails admin = User.withDefaultPasswordEncoder()
        .username("admin")
        .password("admin123")
        .roles("ADMIN")
        .build();
    return new InMemoryUserDetailsManager(user, admin);
}
```
