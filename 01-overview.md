# 01 总览与路线：Java+AI 转型与实战全景图
> 本文档为 Java+AI 转型与实战系列的总纲与导航。
目标：厘清 Java 程序员学 AI / 转 AI 的完整链路、模块顺序、学习路线与核心概念。
>

---

## 1. 全景导航
### 1.1 模块先后顺序
```mermaid
flowchart TB
    M0[第0篇 总纲导航] --> M1[第1篇 入门：基础认知与 SpringBoot 跑通]
    M1 --> M2[第2篇 核心：RAG 知识库构建与检索调优]
    M2 --> M3[第3篇 工程：生产排坑与降本增效]
    M3 --> M4[第4篇 架构：合规、多租户与私有化平台]
    M2 --> M5[第5篇 进阶：Agent 与 Function Calling]
    M4 --> M6[第6篇 冲刺：简历、面试与真实项目]
    M5 --> M6
```

### <font style="color:rgb(15, 17, 21);">1.2 学习路线</font>
| <font style="color:rgb(15, 17, 21);">阶段</font> | <font style="color:rgb(15, 17, 21);">目标</font> | <font style="color:rgb(15, 17, 21);">关键能力</font> | <font style="color:rgb(15, 17, 21);">对应模块</font> |
| --- | --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">入门</font> | <font style="color:rgb(15, 17, 21);">能跑通大模型对话</font> | <font style="color:rgb(15, 17, 21);">SpringBoot 对接、流式输出、多轮记忆</font> | <font style="color:rgb(15, 17, 21);">第1篇</font> |
| <font style="color:rgb(15, 17, 21);">核心</font> | <font style="color:rgb(15, 17, 21);">能做企业知识库</font> | <font style="color:rgb(15, 17, 21);">文档解析、切片、Milvus、混合检索、重排序</font> | <font style="color:rgb(15, 17, 21);">第2篇</font> |
| <font style="color:rgb(15, 17, 21);">工程</font> | <font style="color:rgb(15, 17, 21);">能上生产</font> | <font style="color:rgb(15, 17, 21);">限流降级熔断、SSE 稳定、降本、监控</font> | <font style="color:rgb(15, 17, 21);">第3篇</font> |
| <font style="color:rgb(15, 17, 21);">架构</font> | <font style="color:rgb(15, 17, 21);">能做平台</font> | <font style="color:rgb(15, 17, 21);">鉴权、合规、多租户、私有化 AI 平台</font> | <font style="color:rgb(15, 17, 21);">第4篇</font> |
| <font style="color:rgb(15, 17, 21);">进阶</font> | <font style="color:rgb(15, 17, 21);">能做智能体</font> | <font style="color:rgb(15, 17, 21);">Function Calling、Agent、Graph、多 Agent</font> | <font style="color:rgb(15, 17, 21);">第5篇</font> |
| <font style="color:rgb(15, 17, 21);">冲刺</font> | <font style="color:rgb(15, 17, 21);">能拿 offer</font> | <font style="color:rgb(15, 17, 21);">简历包装、面试题库、完整项目</font> | <font style="color:rgb(15, 17, 21);">第6篇</font> |


---

## <font style="color:rgb(15, 17, 21);">2. Java+AI 能力地图</font>
### <font style="color:rgb(15, 17, 21);">2.1 从 CURD 到 AI 应用工程师</font>
<font style="color:rgb(15, 17, 21);">AI 完整发展链路，从简单到智能：</font>

