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