# 02 SpringBoot 接入：基础认知与快速跑通
> 本文档聚焦 Java+AI 入门：框架选型、SpringBoot 对接大模型、流式输出、多轮对话记忆、常用工具类、Starter 依赖整合。
>

---

## 1. 框架选型
## 1.1 AgentScope 框架简介
AgentScope Java 是阿里巴巴开源的 Agent 编程框架，用于构建基于大语言模型（LLM）的智能体应用。2.0 是一次较大规模的重构，核心定位是**智能体工程化**，在多 Agent 协同与生产底座方面能力突出[reference:0]。

AgentScope Java 需要 JDK 17 及以上版本，构建工具推荐 Maven 3.9+。推荐的入口是 `HarnessAgent`，它把工作区、长期记忆、会话持久化、子 agent、沙箱等工程能力打包在一个 builder 里[reference:1]。

核心依赖：

```xml
<dependency>
  <groupId>io.agentscope</groupId>
  <artifactId>agentscope-harness</artifactId>
  <version>${agentscope.version}</version>
</dependency>
```

<font style="color:rgb(15, 17, 21);">如果只需要裸 ReActAgent 的框架 API（不需要工作区/持久化/子 agent/沙箱），</font><font style="color:rgb(15, 17, 21);"> 足够提供 agent 本身。特定模型提供商的 Chat Model 与 formatter 位于独立的 </font><font style="color:rgb(15, 17, 21);"> 模型扩展模块中</font><font style="color:rgb(15, 17, 21);">。</font>

<font style="color:rgb(15, 17, 21);">HarnessAgent 快速示例：</font>

```java
HarnessAgent agent = HarnessAgent.builder()
.name("note-taker")
.sysPrompt("你是一个帮助用户做笔记的助手。")
.model("dashscope:qwen-plus")
.workspace(Paths.get(".agentscope/workspace"))
.compaction(CompactionConfig.builder()
            .triggerMessages(30)
            .keepMessages(10)
            .build())
.build();

RuntimeContext ctx = RuntimeContext.builder()
.sessionId("demo-session")
.userId("alice")
.build();

agent.call(new UserMessage("我叫天宇，今天准备一个关于 ReAct 的技术分享。"), ctx).block();
agent.call(new UserMessage("我叫什么？我今天要干什么？"), ctx).block();
```

<font style="color:rgb(15, 17, 21);">HarnessAgent 跑通三件事：工作区驱动的人格（AGENTS.md）、会话自动持久化（相同 sessionId 的第二轮记得第一轮）、对话压缩（超阈值后自动压缩 + 长期事实落到 MEMORY.md）</font><font style="color:rgb(15, 17, 21);">。</font>

## <font style="color:rgb(15, 17, 21);">1.2 四框架对比更新</font>
| <font style="color:rgb(15, 17, 21);">框架</font> | <font style="color:rgb(15, 17, 21);">定位</font> | <font style="color:rgb(15, 17, 21);">核心优势</font> | <font style="color:rgb(15, 17, 21);">适用场景</font> |
| --- | --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">LangChain4j</font> | <font style="color:rgb(15, 17, 21);">一站式大模型应用开发工具箱</font> | <font style="color:rgb(15, 17, 21);">组件最丰富、社区最成熟、生态最广</font> | <font style="color:rgb(15, 17, 21);">通用 Java AI 应用、RAG、工具调用</font> |
| <font style="color:rgb(15, 17, 21);">Spring AI</font> | <font style="color:rgb(15, 17, 21);">Spring 官方 AI 集成框架</font> | <font style="color:rgb(15, 17, 21);">Spring 生态原生集成、标准化强</font> | <font style="color:rgb(15, 17, 21);">SpringBoot 3.x 项目</font> |
| <font style="color:rgb(15, 17, 21);">AgentScope</font> | <font style="color:rgb(15, 17, 21);">智能体工程化框架</font> | <font style="color:rgb(15, 17, 21);">多 Agent 协同、生产底座完善（沙箱、权限、审计）、原生支持多智能体</font> | <font style="color:rgb(15, 17, 21);">复杂多 Agent 业务系统、生产级 Agent</font> |
| <font style="color:rgb(15, 17, 21);">Dify</font> | <font style="color:rgb(15, 17, 21);">低代码 AI 应用平台</font> | <font style="color:rgb(15, 17, 21);">快速验证、零代码</font> | <font style="color:rgb(15, 17, 21);">需求验证阶段、非技术团队</font> |


