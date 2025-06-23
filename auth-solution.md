# Giải pháp Authentication & Authorization cho API Service

## 1. Tổng quan
Để bảo vệ các API service, cần đảm bảo chỉ những người dùng hợp lệ mới được truy cập và mỗi người dùng chỉ được thực hiện các thao tác phù hợp với vai trò (role) của mình. Việc này giúp ngăn chặn truy cập trái phép, bảo vệ dữ liệu và tuân thủ các yêu cầu bảo mật.

## 2. Luồng xác thực và phân quyền
- Khi client gửi request đến các endpoint như `/api/todos/**`, hệ thống sẽ kiểm tra xem request có thông tin xác thực hợp lệ không (thông qua cookie, basic auth, hoặc token).
- Nếu không có hoặc thông tin xác thực sai, server trả về HTTP 403 Forbidden.
- Nếu xác thực thành công, hệ thống tiếp tục kiểm tra quyền (role) của user:
  - Nếu là `USER`, chỉ cho phép các request GET (đọc dữ liệu), các thao tác POST/PUT/DELETE sẽ bị từ chối (403).
  - Nếu là `ADMIN`, cho phép thực hiện mọi thao tác (GET, POST, PUT, DELETE).

## 3. Cách hoạt động của Spring Security trong giải pháp này
- **Spring Security** là framework bảo mật mạnh mẽ, tích hợp sẵn với Spring Boot, hỗ trợ nhiều phương thức xác thực (basic, form, token, OAuth2, ...).
- Trong cấu hình, ta sử dụng `SecurityFilterChain` để định nghĩa các rule:
  - `.requestMatchers("/api/todos/**").authenticated()`: Bắt buộc xác thực với mọi request đến `/api/todos/**`.
  - `.requestMatchers(HttpMethod.GET, "/api/todos/**").hasAnyRole("USER", "ADMIN")`: Chỉ cho phép user có role USER hoặc ADMIN thực hiện GET.
  - `.requestMatchers(HttpMethod.POST, "/api/todos/**").hasRole("ADMIN")`: Chỉ ADMIN được phép POST.
  - Tương tự cho PUT, DELETE.
- Nếu request không đủ quyền, Spring Security sẽ trả về 403.
- Nếu không xác thực, cũng trả về 403 nhờ cấu hình `.authenticationEntryPoint((request, response, authException) -> response.sendError(403))`.
- Demo sử dụng **Basic Auth** (user nhập username/password khi gọi API), thuận tiện cho kiểm thử và minh họa.

## 4. Lý do chọn giải pháp này
- **Đơn giản, dễ mở rộng:** Spring Security hỗ trợ nhiều phương thức xác thực, dễ dàng chuyển sang cookie, JWT, OAuth2 nếu cần.
- **Tích hợp sâu với Spring Boot:** Không cần viết nhiều code thủ công, chỉ cần cấu hình.
- **Quản lý role rõ ràng:** Có thể mở rộng thêm nhiều role, rule phức tạp hơn nếu hệ thống lớn lên.
- **Dễ kiểm thử:** Có thể dùng curl, Postman, hoặc các công cụ HTTP client khác để kiểm tra.

## 5. Mở rộng
- Nếu muốn xác thực qua **cookie** (session), chỉ cần bật lại `.formLogin()` trong cấu hình.
- Nếu muốn xác thực qua **token** (JWT), có thể tích hợp thêm filter để kiểm tra token trong header Authorization.
- Có thể kết nối với database hoặc hệ thống quản lý user thực tế thay vì user hardcode trong bộ nhớ.

## 6. Demo cấu hình chính trong `SecurityConfig.java`
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
