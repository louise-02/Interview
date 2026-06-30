Spring Boot 集成 WebSocket，用于**全双工**实时通信（工位推送、设备状态、消息通知等）。

常见两种写法：**`@ServerEndpoint`（JSR-356，项目里常用）** 和 **Spring `WebSocketHandler`**。

---

# 1、依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

---

# 2、两种写法对比

| | @ServerEndpoint | Spring WebSocketHandler |
| -- | --------------- | ----------------------- |
| 风格 | JSR-356 注解（`@OnOpen` / `@OnMessage`） | Spring 接口回调 |
| 配置 | `ServerEndpointExporter` | `WebSocketConfigurer` |
| 路径参数 | `@PathParam("username")` 支持 | 需自己从 URI 解析 |
| 注入 Bean | 类**不是**标准单例 Bean，需 `SpringUtil.getBean` 或 `Configurator` | 可直接 `@Resource` |
| 适用 | 业务项目、按用户/工位管理连接 | 简单 echo、框架化封装 |

---

# 3、方式一：@ServerEndpoint（项目常用）

## 注解说明

| 注解 | 作用 |
| ---- | ---- |
| `@ServerEndpoint` | 标记 WebSocket 服务端，定义连接路径 |
| `@OnOpen` | 连接建立时处理（鉴权、入池、推送初始数据） |
| `@OnClose` | 连接关闭时处理（出池、计数减一） |
| `@OnMessage` | 收到客户端消息（心跳等） |
| `@OnError` | 业务异常处理 |

路径参数用 `@PathParam`：`/tvSocket/{username}` → `@PathParam("username")`。

## 配置类

```java
@Configuration
public class WebSocketConfig {

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

## Endpoint 类（业务简化版）

基于 TV 大屏推送场景简化，保留核心结构：按**工位**管理连接、连接时推 queue/list、业务变更时 `refreshQueue` / `refreshList`。

```java
@Slf4j
@Component
@ServerEndpoint(value = "/tvSocket/{username}")
public class TvSocketServer {

    private static final AtomicInteger ONLINE_COUNT = new AtomicInteger(0);

    /** key: stationId，value: 该工位下所有连接 */
    public static final ConcurrentHashMap<Long, List<TvSocketServer>> SOCKET_LIST_MAP = new ConcurrentHashMap<>();

    public static final List<String> CURRENT_USER_LIST = new Vector<>();

    public static final ReadWriteLock LOCK = new ReentrantReadWriteLock();

    private Session session;
    private SysUser sysUser;
    private Station station;
    /** 是否已完成入池（用于 onClose 准确减在线数） */
    private boolean online = false;

    // ---------- 连接建立 ----------

    @OnOpen
    public void onOpen(Session session, @PathParam("username") String username) throws IOException {
        this.session = session;
        this.sysUser = SpringUtil.getBean(SysUserService.class).getUserByCount(username);

        if (sysUser == null) {
            sendError(String.format("%s user not exists!", username));
            session.close();
            return;
        }

        this.station = SpringUtil.getBean(UserStationService.class).getUserFirstStation(sysUser.getId());
        if (station == null) {
            sendError(String.format("user %s not have station!", username));
            session.close();
            return;
        }

        // 校验全部通过后再计入在线、入池
        this.online = true;
        CURRENT_USER_LIST.add(username);
        addToPool(station.getTid());
        log.info("user connect : {} | online : {}", username, ONLINE_COUNT.incrementAndGet());

        List<Queue> queueList = SpringUtil.getBean(QueueService.class).tvList();
        sendQueue(queueList);

        List<Setups> setupsList = SpringUtil.getBean(SetupsService.class).listByStationId(station.getTid());
        sendList(setupsList);
    }

    // ---------- 连接关闭 ----------

    @OnClose
    public void onClose() {
        if (online) {
            online = false;
            if (sysUser != null) {
                CURRENT_USER_LIST.remove(sysUser.getAccount());
                log.info("user exit : {} | online : {}", sysUser.getAccount(), ONLINE_COUNT.decrementAndGet());
            }
            if (station != null) {
                removeFromPool(station.getTid());
            }
        }
    }

    // ---------- 客户端消息（心跳） ----------

    @OnMessage
    public void onMessage(String message) throws IOException {
        if (sysUser == null) {
            return;
        }
        log.info("user {} | message : {}", sysUser.getAccount(), message);
        sendMessage("ok");
    }

    @OnError
    public void onError(Throwable error) {
        String account = sysUser != null ? sysUser.getAccount() : "unknown";
        log.error("user error : {}", account, error);
        try {
            if (session != null && session.isOpen()) {
                sendError(error.getMessage());
            }
        } catch (IOException e) {
            log.warn("onError send failed, user={}", account, e);
        }
    }

    // ---------- 推送 queue（异步 + 双指针取当前工位前后若干条） ----------

    private void sendQueue(List<Queue> queueList) {
        ThreadUtil.execute(() -> {
            LinkedList<Queue> window = pickQueueWindow(queueList, station.getStationNo(), 3);
            sendSuccess(new TvResponse(window, null));
        });
    }

    // ---------- 推送 list（按 postNo 分组排序） ----------

    private void sendList(List<Setups> setupsList) {
        ThreadUtil.execute(() -> {
            List<TvSetupsResponse> list = groupByPostNo(setupsList);
            sendSuccess(new TvResponse(null, list));
        });
    }

    // ---------- 业务侧调用：刷新所有在线大屏 ----------