**选型原则**

1. **按 SpringBoot 版本与官方集成选型**
    - SpringBoot 3.x + 官方集成 → Spring AI
    - SpringBoot 2.x + 丰富 Agent/RAG → LangChain4j
    - 注意：Spring AI 与 LangChain4j 不要混用
2. **按业务场景与复杂度选型**
    - 快速原型 / 通用 RAG → LangChain4j 或 Spring AI
    - 复杂多 Agent 业务系统 → AgentScope
3. **按开发阶段选型**
    - PoC 阶段：LangChain4j 更适合快速搭建
    - 生产阶段：AgentScope 更适合复杂多 Agent 生产系统
    - 快速验证：Dify 适合低代码验证；生产级深度定制仍建议 Java 自研集成

> 踩坑提醒：LangChain4j 版本和 SpringBoot 版本必须对应，0.32.0 适配 SpringBoot 2.7.x，乱升版本会直接报 `NoSuchMethodError`。
>

---

## 2. 基础参数
+ **Prompt 结构化**：角色 + 指令 + 输出格式
+ **temperature**：0~1，越高越发散，越低越严谨
    - 0.2：知识库
    - 0.7：客服系统或问答系统
+ **max_tokens**：控制输出长度
+ **model**：按场景选择模型等级

---

## 3. SpringBoot 对接通义千问核心流程
```mermaid
sequenceDiagram
    participant U as 用户
    participant C as Controller
    participant S as ChatService
    participant Q as 通义千问
    U->>C: 提问
    C->>S: 调用 chat(question)
    S->>Q: 发送 messages + 参数
    Q-->>S: 返回回答
    S-->>C: 返回结果
    C-->>U: 展示答案
```

### <font style="color:rgb(15, 17, 21);">3.1 核心依赖</font>
```xml
<!-- LangChain4j 核心 -->
<dependency>
  <groupId>dev.langchain4j</groupId>
  <artifactId>langchain4j</artifactId>
  <version>0.31.0</version>
</dependency>
<!-- 通义千问适配 -->
<dependency>
  <groupId>dev.langchain4j</groupId>
  <artifactId>langchain4j-dashscope</artifactId>
  <version>0.31.0</version>
</dependency>
```

<font style="color:rgb(15, 17, 21);">注意：0.31.0 是稳定版，新版本 API 变动较大，生产环境建议锁版本。</font>

---

## <font style="color:rgb(15, 17, 21);">4. 流式输出 SSE</font>
<font style="color:rgb(15, 17, 21);">流式输出不是让大模型回复更快，而是让用户感知到“AI 正在回答”，减少等待焦虑。</font>

<font style="color:rgb(15, 17, 21);">常见方案对比：</font>

| <font style="color:rgb(15, 17, 21);">方案</font> | <font style="color:rgb(15, 17, 21);">适合场景</font> | <font style="color:rgb(15, 17, 21);">推荐度</font> |
| --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">一次性返回字符串</font> | <font style="color:rgb(15, 17, 21);">简单 Demo、短答案</font> | <font style="color:rgb(15, 17, 21);">★</font> |
| <font style="color:rgb(15, 17, 21);">SSE</font> | <font style="color:rgb(15, 17, 21);">网页端、后台系统、企业应用</font> | <font style="color:rgb(15, 17, 21);">★★★★★</font> |
| <font style="color:rgb(15, 17, 21);">WebSocket</font> | <font style="color:rgb(15, 17, 21);">高并发聊天、IM</font> | <font style="color:rgb(15, 17, 21);">★★★★</font> |
| <font style="color:rgb(15, 17, 21);">长轮询</font> | <font style="color:rgb(15, 17, 21);">兼容旧浏览器</font> | <font style="color:rgb(15, 17, 21);">★★</font> |


