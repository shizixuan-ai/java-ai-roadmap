# 06 企业架构与合规：鉴权、多租户与私有化平台
> 本文档聚焦企业级 AI 架构：大模型接口鉴权、敏感词过滤、权限审计、多租户隔离、多租户 RAG、私有化 AI 平台。
>

---

## 1. 大模型接口鉴权
为什么比普通接口更重要：

+ 防未授权调用
+ 防密钥泄露
+ 防恶意刷量
+ 防责任不清

### 1.1 生产级 AI 网关 5 层设计
<!-- 这是一个文本绘图，源码为：flowchart TB
    A[接入层] --> B[鉴权层]
    B --> C[限流层]
    C --> D[配额层]
    D --> E[审计层] -->
![](https://cdn.nlark.com/yuque/__mermaid_v3/d15238628a5c76e81ffc4e6c0deef306.svg)

### <font style="color:rgb(15, 17, 21);">1.2 鉴权方式</font>
+ **<font style="color:rgb(15, 17, 21);">AK/SK 签名</font>**<font style="color:rgb(15, 17, 21);">：适合服务端调用、第三方系统接入</font>
+ **<font style="color:rgb(15, 17, 21);">JWT</font>**<font style="color:rgb(15, 17, 21);">：适合前端用户调用，配合短期过期、刷新令牌、黑名单</font>

### <font style="color:rgb(15, 17, 21);">1.3 普通登录态为什么不够用</font>
<font style="color:rgb(15, 17, 21);">普通登录态解决“用户是否登录”，AI 接口还需要回答：</font>

+ <font style="color:rgb(15, 17, 21);">这个应用能不能调用大模型？</font>
+ <font style="color:rgb(15, 17, 21);">这个用户有没有权限使用某个模型？</font>
+ <font style="color:rgb(15, 17, 21);">这个 Key 有没有过期？</font>
+ <font style="color:rgb(15, 17, 21);">这个用户今天用了多少 Token？</font>
+ <font style="color:rgb(15, 17, 21);">这个应用有没有超过配额？</font>
+ <font style="color:rgb(15, 17, 21);">这次调用能不能审计？</font>

### <font style="color:rgb(15, 17, 21);">1.4 上线检查清单</font>
<font style="color:rgb(15, 17, 21);">✅</font><font style="color:rgb(15, 17, 21);"> 所有大模型请求经过网关</font><font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">✅</font><font style="color:rgb(15, 17, 21);"> AK/SK 签名已实现</font><font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">✅</font><font style="color:rgb(15, 17, 21);"> 时间戳防重放</font><font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">✅</font><font style="color:rgb(15, 17, 21);"> Nonce 防重复请求</font><font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">✅</font><font style="color:rgb(15, 17, 21);"> 限流规则已配置</font><font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">✅</font><font style="color:rgb(15, 17, 21);"> 配额规则已配置</font><font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">✅</font><font style="color:rgb(15, 17, 21);"> 审计日志已落库</font><font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">✅</font><font style="color:rgb(15, 17, 21);"> 异常场景已降级</font>

### <font style="color:rgb(15, 17, 21);">1.5 鉴权过滤器完整代码</font>
```java
@Component
@Slf4j
public class AiGatewayAuthFilter implements Filter {

    @Resource
    private SignService signService;

    @Resource
    private RateLimitService rateLimitService;

    @Resource
    private QuotaService quotaService;

    @Resource
    private AuditLogService auditLogService;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse resp = (HttpServletResponse) response;

        String ak = req.getHeader("X-AK");
        String timestamp = req.getHeader("X-Timestamp");
        String nonce = req.getHeader("X-Nonce");
        String signature = req.getHeader("X-Signature");
        String requestId = UUID.randomUUID().toString();

        try {
            // 1. 校验参数
            if (StrUtil.isBlank(ak) || StrUtil.isBlank(timestamp)
                    || StrUtil.isBlank(nonce) || StrUtil.isBlank(signature)) {
                resp.setStatus(HttpStatus.UNAUTHORIZED.value());
                resp.getWriter().write("缺少鉴权参数");
                return;
            }

            // 2. 验签
            boolean pass = signService.verify(ak, timestamp, nonce, getParams(req), signature);
            if (!pass) {
                resp.setStatus(HttpStatus.UNAUTHORIZED.value());
                resp.getWriter().write("签名验证失败");
                return;
            }

            // 3. 限流
            if (!rateLimitService.acquire(ak)) {
                resp.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
                resp.getWriter().write("请求过于频繁");
                return;
            }

            // 4. 配额检查
            if (!quotaService.checkQuota(ak, resolveModel(req), estimateTokens(req))) {
                resp.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
                resp.getWriter().write("配额不足");
                return;
            }

            // 5. 放行
            chain.doFilter(request, response);

            // 6. 异步审计
            auditLogService.saveAuditLog(requestId, ak, req);

        } catch (Exception e) {
            log.error("AI网关鉴权失败，requestId={}", requestId, e);
            resp.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
            resp.getWriter().write("服务繁忙");
        }
    }
}
```

---

## <font style="color:rgb(15, 17, 21);">2. 敏感词 + 权限 + 审计日志</font>
<font style="color:rgb(15, 17, 21);">AI 系统完整合规防护体系，三大核心模块：</font>

1. <font style="color:rgb(15, 17, 21);">内容合规：DFA 高性能敏感词过滤器，输入输出双重拦截</font>
2. <font style="color:rgb(15, 17, 21);">权限合规：三级权限管控，RAG 知识库多租户数据隔离</font>
3. <font style="color:rgb(15, 17, 21);">审计合规：AOP 自动采集全链路操作日志</font>

### <font style="color:rgb(15, 17, 21);">2.1 敏感词方案</font>
+ <font style="color:rgb(15, 17, 21);">方案一：本地 DFA 前缀树，轻量、无依赖</font>
+ <font style="color:rgb(15, 17, 21);">方案二：云内容安全 API，适合等保、公网业务、企业级场景</font>
+ <font style="color:rgb(15, 17, 21);">推荐生产：本地 DFA 第一层 + 云 API 第二层</font>

#### <font style="color:rgb(15, 17, 21);">2.1.1 DFA 敏感词过滤器完整代码</font>
```java
@Component
public class SensitiveWordFilter {

    private final Map<String, Object> dfaMap = new HashMap<>();

    @PostConstruct
    public void init() {
        List<String> sensitiveWords = SensitiveWordHolder.getAllSensitiveWords();
        for (String word : sensitiveWords) {
            addWord(word);
        }
    }

    private void addWord(String word) {
        Map<String, Object> currentMap = dfaMap;
        for (int i = 0; i < word.length(); i++) {
            char c = word.charAt(i);
            Object nextMap = currentMap.get(String.valueOf(c));
            if (nextMap == null) {
                Map<String, Object> newMap = new HashMap<>();
                newMap.put("isEnd", false);
                currentMap.put(String.valueOf(c), newMap);
                currentMap = newMap;
            } else {
                currentMap = (Map<String, Object>) nextMap;
            }
            if (i == word.length() - 1) {
                currentMap.put("isEnd", true);
            }
        }
    }

    public List<String> checkSensitiveWord(String text) {
        List<String> result = new ArrayList<>();
        for (int i = 0; i < text.length(); i++) {
            int len = matchWord(text, i);
            if (len > 0) {
                result.add(text.substring(i, i + len));
                i += len - 1;
            }
        }
        return result;
    }

    public String replaceSensitiveWord(String text) {
        List<String> words = checkSensitiveWord(text);
        for (String word : words) {
            String replace = "*".repeat(word.length());
            text = text.replace(word, replace);
        }
        return text;
    }
}
```

### <font style="color:rgb(15, 17, 21);">2.2 敏感词过滤核心原则</font>
+ <font style="color:rgb(15, 17, 21);">违规内容进不来、敏感信息出不去</font>
+ <font style="color:rgb(15, 17, 21);">输入输出双重过滤</font>
+ <font style="color:rgb(15, 17, 21);">敏感词库可配置，不要硬编码</font>
+ <font style="color:rgb(15, 17, 21);">校验日志必须留存</font>

### <font style="color:rgb(15, 17, 21);">2.3 三级权限管控</font>
+ <font style="color:rgb(15, 17, 21);">功能权限：普通员工/管理员/超级管理员</font>
+ <font style="color:rgb(15, 17, 21);">数据权限：检索知识库文档自动过滤所属部门</font>
+ <font style="color:rgb(15, 17, 21);">工具权限：限制角色可调用工具</font>

### <font style="color:rgb(15, 17, 21);">2.4 审计日志</font>
+ <font style="color:rgb(15, 17, 21);">用户提问、文档上传、向量库新增、工具调用、管理员配置修改全部记录</font>
+ <font style="color:rgb(15, 17, 21);">审计日志字段：userId、userName、deptId、operationType、content、ip、createTime、success</font>
+ <font style="color:rgb(15, 17, 21);">异步写入，不阻塞主流程</font>

### <font style="color:rgb(15, 17, 21);">2.5 三级权限管控 + 审计日志代码</font>
#### <font style="color:rgb(15, 17, 21);">登录用户上下文</font>
```java
@Component
public class SecurityContextHolder {
    private static final ThreadLocal<LoginUser> USER_THREAD_LOCAL = new ThreadLocal<>();

    public static void setCurrentUser(LoginUser loginUser) {
        USER_THREAD_LOCAL.set(loginUser);
    }

    public static LoginUser getCurrentUser() {
        return USER_THREAD_LOCAL.get();
    }

    public static void clear() {
        USER_THREAD_LOCAL.remove();
    }
}
```

#### <font style="color:rgb(15, 17, 21);">权限校验 AOP 切面</font>
```java
@Aspect
@Component
public class PermissionAspect {

    @Around("@annotation(requiresPermission)")
    public Object checkPermission(ProceedingJoinPoint point,
                                  RequiresPermission requiresPermission) throws Throwable {
        LoginUser user = SecurityContextHolder.getCurrentUser();

        if (!user.getRoles().contains(requiresPermission.role())) {
            throw new PermissionException("当前账号无此操作权限");
        }

        DataPermissionHelper.setCurrentDeptId(user.getDeptId());
        try {
            return point.proceed();
        } finally {
            DataPermissionHelper.clear();
        }
    }
}
```

#### <font style="color:rgb(15, 17, 21);">审计日志 AOP 切面</font>
```java
@Aspect
@Component
public class OperationLogAspect {

    @Autowired
    private OperationLogMapper logMapper;

    @Around("@annotation(operationLog)")
    public Object recordLog(ProceedingJoinPoint joinPoint,
                            OperationLog operationLog) throws Throwable {
        LoginUser user = SecurityContextHolder.getCurrentUser();
        OperationLog log = new OperationLog();
        log.setUserId(user.getUserId());
        log.setUserName(user.getUserName());
        log.setDeptId(user.getDeptId());
        log.setOperationType(operationLog.type());
        log.setContent(JSON.toJSONString(joinPoint.getArgs()));
        log.setIp(ServletUtil.getClientIP());
        log.setCreateTime(LocalDateTime.now());
        log.setSuccess(1);

        Object result;
        try {
            result = joinPoint.proceed();
        } catch (Throwable e) {
            log.setSuccess(0);
            throw e;
        } finally {
            CompletableFuture.runAsync(() -> logMapper.insert(log));
        }
        return result;
    }
}
```

### <font style="color:rgb(15, 17, 21);">2.6 新手落地必踩 5 个致命合规坑</font>
1. <font style="color:rgb(15, 17, 21);">只校验用户输入，忽略 AI 输出审核</font>
2. <font style="color:rgb(15, 17, 21);">权限仅前端控制，后端不做校验</font>
3. <font style="color:rgb(15, 17, 21);">敏感词库杂乱，误杀率过高</font>
4. <font style="color:rgb(15, 17, 21);">审计日志原样存储手机号、身份证</font>
5. <font style="color:rgb(15, 17, 21);">同步写入审计日志，拖慢问答接口</font>

---

## <font style="color:rgb(15, 17, 21);">3. 多租户权限隔离</font>
### <font style="color:rgb(15, 17, 21);">3.1 三种方案</font>
| <font style="color:rgb(15, 17, 21);">方案</font> | <font style="color:rgb(15, 17, 21);">实现方式</font> | <font style="color:rgb(15, 17, 21);">优势</font> | <font style="color:rgb(15, 17, 21);">劣势</font> | <font style="color:rgb(15, 17, 21);">适用场景</font> |
| --- | --- | --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">分集合/分库</font> | <font style="color:rgb(15, 17, 21);">每个部门单独建 Milvus 集合</font> | <font style="color:rgb(15, 17, 21);">隔离级别最高</font> | <font style="color:rgb(15, 17, 21);">运维成本高</font> | <font style="color:rgb(15, 17, 21);">强隔离、子公司/租户</font> |
| <font style="color:rgb(15, 17, 21);">标量字段过滤</font> | <font style="color:rgb(15, 17, 21);">文档打部门 ID、权限标签，检索时过滤</font> | <font style="color:rgb(15, 17, 21);">零侵入、成本低、灵活</font> | <font style="color:rgb(15, 17, 21);">依赖过滤逻辑</font> | <font style="color:rgb(15, 17, 21);">企业内部多部门、中小规模</font> |
| <font style="color:rgb(15, 17, 21);">行级权限管控</font> | <font style="color:rgb(15, 17, 21);">对接权限系统，逐文档配置</font> | <font style="color:rgb(15, 17, 21);">粒度最细</font> | <font style="color:rgb(15, 17, 21);">实现复杂、性能损耗</font> | <font style="color:rgb(15, 17, 21);">涉密文档、精细化权限</font> |


<font style="color:rgb(15, 17, 21);">推荐：绝大多数企业内部知识库，基于 Milvus 标量过滤最优。</font>

### <font style="color:rgb(15, 17, 21);">3.2 生产落地三步</font>
1. <font style="color:rgb(15, 17, 21);">文档入库注入权限元数据：</font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">deptId</font>`<font style="color:rgb(15, 17, 21);">、</font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">permissionLevel</font>`<font style="color:rgb(15, 17, 21);">、</font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">ownerId</font>`
2. <font style="color:rgb(15, 17, 21);">检索环节增加权限过滤，过滤条件下推到向量库层</font>
3. <font style="color:rgb(15, 17, 21);">两级权限管控：部门级 + 文档级</font>

### <font style="color:rgb(15, 17, 21);">3.3 标量字段过滤完整代码</font>
#### <font style="color:rgb(15, 17, 21);">文档入库注入权限元数据</font>
```java
@Service
public class DocumentService {

    @Autowired
    private EmbeddingStore<TextSegment> embeddingStore;

    @Autowired
    private EmbeddingModel embeddingModel;

    public void addFileDocumentWithAuth(MultipartFile file, String deptId,
                                         Integer permissionLevel, String ownerId) throws IOException {
        String fileName = file.getOriginalFilename();
        String rawText = DocumentParserUtil.extractText(file);
        String cleanText = DocumentParserUtil.cleanText(rawText);

        DocumentSplitter splitter = DocumentSplitters.recursive(500, 50);
        List<TextSegment> segments = splitter.split(Document.from(cleanText));

        segments.forEach(segment -> {
            segment.metadata().put("docName", fileName);
            segment.metadata().put("deptId", deptId);
            segment.metadata().put("permissionLevel", String.valueOf(permissionLevel));
            segment.metadata().put("ownerId", ownerId);
        });

        EmbeddingStoreIngestor.builder()
                .documentSplitter(splitter)
                .embeddingModel(embeddingModel)
                .embeddingStore(embeddingStore)
                .build()
                .ingest(segments);
    }
}
```

#### <font style="color:rgb(15, 17, 21);">检索时权限过滤（下推到向量库层）</font>
```java
public String chatWithAuth(String question, String currentDeptId, String currentUserId) {
    // 构造权限过滤条件：部门可见 + 全员可见 + 自己创建的私密文档
    Filter authFilter = Filter.or(
        Filter.eq("permissionLevel", "1"),
        Filter.and(
            Filter.eq("permissionLevel", "2"),
            Filter.eq("deptId", currentDeptId)
        ),
        Filter.and(
            Filter.eq("permissionLevel", "3"),
            Filter.eq("ownerId", currentUserId)
        )
    );

    // 向量检索时带上过滤条件，仅返回权限内的文档
    List<EmbeddingMatch<TextSegment>> matches = embeddingStore.findRelevant(
        embeddingModel.embed(question).content(),
        3,
        0.7,
        authFilter // 核心：权限过滤参数，在Milvus内部执行
    );

    // 拼接上下文、生成答案...
    return chatModel.generate(prompt);
}
```

**<font style="color:rgb(15, 17, 21);">关键点</font>**<font style="color:rgb(15, 17, 21);">：过滤条件是在 Milvus 内部执行的，不会把越权数据返回到 Java 服务层，从根源上避免了数据泄露。</font>

### <font style="color:rgb(15, 17, 21);">3.4 坑</font>
1. <font style="color:rgb(15, 17, 21);">过滤字段不建索引，检索性能暴跌</font>
2. <font style="color:rgb(15, 17, 21);">权限过滤放在生成环节，存在数据泄露风险</font>
3. <font style="color:rgb(15, 17, 21);">直接用前端传的部门 ID，存在越权漏洞</font>
4. <font style="color:rgb(15, 17, 21);">元数据类型不匹配，过滤条件失效</font>
5. <font style="color:rgb(15, 17, 21);">跨部门共享需求没预留</font>

---

## <font style="color:rgb(15, 17, 21);">4. 多租户 RAG：生产级无感隔离</font>
<font style="color:rgb(15, 17, 21);">目标：用户无感知、数据强隔离、检索高准确、运维低成本。</font>

### <font style="color:rgb(15, 17, 21);">4.1 四层隔离</font>
1. **<font style="color:rgb(15, 17, 21);">会话级租户绑定</font>**<font style="color:rgb(15, 17, 21);">：租户标识不依赖前端，从可信渠道解析</font>
    - <font style="color:rgb(15, 17, 21);">企业 IM 嵌入：企微/飞书/钉钉免登授权</font>
    - <font style="color:rgb(15, 17, 21);">专属域名/路径嵌入：网关解析 Host/URI</font>
    - <font style="color:rgb(15, 17, 21);">AK/SK 签名鉴权：第三方系统集成</font>
2. **<font style="color:rgb(15, 17, 21);">路由分发层</font>**<font style="color:rgb(15, 17, 21);">：分级隔离</font>
    - <font style="color:rgb(15, 17, 21);">普通租户：逻辑隔离</font>
    - <font style="color:rgb(15, 17, 21);">核心/涉密租户：物理隔离</font>
    - <font style="color:rgb(15, 17, 21);">超大型客户：独立向量库实例</font>
3. **<font style="color:rgb(15, 17, 21);">向量检索层</font>**<font style="color:rgb(15, 17, 21);">：元数据前置过滤 / 独立集合独立索引 / 二次兜底校验</font>
4. **<font style="color:rgb(15, 17, 21);">模型资源层隔离</font>**<font style="color:rgb(15, 17, 21);">：租户级模型配额、会话级流式输出隔离、租户级合规策略隔离</font>

### <font style="color:rgb(15, 17, 21);">4.2 数字人专属优化</font>
+ <font style="color:rgb(15, 17, 21);">多轮会话上下文强绑定</font>
+ <font style="color:rgb(15, 17, 21);">租户级数字人人设隔离</font>
+ <font style="color:rgb(15, 17, 21);">转人工自动路由</font>
+ <font style="color:rgb(15, 17, 21);">租户级缓存策略</font>

### <font style="color:rgb(15, 17, 21);">4.3 生产必踩 7 坑</font>
1. <font style="color:rgb(15, 17, 21);">线程上下文泄露串租户</font>
2. <font style="color:rgb(15, 17, 21);">租户越开越多，集合爆炸</font>
3. <font style="color:rgb(15, 17, 21);">权限变更后双库数据不一致</font>
4. <font style="color:rgb(15, 17, 21);">大模型生成串租户内容</font>
5. <font style="color:rgb(15, 17, 21);">缓存穿透导致越权访问</font>
6. <font style="color:rgb(15, 17, 21);">数字人形象音色串租户</font>
7. <font style="color:rgb(15, 17, 21);">对话日志混存，审计过不了</font>

### <font style="color:rgb(15, 17, 21);">4.4 租户饥饿问题解决方案</font>
**什么是租户饥饿**<font style="color:rgb(15, 17, 21);">：高峰期一个租户的大量请求占满模型配额，其他租户全部卡顿超时。这是多租户场景下的“吵闹邻居”问题——单个租户过度使用共享资源，导致其他租户无法获得应有的服务。</font>

**解决方案：配额 + 限流 + 舱壁隔离三级防护**

#### <font style="color:rgb(15, 17, 21);">第一级：租户级配额管理</font>
```java
@Service
public class TenantQuotaService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 检查租户配额
     */
    public boolean checkQuota(String tenantId, int tokenCount) {
        String key = "ai:quota:" + tenantId + ":" + LocalDate.now();
        Long used = redisTemplate.opsForValue().get(key);
        if (used == null) used = 0L;

        Long maxQuota = getMaxQuota(tenantId);
        return used + tokenCount <= maxQuota;
    }

    /**
     * 扣减配额
     */
    public void deductQuota(String tenantId, int tokenCount) {
        String key = "ai:quota:" + tenantId + ":" + LocalDate.now();
        redisTemplate.opsForValue().increment(key, tokenCount);
    }

    /**
     * 按租户等级返回不同配额
     */
    private Long getMaxQuota(String tenantId) {
        TenantConfig config = tenantConfigService.getByTenantId(tenantId);
        return switch (config.getLevel()) {
            case "VIP" -> 1000000L;
            case "STANDARD" -> 100000L;
            default -> 10000L;
        };
    }
}
```

#### <font style="color:rgb(15, 17, 21);">第二级：租户级限流（防止单租户占满）</font>
```java
public boolean acquire(String tenantId, int maxPermits) {
    String key = "ai:rate:" + tenantId;
    Long count = redisTemplate.opsForValue().increment(key);
    if (count == 1) {
        redisTemplate.expire(key, 1, TimeUnit.MINUTES);
    }
    return count != null && count <= maxPermits;
}
```

<font style="color:rgb(15, 17, 21);">建议配置：</font>

| <font style="color:rgb(15, 17, 21);">租户等级</font> | <font style="color:rgb(15, 17, 21);">每分钟请求上限</font> | <font style="color:rgb(15, 17, 21);">Token 配额（日）</font> |
| --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">普通租户</font> | <font style="color:rgb(15, 17, 21);">20 次</font> | <font style="color:rgb(15, 17, 21);">10,000</font> |
| <font style="color:rgb(15, 17, 21);">标准租户</font> | <font style="color:rgb(15, 17, 21);">50 次</font> | <font style="color:rgb(15, 17, 21);">100,000</font> |
| <font style="color:rgb(15, 17, 21);">VIP 租户</font> | <font style="color:rgb(15, 17, 21);">100 次</font> | <font style="color:rgb(15, 17, 21);">1,000,000</font> |


#### <font style="color:rgb(15, 17, 21);">第三级：舱壁隔离（ThreadPoolExecutor + Semaphore）</font>
```java
@Component
public class TenantBulkhead {

    private final Map<String, Semaphore> tenantSemaphores = new ConcurrentHashMap<>();

    private static final int DEFAULT_PERMITS = 5;

    public boolean tryAcquire(String tenantId) {
        Semaphore semaphore = tenantSemaphores.computeIfAbsent(
            tenantId, k -> new Semaphore(DEFAULT_PERMITS));
        return semaphore.tryAcquire();
    }

    public void release(String tenantId) {
        Semaphore semaphore = tenantSemaphores.get(tenantId);
        if (semaphore != null) semaphore.release();
    }
}
```

**<font style="color:rgb(15, 17, 21);">核心机制</font>**<font style="color:rgb(15, 17, 21);">：每个租户拥有独立的调用额度，耗尽后无法使用服务；租户级流量隔离，有效防止多租户间的资源争抢与相互干扰，避免个别租户的突发流量影响其他租户的正常使用</font><font style="color:rgb(15, 17, 21);">。</font>

**<font style="color:rgb(15, 17, 21);">超量处理策略</font>**<font style="color:rgb(15, 17, 21);">：</font>

| <font style="color:rgb(15, 17, 21);">使用率</font> | <font style="color:rgb(15, 17, 21);">处理策略</font> |
| --- | --- |
| <font style="color:rgb(15, 17, 21);">80%</font> | <font style="color:rgb(15, 17, 21);">提醒</font> |
| <font style="color:rgb(15, 17, 21);">90%</font> | <font style="color:rgb(15, 17, 21);">预警</font> |
| <font style="color:rgb(15, 17, 21);">100%</font> | <font style="color:rgb(15, 17, 21);">降级（切换小模型/返回缓存）</font> |
| <font style="color:rgb(15, 17, 21);">120%</font> | <font style="color:rgb(15, 17, 21);">限流（拒绝新请求）</font> |


---

## <font style="color:rgb(15, 17, 21);">5. 企业级私有化 AI 平台</font>
### <font style="color:rgb(15, 17, 21);">5.1 五层架构
</font><!-- 这是一张图片，ocr 内容为：智能客服 知识助手 业务应用层 代码助手 文档处理 接入网关层 监控 统一API 限流 鉴权 配额 运维监控体系 安全合规体系 对话引擎 对话调用 内容安全 RAG知识库 能力中台层 工具调用 PROMPT管理 弹性扩缩容 智能调度 容灾降级 多模型适配 模型服务层 量化加速 GPU集群 分布式存储 基础设施层 K8S编排 对象存储 -->
![](https://cdn.nlark.com/yuque/0/2026/png/21950178/1790046096084-12c067fe-61c2-4357-8ae3-f1b04fc4ac12.png)
### <font style="color:rgb(15, 17, 21);">5.2 各层职责</font>
1. **<font style="color:rgb(15, 17, 21);">基础设施层</font>**<font style="color:rgb(15, 17, 21);">：GPU 算力池化、存储、网络、K8s 编排</font>
2. **<font style="color:rgb(15, 17, 21);">模型服务层</font>**<font style="color:rgb(15, 17, 21);">：多模型统一适配、智能路由、弹性扩缩容、容灾降级、量化加速</font>
3. **<font style="color:rgb(15, 17, 21);">能力中台层</font>**<font style="color:rgb(15, 17, 21);">：RAG 知识库、对话引擎、工具调用、内容安全、Prompt 管理</font>
4. **<font style="color:rgb(15, 17, 21);">接入网关层</font>**<font style="color:rgb(15, 17, 21);">：统一协议、身份鉴权、流量防护、配额管理、全链路埋点</font>
5. **<font style="color:rgb(15, 17, 21);">业务应用层</font>**<font style="color:rgb(15, 17, 21);">：人事、研发、业务、运营等 AI 应用</font>

#### <font style="color:rgb(15, 17, 21);">5.2.1 Java 开发在各层的职责</font>
| <font style="color:rgb(15, 17, 21);">层级</font> | <font style="color:rgb(15, 17, 21);">Java 开发核心任务</font> | <font style="color:rgb(15, 17, 21);">不需要深入的部分</font> |
| --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">基础设施层</font> | <font style="color:rgb(15, 17, 21);">配合做服务容器化、K8s 部署配置、健康检查接口开发</font> | <font style="color:rgb(15, 17, 21);">GPU 驱动、算力调度算法、网络底层</font> |
| <font style="color:rgb(15, 17, 21);">模型服务层</font> | <font style="color:rgb(15, 17, 21);">实现模型路由层、统一 API 封装、配额与限流逻辑</font> | <font style="color:rgb(15, 17, 21);">模型本身的部署、量化、加速（由算法工程团队负责）</font> |
| <font style="color:rgb(15, 17, 21);">能力中台层</font> | **100% 是 Java 后端核心工作**<font style="color:rgb(15, 17, 21);">——接口设计、业务逻辑、性能优化、RAG 服务、对话引擎、工具调用框架、内容安全、Prompt 管理</font> | <font style="color:rgb(15, 17, 21);">—</font> |
| <font style="color:rgb(15, 17, 21);">接入网关层</font> | **核心主战场**<font style="color:rgb(15, 17, 21);">——鉴权过滤器、限流规则、配额管理、全链路埋点、协议适配（REST/SSE/WebSocket）</font> | <font style="color:rgb(15, 17, 21);">—</font> |
| <font style="color:rgb(15, 17, 21);">业务应用层</font> | <font style="color:rgb(15, 17, 21);">基于平台能力快速搭建业务 AI 应用，对接前端/企业微信/钉钉</font> | <font style="color:rgb(15, 17, 21);">—</font> |


**一句话总结**<font style="color:rgb(15, 17, 21);">：能力中台层和接入网关层是 Java 开发者的核心阵地，基础设施层和模型服务层只需懂交互边界和调用方式，不需要深入底层实现。</font>

### <font style="color:rgb(15, 17, 21);">5.3 两大横向支撑</font>
+ **<font style="color:rgb(15, 17, 21);">安全合规体系</font>**<font style="color:rgb(15, 17, 21);">：数据安全、权限管控、审计合规、内容安全</font>
+ **<font style="color:rgb(15, 17, 21);">运维监控体系</font>**<font style="color:rgb(15, 17, 21);">：GPU 监控、日志中心、成本核算、自动化运维</font>

### <font style="color:rgb(15, 17, 21);">5.4 落地路线</font>
1. <font style="color:rgb(15, 17, 21);">基础平台搭建：1-2 个月</font>
2. <font style="color:rgb(15, 17, 21);">能力完善推广：2-3 个月</font>
3. <font style="color:rgb(15, 17, 21);">平台成熟运营：3-6 个月</font>

### <font style="color:rgb(15, 17, 21);">5.5 架构设计核心原则</font>
+ <font style="color:rgb(15, 17, 21);">分层解耦</font>
+ <font style="color:rgb(15, 17, 21);">能力下沉</font>
+ <font style="color:rgb(15, 17, 21);">可扩展性</font>
+ <font style="color:rgb(15, 17, 21);">安全可控</font>
+ <font style="color:rgb(15, 17, 21);">成本优先</font>

---

## <font style="color:rgb(15, 17, 21);">6. 面试&简历亮点提炼</font>
### <font style="color:rgb(15, 17, 21);">6.1 中级开发（3-5 年）</font>
> <font style="color:rgb(15, 17, 21);">参与企业级私有化 AI 平台的能力中台设计与开发，负责 RAG 知识库、对话引擎模块的落地，实现多租户数据隔离。封装通用 AI 能力支撑多个业务应用接入，大幅提升业务 AI 需求落地效率。</font>
>

### <font style="color:rgb(15, 17, 21);">6.2 高级开发（5 年+）</font>
> <font style="color:rgb(15, 17, 21);">负责企业 AI 平台接入网关与能力中台层架构设计，实现多模型统一调用、智能路由、配额限流体系。设计多租户 RAG 知识库中台，支撑全公司十余个业务场景复用，整体大模型调用成本下降 40%。</font>
>

### <font style="color:rgb(15, 17, 21);">6.3 架构师岗</font>
> <font style="color:rgb(15, 17, 21);">主导企业级私有化 AI 平台整体架构设计，采用五层架构实现算力池化、能力中台化、接入统一化。建立多模型智能调度、全链路安全合规与成本管控体系，支撑全公司 AI 应用快速落地，算力资源利用率提升 50%，项目交付周期缩短 60%。</font>
>

