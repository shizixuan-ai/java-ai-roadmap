# 07 简历面试与项目：包装、题库与真实落地
> 本文档聚焦简历包装、面试高频考点、企业级智能客服项目、PDF 摘要、HR 助手、企业微信对接、转型避坑。
>

---

## 1. 简历包装
### 1.1 坑 1：只写“调用大模型 API”
> 负责对接通义千问大模型，实现智能问答功能。
>

这句话等于告诉面试官“我谁都能替代”。

**正面写法：**

> 基于 RAG 架构实现企业知识库问答系统，对接通义千问大模型；设计混合检索 + 重排序方案，将问答准确率从 72% 提升至 88%；接入 Sentinel 限流降级保障服务稳定性，支持 50 人同时在线使用；上线后覆盖 80% 内部重复咨询，人工接待量下降 50%。
>

核心判断标准：去掉大模型/AI 相关词，剩下的内容还能不能体现工程能力？

### 1.2 坑 2：堆砌技术名词
技术名词放在“技术栈”一栏就够了，项目描述重点是你解决了什么业务问题。

**正确三段式：**

1. 项目背景：解决了什么业务问题
2. 你的方案：怎么设计和实现
3. 落地结果：带来了什么量化收益

### 1.3 坑 3：项目太 Toy
企业招 AI 工程师是来解决生产问题的。生产级项目加分项：

| 维度 | 加分描述 | 体现能力 |
| --- | --- | --- |
| 权限隔离 | 按部门/租户隔离知识库，敏感文档物理隔离 | 多租户、安全设计 |
| 限流熔断 | 接入 Sentinel，大模型接口超时自动降级 | 高可用、稳定性 |
| 成本优化 | 对话历史裁剪 + 检索去重，Token 消耗下降 40% | 成本意识、工程优化 |
| 可观测性 | AOP 埋点，监控调用量/响应时长/错误率/Token | 生产运维能力 |
| 合规设计 | 输入输出双层敏感词过滤，审计日志全记录 | 合规意识 |
| 性能数据 | 压测支持 50 并发，平均响应 2 秒，首字延迟 300ms | 性能意识 |


> 简历可以适当包装，但绝对不要瞎编。面试官深挖两句就能看出来。
>

---

## 2. 企业级智能客服项目
### 2.1 需求
+ 覆盖 80% 常见内部咨询
+ 人工接待量下降 50%
+ 平均响应从 10 分钟降到 2 秒内
+ 支持 50 人同时在线

### 2.2 功能分级
+ P0：智能问答、知识库管理、权限管控
+ P1：工具调用、管理后台
+ P2：多端接入

### 2.3 技术选型
| 层级 | 选型 |
| --- | --- |
| 后端 | Spring Boot 2.7.18 |
| 大模型 | 通义千问 API / 本地 Qwen2-7B |
| 向量库 | Milvus 2.3 |
| 关系库 | MySQL 8.0 |
| 缓存 | Redis + Caffeine |
| 流量防护 | Sentinel |
| 前端 | Vue3 + 企业微信 JSAPI |


### 2.4 核心模块
1. 对话交互模块
2. RAG 知识库模块
3. 工具调用模块
4. 权限管控模块
5. 管理后台模块
6. 高可用与合规模块

### 2.5 核心数据库
+ `kb_document`
+ `kb_document_chunk`
+ `chat_session`
+ `chat_record`
+ `operation_log`

### 2.6 用户问答全链路
1. 接入层：企业微信发消息，Sentinel 限流
2. 权限层：校验身份、部门权限
3. 缓存层：精确匹配 + 语义匹配
4. 检索层：查询改写 + 混合检索，召回 Top20
5. 精排层：Rerank 选出 Top5，权限过滤
6. 模型层：拼装 Prompt，按复杂度路由模型
7. 返回层：SSE 流式输出
8. 后置处理：异步记录日志、统计 Token、更新缓存、敏感词二次校验

### 2.7 文档入库全链路
1. 管理员上传文档，校验权限与文件格式
2. 解析文档内容，清洗无效格式、页眉页脚、乱码
3. 按语义智能切片，保留上下文重叠
4. 调用 Embedding 模型生成向量
5. 向量存入 Milvus，返回唯一 vector_id
6. 切片内容 + vector_id + 文档信息存入 MySQL
7. 更新文档状态为上线，清空相关问答缓存
8. 记录操作审计日志

### 2.8 项目落地节奏
| 阶段 | 时间 | 验收标准 |
| --- | --- | --- |
| 基础可用版 | 3 天 | 能上传文档并回答问题，单轮准确率 ≥70% |
| 核心功能版 | 5 天 | 多轮对话、流式首字 <1s，准确率 ≥80% |
| 企业级增强 | 4 天 | 权限隔离生效，50 路并发稳定，合规通过 |
| 上线试运行 | 3 天 | 99.9% 可用性，核心功能无 Bug |


