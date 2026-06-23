## Bài 5 — Thiết kế quy trình & Prompt cho Rate Limiter (SafePay)

I. Ý đồ thiết kế quy trình 3 bước
Quy trình 3 bước được thiết kế dựa trên nguyên tắc "Top-Down Approach" (Đi từ tổng quan đến chi tiết) trong thiết kế phần mềm:

Bước 1 (Định hình tư duy thuật toán): Không vội vàng viết code ngay. Việc yêu cầu AI phân tích các thuật toán giúp ta chọn được chiến lược phù hợp nhất với đặc thù nghiệp vụ tài chính (cần độ chính xác cao hay cần hiệu năng mượt mà).

Bước 2 (Giải quyết bài toán mở rộng - Scalability): Đưa AI vào một bối cảnh thực tế của hệ thống Microservices/Distributed. Việc bắt lỗi bộ nhớ cục bộ (local cache) giúp AI đề xuất đúng công nghệ lưu trữ tập trung (Redis), làm tiền đề vững chắc cho bước code.

Bước 3 (Triển khai & Quản trị rủi ro): Yêu cầu AI áp dụng kỹ thuật CoT (Chain of Thought) để viết code Spring Boot không chỉ chạy được, mà còn phải chịu lỗi tốt (Fault-tolerant) bằng cơ chế Fallback khi Redis sập, đảm bảo tính sẵn sàng (Availability) cho ứng dụng "SafePay".

II. Danh sách 3 Prompt sử dụng
Prompt 1 (Bước 1):

Đóng vai trò là System Architect. Tôi đang xây dựng API Gateway cho ứng dụng tài chính "SafePay". Cần thiết lập tính năng Rate Limiting với quy tắc: Mỗi API Key tối đa 100 requests/1 phút, vượt quá trả về HTTP 429. Hãy tư vấn ít nhất 2 thuật toán phổ biến (ví dụ: Token Bucket và Fixed Window) và phân tích/so sánh ưu, nhược điểm của chúng khi triển khai trên nền tảng Java. Cuối cùng, hãy gợi ý thuật toán nào phù hợp nhất cho SafePay.

Prompt 2 (Bước 2):

Giả định hệ thống SafePay hiện tại đang được scale ra chạy trên một cụm 3 máy chủ (Distributed System) có load balancer đứng trước. Việc lưu trữ biến đếm rate limit trong bộ nhớ cục bộ (Local memory) của từng máy chủ sẽ dẫn đến sai lệch dữ liệu. Hãy phân tích ngắn gọn tại sao lại sai lệch, và đề xuất chi tiết giải pháp sử dụng Redis làm kho lưu trữ tập trung để giải quyết vấn đề này.

Prompt 3 (Bước 3):

Đóng vai trò là Senior Java Developer. Hãy áp dụng tư duy từng bước (Chain of Thought) để viết mã nguồn Java Spring Boot triển khai Rate Limiter phân tán (sử dụng thuật toán Fixed Window) với Spring Data Redis. Yêu cầu bắt buộc:

Xử lý logic 100 requests / 1 phút / API Key.

Có cơ chế Fallback bảo vệ ứng dụng: Nếu Redis đột ngột sập (Timeout/Connection Refused), hệ thống sẽ xử lý thế nào để không làm gián đoạn toàn bộ giao dịch của SafePay?

Sử dụng @Slf4j để ghi log các tình huống (block request, lỗi Redis, kích hoạt fallback).

III. Minh chứng thực tế (Log Chat)
Dưới đây là log chat chi tiết phản hồi của AI cho từng Prompt.

💬 Lượt Chat 1: Phân tích và lựa chọn thuật toán
AI Assistant:
Chào bạn, với tư cách là System Architect cho hệ thống tài chính "SafePay", tôi hiểu rằng tính chính xác, hiệu năng và bảo mật là ưu tiên hàng đầu. Dưới đây là phân tích 2 thuật toán Rate Limiting phổ biến nhất và so sánh ưu/nhược điểm khi triển khai trên Java:

1. Thuật toán Fixed Window Counters (Cửa sổ cố định)

Cơ chế: Chia thời gian thành các "cửa sổ" cố định (ví dụ: 00:00 - 00:01, 00:01 - 00:02). Mỗi API Key có một biến đếm trong cửa sổ đó. Nếu request đến, tăng biến đếm. Nếu biến đếm > 100, chặn request (trả về 429) cho đến khi chuyển sang cửa sổ thời gian tiếp theo.