### <font style="color:rgb(15, 17, 21);">4.1 SSE 注意事项</font>
1. **<font style="color:rgb(15, 17, 21);">异步线程池配置</font>**<font style="color:rgb(15, 17, 21);">：避免占用请求线程导致超时阻塞。</font>
2. **<font style="color:rgb(15, 17, 21);">前端 EventSource 接收</font>**<font style="color:rgb(15, 17, 21);">：逐字追加 DOM。</font>
3. **<font style="color:rgb(15, 17, 21);">测试方式</font>**<font style="color:rgb(15, 17, 21);">：浏览器直接访问或 Apifox/Postman 使用 SSE 调试。</font>
4. **<font style="color:rgb(15, 17, 21);">常见坑</font>**<font style="color:rgb(15, 17, 21);">：</font>
    - <font style="color:rgb(15, 17, 21);">大模型调用未开启 </font><font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">incrementalOutput(true)</font><font style="color:rgb(15, 17, 21);">（核心。不开启的话，大模型还是会一次性返回）</font>
    - <font style="color:rgb(15, 17, 21);">返回类型未写 </font><font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">produces = MediaType.TEXT_EVENT_STREAM_VALUE</font>
    - <font style="color:rgb(15, 17, 21);">前端跨域</font>
    - <font style="color:rgb(15, 17, 21);">超时时间太短</font>
5. **<font style="color:rgb(15, 17, 21);">真实项目还要考虑</font>**<font style="color:rgb(15, 17, 21);">：</font>
    - <font style="color:rgb(15, 17, 21);">连接断开处理</font>
    - <font style="color:rgb(15, 17, 21);">错误重试</font>
    - <font style="color:rgb(15, 17, 21);">内容拼接</font>
    - <font style="color:rgb(15, 17, 21);">代码块、Markdown 渲染</font>
    - <font style="color:rgb(15, 17, 21);">权限校验、限流、配额、日志审计</font>

### <font style="color:rgb(15, 17, 21);">4.2 前端 EventSource 示例</font>
```javascript
const source = new EventSource('http://localhost:8080/api/chat/stream?message=用Java写一个冒泡排序');
source.onmessage = function(event) {
  const content = event.data;
  document.getElementById('answer').innerHTML += content;
};
source.onerror = function(error) {
  console.error('SSE连接失败', error);
  source.close();
};
```

### <font style="color:rgb(15, 17, 21);">4.3 超时设置</font>
```java
SseEmitter emitter = new SseEmitter(60 * 1000L); // 60秒
```

### 4.4 WebFlux 和 SSE 如何选择
SSE 是一种**协议**（Server-Sent Events），WebFlux 是一种**编程模型**（响应式非阻塞）。两者不是互斥关系——WebFlux 可以用来实现 SSE，Spring MVC 也可以用 `SseEmitter` 实现 SSE。

| 维度 | Spring MVC + SseEmitter | WebFlux + Flux |
| --- | --- | --- |
| 编程模型 | 命令式、同步阻塞 | 响应式、异步非阻塞 |
| 并发能力 | 低（每个连接占用一个线程） | 高（基于事件循环，Netty 驱动） |
| 资源占用 | 高 | 低 |
| 学习曲线 | 低，Java 开发者熟悉 | 高，需要理解 Reactor 范式 |
| 适用场景 | 本地调试、轻量级服务、已有 SpringMVC 项目 | 生产环境、高并发实时流处理、已有 WebFlux 技术栈 |
| 迁移成本 | 低 | 如果已有 SpringMVC 项目，切换成本高 |


**选型建议**：