### 2.9 核心代码案例
#### 用户问答全链路核心代码
```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    @Resource
    private RagChatService ragChatService;

    @Resource(name = "chatStreamExecutor")
    private ThreadPoolTaskExecutor streamExecutor;

    /**
     * 带权限的流式问答
     */
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamChat(@RequestParam String question) {
        SseEmitter emitter = new SseEmitter(300000L);

        // 从登录上下文获取当前用户信息，杜绝前端越权传参
        LoginUser currentUser = SecurityContextHolder.getCurrentUser();

        streamExecutor.execute(() -> {
            try {
                // 1. 查询改写
                String rewrittenQuery = ragChatService.rewriteQuery(question);

                // 2. 混合检索 + 权限过滤
                List<TextSegment> segments = ragChatService.hybridSearch(
                        rewrittenQuery, currentUser.getDeptId(), currentUser.getUserId());

                // 3. Rerank 重排
                List<TextSegment> reranked = ragChatService.rerank(rewrittenQuery, segments);

                // 4. 组装 Prompt
                String prompt = PromptTemplateUtil.getRagPrompt(
                        reranked.stream().map(TextSegment::text).collect(Collectors.joining("\n")),
                        question);

                // 5. 流式调用大模型
                ragChatService.streamChat(prompt, token ->
                        SseUtil.sendEvent(emitter, token));

                SseUtil.sendComplete(emitter);

                // 6. 后置处理：异步记录日志、更新缓存
                ragChatService.asyncPostProcess(currentUser, question, emitter);

            } catch (Exception e) {
                emitter.completeWithError(e);
            }
        });

        return emitter;
    }
}
```

#### 知识库文档入库核心代码
```java
@Service
public class DocumentIngestService {

    @Resource
    private DocumentParserUtil parserUtil;

    @Resource
    private EmbeddingModel embeddingModel;

    @Resource
    private EmbeddingStore<TextSegment> embeddingStore;

    /**
     * 文档入库全链路
     */
    public void ingestDocument(MultipartFile file, LoginUser currentUser,
                               Integer permissionLevel) throws IOException {
        // 1. 解析文档
        String rawText = parserUtil.extractText(file);

        // 2. 清洗无效内容
        String cleanText = parserUtil.cleanText(rawText);

        // 3. 智能切片（重叠窗口策略）
        DocumentSplitter splitter = DocumentSplitters.recursive(500, 50);
        List<TextSegment> segments = splitter.split(Document.from(cleanText));

        // 4. 注入权限元数据
        segments.forEach(segment -> {
            segment.metadata().put("docName", file.getOriginalFilename());
            segment.metadata().put("deptId", currentUser.getDeptId());
            segment.metadata().put("permissionLevel", String.valueOf(permissionLevel));
            segment.metadata().put("ownerId", currentUser.getUserId());
            segment.metadata().put("uploadTime", LocalDateTime.now().toString());
        });

        // 5. 向量化入库
        EmbeddingStoreIngestor.builder()
        .documentSplitter(splitter)
        .embeddingModel(embeddingModel)
        .embeddingStore(embeddingStore)
        .build()
        .ingest(segments);

        // 6. 清空相关问答缓存
        cacheService.evictByDeptId(currentUser.getDeptId());

        // 7. 记录审计日志
        operationLogService.log(currentUser, "上传文档", file.getOriginalFilename());
    }
}
```

#### 企业微信对接核心代码
```java
@RestController
@RequestMapping("/api/wxwork")
public class WxWorkController {

    @Autowired
    private WxCpService wxCpService;

    @Autowired
    private RagChatService ragChatService;

    @Autowired
    private ThreadPoolTaskExecutor taskExecutor;

    /**
     * 接收企业微信消息
     * 必须5秒内返回success，业务逻辑异步处理
     */
    @PostMapping("/callback")
    public String receiveMessage(String msgSignature, String timestamp,
                                 String nonce, @RequestBody String requestBody) {
        try {
            WxCpXmlMessage message = WxCpXmlMessage.fromEncryptedXml(
                requestBody, wxCpService.getConfigStorage(),
                msgSignature, timestamp, nonce);

            if (!"text".equals(message.getMsgType())) {
                return WxCpXmlOutMessage.TEXT()
                .content("暂不支持其他类型消息，请输入文字提问")
                .build().toEncryptedXml(wxCpService.getConfigStorage());
            }

            String userId = message.getFromUserName();
            String question = message.getContent();

            // 异步处理问答，避免5秒超时
            taskExecutor.execute(() -> {
                try {
                    List<Long> deptIds = wxCpService.getUserService()
                    .getById(userId).getDeptIds();
                    String deptId = deptIds.isEmpty() ? "" : String.valueOf(deptIds.get(0));

                    // 复用带权限的RAG问答服务
                    String answer = ragChatService.chatWithAuth(question, deptId, userId);

                    WxCpTextMessage textMessage = new WxCpTextMessage();
                    textMessage.setContent(answer);
                    wxCpService.getMessageService().send(userId, textMessage);
                } catch (Exception e) {
                    // 异常兜底
                }
            });

            return "success";
        } catch (Exception e) {
            return "success";
        }
    }
}
```