Ưu điểm (Trong Java): Cực kỳ dễ cài đặt bằng các cấu trúc dữ liệu đơn giản như ConcurrentHashMap (nếu chạy local) hoặc Redis INCR kèm TTL. Tốn rất ít bộ nhớ.

Nhược điểm: Mắc lỗi "Giao thoa mép cửa sổ" (Boundary/Spike problem). Ví dụ: User gửi 100 req ở giây 0:59 và 100 req ở giây 1:01. Hệ thống vẫn chấp nhận, nhưng thực tế trong 2 giây đó có tới 200 req tràn vào hệ thống, có thể gây quá tải.

2. Thuật toán Token Bucket (Xô mộc bài)

Cơ chế: Cấp cho mỗi API Key một "cái xô" chứa tối đa 100 tokens. Hệ thống tự động thêm 100 tokens vào xô mỗi phút (hoặc ~1.66 token/giây). Mỗi request tốn 1 token. Nếu xô rỗng, request bị từ chối.

Ưu điểm (Trong Java): Xử lý luồng request rất mượt mà. Cho phép "Burst traffic" (tức là user có thể dùng một lúc 100 req ngay lập tức nếu xô đầy). Có các thư viện Java hỗ trợ cực tốt như Bucket4j hoặc Resilience4j.

Nhược điểm: Logic cập nhật số lượng token phức tạp hơn (cần tính toán thời gian chênh lệch để refill token). Trong môi trường phân tán, quản lý trạng thái token đòi hỏi cơ chế lock chặt chẽ hơn.

💡 Khuyến nghị cho SafePay:
Đối với một API Gateway tài chính mới xây dựng, tôi khuyên dùng Fixed Window (nhưng nâng cấp lên một chút thành Sliding Window Log nếu có nguồn lực) hoặc dùng Fixed Window cơ bản với Redis cho giai đoạn đầu. Lý do: Tính đơn giản, dễ debug, và kiểm soát chính xác lượng request trong một phút. Lỗi mép cửa sổ của Fixed Window có thể chấp nhận được nếu Backend có hệ thống Auto-scaling tốt.

💬 Lượt Chat 2: Kịch bản giả định phân tán với Redis
AI Assistant:
Chào bạn, câu hỏi "What-if" của bạn chạm đúng vào vấn đề cốt lõi của các hệ thống Microservices.

1. Tại sao Local Memory lại gây sai lệch?

Vấn đề Inconsistency (Không nhất quán): Khi có 3 máy chủ (A, B, C) đứng sau Load Balancer. User gửi request thứ 1 -> Load balancer đẩy vào Server A (Server A đếm: 1). Request thứ 2 -> đẩy vào Server B (Server B đếm: 1).

Hậu quả: Thay vì giới hạn user ở 100 request tổng cộng, thực tế user có thể gửi tới 300 requests (mỗi server chịu 100 req) vì các server không chia sẻ bộ nhớ cho nhau. Điều này phá vỡ hoàn toàn quy tắc Rate Limiting.

2. Giải pháp: Sử dụng Redis làm kho lưu trữ tập trung (Centralized Data Store)
Để giải quyết, ta đưa state (biến đếm) ra khỏi các máy chủ API Gateway và lưu vào Redis - một In-Memory Database cực nhanh.

Kiến trúc: Cả Server A, B và C đều gọi vào cùng một cluster Redis để kiểm tra và tăng biến đếm.

Cơ chế hoạt động (Dựa trên Fixed Window):

Key Design: Tạo key trong Redis theo định dạng: rate_limit:{api_key}:{minute_timestamp}. Ví dụ: rate_limit:abc123:202606230910.

Atomic Operation: Các Server dùng lệnh INCR của Redis. Lệnh này đảm bảo tính nguyên tử (Atomic), dù 3 server cùng gọi một lúc cũng không bị Race Condition.

TTL (Time-To-Live): Đặt EXPIRE cho key là 60 giây. Sau 1 phút, Redis tự xóa key cũ, giải phóng bộ nhớ.

Sử dụng Redis giúp hệ thống SafePay duy trì "Single Source of Truth" (Nguồn chân lý duy nhất) cho biến đếm rate limit, bất kể scale lên bao nhiêu instance.

💬 Lượt Chat 3: Code Generation với tư duy CoT (Chain of Thought)
AI Assistant:
Chào bạn, tôi là Senior Java Developer. Để giải quyết bài toán của SafePay một cách an toàn và tối ưu nhất, tôi sẽ áp dụng tư duy từng bước (Chain of Thought).

🧠 Tư duy từng bước (Chain of Thought):