+ 如果项目已有 SpringMVC 技术栈，**优先用 SseEmitter**，不要为了流式输出引入整个 WebFlux 依赖栈。
+ 如果系统本身就在用 WebFlux（如 Spring Cloud Gateway），**直接用 Flux** 集成成本最低。
+ 低并发（50 连接以内）时，两者表现几乎无差别，内存增长平稳，CPU 使用率均低于 5%。
+ 高并发场景（数百以上长连接），WebFlux 的资源优势才真正体现。
+ 务实做法：MVC + 薄接缝——核心逻辑与 web 传输解耦，核心只把 token 吐给一个最小的流式 sink 接口，web 层用薄适配桥到 `SseEmitter`。日后想换 WebFlux 只需改 web 层适配，核心逻辑不动。

---

## <font style="color:rgb(15, 17, 21);">5. 多轮对话记忆</font>
<font style="color:rgb(15, 17, 21);">本质：每次提问时，把历史聊天记录一起打包发给大模型。</font><font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">核心逻辑：后端维护历史消息，每次全量传给大模型。剩下是存储介质、裁剪策略、过期策略的工程化优化。</font>

### <font style="color:rgb(15, 17, 21);">5.1 新手必踩 4 坑</font>
1. <font style="color:rgb(15, 17, 21);">只存用户消息，不存 AI 回复</font>
2. <font style="color:rgb(15, 17, 21);">历史消息不裁剪，token 费用爆</font>
3. <font style="color:rgb(15, 17, 21);">sessionId 不隔离，不同用户串消息</font>
4. <font style="color:rgb(15, 17, 21);">本地版部署到集群，会话丢失</font>

### <font style="color:rgb(15, 17, 21);">5.2 生产优化</font>
+ <font style="color:rgb(15, 17, 21);">按用户配额限制会话数</font>
+ <font style="color:rgb(15, 17, 21);">摘要式长记忆</font>
+ <font style="color:rgb(15, 17, 21);">敏感内容校验</font>
+ <font style="color:rgb(15, 17, 21);">流式输出 + 对话记忆结合</font>

#### <font style="color:rgb(15, 17, 21);">5.2.1 摘要式长记忆详解</font>
**字数范围**<font style="color:rgb(15, 17, 21);">：摘要控制在 </font>**200 字以内**<font style="color:rgb(15, 17, 21);">，代替完整历史上下文。实测可将 Token 消耗降低 80% 以上。</font>

**使用什么模型压缩**<font style="color:rgb(15, 17, 21);">：使用</font>**便宜的小模型**<font style="color:rgb(15, 17, 21);">（如通义千问 lite / qwen-turbo）做会话摘要，不用主力大模型，成本极低。</font>

**摘要内容包含**<font style="color:rgb(15, 17, 21);">：</font>

+ <font style="color:rgb(15, 17, 21);">用户的核心意图和问题主题</font>
+ <font style="color:rgb(15, 17, 21);">AI 已给出的关键结论和数据</font>
+ <font style="color:rgb(15, 17, 21);">未解决的待办事项或后续追问方向</font>
+ <font style="color:rgb(15, 17, 21);">涉及的业务实体（用户ID、单据号、政策名称等）</font>

**触发时机**<font style="color:rgb(15, 17, 21);">：当会话长度超过预设阈值（如超过 8-10 轮或 Token 超过 3000）时，自动触发摘要压缩，用摘要替换原始历史消息。</font>

---

## <font style="color:rgb(15, 17, 21);">6. 常用工具类</font>
| <font style="color:rgb(15, 17, 21);">工具类</font> | <font style="color:rgb(15, 17, 21);">作用</font> |
| --- | --- |
| <font style="color:rgb(15, 17, 21);">Token 计数工具</font> | <font style="color:rgb(15, 17, 21);">成本统计、上下文长度校验</font> |
| <font style="color:rgb(15, 17, 21);">文本清洗工具</font> | <font style="color:rgb(15, 17, 21);">去乱码、多余空格、换行、无效字符</font> |
| <font style="color:rgb(15, 17, 21);">SSE 格式转换工具</font> | <font style="color:rgb(15, 17, 21);">统一事件格式、心跳保活</font> |
| <font style="color:rgb(15, 17, 21);">Prompt 模板工具</font> | <font style="color:rgb(15, 17, 21);">统一管理提示词，支持变量填充</font> |
| <font style="color:rgb(15, 17, 21);">向量维度校验工具</font> | <font style="color:rgb(15, 17, 21);">避免入库时维度不匹配、全 0 向量</font> |


