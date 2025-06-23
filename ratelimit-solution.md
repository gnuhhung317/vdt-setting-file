# Giải pháp Rate Limit cho API Service

## 1. Yêu cầu
- Giới hạn số lượng request đến endpoint của API service: nếu quá 10 request trong 1 phút thì các request sau trả về HTTP 409.

## 2. Giải pháp sử dụng
- Sử dụng **Servlet Filter** (`RateLimitFilter.java`) để kiểm soát số lượng request theo IP.
- Lưu trữ số lượng request mỗi IP trong 1 phút bằng `ConcurrentHashMap`.
- Nếu quá giới hạn, trả về HTTP 409 với thông báo "Too many requests".

### Code mẫu:
```java
@Component
public class RateLimitFilter implements Filter {
    private static final int MAX_REQUESTS_PER_MINUTE = 10;
    private final Map<String, RequestCounter> requestCounts = new ConcurrentHashMap<>();

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String clientIp = request.getRemoteAddr();
        long currentMinute = Instant.now().getEpochSecond() / 60;

        requestCounts.compute(clientIp, (ip, counter) -> {
            if (counter == null || counter.minute != currentMinute) {
                return new RequestCounter(currentMinute, 1);
            } else {
                counter.count++;
                return counter;
            }
        });

        RequestCounter counter = requestCounts.get(clientIp);
        if (counter.count > MAX_REQUESTS_PER_MINUTE) {
            ((HttpServletResponse) response).setStatus(409);
            response.getWriter().write("Too many requests");
            return;
        }

        chain.doFilter(request, response);
    }

    private static class RequestCounter {
        long minute;
        int count;

        RequestCounter(long minute, int count) {
            this.minute = minute;
            this.count = count;
        }
    }
}
```
