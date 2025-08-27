# HMAC 簽章驗證機制

```mermaid

```

## 角色與構件

* Client：前端或外部呼叫者
* Gateway (Spring Cloud Gateway)
  * JwtGatewayFilter：驗證 JWT、查詢 Redis session、將使用者資訊放入 Header
  * HmacSignGlobalFilter：僅對 /portal/ 路徑進行 HMAC 簽章（新增 HMAC headers）
* BFF（admin-portal）
  * HmacVerificationFilter：僅對 /portal/ 驗證 HMAC（包含 body 雜湊、時間飄移、簽章驗證）
  * Controller：直接透過 request.getHeader("X-User-Account") 取得使用者帳號（必要時也可帶到 Feign）
* Auth-Service：業務服務（此段不實作 HMAC）
* Redis：存放「登入後的使用者會話」

## 設計原則

* 使用者身分由 Gateway 透過 Header (X-User-Account) 傳遞；前端不傳遞使用者身分。
* HMAC 只保障 Gateway→BFF 的請求完整性與不可否認性。
* BFF→Auth-Service 可選擇透傳 X-User-Account 供審計或多租戶分流（非強制）。

## 請求生命週期

### 1. 登入 /auth/login

* Client 呼叫 Gateway：POST /auth/login（白名單，不經過 JWT/HMAC）。
* Auth-Service 驗證帳號密碼 → 產生 JWT（內含 userAccount claim）。
* 在 Redis 設定 session:{userAccount} → 序列化的 UserSession（包含 permissions 等）。
* JWT 回到 Client 保存。

### 2. 呼叫受保護的 BFF API [/portal/**]

#### 2.1 Gateway：JWT & Session 檢查

JwtGatewayFilter：

* 解析 Authorization: Bearer[JWT]，取得 userAccount。
* Redis 檢查 session:{userAccount} 是否存在／未過期。
* 將下列 Header 加入／覆蓋到往下游的請求：
  * X-User-Account: {userAccount}
  * X-User-Permissions: permA,permB,...（可選）

#### 2.2 Gateway：HMAC 簽章（僅 /portal/**）

HmacSignGlobalFilter 讀取請求 body（可為空），計算 SHA-256。

建立規範化字串（兩端需一致）：
> {METHOD}\n{PATH?QUERY}\n{TIMESTAMP}\n{NONCE}\n{BODY_SHA256}

以共享密鑰 HMAC-SHA256 → Base64，新增下列 Header：

* X-Client-Id
* X-Timestamp（毫秒）
* X-Nonce（隨機字串）
* X-Content-SHA256（Hex）
* X-Signature（Base64）

連同 X-User-Account 一併轉發至 BFF。

#### 2.3 BFF：HMAC 驗章（僅 /portal/**）

HmacVerificationFilter：

* 讀取並重新計算 SHA-256；比對 X-Content-SHA256。
* 重建規範化字串；使用同一密鑰重新計算簽章，比對 X-Signature。

驗章通過才交由 Controller 處理。

#### 2.4 BFF：Controller 讀取 Header + 業務流程

Controller 直接讀取 Header：
依照既有分層：Controller → UseCase → UseCaseImpl → FeignClient（呼叫 Auth-Service）。

## 程式碼實作

### Gateway HMAC 簽章過濾器

```java
package com.casha.gateway.filter;

import com.casha.common.bff.hmac.HmacSigner;
import com.casha.gateway.config.GatewayHmacProperties;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.core.io.buffer.DataBufferFactory;
import org.springframework.core.io.buffer.DataBufferUtils;
import org.springframework.http.HttpHeaders;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpRequestDecorator;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.Instant;
import java.util.UUID;

@Component
@RequiredArgsConstructor
@Slf4j
@EnableConfigurationProperties(GatewayHmacProperties.class)
public class HmacSignGlobalFilter implements GlobalFilter, Ordered {

    private final GatewayHmacProperties props;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, org.springframework.cloud.gateway.filter.GatewayFilterChain chain) {
        if (!props.isEnabled()) return chain.filter(exchange);

        ServerHttpRequest req = exchange.getRequest();
        String path = req.getURI().getRawPath();
        if (!StringUtils.hasText(path) || !path.startsWith(props.getSignPathPrefix())) {
            return chain.filter(exchange); // 只簽 /portal/**
        }

        return DataBufferUtils.join(req.getBody())
                .defaultIfEmpty(exchange.getResponse().bufferFactory().wrap(new byte[0]))
                .flatMap(buf -> {
                    byte[] bodyBytes = new byte[buf.readableByteCount()];
                    buf.read(bodyBytes);
                    DataBufferUtils.release(buf);

                    String ts = String.valueOf(Instant.now().toEpochMilli());
                    String nonce = UUID.randomUUID().toString().replace("-", "");
                    String bodySha = HmacSigner.sha256Hex(bodyBytes);
                    String method = (req.getMethod() != null) ? req.getMethod().name() : "GET";
                    String pathWithQuery = req.getURI().getRawPath()
                            + (req.getURI().getRawQuery() == null ? "" : "?" + req.getURI().getRawQuery());
                    String canonical = HmacSigner.canonical(method, pathWithQuery, ts, nonce, bodySha);
                    String sig = HmacSigner.hmacBase64(props.getSecret(), canonical);

                    HttpHeaders newHeaders = new HttpHeaders();
                    newHeaders.putAll(req.getHeaders());
                    newHeaders.set("X-Client-Id", props.getClientId());
                    newHeaders.set("X-Timestamp", ts);
                    newHeaders.set("X-Nonce", nonce);
                    newHeaders.set("X-Content-SHA256", bodySha);
                    newHeaders.set("X-Signature", sig);

                    ServerHttpRequest decorated = new ServerHttpRequestDecorator(req) {
                        @Override
                        public HttpHeaders getHeaders() {
                            HttpHeaders h = new HttpHeaders();
                            h.putAll(newHeaders);
                            if (bodyBytes.length > 0) {
                                h.setContentLength(bodyBytes.length);
                            }
                            return h;
                        }

                        @Override
                        public Flux<DataBuffer> getBody() {
                            DataBufferFactory factory = exchange.getResponse().bufferFactory();
                            return Flux.just(factory.wrap(bodyBytes));
                        }
                    };

                    return chain.filter(exchange.mutate().request(decorated).build());
                });
    }

    /**
     * 放在 JwtGatewayFilter(-1) 之後
     */
    @Override
    public int getOrder() {
        return 0;
    }
}
```