### <font style="color:rgb(15, 17, 21);">6.1 前置依赖</font>
<font style="color:rgb(15, 17, 21);">xml</font>

```xml
<!-- Spring Web 基础依赖 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <version>2.7.18</version>
</dependency>
<!-- JTokkit Token 计数库 -->
<dependency>
  <groupId>me.saro</groupId>
  <artifactId>jtokkit</artifactId>
  <version>1.0.0</version>
</dependency>
```

<font style="color:rgb(15, 17, 21);">说明：文本清洗、Prompt 模板、向量校验三个工具类为纯 JDK 实现，无需第三方依赖。</font>

### <font style="color:rgb(15, 17, 21);">6.2 Token 计数工具类</font>
<font style="color:rgb(15, 17, 21);">java</font>

```java
public class TokenCountUtil {
    public static int countTokens(String text) {
        if (text == null || text.isEmpty()) return 0;
        return JTokkit.countTokens(text);
    }
    public static boolean isOverLimit(String text, int maxTokens) {
        return countTokens(text) > maxTokens;
    }
}
```

### <font style="color:rgb(15, 17, 21);">6.3 文本清洗工具类</font>
<font style="color:rgb(15, 17, 21);">java</font>

```java
public class TextCleanUtil {
    public static String clean(String content) {
        if (content == null) return "";
        content = content.replaceAll("[\\t\\s]+", " ");
        content = content.replaceAll("\\n{3,}", "\n\n");
        content = content.replaceAll("机密|内部资料|第\\d+页共\\d+页", "");
        return content.trim();
    }
}
```

### <font style="color:rgb(15, 17, 21);">6.4 SSE 格式转换工具类</font>
<font style="color:rgb(15, 17, 21);">java</font>

```java
public class SseUtil {
    public static void sendEvent(SseEmitter emitter, String content) {
        try {
            emitter.send(SseEmitter.event().name("message").data(content));
        } catch (Exception e) {
            throw new RuntimeException("SSE消息发送失败", e);
        }
    }
    public static void sendComplete(SseEmitter emitter) {
        try {
            emitter.send(SseEmitter.event().name("done").data("[DONE]"));
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    }
    public static void sendHeartbeat(SseEmitter emitter) {
        try {
            emitter.send(SseEmitter.event().comment("heartbeat"));
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    }
}
```

### <font style="color:rgb(15, 17, 21);">6.5 Prompt 模板工具类</font>
<font style="color:rgb(15, 17, 21);">java</font>

```java
public class PromptTemplateUtil {
    private static final String RAG_ANSWER_TEMPLATE =
    "你是企业内部智能客服，请仅基于以下参考文档回答用户问题。\n" +
    "如果参考文档中没有答案，请直接回复：暂无相关信息，请联系人工客服。\n\n" +
    "参考文档：\n{context}\n\n" +
    "用户问题：\n{question}";

    public static String getRagPrompt(String context, String question) {
        Map<String, String> params = new HashMap<>(2);
        params.put("context", context);
        params.put("question", question);
        return fillTemplate(RAG_ANSWER_TEMPLATE, params);
    }

    public static String fillTemplate(String template, Map<String, String> params) {
        String result = template;
        for (Map.Entry<String, String> entry : params.entrySet()) {
            result = result.replace("{" + entry.getKey() + "}", entry.getValue());
        }
        return result;
    }
}
```

### <font style="color:rgb(15, 17, 21);">6.6 向量维度校验工具类</font>
<font style="color:rgb(15, 17, 21);">java</font>