#### 面试项目数据速查
| 指标 | 数据 |
| --- | --- |
| 准确率提升 | 从 72% 提升至 88% |
| 成本降低 | 大模型调用成本降低 60%-70% |
| 并发支持 | 50 路并发稳定运行 |
| 首字延迟 | < 1 秒 |
| 覆盖咨询量 | 覆盖 80% 内部常见咨询 |
| 人工接待量下降 | 50% |
| 系统可用性 | 99.9% |
| Token 节省 | 输入 Token 整体下降 30%-40% |
| 缓存命中率 | ≥ 40% |


### 2.10 简历亮点提炼
**初级开发（1-3 年）：**

> 负责企业内部智能客服系统的开发与落地，基于 Spring Boot + LangChain4j 对接大模型，实现 RAG 知识库问答功能，支持多格式文档解析、多轮对话与流式输出。接入 Milvus 向量数据库实现混合检索，优化问答准确率；使用 Sentinel 做接口限流，保障系统稳定性。项目上线后覆盖 80% 内部常见咨询，提升员工办事效率。
>

**中级开发（3-5 年）：**

> 主导企业内部智能客服系统的架构设计与核心模块开发，采用分层架构解耦业务与 AI 能力，支持多端接入。设计实现两级问答缓存、模型分级路由方案，将大模型调用成本降低 60%；采用混合检索 + Rerank 方案，问答准确率提升至 85% 以上。落地部门级数据权限隔离、全链路审计与敏感词过滤，满足企业合规要求，系统可用性达 99.9%。
>

**高级开发/架构师（5 年+）：**

> 负责企业内部智能客服平台的整体架构设计与技术选型，从 0 到 1 搭建完整的 RAG 智能问答体系，覆盖知识库管理、对话交互、工具调用、权限管控全模块。建立全链路成本管控体系，通过缓存复用、模型分级、上下文裁剪等手段，综合降本 70%；设计高可用防护体系，支持限流熔断与降级兜底，保障高峰期稳定运行。项目上线后覆盖公司全员内部咨询场景，人工接待量下降 50%，大幅降低行政人事支持成本，实现显著的业务提效与降本价值。
>

---

## 3. PDF 智能摘要
### 3.1 核心难点
1. 长文档超限
2. 摘要遗漏前半部分
3. 业务需求不统一

### 3.2 方案：MapReduce 思路
+ Map：长文档切段，每段单独生成摘要
+ Reduce：合并所有分段摘要，生成总摘要

### 3.3 优化
+ 按章节边界切分
+ 分级摘要
+ 缓存摘要结果

### 3.4 简历描述
> 设计并实现企业文档智能摘要模块，基于 LangChain4j + PDFBox，采用分段摘要 + 合并总摘要的 MapReduce 方案解决长文档上下文超限问题；支持合同/报告/通用三种定制摘要模板，单份 50 页文档处理时间从 15 分钟降至 30 秒，法务/产品团队审阅效率提升 90%。
>

---

## 4. HR 助手案例
### 4.1 选场景标准
问题重复率高、答案有明确文档依据、边界清晰不涉及复杂决策。

### 4.2 技术方案
| 层级 | 选型 | 说明 |
| --- | --- | --- |
| 接入端 | 企业微信 | 员工直接在企微提问 |
| 后端 | SpringBoot 2.7 | Java 开发者最熟悉 |
| 大模型 | 通义千问 turbo | 性价比高，中文效果好 |
| 向量库 | Milvus 单机版 | 500 人公司政策文档量不大 |
| 权限 | 按部门隔离 | 薪资、绩效文档物理隔离 |


### 4.3 核心架构流程
1. 员工在企微提问 → 企微回调 SpringBoot 接口
2. 查询改写（口语化问题转标准检索 query）
3. Milvus 向量检索 TopK 相关政策片段
4. 片段 + 问题组装 Prompt → 通义千问生成回答
5. 回答返回企微 → 员工看到答案
6. 解决不了的问题 → 一键转 HR 人工

### 4.4 踩过的 3 个坑
1. 文档不清洗，准确率只有 60%
2. 员工提问太口语化，检索不到
3. 薪资绩效文档怕泄露，不敢放一起