Bước 1 - Lựa chọn công nghệ: Tôi sẽ dùng Spring Boot, spring-boot-starter-data-redis.

Bước 2 - Logic Redis: Sử dụng Redis INCR để tăng đếm và EXPIRE để set thời gian hết hạn (60s). Thay vì gửi 2 lệnh riêng biệt rủi ro mạng, tôi gộp chúng bằng Redis Lua Script để đảm bảo tính nguyên tử tuyệt đối.

Bước 3 - Cơ chế Fallback (Bảo vệ khi Redis sập): SafePay là app tài chính, ưu tiên là không được làm sập cả hệ thống thanh toán chỉ vì tính năng phụ (Rate limiter) bị lỗi.

Chiến lược: Fail-Open kết hợp Local Fallback. Khi Redis sập (ném ra RedisConnectionFailureException), hệ thống sẽ catch lỗi này, ghi log mức ERROR để báo động (Alerting), và tạm thời cho phép request đi qua (Fail-open) hoặc chuyển sang dùng ConcurrentHashMap cục bộ làm giải pháp tạm thời trong 1 phút chờ Redis phục hồi. Ở đây, tôi sẽ code cơ chế Local In-Memory Fallback đơn giản để chặn lạm dụng ngay cả khi Redis chết.

Bước 4 - Triển khai mã nguồn:

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.RedisConnectionFailureException;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.Collections;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

@Slf4j
@Service
public class RateLimiterService {

    private final StringRedisTemplate redisTemplate;
    private final DefaultRedisScript<Long> rateLimitScript;
    
    // Local Fallback Cache (Cho trường hợp Redis sập)
    private final Map<String, AtomicInteger> localFallbackCache = new ConcurrentHashMap<>();
    private long currentFallbackMinute = Instant.now().getEpochSecond() / 60;

    private static final int MAX_REQUESTS_PER_MINUTE = 100;

    @Autowired
    public RateLimiterService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
        
        // Lua script: Tăng biến đếm, nếu là lần đầu tiên (count == 1) thì set TTL 60 giây
        String luaCode = 
                "local current = redis.call('INCR', KEYS[1]) " +
                "if tonumber(current) == 1 then " +
                "    redis.call('EXPIRE', KEYS[1], 60) " +
                "end " +
                "return current;";
        this.rateLimitScript = new DefaultRedisScript<>(luaCode, Long.class);
    }

    /**
     * Kiểm tra API Key có được phép truy cập không
     * @return true nếu hợp lệ, false nếu vượt quá 100 requests (trả về 429)
     */
    public boolean isAllowed(String apiKey) {
        long currentMinute = Instant.now().getEpochSecond() / 60;
        String redisKey = "rate_limit:" + apiKey + ":" + currentMinute;

        try {
            // Thực thi Lua script trên Redis
            Long requestCount = redisTemplate.execute(
                    rateLimitScript, 
                    Collections.singletonList(redisKey)
            );

            if (requestCount != null && requestCount > MAX_REQUESTS_PER_MINUTE) {
                log.warn("RATE LIMIT EXCEEDED: API Key [{}] reached {} requests.", apiKey, requestCount);
                return false;
            }
            
            return true;

        } catch (RedisConnectionFailureException | org.springframework.data.redis.RedisSystemException e) {
            // FALLBACK MECHANISM: Kích hoạt khi Redis sập
            log.error("REDIS DOWN! Kích hoạt cơ chế Fallback Local Memory cho Rate Limiter. Chi tiết lỗi: {}", e.getMessage());
            return fallbackCheck(apiKey, currentMinute);
        } catch (Exception e) {
            // Lỗi không xác định, Fail-open để không làm gián đoạn SafePay
            log.error("Lỗi không xác định tại Rate Limiter, Fail-open áp dụng. Lỗi: {}", e.getMessage());
            return true; 
        }
    }

    /**
     * Logic Local Fallback Memory
     */
    private boolean fallbackCheck(String apiKey, long currentMinute) {
        // Reset cache cục bộ nếu bước sang phút mới
        if (currentMinute > currentFallbackMinute) {
            localFallbackCache.clear();
            currentFallbackMinute = currentMinute;
        }

        String localKey = apiKey + ":" + currentMinute;
        localFallbackCache.putIfAbsent(localKey, new AtomicInteger(0));
        int count = localFallbackCache.get(localKey).incrementAndGet();

        if (count > MAX_REQUESTS_PER_MINUTE) {
            log.warn("FALLBACK RATE LIMIT EXCEEDED: API Key [{}] blocked by Local Cache.", apiKey);
            return false;
        }
        return true;
    }
}
```