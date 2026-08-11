# 原生 webSocket 应用
## 配置

```JAVA
@Configuration
@EnableWebSocket
public class RawWebSocketConfig implements WebSocketConfigurer {

    @Autowired
    private ChatWebSocketHandler chatHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatHandler, "/ws/chat")
            .setAllowedOriginPatterns("*")
            .addInterceptors(new JwtHandshakeInterceptor());
    }
}
```

## Handler

```JAVA
@Component
@Slf4j
public class ChatWebSocketHandler extends TextWebSocketHandler {

    // 维护所有连接（线程安全）
    private final ConcurrentHashMap<String, WebSocketSession>
        sessions = new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(
            WebSocketSession session) throws Exception {
        String username = getUsername(session);
        sessions.put(session.getId(), session);
        log.info("连接建立: sessionId={}, user={}",
            session.getId(), username);

        // 发送欢迎消息
        session.sendMessage(new TextMessage(
            "{\"type\":\"WELCOME\",\"content\":\"连接成功\"}"));
    }

    @Override
    protected void handleTextMessage(WebSocketSession session,
            TextMessage message) throws Exception {
        String payload = message.getPayload();
        log.info("收到消息: {}", payload);

        // 广播给所有人
        broadcast(payload, session.getId());
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session,
            CloseStatus status) throws Exception {
        sessions.remove(session.getId());
        log.info("连接关闭: sessionId={}, status={}",
            session.getId(), status);
    }

    @Override
    public void handleTransportError(WebSocketSession session,
            Throwable exception) throws Exception {
        log.error("传输异常: sessionId={}",
            session.getId(), exception);
        sessions.remove(session.getId());
        session.close(CloseStatus.SERVER_ERROR);
    }

    // 广播给所有在线用户（除发送者）
    private void broadcast(String message,
            String excludeSessionId) {
        sessions.forEach((id, session) -> {
            if (!id.equals(excludeSessionId)
                    && session.isOpen()) {
                try {
                    session.sendMessage(new TextMessage(message));
                } catch (IOException e) {
                    log.error("发送失败: sessionId={}", id, e);
                }
            }
        });
    }

    // 发给特定用户
    public void sendToUser(String sessionId, String message) {
        WebSocketSession session = sessions.get(sessionId);
        if (session != null && session.isOpen()) {
            try {
                session.sendMessage(new TextMessage(message));
            } catch (IOException e) {
                log.error("发送失败", e);
            }
        }
    }

    private String getUsername(WebSocketSession session) {
        return (String) session.getAttributes()
            .getOrDefault("username", "anonymous");
    }
}
```

# WebSocket Client
Spring 提供了原生的 `WebSocketClient` 接口，直接用 Spring 生态。

## 依赖

```XML
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

## 两种实现

```
StandardWebSocketClient  → 基于 JSR-356（Java EE WebSocket 标准）
JettyWebSocketClient     → 基于 Jetty（需要额外依赖）

Spring Boot 默认用 StandardWebSocketClient，不需要额外引入
```
## 基础使用
```JAVA
@Component
@Slf4j
public class SpringWebSocketClient {

    private WebSocketSession session;

    public void connect(String url) throws Exception {
        WebSocketClient client = new StandardWebSocketClient();

        // 连接，传入 Handler 处理消息
        ListenableFuture<WebSocketSession> future =
            client.doHandshake(
                new MyWebSocketHandler(),
                url
            );

        // 阻塞等待连接完成
        this.session = future.get(10, TimeUnit.SECONDS);
        log.info("连接成功: {}", session.getId());
    }

    public void send(String message) throws IOException {
        if (session != null && session.isOpen()) {
            session.sendMessage(new TextMessage(message));
        }
    }