    public static void refreshQueue() {
        ThreadUtil.execute(() -> {
            List<Queue> queueList = SpringUtil.getBean(QueueService.class).tvList();
            Lock lock = LOCK.readLock();
            lock.lock();
            try {
                for (Map.Entry<Long, List<TvSocketServer>> entry : SOCKET_LIST_MAP.entrySet()) {
                    for (TvSocketServer server : entry.getValue()) {
                        server.sendQueue(queueList);
                    }
                    refreshList(entry.getKey());
                }
            } finally {
                lock.unlock();
            }
        });
    }

    public static void refreshList(Long stationId) {
        ThreadUtil.execute(() -> {
            Lock lock = LOCK.readLock();
            lock.lock();
            try {
                List<TvSocketServer> servers = SOCKET_LIST_MAP.get(stationId);
                if (servers == null || servers.isEmpty()) {
                    return;
                }
                List<Setups> setupsList = SpringUtil.getBean(SetupsService.class).listByStationId(stationId);
                for (TvSocketServer server : servers) {
                    server.sendList(setupsList);
                }
            } finally {
                lock.unlock();
            }
        });
    }

    // ---------- 发送封装（统一 JSON 响应） ----------

    public void sendSuccess(Object data) {
        try {
            sendMessage(JSON.toJSONString(ResponseData.success(data)));
        } catch (IOException e) {
            log.warn("sendSuccess failed, user={}", sysUser != null ? sysUser.getAccount() : "unknown", e);
        }
    }

    public void sendError(String msg) throws IOException {
        sendMessage(JSON.toJSONString(ResponseData.error(msg)));
    }

    public void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
    }

    // ---------- 连接池 ----------

    private void addToPool(Long stationId) {
        Lock lock = LOCK.writeLock();
        lock.lock();
        try {
            List<TvSocketServer> list = SOCKET_LIST_MAP.computeIfAbsent(stationId, k -> new ArrayList<>());
            list.add(this);
        } finally {
            lock.unlock();
        }
    }

    private void removeFromPool(Long stationId) {
        Lock lock = LOCK.writeLock();
        lock.lock();
        try {
            List<TvSocketServer> list = SOCKET_LIST_MAP.get(stationId);
            if (list != null) {
                list.remove(this);
                if (list.isEmpty()) {
                    SOCKET_LIST_MAP.remove(stationId);
                }
            }
        } finally {
            lock.unlock();
        }
    }

    // pickQueueWindow / groupByPostNo：按业务规则筛选 queue 窗口、按 postNo 分组，略
}
```

**整体流程**

```
客户端连接 /tvSocket/{username}
    → onOpen：校验用户 → 查工位 → 入 SOCKET_LIST_MAP
    → 推 queue + list 初始数据
业务变更（如 MES 更新）
    → refreshQueue() / refreshList(stationId) 推送给在线连接
客户端定时发消息
    → onMessage 回 ok（心跳）
断开
    → onClose：出池、在线数 -1
```

## 为什么用 SpringUtil.getBean

`@ServerEndpoint` **每个连接**会 new 一个实例，**不能**直接 `@Autowired`。项目里用 Hutool：

```java
SpringUtil.getBean(QueueService.class)
```

| 做法 | 说明 |
| ---- | ---- |
| `SpringUtil.getBean` | 项目常用 |
| `ServerEndpointConfig.Configurator` | 官方推荐，可替代 SpringUtil |

## 前端连接

```javascript
const ws = new WebSocket(`ws://localhost:8080/tvSocket/${username}`);
ws.onmessage = (e) => {
    const res = JSON.parse(e.data);   // ResponseData 结构
    console.log(res);
};
// 心跳
setInterval(() => ws.send('ping'), 30000);
```

## 业务里主动刷新

```java
// queue 数据变更后，刷新所有在线 TV 大屏
TvSocketServer.refreshQueue();

// 某工位 setups 变更
TvSocketServer.refreshList(stationId);
```

---

# 4、方式二：Spring WebSocketHandler（备选）

Spring 原生写法，Handler 可直接注入 Bean，适合简单场景：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(demoWebSocketHandler(), "/ws/demo")
                .setAllowedOriginPatterns("*");
    }

    @Bean
    public DemoWebSocketHandler demoWebSocketHandler() {
        return new DemoWebSocketHandler();
    }
}
```

```java
public class DemoWebSocketHandler extends TextWebSocketHandler {
    private static final ConcurrentHashMap<String, WebSocketSession> SESSIONS = new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        SESSIONS.put(session.getId(), session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        session.sendMessage(new TextMessage("echo: " + message.getPayload()));
    }

    public static void broadcast(String text) {
        SESSIONS.values().forEach(s -> {
            try {
                if (s.isOpen()) s.sendMessage(new TextMessage(text));
            } catch (Exception ignored) { }
        });
    }
}
```

---

# 5、STOMP（订阅/广播，可选）

需要 `/topic` 主题订阅时用 STOMP + `@EnableWebSocketMessageBroker`，与上面两种写法**选其一**即可，一般工业推送用 `@ServerEndpoint` 足够。

---

# 6、注意

| 点 | 说明 |
| -- | ---- |
| 连接池 | `SOCKET_LIST_MAP` 按 stationId 管理；入池/出池用**写锁** |
| 刷新推送 | `refreshQueue` / `refreshList` 用**读锁**遍历，避免并发修改 |
| 异步推送 | `ThreadUtil.execute` 避免阻塞 WebSocket 线程 |
| 注入 Service | `SpringUtil.getBean`，不要直接 `@Autowired` |
| 心跳 | `@OnMessage` 收到消息后回 `ok` |
| 统一响应 | `ResponseData.success/error` 包一层 JSON |
| 集群 | 静态 Map 只在单机有效；多实例需 Redis/MQ 转发 |
| 网关 | Nginx 需配置 `Upgrade`、`Connection` 头 |