```java
public class VectorValidateUtil {
    public static boolean validateDimension(float[] vector, int expectDimension) {
        if (vector == null || vector.length == 0) return false;
        return vector.length == expectDimension;
    }
    public static boolean isZeroVector(float[] vector) {
        if (vector == null || vector.length == 0) return true;
        for (float v : vector) {
            if (v != 0f) return false;
        }
        return true;
    }
}
```

---

## <font style="color:rgb(15, 17, 21);">7. 5 个 Starter 依赖整合</font>
| <font style="color:rgb(15, 17, 21);">序号</font> | <font style="color:rgb(15, 17, 21);">Starter</font> | <font style="color:rgb(15, 17, 21);">用途</font> | <font style="color:rgb(15, 17, 21);">适用场景</font> |
| --- | --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">1</font> | <font style="color:rgb(15, 17, 21);">spring-ai-openai-spring-boot-starter</font> | <font style="color:rgb(15, 17, 21);">大模型调用（OpenAI 兼容接口）</font> | <font style="color:rgb(15, 17, 21);">通用大模型对话、Function Calling</font> |
| <font style="color:rgb(15, 17, 21);">2</font> | <font style="color:rgb(15, 17, 21);">langchain4j-spring-boot-starter</font> | <font style="color:rgb(15, 17, 21);">Agent 框架（RAG/Agent/记忆/工具）</font> | <font style="color:rgb(15, 17, 21);">完整 Agent 应用开发</font> |
| <font style="color:rgb(15, 17, 21);">3</font> | <font style="color:rgb(15, 17, 21);">milvus-sdk-java</font> | <font style="color:rgb(15, 17, 21);">向量数据库</font> | <font style="color:rgb(15, 17, 21);">RAG 知识库、长期记忆</font> |
| <font style="color:rgb(15, 17, 21);">4</font> | <font style="color:rgb(15, 17, 21);">spring-boot-starter-webflux</font> | <font style="color:rgb(15, 17, 21);">流式输出（SSE）</font> | <font style="color:rgb(15, 17, 21);">大模型流式响应</font> |
| <font style="color:rgb(15, 17, 21);">5</font> | <font style="color:rgb(15, 17, 21);">spring-boot-configuration-processor</font> | <font style="color:rgb(15, 17, 21);">配置管理</font> | <font style="color:rgb(15, 17, 21);">大模型参数配置化</font> |


### <font style="color:rgb(15, 17, 21);">7.1 完整 pom.xml</font>
```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
  </parent>
  <groupId>com.example</groupId>
  <artifactId>springboot-ai-starter</artifactId>
  <version>1.0.0</version>
  <properties>
    <java.version>17</java.version>
    <langchain4j.version>0.35.0</langchain4j.version>
    <milvus.version>2.4.4</milvus.version>
  </properties>
  <dependencies>
    <!-- 1. Spring AI：大模型调用 -->
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
      <version>1.0.0</version>
    </dependency>
    <!-- 2. LangChain4j：Agent 框架 -->
    <dependency>
      <groupId>dev.langchain4j</groupId>
      <artifactId>langchain4j-spring-boot-starter</artifactId>
      <version>${langchain4j.version}</version>
    </dependency>
    <dependency>
      <groupId>dev.langchain4j</groupId>
      <artifactId>langchain4j-dashscope-spring-boot-starter</artifactId>
      <version>${langchain4j.version}</version>
    </dependency>
    <!-- 3. Milvus：向量数据库 -->
    <dependency>
      <groupId>io.milvus</groupId>
      <artifactId>milvus-sdk-java</artifactId>
      <version>${milvus.version}</version>
    </dependency>
    <!-- 4. WebFlux：流式输出 -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <!-- 5. 配置管理 -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-configuration-processor</artifactId>
      <optional>true</optional>
    </dependency>
    <!-- Web -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Lombok -->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>
  </dependencies>
</project>
```

> **<font style="color:rgb(15, 17, 21);">避坑提醒</font>**<font style="color:rgb(15, 17, 21);">：Spring AI 1.0.0 要求 SpringBoot 3.2+；如果还在用 SpringBoot 2.7，用 LangChain4j 更合适。</font>
>