    public void close() throws IOException {
        if (session != null && session.isOpen()) {
            session.close();
        }
    }
}
```

## WebSocketHandler 实现

```JAVA
@Slf4j
public class MyWebSocketHandler
        extends TextWebSocketHandler {

    /**
     * 连接建立
     */
    @Override
    public void afterConnectionEstablished(
            WebSocketSession session) throws Exception {
        log.info("连接建立: sessionId={}, uri={}",
            session.getId(), session.getUri());
    }

    /**
     * 收到文本消息
     */
    @Override
    protected void handleTextMessage(
            WebSocketSession session,
            TextMessage message) throws Exception {
        String payload = message.getPayload();
        log.info("收到消息: {}", payload);
        // 业务处理
    }

    /**
     * 收到二进制消息
     */
    @Override
    protected void handleBinaryMessage(
            WebSocketSession session,
            BinaryMessage message) throws Exception {
        byte[] bytes = message.getPayload().array();
        log.info("收到二进制消息: {} bytes", bytes.length);
    }

    /**
     * 收到 Pong 帧（心跳响应）
     */
    @Override
    protected void handlePongMessage(
            WebSocketSession session,
            PongMessage message) throws Exception {
        log.debug("收到 Pong 心跳响应");
    }

    /**
     * 连接关闭
     */
    @Override
    public void afterConnectionClosed(
            WebSocketSession session,
            CloseStatus status) throws Exception {
        log.info("连接关闭: status={}", status);
    }

    /**
     * 传输异常
     */
    @Override
    public void handleTransportError(
            WebSocketSession session,
            Throwable exception) throws Exception {
        log.error("传输异常", exception);
        session.close(CloseStatus.SERVER_ERROR);
    }
}
```

## 带认证的连接

```JAVA
public void connectWithAuth(String url, String token)
        throws Exception {

    WebSocketClient client = new StandardWebSocketClient();

    // 设置请求头
    WebSocketHttpHeaders headers = new WebSocketHttpHeaders();
    headers.add("Authorization", "Bearer " + token);
    headers.add("X-Custom-Header", "value");

    // 带 header 连接
    this.session = client.doHandshake(
        new MyWebSocketHandler(),
        headers,
        URI.create(url)
    ).get(10, TimeUnit.SECONDS);
}
```

## 完整实现

```JAVA
@Component
@Slf4j
public class SpringWebSocketClient
        implements SmartLifecycle {

    @Value("${ws.server.url}")
    private String serverUrl;

    @Value("${ws.server.token}")
    private String token;

    private volatile WebSocketSession session;
    private volatile boolean running = false;
    private int reconnectDelay = 1000;
    private static final int MAX_DELAY = 30000;

    private final WebSocketClient wsClient =
        new StandardWebSocketClient();
    private final ObjectMapper objectMapper =
        new ObjectMapper();
    private ScheduledFuture<?> heartbeatTask;

    @Autowired
    private TaskScheduler taskScheduler;

    // ══ 连接管理 ════════════════════════════════

    @Override
    public void start() {
        running = true;
        doConnect();
    }

    private void doConnect() {
        if (!running) return;
        try {
            log.info("连接 WebSocket: {}", serverUrl);
            WebSocketHttpHeaders headers =
                new WebSocketHttpHeaders();
            headers.add("Authorization", "Bearer " + token);

            this.session = wsClient.doHandshake(
                new ClientHandler(),
                headers,
                URI.create(serverUrl)
            ).get(10, TimeUnit.SECONDS);

            reconnectDelay = 1000;  // 连接成功重置间隔
            startHeartbeat();
            log.info("连接成功");

        } catch (Exception e) {
            log.error("连接失败，{}ms 后重试", reconnectDelay, e);
            scheduleReconnect();
        }
    }

    private void scheduleReconnect() {
        if (!running) return;
        taskScheduler.schedule(
            this::doConnect,
            Instant.now().plusMillis(reconnectDelay));
        reconnectDelay = Math.min(reconnectDelay * 2, MAX_DELAY);
    }

    // ══ 心跳 ══════════════════════════════════

    private void startHeartbeat() {
        stopHeartbeat();
        heartbeatTask = taskScheduler.scheduleAtFixedRate(
            this::sendPing,
            Instant.now().plusSeconds(30),
            Duration.ofSeconds(30));
    }

    private void stopHeartbeat() {
        if (heartbeatTask != null) {
            heartbeatTask.cancel(false);
        }
    }

    private void sendPing() {
        if (session != null && session.isOpen()) {
            try {
                session.sendMessage(new PingMessage());
                log.debug("发送 Ping 心跳");
            } catch (IOException e) {
                log.warn("心跳发送失败", e);
            }
        }
    }

    // ══ 发送消息 ══════════════════════════════

    public boolean send(Object payload) {
        if (session == null || !session.isOpen()) {
            log.warn("未连接，发送失败");
            return false;
        }
        try {
            String json = objectMapper.writeValueAsString(payload);
            session.sendMessage(new TextMessage(json));
            return true;
        } catch (Exception e) {
            log.error("发送失败", e);
            return false;
        }
    }

    public boolean sendText(String text) {
        if (session == null || !session.isOpen()) return false;
        try {
            session.sendMessage(new TextMessage(text));
            return true;
        } catch (IOException e) {
            log.error("发送失败", e);
            return false;
        }
    }

    public boolean isConnected() {
        return session != null && session.isOpen();
    }

    // ══ 生命周期 ══════════════════════════════

    @Override
    public void stop() {
        running = false;
        stopHeartbeat();
        if (session != null && session.isOpen()) {
            try {
                session.close(CloseStatus.NORMAL);
            } catch (IOException e) {
                log.warn("关闭连接异常", e);
            }
        }
    }

    @Override
    public boolean isRunning() {
        return running;
    }

    // ══ 消息处理器 ════════════════════════════

    class ClientHandler extends TextWebSocketHandler {

        @Override
        public void afterConnectionEstablished(
                WebSocketSession session) {
            log.info("连接建立: {}", session.getId());
        }

        @Override
        protected void handleTextMessage(
                WebSocketSession session,
                TextMessage message) {
            String payload = message.getPayload();
            try {
                JsonNode node = objectMapper.readTree(payload);
                String type = node.path("type").asText();
                dispatch(type, node);
            } catch (Exception e) {
                log.error("消息处理异常: {}", payload, e);
            }
        }

        @Override
        protected void handlePongMessage(
                WebSocketSession session,
                PongMessage message) {
            log.debug("收到 Pong");
        }

        @Override
        public void handleTransportError(
                WebSocketSession session,
                Throwable exception) {
            log.error("传输异常，准备重连", exception);
            stopHeartbeat();
            scheduleReconnect();
        }

        @Override
        public void afterConnectionClosed(
                WebSocketSession session,
                CloseStatus status) {
            log.info("连接关闭: {}", status);
            stopHeartbeat();
            // 非正常关闭则重连
            if (running && !CloseStatus.NORMAL.equals(status)) {
                scheduleReconnect();
            }
        }
    }

    // ══ 消息分发 ══════════════════════════════

    private final Map<String, MessageHandler> handlerMap =
        new ConcurrentHashMap<>();

    public void registerHandler(String type,
            MessageHandler handler) {
        handlerMap.put(type, handler);
    }

    private void dispatch(String type, JsonNode message) {
        MessageHandler handler = handlerMap.get(type);
        if (handler != null) {
            handler.handle(message);
        } else {
            log.warn("没有处理器: type={}", type);
        }
    }

    @FunctionalInterface
    public interface MessageHandler {
        void handle(JsonNode message);
    }
}
```