### 4.5 建议
+ 别一上来做大平台，先找具体场景
+ 文档质量是生命线
+ 权限隔离从第一天就做

---

## 5. 一键对接企业微信
### 5.1 对接方案选型
| 方案 | 实现方式 | 优势 | 劣势 | 适用场景 |
| --- | --- | --- | --- | --- |
| 群机器人 | Webhook 调用 | 配置最简单 | 功能单一，无权限、无多轮对话 | 小团队群内工具 |
| 自建应用 | 企业内部自建应用 | 功能完整，可对接组织架构 | 需要简单开发配置 | 企业内部全员知识库 |
| 第三方应用 | 上架企业微信第三方市场 | 可对外提供服务 | 审核严格 | 对外商业化 SaaS |


企业内部知识库落地，自建应用是最优解。

### 5.2 核心依赖
```xml
<dependency>
    <groupId>com.github.binarywang</groupId>
    <artifactId>weixin-java-cp</artifactId>
    <version>4.5.0</version>
</dependency>
```

### 5.3 消息回调与问答实现
企业微信硬性规则：回调接口必须在 5 秒内返回 success，否则重试 3 次。
所以不能同步等大模型生成答案，要先返回 success，再异步处理问答，最后用客服消息推送结果。

### 5.4 新手必踩 5 坑
1. 回调 URL 校验一直失败
2. 消息重复收到好几次
3. 调用发送消息接口报错 42001
4. 用户收不到消息
5. 用户 ID 和部门 ID 对不上权限体系

---

## 6. 面试高频考点
### 6.1 基础概念题
+ RAG 的完整流程是什么？
+ RAG 有哪些常见优化方法？
+ 流式输出怎么实现？
+ Function Calling 的执行流程是什么？

### 6.2 工程场景题
+ 大模型接口不稳定怎么处理？
+ 怎么做限流降级？
+ 怎么控制调用成本？
+ 怎么保证数据安全？
+ 怎么实现部门级权限隔离？

### 6.3 项目深挖题
+ 你做的项目准确率多少？
+ 遇到过最棘手的问题是什么？
+ 怎么解决的？
+ 如果再做一次你会怎么优化？

### 6.4 方案设计题
+ 给你一个业务场景，你怎么设计整体方案？
+ 怎么做内部知识库？
+ 怎么做智能客服？
+ 怎么做文档总结系统？

---

## 7. 转型避坑指南
1. **放弃 Java，去学 Python 卷算法**
卷不过科班，丢掉工程优势。正确方向：立足 Java，走 AI 工程化落地路线。
2. **只会调 API，不做完整项目**
只会调 API 没有竞争力。正确方向：做完整生产级项目。
3. **光学不练，收藏等于学会**
面试一问细节就露馅。正确方向：动手做完整智能客服项目。
4. **盲目追新，丢了根基**
Java 工程能力是根基，AI 是增量能力。

---

## 8. Agent 生产踩坑实录（面试话术）
### 8.1 记忆错乱
+ 现象：A 用户看到 B 用户政策
+ 根因：向量召回没加租户隔离；sessionId 生成有 bug；没有相关性阈值过滤
+ 解决：向量召回强制带 `tenant_id + user_id`；sessionId 和用户绑定；相关性阈值过滤

### 8.2 任务死循环
+ 现象：Agent 卡住不动，token 疯狂消耗
+ 根因：没有最大执行步数；失败任务无限重试；工具返回格式不对
+ 解决：最大执行步数 15；失败重试上限 3 次 + 指数退避；工具结果格式化

### 8.3 输出幻觉
+ 现象：Agent 编造不存在的数据和政策
+ 根因：没有检索结果相关性校验；Prompt 没有禁止编造；关键参数没锁定
+ 解决：空结果必拒答；Prompt 强约束；关键业务参数用实体记忆锁定

### 8.4 面试话术模板
> 我们 Agent 上线后主要遇到三类生产问题：
第一是记忆错乱，多租户场景下召回没有加租户隔离，导致用户信息串了。解决方式是向量召回强制带 tenant_id + user_id 过滤条件，sessionId 和用户绑定，召回结果加相关性阈值过滤。
第二是任务死循环，复杂任务规划没有步数限制，失败任务无限重试。解决方式是设置最大执行步数 15 步，失败重试上限 3 次加指数退避，重试耗尽降级跳过，工具返回结果统一格式化。
第三是输出幻觉，知识库没召回时大模型自己编数据。解决方式是检索为空时明确告诉大模型“无相关信息禁止编造”，System Prompt 加强约束，关键业务参数用实体记忆锁定注入。
这三个问题解决后，Agent 回答准确率从 72% 提升到 91%，生产故障基本清零。
>

