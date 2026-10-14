# 05 生产工程化：排坑、高可用、降本与监控
> 本文档聚焦 AI 应用上生产：限流降级熔断、SSE 稳定、Token 降本、性能优化、监控告警、生产 10 大坑。
>

---

## 1. AI 接口高可用：限流、降级、熔断
AI 接口高可用 3 个核心难点：

1. 响应耗时长，线程资源占用极高
2. 外部依赖不稳定，故障不可控
3. 算力资源消耗高，过载恢复极慢

三层防护架构：

<!-- 这是一个文本绘图，源码为：flowchart LR
    A[流量入口] --> B[限流]
    B --> C[降级]
    C --> D[熔断]
    D --> E[大模型服务]
 -->
![](https://cdn.nlark.com/yuque/__mermaid_v3/4d9e0623f9884650ad22b18b7f46b86a.svg)

### <font style="color:rgb(15, 17, 21);">1.1 限流</font>
+ <font style="color:rgb(15, 17, 21);">QPS 限流 + 并发线程数限流双重防护</font>
+ <font style="color:rgb(15, 17, 21);">匀速排队，不直接拒绝</font>
+ <font style="color:rgb(15, 17, 21);">流式 SSE 接口用信号量控制并发连接数</font>

### <font style="color:rgb(15, 17, 21);">1.2 降级</font>
+ <font style="color:rgb(15, 17, 21);">慢调用比例降级</font>
+ <font style="color:rgb(15, 17, 21);">优先返回缓存答案</font>
+ <font style="color:rgb(15, 17, 21);">适用：早高峰、大模型抖动、配额不足</font>

### <font style="color:rgb(15, 17, 21);">1.3 熔断</font>
+ <font style="color:rgb(15, 17, 21);">异常比例熔断</font>
+ <font style="color:rgb(15, 17, 21);">大模型 429、超时、配额耗尽、服务不可用都算异常</font>
+ <font style="color:rgb(15, 17, 21);">状态流转：关闭 → 打开 → 半开 → 关闭</font>

### <font style="color:rgb(15, 17, 21);">1.4 三级兜底</font>
| <font style="color:rgb(15, 17, 21);">层级</font> | <font style="color:rgb(15, 17, 21);">触发条件</font> | <font style="color:rgb(15, 17, 21);">策略</font> |
| --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">一级</font> | <font style="color:rgb(15, 17, 21);">限流触发</font> | <font style="color:rgb(15, 17, 21);">友好排队提示</font> |
| <font style="color:rgb(15, 17, 21);">二级</font> | <font style="color:rgb(15, 17, 21);">降级/熔断</font> | <font style="color:rgb(15, 17, 21);">优先本地缓存答案</font> |
| <font style="color:rgb(15, 17, 21);">三级</font> | <font style="color:rgb(15, 17, 21);">缓存未命中/故障持续</font> | <font style="color:rgb(15, 17, 21);">常见问题导航 + 人工客服入口</font> |


### 1.5 Sentinel 限流降级熔断完整代码
#### 依赖
```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    <version>2021.0.5.0</version>
</dependency>

```

#### <font style="color:rgb(15, 17, 21);">限流规则配置</font>
```java
@PostConstruct
public void initFlowRules() {
    List<FlowRule> rules = new ArrayList<>();

    // QPS限流规则
    FlowRule qpsRule = new FlowRule();
    qpsRule.setResource("chat_interface");
    qpsRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    qpsRule.setCount(20);
    qpsRule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);
    qpsRule.setMaxQueueingTimeMs(30000);
    rules.add(qpsRule);

    // 并发线程数限流规则（AI接口核心防护）
    FlowRule threadRule = new FlowRule();
    threadRule.setResource("chat_interface");
    threadRule.setGrade(RuleConstant.FLOW_GRADE_THREAD);
    threadRule.setCount(10);
    rules.add(threadRule);

    FlowRuleManager.loadRules(rules);
}
```

#### <font style="color:rgb(15, 17, 21);">业务接口使用注解</font>
```java
@SentinelResource(
    value = "chat_interface",
    blockHandler = "chatBlockHandler",
    fallback = "chatFallback"
)
public String chat(String question) {
// 调用大模型
return dashScope.call(request).getOutput().getChoices().get(0).getMessage().getContent();
}

/**
 * 限流/熔断触发的阻塞处理方法
 * 参数必须和原方法一致，最后加一个 BlockException 参数
 */
public String chatBlockHandler(String question, BlockException e) {
    return "当前咨询人数较多，请稍后再试，或查看底部常见问题文档";
}

/**
 * 降级触发的兜底方法：优先返回缓存答案
 */
public String chatFallback(String question) {
String cacheAnswer = chatAnswerCache.getIfPresent(question.trim().toLowerCase());
if (cacheAnswer != null) return cacheAnswer;
return "当前系统繁忙，请稍后重试，您也可以查看常见问题文档快速获取答案";
}
```

#### <font style="color:rgb(15, 17, 21);">流式 SSE 接口专属限流（信号量）</font>
```java
private final Semaphore sseSemaphore = new Semaphore(8);

@SentinelResource(value = "chat_stream", blockHandler = "streamBlockHandler")
public SseEmitter chatStream(String question) {
if (!sseSemaphore.tryAcquire()) {
    throw new BlockException("并发数超限") {};
}
SseEmitter emitter = new SseEmitter(60000L);
emitter.onCompletion(sseSemaphore::release);
emitter.onError(e -> sseSemaphore.release());
return emitter;
}
```

### <font style="color:rgb(15, 17, 21);">1.6 新手必踩 6 坑</font>
1. <font style="color:rgb(15, 17, 21);">只做 QPS 限流，不控并发数</font>
2. <font style="color:rgb(15, 17, 21);">兜底方法写太复杂，自己也报错</font>
3. <font style="color:rgb(15, 17, 21);">熔断时间设置不合理，反复雪崩</font>
4. <font style="color:rgb(15, 17, 21);">所有接口共用一个资源名，限流混乱</font>
5. <font style="color:rgb(15, 17, 21);">没有监控，出了问题全靠猜</font>
6. <font style="color:rgb(15, 17, 21);">流式接口没做全链路限流，连接池爆了</font>

---

## <font style="color:rgb(15, 17, 21);">2. SSE 上生产：断线重连 + 超时降级</font>
<font style="color:rgb(15, 17, 21);">生产必死 4 个坑：</font>

1. <font style="color:rgb(15, 17, 21);">断连即丢失进度</font>
2. <font style="color:rgb(15, 17, 21);">无超时管控</font>
3. <font style="color:rgb(15, 17, 21);">无异常兜底</font>
4. <font style="color:rgb(15, 17, 21);">无资源隔离</font>

### <font style="color:rgb(15, 17, 21);">2.1 全链路检查清单</font>
| <font style="color:rgb(15, 17, 21);">节点</font> | <font style="color:rgb(15, 17, 21);">检查项</font> | <font style="color:rgb(15, 17, 21);">配置值</font> |
| --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">大模型客户端</font> | <font style="color:rgb(15, 17, 21);">读超时</font> | <font style="color:rgb(15, 17, 21);">≥5 分钟</font> |
| <font style="color:rgb(15, 17, 21);">大模型客户端</font> | <font style="color:rgb(15, 17, 21);">响应读取方式</font> | <font style="color:rgb(15, 17, 21);">逐行流读取，禁止 string()</font> |
| <font style="color:rgb(15, 17, 21);">后端服务</font> | <font style="color:rgb(15, 17, 21);">SseEmitter 超时</font> | <font style="color:rgb(15, 17, 21);">≥5 分钟</font> |
| <font style="color:rgb(15, 17, 21);">后端服务</font> | <font style="color:rgb(15, 17, 21);">线程池隔离</font> | <font style="color:rgb(15, 17, 21);">流式单独线程池</font> |
| <font style="color:rgb(15, 17, 21);">后端服务</font> | <font style="color:rgb(15, 17, 21);">异常回调</font> | <font style="color:rgb(15, 17, 21);">onCompletion/onTimeout/onError 全注册</font> |
| <font style="color:rgb(15, 17, 21);">后端服务</font> | <font style="color:rgb(15, 17, 21);">心跳保活</font> | <font style="color:rgb(15, 17, 21);">每 10 秒发注释心跳</font> |
| <font style="color:rgb(15, 17, 21);">Nginx/网关</font> | <font style="color:rgb(15, 17, 21);">proxy_buffering</font> | <font style="color:rgb(15, 17, 21);">off</font> |
| <font style="color:rgb(15, 17, 21);">Nginx/网关</font> | <font style="color:rgb(15, 17, 21);">X-Accel-Buffering</font> | <font style="color:rgb(15, 17, 21);">no</font> |
| <font style="color:rgb(15, 17, 21);">Nginx/网关</font> | <font style="color:rgb(15, 17, 21);">proxy_read_timeout</font> | <font style="color:rgb(15, 17, 21);">≥5 分钟</font> |
| <font style="color:rgb(15, 17, 21);">前端</font> | <font style="color:rgb(15, 17, 21);">断线重连</font> | <font style="color:rgb(15, 17, 21);">EventSource 自带 + offset 断点续传</font> |


### <font style="color:rgb(15, 17, 21);">2.2 生产级能力</font>
1. <font style="color:rgb(15, 17, 21);">断线重连 + 消息序号缓存</font>
2. <font style="color:rgb(15, 17, 21);">线程池隔离 + 分级超时控制</font>
3. <font style="color:rgb(15, 17, 21);">异常重试 + 分级降级兜底</font>
4. <font style="color:rgb(15, 17, 21);">单用户并发限制</font>

### 2.3 SSE 全链路代码案例
#### 大模型客户端读超时配置（OkHttp）
```java
OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(Duration.ofSeconds(10))
        .readTimeout(Duration.ofMinutes(5))   // 流式必须设长
        .writeTimeout(Duration.ofSeconds(30))
        .build();
```

#### <font style="color:rgb(15, 17, 21);">流式读取（逐行，不要用 string()）</font>
```java
try (Response response = client.newCall(request).execute()) {
    BufferedReader reader = new BufferedReader(
        new InputStreamReader(response.body().byteStream()));
    String line;
    while ((line = reader.readLine()) != null) {
        if (line.startsWith("data:")) {
            String data = line.substring(5).trim();
            emitter.send(SseEmitter.event().data(data));
        }
    }
    emitter.complete();
}
```

#### <font style="color:rgb(15, 17, 21);">SSE 线程池隔离</font>
```java
@Configuration
public class SseThreadPoolConfig {
    @Bean("sseExecutor")
    public ThreadPoolTaskExecutor sseExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("sse-stream-");
        executor.setRejectedExecutionHandler((r, e) -> {
            // 降级处理：返回完整回答而不是流式
        });
        executor.initialize();
        return executor;
    }
}
```

#### <font style="color:rgb(15, 17, 21);">异常回调全注册</font>
```java
SseEmitter emitter = new SseEmitter(300000L);
emitter.onCompletion(() -> log.info("SSE连接正常结束"));
emitter.onTimeout(() -> { log.warn("SSE连接超时"); emitter.complete(); });
emitter.onError(e -> { log.error("SSE连接异常", e); emitter.complete(); });
```

#### <font style="color:rgb(15, 17, 21);">心跳保活</font>
```java
ScheduledExecutorService heartbeat = Executors.newSingleThreadScheduledExecutor();
heartbeat.scheduleAtFixedRate(() -> {
    try {
        emitter.send(SseEmitter.event().comment("heartbeat"));
    } catch (Exception e) {
        heartbeat.shutdown();
    }
}, 10, 10, TimeUnit.SECONDS);
emitter.onCompletion(() -> heartbeat.shutdown());
```

#### <font style="color:rgb(15, 17, 21);">Nginx 配置</font>
```nginx
location /api/chat/stream {
  proxy_pass http://backend;
  proxy_buffering off;
  proxy_cache off;
  chunked_transfer_encoding on;
  proxy_connect_timeout 10s;
  proxy_read_timeout 300s;
  proxy_send_timeout 300s;
  add_header X-Accel-Buffering no;
  add_header Cache-Control no-cache;
}
```

#### <font style="color:rgb(15, 17, 21);">前端断线重连与 Last-Event-ID</font>
**<font style="color:rgb(15, 17, 21);">前端 EventSource 基础用法（浏览器自带断线重连）</font>**

```html
<script>
  // 浏览器自带断线自动重连，无需手动 close + 重新 new EventSource
  const evtSource = new EventSource('/api/chat/stream?sessionId=abc123');

  // 接收消息：event.lastEventId 为服务端设置的 id
  evtSource.onmessage = function(event) {
    document.getElementById('answer').innerHTML += event.data;
  };

  // 监听自定义事件
  evtSource.addEventListener('token', function(event) {
    document.getElementById('answer').innerHTML += event.data;
  });

  // 断线：浏览器约3秒后自动重连，并自动携带 Last-Event-ID 请求头
  evtSource.onerror = function(error) {
    console.warn('SSE连接出错，浏览器将自动重连...', error);
    // 注意：不要在这里调用 evtSource.close()，否则会阻止自动重连
  };
</script>
```

**<font style="color:rgb(15, 17, 21);">后端读取 Last-Event-ID 断点续传</font>**

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamChat(
    @RequestParam String sessionId,
    @RequestParam(required = false) String question,
    @RequestHeader(value = "Last-Event-ID", required = false) String lastEventId) {

    SseEmitter emitter = new SseEmitter(300000L);

    // 断线重连：补发断点之后的消息
    if (lastEventId != null && !lastEventId.isEmpty()) {
        int lastId = Integer.parseInt(lastEventId);
        List<ChatMessage> cached = chatStreamService.getCachedMessages(sessionId);
        for (ChatMessage msg : cached) {
            if (msg.getSequence() > lastId) {
                try {
                    emitter.send(SseEmitter.event()
                                 .id(String.valueOf(msg.getSequence()))
                                 .name(msg.getType().name().toLowerCase())
                                 .data(msg.getContent()));
                } catch (IOException e) {
                    return emitter; // 又断了，等下次重连
                }
            }
        }
        if (chatStreamService.isGenerationComplete(sessionId)) {
            emitter.complete();
            return emitter;
        }
    }

    // 继续生成剩余内容
    streamExecutor.execute(() -> {
        try {
            chatStreamService.generateAnswer(sessionId, question, emitter);
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });

    return emitter;
}
```

**<font style="color:rgb(15, 17, 21);">服务端发送消息必须设置事件 id</font>**

```java
public void pushMessage(String sessionId, String content, MessageType type) {
    ChatMessage msg = cacheMessage(sessionId, content, type);
    SseEmitter emitter = activeEmitters.get(sessionId);
    if (emitter == null) return; // 用户离线，消息已缓存

    try {
        emitter.send(SseEmitter.event()
                     .id(String.valueOf(msg.getSequence()))   // 关键：设置 id，浏览器才会记录
                     .name(type.name().toLowerCase())
                     .data(msg.getContent()));
    } catch (IOException e) {
        log.warn("SSE发送失败，消息已缓存待重连，序号:{}", msg.getSequence());
    }
}

private ChatMessage cacheMessage(String sessionId, String content, MessageType type) {
    List<ChatMessage> cache = messageCache
    .computeIfAbsent(sessionId, k -> Collections.synchronizedList(new ArrayList<>()));
    int seq = cache.size() + 1;
    ChatMessage msg = new ChatMessage(seq, content, type, System.currentTimeMillis());
    cache.add(msg);
    // 缓存容量上限，防止内存溢出
    if (cache.size() > 100) {
        cache.subList(0, cache.size() - 100).clear();
    }
    return msg;
}
```

**<font style="color:rgb(15, 17, 21);">注意事项</font>**

1. <font style="color:rgb(15, 17, 21);">必须设置事件 id：只有 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">SseEmitter.event().id(...)</font>`<font style="color:rgb(15, 17, 21);"> 设置了 id，浏览器才会在重连时发送 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">Last-Event-ID</font>`<font style="color:rgb(15, 17, 21);"> 头。</font>
2. <font style="color:rgb(15, 17, 21);">不要手动关闭 EventSource：在 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">onerror</font>`<font style="color:rgb(15, 17, 21);"> 中调用 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">evtSource.close()</font>`<font style="color:rgb(15, 17, 21);"> 会阻止浏览器自动重连。</font>
3. <font style="color:rgb(15, 17, 21);">服务端必须缓存已发送消息：内存或 Redis，给每条消息分配递增序号。</font>
4. <font style="color:rgb(15, 17, 21);">缓存容量必须设上限：防止长时间会话导致内存溢出。</font>
5. <font style="color:rgb(15, 17, 21);">断点续传后继续生成：如果回答未生成完，服务端应继续生成剩余内容并推送新事件。</font>

---

## <font style="color:rgb(15, 17, 21);">3. Token 降本增效</font>
<font style="color:rgb(15, 17, 21);">Token 成本构成：</font>

+ <font style="color:rgb(15, 17, 21);">输入 token 费用：用户问题、历史对话、检索文档</font>
+ <font style="color:rgb(15, 17, 21);">输出 token 费用：大模型生成内容，通常比输入贵 2-3 倍</font>

<font style="color:rgb(15, 17, 21);">企业知识库场景，输入 token 占 70% 以上。</font>

### <font style="color:rgb(15, 17, 21);">3.1 降本技巧</font>
1. <font style="color:rgb(15, 17, 21);">问答缓存复用：降本 40%</font>
2. <font style="color:rgb(15, 17, 21);">模型分级调用：降本 25%</font>
3. <font style="color:rgb(15, 17, 21);">上下文精简与裁剪：降本 15%</font>
4. <font style="color:rgb(15, 17, 21);">输出长度控制：降本 10%</font>
5. <font style="color:rgb(15, 17, 21);">批量请求合并</font>
6. <font style="color:rgb(15, 17, 21);">本地模型替代公网 API</font>

### <font style="color:rgb(15, 17, 21);">3.2 Java 代码技巧</font>
+ <font style="color:rgb(15, 17, 21);">对话历史自动裁剪，只保留最近 N 轮</font>
+ <font style="color:rgb(15, 17, 21);">检索结果去重 + 截断</font>
+ <font style="color:rgb(15, 17, 21);">Prompt 模板精简</font>
+ <font style="color:rgb(15, 17, 21);">TopK 动态调整</font>

### <font style="color:rgb(15, 17, 21);">3.3 降本技巧代码案例</font>
#### <font style="color:rgb(15, 17, 21);">技巧1：对话历史自动裁剪</font>
```java
public class ChatHistoryTrimmer {
    private static final int KEEP_ROUNDS = 5;

    public static List<Message> trim(List<Message> messages) {
        if (messages == null || messages.size() <= KEEP_ROUNDS * 2 + 1) {
            return messages;
        }
        List<Message> result = new ArrayList<>();
        result.add(messages.get(0)); // 系统提示词永远保留
        int start = messages.size() - KEEP_ROUNDS * 2;
        result.addAll(messages.subList(start, messages.size()));
        return result;
    }
}
```

> 实测数据：一个客服场景，保留全部历史平均单次输入 1800 Token，裁剪到最近 5 轮后降到 950 Token，直接省 47%。
>

#### <font style="color:rgb(15, 17, 21);">技巧2：检索结果去重 + 截断</font>
<font style="color:rgb(15, 17, 21);">java</font>

```plain
public class ChunkDeduplicator {
    private static final double SIMILARITY_THRESHOLD = 0.8;
    private static final int MAX_CHUNK_LENGTH = 500;

    public static List<String> process(List<String> chunks) {
        List<String> result = new ArrayList<>();
        for (String chunk : chunks) {
            chunk = chunk.length() > MAX_CHUNK_LENGTH
                    ? chunk.substring(0, MAX_CHUNK_LENGTH) : chunk;
            boolean isDuplicate = result.stream()
                    .anyMatch(exist -> similarity(chunk, exist) > SIMILARITY_THRESHOLD);
            if (!isDuplicate) result.add(chunk);
        }
        return result;
    }

    private static double similarity(String s1, String s2) {
        if (s1.contains(s2) || s2.contains(s1)) return 1.0;
        int lcsLength = longestCommonSubstring(s1, s2);
        return (double) lcsLength / Math.min(s1.length(), s2.length());
    }
}
```

<font style="color:rgb(15, 17, 21);">实测效果：10 万文档知识库，去重前平均每次送 4 个片段共 1800 字，去重后剩 2.5 个片段约 1100 字，省 39% Token。</font>

#### <font style="color:rgb(15, 17, 21);">技巧3：Prompt 模板精简</font>
**<font style="color:rgb(15, 17, 21);">反面例子（约120Token）</font>**<font style="color:rgb(15, 17, 21);">：</font>

> <font style="color:rgb(15, 17, 21);">你现在是一个非常专业的智能客服助手，你需要非常耐心地回答用户的问题，请注意，你只能根据提供的参考文档来回答，绝对不可以自己编造内容...</font>
>

**<font style="color:rgb(15, 17, 21);">精简版本（约25Token）</font>**<font style="color:rgb(15, 17, 21);">：</font>

> <font style="color:rgb(15, 17, 21);">你是企业智能客服，仅基于参考文档回答，无答案则回复：暂无相关信息。</font>
>

<font style="color:rgb(15, 17, 21);">Token 少了 80%。</font>**<font style="color:rgb(15, 17, 21);">精简原则：去掉形容词副词、去掉重复约束、角色设定一句话、输出格式用最短指令。</font>**

#### <font style="color:rgb(15, 17, 21);">技巧4：TopK 动态调整</font>
```java
public static int dynamicTopK(String question) {
if (question.length() < 20 || isFactQuestion(question)) {
    return 2; // 短问题、事实类问题，TopK=2足够
}
return 5; // 长问题、复杂推理类，TopK=5
}
```

#### <font style="color:rgb(15, 17, 21);">技巧5：模型分级路由</font>
```java
@Service
public class ModelRouterService {
    @Autowired
    private ChatLanguageModel smallModel;

    @Autowired
    private ChatLanguageModel strongModel;

    public String routeChat(String question) {
        String cacheAnswer = chatCache.getIfPresent(question);
        if (cacheAnswer != null) return cacheAnswer;

        if (isSimpleQuestion(question)) {
            return smallModel.generate(question);
        }
        return strongModel.generate(question);
    }

    private boolean isSimpleQuestion(String question) {
        if (commonQuestionSet.contains(question)) return true;
        return question.length() < 20 && question.contains("怎么查");
    }
}
```

> <font style="color:rgb(15, 17, 21);">实测效果：80% 的普通问题都能用小模型解决，整体成本再降 25%，用户体验完全没差别。</font>
>

### <font style="color:rgb(15, 17, 21);">3.4 优化项与单省比例</font>
| <font style="color:rgb(15, 17, 21);">优化项</font> | <font style="color:rgb(15, 17, 21);">单省比例</font> |
| --- | --- |
| <font style="color:rgb(15, 17, 21);">对话历史裁剪</font> | <font style="color:rgb(15, 17, 21);">30%-50%</font> |
| <font style="color:rgb(15, 17, 21);">检索去重截断</font> | <font style="color:rgb(15, 17, 21);">20%-40%</font> |
| <font style="color:rgb(15, 17, 21);">Prompt 精简</font> | <font style="color:rgb(15, 17, 21);">50%-80%</font> |
| <font style="color:rgb(15, 17, 21);">TopK 动态调整</font> | <font style="color:rgb(15, 17, 21);">10%-30%</font> |


<font style="color:rgb(15, 17, 21);">普通 RAG 场景叠加使用，输入 Token 整体下降 30%-40% 很轻松。</font>

---

## <font style="color:rgb(15, 17, 21);">4. 性能优化</font>
<font style="color:rgb(15, 17, 21);">通义千问性能优化 4 核心手段：</font>

1. <font style="color:rgb(15, 17, 21);">模型分级选型</font>
2. <font style="color:rgb(15, 17, 21);">并发控制 + 请求排队</font>
3. <font style="color:rgb(15, 17, 21);">缓存策略</font>
4. <font style="color:rgb(15, 17, 21);">流式异步输出</font>

### <font style="color:rgb(15, 17, 21);">4.1 必踩坑</font>
+ <font style="color:rgb(15, 17, 21);">盲目调高并发数，触发平台限流</font>
+ <font style="color:rgb(15, 17, 21);">全场景用高价大模型</font>
+ <font style="color:rgb(15, 17, 21);">缓存没做失效机制</font>
+ <font style="color:rgb(15, 17, 21);">队列长度设置过长</font>
+ <font style="color:rgb(15, 17, 21);">所有请求共用线程池</font>

### 4.2 并发控制代码案例（信号量 + 线程池）
```java
@Configuration
public class ModelConcurrencyConfig {
    private static final int MAX_CONCURRENT_REQUESTS = 8;
    private static final int MAX_QUEUE_SIZE = 20;

    @Bean
    public Semaphore modelSemaphore() {
        return new Semaphore(MAX_CONCURRENT_REQUESTS);
    }

    @Bean(name = "modelTaskExecutor")
    public ThreadPoolTaskExecutor modelTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(MAX_CONCURRENT_REQUESTS);
        executor.setMaxPoolSize(MAX_CONCURRENT_REQUESTS);
        executor.setQueueCapacity(MAX_QUEUE_SIZE);
        executor.setThreadNamePrefix("ai-model-task-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        executor.initialize();
        return executor;
    }
}
```

<font style="color:rgb(15, 17, 21);">调用时增加排队控制：</font>

```java
public String chatWithConcurrencyControl(String question) {
boolean acquired = false;
try {
    acquired = modelSemaphore.tryAcquire(30, TimeUnit.SECONDS);
    if (!acquired) return "当前咨询人数较多，请稍后再试";
    return singleChat(question);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return "请求中断，请重试";
} finally {
    if (acquired) modelSemaphore.release();
}
}
```

### <font style="color:rgb(15, 17, 21);">4.3 缓存策略代码案例（Caffeine + Redis 两级缓存）</font>
```java
@Configuration
public class ChatCacheConfig {
    @Bean
    public Cache<String, String> chatAnswerCache() {
        return Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(1, TimeUnit.HOURS)
        .build();
    }
}
```

```java
public String chatWithCache(String question) {
String cacheKey = question.trim().toLowerCase();
String cachedAnswer = chatAnswerCache.getIfPresent(cacheKey);
if (cachedAnswer != null) return cachedAnswer;

String answer = singleChat(question);
chatAnswerCache.put(cacheKey, answer);
return answer;
}
```

> <font style="color:rgb(15, 17, 21);">实测效果：企业知识库场景高频问题占比超 60%，加缓存后大模型实际调用量降 70%，平均响应从 2-3 秒降到几百毫秒。</font>
>

---

## <font style="color:rgb(15, 17, 21);">5. 监控告警</font>
### <font style="color:rgb(15, 17, 21);">5.1 5 类核心指标</font>
1. <font style="color:rgb(15, 17, 21);">调用指标：总调用量、各接口调用量、日活/周活</font>
2. <font style="color:rgb(15, 17, 21);">性能指标：平均响应时长、P95/P99、首字延迟、流式总耗时</font>
3. <font style="color:rgb(15, 17, 21);">错误指标：错误率、错误类型分布、超时次数、限流触发次数</font>
4. <font style="color:rgb(15, 17, 21);">成本指标：总 Token 消耗、输入/输出占比、日均/月均费用</font>
5. <font style="color:rgb(15, 17, 21);">效果指标：问答准确率、点赞/点踩率、badcase 数量、转人工率</font>

### <font style="color:rgb(15, 17, 21);">5.2 4 条核心告警</font>
1. <font style="color:rgb(15, 17, 21);">错误率告警：5 分钟内错误率超过 10%</font>
2. <font style="color:rgb(15, 17, 21);">响应耗时告警：平均响应超过 5 秒或 P95 超过 10 秒</font>
3. <font style="color:rgb(15, 17, 21);">成本告警：当日 Token 消耗超预算</font>
4. <font style="color:rgb(15, 17, 21);">调用量异常告警：较昨日下降 80% 或增长 300%</font>

### <font style="color:rgb(15, 17, 21);">5.</font>3 监控埋点完整实现
#### 自定义注解<font style="color:rgb(15, 17, 21);"> + AOP 切面，零侵入业务代码</font>
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AiMonitor {
    String modelName() default "default";
    String description() default "";
}
```

#### <font style="color:rgb(15, 17, 21);">AOP 切面</font>
```java
@Aspect
@Component
public class AiMonitorAspect {

    @Resource
    private MonitorService monitorService;

    @Around("@annotation(aiMonitor)")
    public Object around(ProceedingJoinPoint joinPoint, AiMonitor aiMonitor) throws Throwable {
        long start = System.currentTimeMillis();
        String userId = UserContext.getUserId();
        String modelName = aiMonitor.modelName();
        String methodName = joinPoint.getSignature().toShortString();

        try {
            Object result = joinPoint.proceed();
            long costMs = System.currentTimeMillis() - start;
            monitorService.recordSuccess(userId, modelName, methodName, costMs);
            return result;
        } catch (Exception e) {
            long costMs = System.currentTimeMillis() - start;
            monitorService.recordFail(userId, modelName, methodName, costMs, e);
            throw e;
        }
    }
}
```

#### <font style="color:rgb(15, 17, 21);">指标采集 Service</font>
```java
@Service
public class MonitorService {

    @Resource
    private StringRedisTemplate redisTemplate;

    private static final String KEY_PREFIX = "ai:monitor:";

    public void recordSuccess(String userId, String modelName, String methodName, long costMs) {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        String baseKey = KEY_PREFIX + date + ":" + modelName;

        redisTemplate.opsForValue().increment(baseKey + ":success");
        redisTemplate.opsForValue().increment(baseKey + ":totalCost", costMs);
        redisTemplate.opsForHyperLogLog().add(baseKey + ":uv", userId);
        redisTemplate.expire(baseKey, 30, TimeUnit.DAYS);
    }

    public void recordFail(String userId, String modelName, String methodName,
                           long costMs, Exception e) {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        String baseKey = KEY_PREFIX + date + ":" + modelName;

        redisTemplate.opsForValue().increment(baseKey + ":fail");
        String errorType = e.getClass().getSimpleName();
        redisTemplate.opsForHash().increment(baseKey + ":errorType", errorType, 1);
        redisTemplate.expire(baseKey, 30, TimeUnit.DAYS);
    }

    public void recordTokenUsage(String userId, String modelName,
                                 int promptTokens, int completionTokens) {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        String baseKey = KEY_PREFIX + date + ":" + modelName;

        redisTemplate.opsForValue().increment(baseKey + ":promptTokens", promptTokens);
        redisTemplate.opsForValue().increment(baseKey + ":completionTokens", completionTokens);
    }
}
```

#### <font style="color:rgb(15, 17, 21);">业务代码使用（零侵入）</font>
```java
@Service
public class ChatService {
    @AiMonitor(modelName = "qwen-turbo", description = "RAG问答接口")
    public String chat(String question) {
        // 业务逻辑：检索 → 组装Prompt → 调用大模型 → 返回答案
        return answer;
    }
}
```

---

## <font style="color:rgb(15, 17, 21);">6. 生产 10 大坑</font>
| <font style="color:rgb(15, 17, 21);">风险等级</font> | <font style="color:rgb(15, 17, 21);">坑点</font> | <font style="color:rgb(15, 17, 21);">核心风险</font> |
| --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">致命</font> | <font style="color:rgb(15, 17, 21);">并发雪崩</font> | <font style="color:rgb(15, 17, 21);">服务整体不可用</font> |
| <font style="color:rgb(15, 17, 21);">致命</font> | <font style="color:rgb(15, 17, 21);">敏感词漏检</font> | <font style="color:rgb(15, 17, 21);">合规处罚风险</font> |
| <font style="color:rgb(15, 17, 21);">致命</font> | <font style="color:rgb(15, 17, 21);">会话串号</font> | <font style="color:rgb(15, 17, 21);">用户数据泄露</font> |
| <font style="color:rgb(15, 17, 21);">致命</font> | <font style="color:rgb(15, 17, 21);">日志泄露敏感数据</font> | <font style="color:rgb(15, 17, 21);">数据安全事故</font> |
| <font style="color:rgb(15, 17, 21);">严重</font> | <font style="color:rgb(15, 17, 21);">流式输出断流</font> | <font style="color:rgb(15, 17, 21);">用户体验崩塌</font> |
| <font style="color:rgb(15, 17, 21);">严重</font> | <font style="color:rgb(15, 17, 21);">Token 超限截断</font> | <font style="color:rgb(15, 17, 21);">回答不完整</font> |
| <font style="color:rgb(15, 17, 21);">严重</font> | <font style="color:rgb(15, 17, 21);">大模型超时</font> | <font style="color:rgb(15, 17, 21);">接口偶发失败</font> |
| <font style="color:rgb(15, 17, 21);">严重</font> | <font style="color:rgb(15, 17, 21);">向量库内存泄漏</font> | <font style="color:rgb(15, 17, 21);">服务 OOM 崩溃</font> |
| <font style="color:rgb(15, 17, 21);">一般</font> | <font style="color:rgb(15, 17, 21);">中文乱码</font> | <font style="color:rgb(15, 17, 21);">展示异常</font> |
| <font style="color:rgb(15, 17, 21);">一般</font> | <font style="color:rgb(15, 17, 21);">依赖版本冲突</font> | <font style="color:rgb(15, 17, 21);">启动失败</font> |


### <font style="color:rgb(15, 17, 21);">6.1 避坑核心原则</font>
1. <font style="color:rgb(15, 17, 21);">边界防护：所有外部依赖都要做超时、重试、熔断、限流</font>
2. <font style="color:rgb(15, 17, 21);">数据校验：所有输入输出都要校验、脱敏、权限判断</font>
3. <font style="color:rgb(15, 17, 21);">可观测性：所有核心链路都要加监控、日志、告警</font>

### <font style="color:rgb(15, 17, 21);">6.2 致命级坑案例</font>
#### <font style="color:rgb(15, 17, 21);">坑1：并发雪崩</font>
**现象**<font style="color:rgb(15, 17, 21);">：早高峰大量用户提问，服务直接卡死，所有接口响应变慢，大面积超时崩溃。</font>

**根因**<font style="color:rgb(15, 17, 21);">：没有做并发控制，请求全部堆到大模型侧，GPU、带宽、线程池被打满，请求持续堆积形成雪崩。</font>

**解决方案**<font style="color:rgb(15, 17, 21);">：</font>

```java
// 接入层做并发限流
FlowRule threadRule = new FlowRule();
threadRule.setResource("chat_interface");
threadRule.setGrade(RuleConstant.FLOW_GRADE_THREAD);
threadRule.setCount(10);
FlowRuleManager.loadRules(Collections.singletonList(threadRule));

// 大模型调用线程池与业务线程池隔离
@Bean("aiModelExecutor")
public ThreadPoolTaskExecutor aiModelExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(8);
    executor.setMaxPoolSize(16);
    executor.setQueueCapacity(50);
    executor.setThreadNamePrefix("ai-model-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
    return executor;
}
```

#### <font style="color:rgb(15, 17, 21);">坑2：敏感词漏检</font>
**<font style="color:rgb(15, 17, 21);">现象</font>**<font style="color:rgb(15, 17, 21);">：敏感词过滤已接入，仍有违规内容被输出。</font>

**<font style="color:rgb(15, 17, 21);">根因</font>**<font style="color:rgb(15, 17, 21);">：只做输入侧过滤，没做输出侧二次校验；词库不全，变体、谐音容易绕过。</font>

**<font style="color:rgb(15, 17, 21);">解决方案</font>**<font style="color:rgb(15, 17, 21);">：</font>

```java
// 输入输出双重过滤
public String chatWithSensitiveCheck(String question) {
// 输入校验
if (!sensitiveWordFilter.checkSensitiveWord(question).isEmpty()) {
    return "提问包含违规内容，请调整后重试";
}

String answer = chatModel.generate(question);

// 输出校验（关键：AI返回内容必须再过一遍）
if (!sensitiveWordFilter.checkSensitiveWord(answer).isEmpty()) {
    return "抱歉，生成的回答包含不合规内容，已拦截";
}
return answer;
}
```

#### <font style="color:rgb(15, 17, 21);">坑3：会话串号</font>
**<font style="color:rgb(15, 17, 21);">现象</font>**<font style="color:rgb(15, 17, 21);">：A 用户提问，返回了 B 用户的对话内容。</font>

**<font style="color:rgb(15, 17, 21);">根因</font>**<font style="color:rgb(15, 17, 21);">：sessionId 使用全局变量或缓存 key 重复；ThreadLocal 使用后未清理。</font>

**<font style="color:rgb(15, 17, 21);">解决方案</font>**<font style="color:rgb(15, 17, 21);">：</font>

```java
// sessionId 采用「用户ID + UUID」唯一标识
String sessionId = userId + "_" + UUID.randomUUID().toString();

// ThreadLocal 用完必须清理
try {
    SecurityContextHolder.setCurrentUser(loginUser);
    // 业务逻辑
} finally {
    SecurityContextHolder.clear(); // 关键：防止线程池复用导致上下文泄露
}
```

---

## <font style="color:rgb(15, 17, 21);">7. Agent 生产踩坑实录</font>
### <font style="color:rgb(15, 17, 21);">7.1 记忆错乱</font>
+ <font style="color:rgb(15, 17, 21);">现象：A 用户看到 B 用户政策，新会话记得上个用户隐私</font>
+ <font style="color:rgb(15, 17, 21);">根因：向量召回没加租户隔离；sessionId 生成有 bug；没有相关性阈值过滤</font>
+ <font style="color:rgb(15, 17, 21);">解决：向量召回强制带 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">tenant_id + user_id</font>`<font style="color:rgb(15, 17, 21);">；sessionId 和用户绑定；相关性阈值过滤</font>

### <font style="color:rgb(15, 17, 21);">7.2 任务死循环</font>
+ <font style="color:rgb(15, 17, 21);">现象：Agent 卡住不动，token 疯狂消耗</font>
+ <font style="color:rgb(15, 17, 21);">根因：没有最大执行步数；失败任务无限重试；工具返回格式不对</font>
+ <font style="color:rgb(15, 17, 21);">解决：最大执行步数 15；失败重试上限 3 次 + 指数退避；工具结果格式化</font>

### <font style="color:rgb(15, 17, 21);">7.3 输出幻觉</font>
+ <font style="color:rgb(15, 17, 21);">现象：Agent 编造不存在的数据和政策</font>
+ <font style="color:rgb(15, 17, 21);">根因：没有检索结果相关性校验；Prompt 没有禁止编造；关键参数没锁定</font>
+ <font style="color:rgb(15, 17, 21);">解决：空结果必拒答；Prompt 强约束；关键业务参数用实体记忆锁定</font>