1. **<font style="color:rgb(15, 17, 21);">LLM</font>**<font style="color:rgb(15, 17, 21);">：能说话</font>
2. **<font style="color:rgb(15, 17, 21);">RAG</font>**<font style="color:rgb(15, 17, 21);">：有知识、不幻觉</font>
3. **<font style="color:rgb(15, 17, 21);">FunctionCall</font>**<font style="color:rgb(15, 17, 21);">：能调用单个 Java 函数</font>
4. **<font style="color:rgb(15, 17, 21);">ToolCall</font>**<font style="color:rgb(15, 17, 21);">：能编排多工具</font>
5. **<font style="color:rgb(15, 17, 21);">Skills</font>**<font style="color:rgb(15, 17, 21);">：有业务经验、会按流程干</font>
6. **<font style="color:rgb(15, 17, 21);">MCP</font>**<font style="color:rgb(15, 17, 21);">：标准连接、易集成</font>
7. **<font style="color:rgb(15, 17, 21);">Agent</font>**<font style="color:rgb(15, 17, 21);">：自主思考、自动完成任务</font>

<font style="color:rgb(15, 17, 21);">一句话总结：</font><font style="color:rgb(15, 17, 21);">
</font>**<font style="color:rgb(15, 17, 21);">LLM 是大脑，RAG 是书本，FunctionCall 是手，ToolCall 是手臂，Skills 是经验，MCP 是神经，Agent 是完整的 AI 员工。</font>**

### <font style="color:rgb(15, 17, 21);">2.2 AI 技术分层与 Java 类比</font>
| <font style="color:rgb(15, 17, 21);">AI 层</font> | <font style="color:rgb(15, 17, 21);">作用</font> | <font style="color:rgb(15, 17, 21);">Java 后端类比</font> |
| --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">LLM 大语言模型</font> | <font style="color:rgb(15, 17, 21);">AI 的大脑</font> | <font style="color:rgb(15, 17, 21);">接口服务</font> |
| <font style="color:rgb(15, 17, 21);">RAG 检索增强</font> | <font style="color:rgb(15, 17, 21);">解决幻觉、知识过时</font> | <font style="color:rgb(15, 17, 21);">数据库 / 搜索引擎</font> |
| <font style="color:rgb(15, 17, 21);">FunctionCall 函数调用</font> | <font style="color:rgb(15, 17, 21);">调用 Java 接口底层机制</font> | <font style="color:rgb(15, 17, 21);">单个方法调用</font> |
| <font style="color:rgb(15, 17, 21);">ToolCall 工具调用</font> | <font style="color:rgb(15, 17, 21);">多工具编排</font> | <font style="color:rgb(15, 17, 21);">服务编排</font> |
| <font style="color:rgb(15, 17, 21);">Skills 技能</font> | <font style="color:rgb(15, 17, 21);">业务流程变成可复用专家经验</font> | <font style="color:rgb(15, 17, 21);">业务手册 / 最佳实践</font> |
| <font style="color:rgb(15, 17, 21);">MCP 协议</font> | <font style="color:rgb(15, 17, 21);">统一工具连接</font> | <font style="color:rgb(15, 17, 21);">统一接口规范</font> |
| <font style="color:rgb(15, 17, 21);">Agent 智能体</font> | <font style="color:rgb(15, 17, 21);">自动干活</font> | <font style="color:rgb(15, 17, 21);">自动化流程引擎</font> |


### <font style="color:rgb(15, 17, 21);">2.3 LLM 天生三大缺陷</font>
1. **<font style="color:rgb(15, 17, 21);">幻觉</font>**<font style="color:rgb(15, 17, 21);">：胡说八道</font>
2. **<font style="color:rgb(15, 17, 21);">知识截止</font>**<font style="color:rgb(15, 17, 21);">：训练后不更新</font>
3. **<font style="color:rgb(15, 17, 21);">无执行能力</font>**<font style="color:rgb(15, 17, 21);">：不能查库、不能调接口</font>

<font style="color:rgb(15, 17, 21);">后续所有技术，都是为了补这三个缺陷。</font>

<!-- 这是一个文本绘图，源码为：flowchart LR -->
    A[LLM 三大缺陷] --> B[幻觉]
    A --> C[知识截止]
    A --> D[无执行能力]
    B --> E[RAG]
    C --> E
    D --> F[Function Calling / Tool Calling]
    F --> G[Agent]
    E --> G -->