### BFF HMAC 驗證過濾器

```java
package com.casha.common.bff.hmac;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletRequestWrapper;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 5)
@RequiredArgsConstructor
public class HmacVerificationFilter implements Filter {

    private final HmacConfig props;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) request;
        HttpServletResponse httpRes = (HttpServletResponse) response;

        if (!props.isEnabled() || !httpReq.getRequestURI().startsWith("/portal/")) {
            chain.doFilter(request, response);
            return;
        }

        CachedRequestWrapper wrapped = new CachedRequestWrapper(httpReq);

        String ts = wrapped.getHeader("X-Timestamp");
        String nonce = wrapped.getHeader("X-Nonce");
        String bodySha = wrapped.getHeader("X-Content-SHA256");
        String sig = wrapped.getHeader("X-Signature");
        String clientId = wrapped.getHeader("X-Client-Id");

        if (ts == null || nonce == null || bodySha == null || sig == null || clientId == null) {
            httpRes.sendError(HttpServletResponse.SC_UNAUTHORIZED, "HMAC headers missing");
            return;
        }

        // 驗證 body SHA
        String realBodySha = HmacSigner.sha256Hex(wrapped.getBody());
        if (!realBodySha.equals(bodySha)) {
            httpRes.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Body SHA mismatch");
            return;
        }

        String pathWithQuery = httpReq.getRequestURI() + (httpReq.getQueryString() == null ? "" : "?" + httpReq.getQueryString());
        String canonical = HmacSigner.canonical(httpReq.getMethod(), pathWithQuery, ts, nonce, bodySha);
        String expected = HmacSigner.hmacBase64(props.getSecret(), canonical);
        if (!expected.equals(sig)) {
            httpRes.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid signature");
            return;
        }

        chain.doFilter(wrapped, response);
    }

    /**
     * 可重複讀取 Body
     */
    static class CachedRequestWrapper extends HttpServletRequestWrapper {
        private final byte[] body;

        CachedRequestWrapper(HttpServletRequest request) throws IOException {
            super(request);
            try (InputStream is = request.getInputStream()) {
                this.body = is.readAllBytes();
            }
        }

        byte[] getBody() {
            return body;
        }

        @Override
        public ServletInputStream getInputStream() {
            ByteArrayInputStream bais = new ByteArrayInputStream(body);
            return new ServletInputStream() {
                @Override
                public int read() {
                    return bais.read();
                }

                @Override
                public boolean isFinished() {
                    return bais.available() == 0;
                }

                @Override
                public boolean isReady() {
                    return true;
                }

                @Override
                public void setReadListener(ReadListener readListener) {
                }
            };
        }
    }

    @Component
    @ConfigurationProperties(prefix = "casha.hmac")
    public static class HmacConfig {
        private boolean enabled = true;
        private String secret;

        public boolean isEnabled() {
            return enabled;
        }

        public void setEnabled(boolean enabled) {
            this.enabled = enabled;
        }

        public String getSecret() {
            return secret;
        }

        public void setSecret(String secret) {
            this.secret = secret;
        }
    }
}
```

### HMAC 簽章工具類

```java
package com.casha.common.bff.hmac;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.Base64;

public final class HmacSigner {

    private HmacSigner() {
    }

    public static String sha256Hex(byte[] body) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] d = md.digest(body == null ? new byte[0] : body);
            StringBuilder sb = new StringBuilder(d.length * 2);
            for (byte b : d) sb.append(String.format("%02x", b));
            return sb.toString();
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }

    public static String canonical(String method, String pathWithQuery, String ts, String nonce, String bodySha256) {
        return String.join("\n", method.toUpperCase(), pathWithQuery, ts, nonce, bodySha256 == null ? "" : bodySha256);
    }

    public static String hmacBase64(String secret, String canonical) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
            return Base64.getEncoder().encodeToString(mac.doFinal(canonical.getBytes(StandardCharsets.UTF_8)));
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }
}
```