![](https://cdn.nlark.com/yuque/__mermaid_v3/bbcffc86fb44ed565c9eba615370f429.svg)

---

## <font style="color:rgb(15, 17, 21);">3. 转型路线与阶段</font>
| <font style="color:rgb(15, 17, 21);">阶段</font> | <font style="color:rgb(15, 17, 21);">学习周期</font> | <font style="color:rgb(15, 17, 21);">薪资涨幅/年薪</font> | <font style="color:rgb(15, 17, 21);">岗位</font> | <font style="color:rgb(15, 17, 21);">核心能力</font> |
| --- | --- | --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">入门级 AI 应用开发</font> | <font style="color:rgb(15, 17, 21);">1-2 个月</font> | <font style="color:rgb(15, 17, 21);">10%-20%</font> | <font style="color:rgb(15, 17, 21);">Java 开发（AI 方向）</font> | <font style="color:rgb(15, 17, 21);">调 API、基础 RAG、集成现有系统</font> |
| <font style="color:rgb(15, 17, 21);">进阶级 AI 系统开发</font> | <font style="color:rgb(15, 17, 21);">3-6 个月</font> | <font style="color:rgb(15, 17, 21);">30%-50%</font> | <font style="color:rgb(15, 17, 21);">AI 应用开发工程师</font> | <font style="color:rgb(15, 17, 21);">生产级 RAG、高可用、合规、多端接入</font> |
| <font style="color:rgb(15, 17, 21);">高级 AI 解决方案架构师</font> | <font style="color:rgb(15, 17, 21);">6-12 个月</font> | <font style="color:rgb(15, 17, 21);">50W+</font> | <font style="color:rgb(15, 17, 21);">AI 解决方案架构师</font> | <font style="color:rgb(15, 17, 21);">方案设计、多模型选型、成本管控、跨部门推动</font> |
| <font style="color:rgb(15, 17, 21);">资深 AI 平台架构师</font> | <font style="color:rgb(15, 17, 21);">1-2 年</font> | <font style="color:rgb(15, 17, 21);">80W+</font> | <font style="color:rgb(15, 17, 21);">AI 平台架构师</font> | <font style="color:rgb(15, 17, 21);">企业级 AI 平台、分布式、算力调度、多租户</font> |


### <font style="color:rgb(15, 17, 21);">3.1 入门级 AI 应用开发（1-2 个月）</font>
+ <font style="color:rgb(15, 17, 21);">熟练调用主流大模型 API，能实现基础 AI 对话功能</font>
+ <font style="color:rgb(15, 17, 21);">理解 RAG 基本原理，能搭建简单知识库问答</font>
+ <font style="color:rgb(15, 17, 21);">能把 AI 能力无缝集成到现有 Java 业务系统</font>
+ <font style="color:rgb(15, 17, 21);">知识点：大模型基础概念、Java 对接通义千问/文心一言、LangChain4j 基础、RAG 最简实现</font>
+ <font style="color:rgb(15, 17, 21);">实战项目：简单企业内部知识库问答</font>

### <font style="color:rgb(15, 17, 21);">3.2 进阶级 AI 系统开发（3-6 个月）</font>
+ <font style="color:rgb(15, 17, 21);">能独立开发完整生产级 RAG 知识库系统</font>
+ <font style="color:rgb(15, 17, 21);">懂性能优化、高可用设计、合规管控</font>
+ <font style="color:rgb(15, 17, 21);">知识点：RAG 深度优化、本地大模型部署与量化、企业级能力、多端接入</font>
+ <font style="color:rgb(15, 17, 21);">实战项目：企业智能客服系统、内部知识库平台</font>

### <font style="color:rgb(15, 17, 21);">3.3 高级 AI 解决方案架构师（6-12 个月）</font>
+ <font style="color:rgb(15, 17, 21);">能根据业务需求设计完整 AI 落地方案</font>
+ <font style="color:rgb(15, 17, 21);">懂多模型选型、成本管控、架构设计</font>
+ <font style="color:rgb(15, 17, 21);">能跨部门推动项目落地，拿到可量化业务结果</font>

### <font style="color:rgb(15, 17, 21);">3.4 资深 AI 平台架构师（1-2 年）</font>
+ <font style="color:rgb(15, 17, 21);">能设计企业级统一 AI 平台</font>
+ <font style="color:rgb(15, 17, 21);">懂分布式架构、算力调度、多模型管理</font>
+ <font style="color:rgb(15, 17, 21);">能支撑全公司 AI 应用落地</font>

---

## <font style="color:rgb(15, 17, 21);">4. 避坑指南：转型最容易走的 4 条弯路</font>
1. **<font style="color:rgb(15, 17, 21);">放弃 Java，去学 Python 卷算法</font>**<font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">从零卷算法卷不过科班，反而丢掉工程优势。正确方向：立足 Java 技术栈，走 AI 工程化落地路线。</font>
2. **<font style="color:rgb(15, 17, 21);">只会调 API，不做完整项目</font>**<font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">只会调 API 没有竞争力。正确方向：做完整生产级项目，加入高可用、权限、合规、性能优化。</font>
3. **<font style="color:rgb(15, 17, 21);">光学不练，收藏等于学会</font>**<font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">面试一问细节就露馅。正确方向：动手做完整智能客服项目，写进简历。</font>
4. **<font style="color:rgb(15, 17, 21);">盲目追新，丢了根基</font>**<font style="color:rgb(15, 17, 21);">
</font><font style="color:rgb(15, 17, 21);">Java 工程能力是根基，AI 是增量能力。根基稳了，增量才有价值。</font>

---

## <font style="color:rgb(15, 17, 21);">5. 面试高频考点概览</font>
### <font style="color:rgb(15, 17, 21);">5.1 基础概念题</font>
+ <font style="color:rgb(15, 17, 21);">RAG 的完整流程是什么？</font>
+ <font style="color:rgb(15, 17, 21);">RAG 有哪些常见优化方法？</font>
+ <font style="color:rgb(15, 17, 21);">流式输出怎么实现？</font>
+ <font style="color:rgb(15, 17, 21);">Function Calling 的执行流程是什么？</font>

### <font style="color:rgb(15, 17, 21);">5.2 工程场景题</font>
+ <font style="color:rgb(15, 17, 21);">大模型接口不稳定怎么处理？</font>
+ <font style="color:rgb(15, 17, 21);">怎么做限流降级？</font>
+ <font style="color:rgb(15, 17, 21);">怎么控制调用成本？</font>
+ <font style="color:rgb(15, 17, 21);">怎么保证数据安全？</font>
+ <font style="color:rgb(15, 17, 21);">怎么实现部门级权限隔离？</font>

### <font style="color:rgb(15, 17, 21);">5.3 项目深挖题</font>
+ <font style="color:rgb(15, 17, 21);">你做的项目准确率多少？</font>
+ <font style="color:rgb(15, 17, 21);">遇到过最棘手的问题是什么？</font>
+ <font style="color:rgb(15, 17, 21);">怎么解决的？</font>
+ <font style="color:rgb(15, 17, 21);">如果再做一次你会怎么优化？</font>

### <font style="color:rgb(15, 17, 21);">5.4 方案设计题</font>
+ <font style="color:rgb(15, 17, 21);">给你一个业务场景，你怎么设计整体方案？</font>
+ <font style="color:rgb(15, 17, 21);">怎么做内部知识库？</font>
+ <font style="color:rgb(15, 17, 21);">怎么做智能客服？</font>
+ <font style="color:rgb(15, 17, 21);">怎么做文档总结系统？</font>

