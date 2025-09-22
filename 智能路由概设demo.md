# 

# 数据服务层（智能引擎）模块概要设计说明书

## 页眉

- 公司名称：深信服科技股份有限公司

- 文件模板编号：SFRD-TS-02-3.4

- 密级：B级(项目组内公开)

- 发布生效日期：待确认

- 模板现行版本：V3.4

## 0. 设计方法参考

### 0.1 概设输入

- 总体设计中模块划分边界：数据服务层（智能引擎）位于业务系统与数据库之间，作为中间层承担“SPL解析-语句转换-数据库路由-执行调度-结果返回”全流程，不侵入业务系统核心逻辑，不直接管理数据库底层资源（依赖基础连接池模块）。

- 模块需实现的对外接口：SPL执行接口（dsl_execute_spl）、SPL语法校验接口（dsl_validate_spl）、自动化测试接口（dsl_test_route、dsl_test_convert）、进程间结果传递消息接口（msg_dsl_result）。

- 模块关联的系统流程：业务侧SPL请求发起→SPL语法校验→AST生成→数据库路由规则匹配→目标SQL转换→数据库连接获取→SQL执行→结果格式化→业务侧结果返回。

### 0.2 撰写前准备

- 核心方案图示：需绘制《数据服务层模块交互图》（含子模块间数据流）、《SPL执行主流程图》（含异常分支），提交专家评审形式为“邮件同步文档+线下评审会（邀请架构师、测试负责人、业务模块负责人参会）”。

- 专家评审输出：需形成评审意见记录表，明确“通过/需修改”结论，修改项需跟踪闭环。

### 0.3 概设输出

- 模块全局数据结构：含TaskContext（任务上下文）、DBInfo（数据库信息）、ResultData（结果数据）等结构体定义，覆盖全流程数据传递需求。

- 接口流程图：含对外API接口（输入输出关系、参数校验逻辑）、进程间消息接口（字段传递路径）的可视化图示。

- 实现接口的所有内部函数：含各子模块核心函数（如dsl_parse_spl、dsl_convert_ast_to_sql）的功能描述、参数列表、流程图。

- 子模块划分及文档合并机制：

    - 子模块划分：按功能拆分为SPL解析子模块、语句转换子模块、路由子模块、执行调度子模块、结果处理子模块，符合“高内聚低耦合”原则。

    - 子模块负责人：SPL解析子模块（张XX）、语句转换子模块（李XX）、路由子模块（王XX）、执行调度子模块（刘XX）、结果处理子模块（陈XX）。

    - 文档合并机制：各子模块负责人按“周同步”节奏更新子模块设计文档，提交至项目共享库（路径：//server/dsl_design/），主文档负责人每周五合并汇总，标注版本号（如V1.0.1_202405XX）。

### 0.4 模块设计方法参考

|表格名称|用途|使用场景|
|---|---|---|
|函数定义表|规范内部函数的名称、参数、返回值、约束|6.1.5~6.5.5 各子模块函数列表章节|
|数据结构表|定义全局/子模块数据结构的字段、类型|5.2 全局数据结构定义、6.1.4~6.5.4 子模块数据结构章节|
|需求跟踪表|关联需求点与设计章节，验证需求覆盖度|2.1 需求跟踪章节|
|方案选型评估表|量化对比核心方案，支撑选型决策|4.2 方案选型章节|
## 1. 介绍

### 1.1 目的

本设计旨在解决业务系统与数据库强耦合的问题：当前业务系统需针对Mysql、Clickhouse、PG、Starrocks等不同数据库编写专属SQL，新增/替换数据库时需大规模修改业务代码；同时缺乏统一的查询入口，难以实现多库协同查询。通过构建数据服务层（智能引擎），支持用户输入SPL语句，自动转换为目标数据库SQL并路由执行，最终返回统一格式结果，实现业务与数据库选型解耦，降低多库适配成本。

### 1.2 定义和缩写

- 【SPL】：结构化查询语言扩展（Structured Query Language Extension），本模块支持的统一查询语言，具备跨数据库查询适配能力，语法兼容ANSI SPL核心规范。

- 【DSL】：数据服务层（Data Service Layer），即本设计的核心模块，承担SPL解析、语句转换、路由、执行、结果处理功能。

- 【IE】：智能引擎（Intelligent Engine），数据服务层的别称，强调其“自动转换、智能路由”特性。

- 【API】：应用程序接口（Application Programming Interface），本模块对外提供的SPL执行、校验等接口。

- 【RPC】：远程过程调用（Remote Procedure Call），本模块与基础连接池模块的通信方式。

- 【AST】：抽象语法树（Abstract Syntax Tree），SPL解析后生成的结构化数据，用于后续语句转换。

- 【STRIDE】：微软威胁建模方法，涵盖欺骗（Spoofing）、篡改（Tampering）、否认（Repudiation）、信息泄露（Information Disclosure）、拒绝服务（Denial of Service）、权限提升（Elevation of Privilege）六大威胁类型，用于模块安全性分析。

- 【DFX】：设计for X（Design for X），包含可测试性、可调试性、可运维性等特性，指导模块非功能需求设计。

### 1.3 参考和引用

1. 需求文档

    - 《数据服务层（智能引擎）需求规格说明书》第2.1节 核心功能需求、第3.1节 性能需求、第4.1节 安全性需求。

2. 总体设计文档

    - 《XX项目总体设计说明书》第4.3节 模块划分（明确DSL模块边界）、第5.2节 接口规划（定义DSL对外接口）。

3. 其他文档

    - 《SPL语法规范（V1.0）》（[http://docs.sangfor.org/pages/viewpage.action?pageId=78901234](http://docs.sangfor.org/pages/viewpage.action?pageId=78901234)）

    - 《多数据库语句转换技术指南》（[http://docs.sangfor.org/pages/viewpage.action?pageId=78901235](http://docs.sangfor.org/pages/viewpage.action?pageId=78901235)）

    - 《软件可测试性技术规范》（[http://docs.sangfor.org/pages/viewpage.action?pageId=42148775](http://docs.sangfor.org/pages/viewpage.action?pageId=42148775)）

## 2. 设计任务书

### 2.1 需求跟踪

|编号|需求点名称|需求点说明验收条件|自检|
|---|---|---|---|
|1.1|SPL输入支持|功能：支持ANSI SPL核心语法（select、join、where、limit等），错误输入返回明确错误码（如1001：语法错误）及位置提示；性能：单条SPL解析耗时≤100ms，并发解析PPS≥1000|通过，见6.1.2 处理流程、6.1.3 关键算法描述|
|1.2|多数据库语句转换|功能：支持将SPL转换为Mysql（5.7+/8.0+）、Clickhouse（22.0+/23.0+）、PG（12+/14+）、Starrocks（2.0+/3.0+）SQL；性能：转换耗时≤150ms，转换准确率≥99.5%（核心语法场景）|通过，见6.2.2 处理流程、6.2.3 关键算法描述|
|1.3|数据库路由|功能：支持按“表归属”“数据类型”“数据库状态”路由（如user表→Mysql、log表→Clickhouse），路由规则可配置；性能：路由决策耗时≤50ms，路由准确率100%（规则配置正确场景）|通过，见6.3.2 处理流程、6.3.3 关键算法描述|
|1.4|执行结果返回|功能：统一结果格式为JSON（含code、msg、data字段），支持结果分页（默认100条/页）；性能：结果格式化耗时≤50ms，单条请求端到端响应时间≤500ms（无大数据量场景）|通过，见6.5.2 处理流程、6.5.3 关键算法描述|
|1.5|接口安全性|功能：对外API需Token认证（Token有效期1小时），防SPL注入（过滤特殊字符如;、--），敏感结果字段（如password）脱敏；性能：认证耗时≤20ms，不影响主流程性能|通过，见[4.6.1.3](4.6.1.3) 安全设计、6.1.2 处理流程|
### 2.2 模块整体目标

|编号|目标项概述|对应的评审标准|自检|
|---|---|---|---|
|2.1|性能|并发用户数≥500；并发任务数≥1000；SPL解析PPS≥1000；单任务端到端响应时间≤500ms（数据量≤1万行）；数据库连接建立时间≤100ms|通过|
|2.2|资源开销|内存占用≤500MB（并发1000任务场景）；磁盘占用≤100MB（日志+配置文件，日志保留15天）；CPU使用率≤80%（并发1000任务场景）|通过|
|2.3|安全性要求|禁止未授权访问API接口（Token认证失败返回401）；防SPL注入（注入测试用例通过率100%）；敏感数据传输加密（HTTPS）；日志无明文密码|通过|
|2.4|可靠性要求|进程故障自动恢复时间≤10s；服务可用性≥99.9%（月度）；任务失败重试机制（默认3次，间隔1s）；配置文件损坏自动恢复（依赖备份）|通过|
|2.5|可测试性|提供自动化测试接口覆盖100%核心功能；支持模拟数据库异常（如离线、执行超时）；测试用例通过率≥95%（核心场景）|通过|
|2.6|可调试性|日志支持分级（DEBUG/INFO/ERROR），含task_id便于追踪单任务全流程；提供调试接口查询任务状态（如dsl_query_task_status）；支持AST节点打印（DEBUG模式）|通过|
|2.7|可运维性|日志存储路径：/var/log/dsl/，按天切割；提供监控接口（dsl_monitor）返回并发数、失败率、资源占用；支持配置热加载（无需重启服务）|通过|
|2.8|可扩展性|新增数据库适配时，仅需开发转换插件（接口标准化），核心代码改动≤5%；路由规则支持动态新增（JSON配置）；并发任务数支持配置扩展（最大10000）|通过|
|2.9|可复用性|SPL解析子模块可独立复用至其他数据查询场景；数据库连接管理逻辑可复用至其他需多库连接的模块；提供SDK（含Java/Python版本）供业务侧调用|通过|
|2.10|兼容性|支持数据库版本：Mysql 5.7+/8.0+、Clickhouse 22.0+/23.0+、PG 12+/14+、Starrocks 2.0+/3.0+；支持操作系统：x86_64/ARM64 Linux（CentOS 7+/Ubuntu 18.04+）|通过|
|2.11|其它|错误码体系统一（1000-1999为模块专属）；支持中英文错误提示（可配置）；单任务最大处理数据量≤10万行（超出返回分页提示）|通过|
### 2.3 流程要求

- 本文档需外部评审（评审参与方：架构师、测试负责人、业务模块负责人、安全经理）。

- 审核人签名：____________________

- 时间：____年__月__日

## 3. 对外接口

### 3.1 API接口

#### 3.1.1 普通API

|名称|功能|参数（参数名、类型、方向、说明）|返回值（JSON格式）|
|---|---|---|---|
|dsl_execute_spl|接收SPL语句，执行查询并返回统一格式结果|spl_content（string，in，SPL语句，非空）；db_route_rule（json，in，可选，路由规则，如{"table_route":{"user":"mysql"}}）；page_num（int，in，可选，页码，默认1）；page_size（int，in，可选，页大小，默认100）|code（int，0成功，1001-1999失败）；msg（string，提示信息）；data（json，结果数据，含total、rows、columns）；task_id（string，任务ID，用于追踪）|
|dsl_validate_spl|校验SPL语句语法正确性|spl_content（string，in，SPL语句，非空）|code（int，0成功，1001语法错误）；msg（string，错误位置/提示）；task_id（string，校验任务ID）|
|dsl_query_task_status|查询任务执行状态|task_id（string，in，任务ID，非空）|code（int，0成功）；msg（string，提示）；status（int，0未开始，1执行中，2成功，3失败）；progress（int，进度百分比，仅执行中有效）|
#### 3.1.2 自动化测试接口

- test_dsl_route：测试数据库路由正确性，参数为spl_content（SPL语句）、expected_db_type（预期数据库类型），返回code（0成功/1002路由错误）、actual_db_type（实际路由结果）。

- test_dsl_convert：测试语句转换正确性，参数为spl_content（SPL语句）、target_db_type（目标数据库类型），返回code（0成功/1003转换错误）、converted_sql（转换后的SQL）。

#### 3.1.3 openAPI

本模块无对外openAPI接口，若后续需扩展，将采用swagger格式归档，遵循规范链接（[https://wiki.sangfor.com/pages/viewpage.action?pageId=64519005](https://wiki.sangfor.com/pages/viewpage.action?pageId=64519005)）。

### 3.2 消息接口

|名称|说明|字段说明|
|---|---|---|
|msg_dsl_result|二进制消息，用于执行调度子模块向结果处理子模块传递数据库执行结果，需校验task_id唯一性（防重复处理）|字段名：msg_type（类型：int，长度：8bit，说明：消息类型，固定为0x01）；task_id（类型：string，长度：64byte，说明：唯一任务ID）；result_code（类型：int，长度：32bit，说明：执行结果码，0成功）；result_data_len（类型：int，长度：32bit，说明：result_data字段长度）；result_data（类型：string，长度：变长，说明：数据库原始执行结果，JSON格式）|
## 4. 概要说明

### 4.1 背景描述

#### 4.1.1 工作原理

数据服务层（智能引擎）的核心工作原理是“统一入口-分层处理-解耦适配”：

1. 统一入口：业务侧通过标准化API（如dsl_execute_spl）传入SPL语句，模块生成唯一task_id标识任务，初始化TaskContext（存储任务全流程数据）；

2. 分层处理：

    - SPL解析层：校验SPL语法，生成AST（抽象语法树），过滤注入风险；

    - 路由层：结合配置规则（如“表-库”映射）和数据库状态（在线/离线），确定目标数据库；

    - 语句转换层：基于AST和目标数据库类型，通过“规则映射+模板填充”生成专属SQL；

    - 执行层：从基础连接池模块获取数据库连接，执行SQL并获取原始结果；

    - 结果处理层：将原始结果格式化为统一JSON，敏感字段脱敏后返回业务侧；

3. 解耦适配：新增数据库时，仅需开发“语句转换插件”和“路由规则”，无需修改核心逻辑；业务侧无需关注数据库类型，仅需编写SPL语句。

关键需求设计思路：为满足“高可扩展性”，采用“插件化架构”设计语句转换子模块；为满足“高可靠性”，引入任务重试机制和进程故障自动恢复（依赖系统公共监控机制）。

#### 4.1.2 应用场景

- 场景1：业务系统多库协同查询

    - 场景描述：企业ERP系统需关联查询Mysql的“用户表”和Clickhouse的“操作日志表”，传统方式需业务侧写两套SQL并手动关联结果；

    - 操作步骤：1. 业务侧编写SPL语句（select [u.id](u.id), u.name, l.op_time from user u join log l on [u.id](u.id)=l.user_id where u.status=1）；2. 调用dsl_execute_spl接口传入SPL；3. 接收返回结果；

    - 可能事件：1. SPL语法错误（返回code=1001及错误位置）；2. Clickhouse离线（返回code=1004及“目标库离线”提示）；

    - 处理结果：模块自动路由至Mysql和Clickhouse，执行关联查询后返回统一JSON结果，业务侧无需手动处理多库数据。

- 场景2：新增Starrocks数据库适配

    - 场景描述：业务侧需将部分报表查询迁移至Starrocks（OLAP数据库），需快速实现SPL到Starrocks SQL的转换；

    - 操作步骤：1. 开发Starrocks语句转换插件（实现标准化转换接口）；2. 配置路由规则（如“report_*表→Starrocks”）；3. 调用test_dsl_convert接口验证转换正确性；4. 业务侧正常调用dsl_execute_spl；

    - 可能事件：插件兼容性问题（转换失败，返回code=1003）；

    - 处理结果：适配完成后，SPL可自动转为Starrocks SQL，转换准确率≥99.5%，业务侧无感知。

#### 4.1.3 对手分析

|竞品/参考产品|优点|缺点|借鉴点|
|---|---|---|---|
|竞品A（某SQL中间件）|支持多数据库连接池管理，成熟稳定，社区活跃|不支持SPL，需业务侧编写多库专属SQL，无统一查询入口|借鉴其连接池管理逻辑，优化数据库连接复用效率；参考其高可用设计（主备连接池）|
|竞品B（某自定义查询引擎）|支持自定义查询语言，灵活度高|语句转换准确率低（≤95%），路由规则配置复杂，不支持国产数据库（如Starrocks）|强化本模块语句转换算法（AST+规则双重校验），简化路由规则配置（JSON格式），优先支持国产数据库|
|业界参考（Apache Calcite）|支持SQL解析、优化，可扩展性强|需二次开发适配SPL，无数据库路由和结果处理能力|借鉴其AST构建逻辑，优化SPL解析效率；参考其插件化架构，设计本模块的语句转换插件|
### 4.2 方案选型

方案选型参考：[http://200.200.1.35/designing/decision/scheme_select.html](http://200.200.1.35/designing/decision/scheme_select.html)

#### 4.2.1 核心需求项确定

核心需求项为“SPL解析引擎选型”“语句转换方案”“数据库路由机制”，三者直接影响模块性能、可扩展性和开发成本，非核心需求项（如日志存储）采用业界通用方案（本地文件+按天切割）。

#### 4.2.2 方案概述

- 方案1：SPL解析用Antlr+自定义语法规则，语句转换用Freemarker模板引擎，路由用Drools规则引擎；

- 方案2：SPL解析用自研递归下降引擎，语句转换用AST直接映射（规则配置化），路由用JSON配置化规则；

- 方案3：SPL解析用第三方开源库（SPL Parser V2.0），语句转换用“模板+映射”混合方案，路由用数据库标签匹配。

#### 4.2.3 选型评估表

|评估准则|权重|方案1（SPL解析：Antlr；转换：Freemarker；路由：Drools）|方案2（SPL解析：自研；转换：AST映射；路由：JSON配置）|方案3（SPL解析：开源库；转换：混合；路由：标签匹配）|
|---|---|---|---|---|
|||打分（1-5）|加权得分|打分（1-5）|
|性能|40%|3.5|1.4|4.5|
|可扩展性|30%|4.0|1.2|4.0|
|开发成本|20%|3.0|0.6|3.5|
|维护成本|10%|3.0|0.3|4.0|
|最终得分|-|-|3.5|-|
#### 4.2.4 备选方案说明

|备选方案名称|本方案的优点|本方案的风险和缺点|
|---|---|---|
|方案1（Antlr+Freemarker+Drools）|成熟稳定，社区支持强；Drools规则引擎支持复杂路由逻辑|性能较差（解析+转换耗时≥300ms）；Drools学习成本高；依赖第三方组件多，部署复杂|
|方案2（自研+AST映射+JSON配置）|性能最优（解析+转换耗时≤200ms）；维护成本低（配置化）；无强依赖第三方组件|自研SPL解析引擎初期bug率可能较高；复杂SPL语句（如多层子查询）转换需额外开发规则|
|方案3（开源库+混合+标签匹配）|开发周期短（开源库可直接用）；标签匹配路由简单易懂|开源库扩展性差（新增SPL语法需改源码）；混合转换方案一致性难保证；不支持复杂路由规则|
#### 4.2.5 最终选择

选择方案2，理由：

1. 最终得分最高（4.1），性能最优（满足“单任务响应≤500ms”需求），可扩展性与方案1持平；

2. 维护成本低（JSON配置化），后续新增数据库时仅需补充转换规则，无需修改引擎；

3. 风险规避：自研解析引擎初期仅支持核心SPL语法，单元测试覆盖率≥90%；复杂SPL转换通过“规则迭代+灰度验证”逐步完善。

### 4.3 静态结构

#### 4.3.1 子模块划分及职责

|子模块名称|核心职责|细化软件单元|与其他子模块的关系|
|---|---|---|---|
|SPL解析子模块|SPL语法校验、注入过滤、AST生成|语法分析单元、注入检测单元、AST构建单元|输出AST至语句转换子模块；接收TaskContext，处理后返回至路由子模块|
|语句转换子模块|基于AST和目标数据库类型，生成专属SQL；管理转换规则（新增/修改）|转换规则管理单元、AST映射单元、SQL构建单元|接收路由子模块传入的“AST+目标库类型”；输出SQL至执行调度子模块|
|路由子模块|解析路由规则、校验数据库状态、确定目标数据库；管理路由规则|规则解析单元、数据库状态校验单元、目标库选择单元|接收SPL解析子模块的TaskContext；输出“目标库信息（DBInfo）”至语句转换子模块|
|执行调度子模块|从基础连接池获取数据库连接、执行SQL、处理执行异常（重试/失败）|连接管理单元、SQL执行单元、异常处理单元|接收语句转换子模块的SQL；输出原始执行结果至结果处理子模块；依赖基础连接池模块|
|结果处理子模块|格式化原始结果（JSON）、敏感字段脱敏、分页处理；返回结果至业务侧|结果解析单元、脱敏处理单元、格式转换单元|接收执行调度子模块的原始结果；输出统一格式结果至业务侧；接收TaskContext更新状态|
#### 4.3.2 公共数据结构

- 结构名：TaskContext（任务上下文）

- 说明：存储任务全流程数据，供所有子模块共享，确保任务状态一致性；

- 结构定义（类C风格）：

```C

```

- 设计意图：通过统一的上下文结构体，避免子模块间数据传递混乱，便于问题追踪（通过task_id关联全流程数据）。

### 4.4 对软件总体架构的影响

|情况分类|是否对总设有影响|具体说明|
|---|---|---|
|有一定影响（涉及≤3个模块）|是|1. 业务模块：需将原直连数据库的接口改为调用本模块API，涉及“用户中心”“报表中心”2个业务模块；2. 基础连接池模块：本模块依赖其提供数据库连接，需新增“多库连接池管理”接口（如get_db_connection_by_type）；3. 无其他核心模块改动，总体架构分层（业务层-数据服务层-数据库层）保持不变|
|新增/调整模块对总设无影响|否|-|
|有较大影响（需重新设计总设）|否|-|
### 4.5 概要流程

#### 4.5.1 SPL执行主流程

1. 业务侧调用dsl_execute_spl接口，传入spl_content、db_route_rule（可选）等参数；

2. 模块生成task_id，初始化TaskContext（status=0，error_code=0）；

3. SPL解析子模块：

    - 注入检测单元过滤SPL中的注入风险字符（如;、--、union）；

    - 语法分析单元校验SPL语法，错误则设置status=6、error_code=1001、error_msg，跳转至步骤8；

    - AST构建单元生成AST，填充至TaskContext，设置status=1；

4. 路由子模块：

    - 规则解析单元解析db_route_rule（无则用默认配置）；

    - 数据库状态校验单元查询目标数据库在线状态（调用基础监控接口）；

    - 目标库选择单元确定目标数据库，填充DBInfo至TaskContext，错误（如无匹配库、库离线）则设置status=6、error_code=1002/1004，跳转至步骤8；设置status=2；

5. 语句转换子模块：

    - 转换规则管理单元加载目标数据库的转换规则；

    - AST映射单元将AST转换为目标SQL；

    - SQL构建单元优化SQL（如Mysql添加limit优化），填充至TaskContext，转换失败则设置status=6、error_code=1003，跳转至步骤8；设置status=3；

6. 执行调度子模块：

    - 连接管理单元从基础连接池获取数据库连接（重试3次，间隔1s）；

    - SQL执行单元执行SQL，获取原始结果，填充raw_result至TaskContext，执行失败则设置status=6、error_code=1004，跳转至步骤8；设置status=4；

7. 结果处理子模块：

    - 结果解析单元解析raw_result，脱敏敏感字段（如password替换为***）；

    - 格式转换单元将结果格式化为JSON，填充fmt_result至TaskContext；设置status=5；

8. 模块返回fmt_result（含code、msg、data）及task_id至业务侧。

#### 4.5.2 数据库路由流程

1. 路由子模块接收SPL解析子模块传递的TaskContext（含spl_content、ast）；

2. 规则解析单元解析spl_content中的表名（如从“select * from user”中提取user表）；

3. 匹配默认路由规则（JSON配置，如{"table_route":{"user":"mysql","log":"clickhouse","report":"starrocks","pg_table":"pg"}}）；

4. 数据库状态校验单元调用基础监控接口（monitor_db_status），查询匹配数据库的在线状态；

5. 若数据库在线，填充DBInfo（db_type、ip、port等）至TaskContext，返回TaskContext至语句转换子模块；

6. 若数据库离线，查询备用数据库（如mysql离线则查mysql_slave），存在则填充DBInfo，不存在则设置error_code=1004。

#### 4.5.3 SPL语法校验流程

1. 业务侧调用dsl_validate_spl接口，传入spl_content；

2. 模块生成校验任务ID（task_id），初始化简易TaskContext（仅含spl_content、task_id）；

3. SPL解析子模块的语法分析单元执行语法校验：

    - 按ANSI SPL核心语法规则逐行校验；

    - 若校验通过，返回code=0、msg=“语法正确”、task_id；

    - 若校验失败，定位错误位置（如“第2行第5列：缺少from关键字”），返回code=1001、msg=错误描述、task_id。

### 4.6 关键特性设计

#### 4.6.1 安全性设计

##### [4.6.1.1](4.6.1.1) 安全兜底机制覆盖

需满足产品已合入的安全兜底项（必填项）：接口认证、输入校验、权限控制、敏感数据保护，参考模板链接（[https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=520981623](https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=520981623)），并提交《模块概要设计安全兜底机制合入checklist.xlsx》至附件。若不满足某兜底项（如暂不支持权限细粒度控制），需由安全经理分析确认，向吴义和（11582）+研发主管发备案申请，邮件结果上传至文档附件。

##### [4.6.1.2](4.6.1.2) 威胁建模分析

由产线安全经理确认需执行威胁建模，采用STRIDE方法，风险识别结果如下：

|威胁类型|风险描述|风险等级|影响范围|
|---|---|---|---|
|欺骗（Spoofing）|未授权用户伪造Token调用API接口|高|接口安全性|
|篡改（Tampering）|攻击者篡改SPL语句（如添加drop table）|高|数据库数据安全|
|否认（Repudiation）|攻击者执行恶意操作后否认（无日志记录）|中|审计追溯|
|信息泄露（Information Disclosure）|结果数据含敏感信息（如身份证号）未脱敏|高|用户数据隐私|
|拒绝服务（DoS）|攻击者发送大量复杂SPL，导致模块CPU占用100%|中|模块可用性|
|权限提升（Elevation of Privilege）|攻击者通过API获取数据库高权限（如root）|低|数据库权限控制|
参考学习资源（[https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=90780583](https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=90780583)），后续将输出《SFRD-SDL-TMSD-1.5_数据服务层（智能引擎）威胁建模分析.xlsx》。

##### [4.6.1.3](4.6.1.3) 安全设计

针对上述威胁，设计如下安全机制：

- 防欺骗：API接口采用Token认证，Token生成规则为“用户ID+时间戳+密钥”（SHA256加密），有效期1小时；Token验证失败返回401，单日失败次数≥10次则封禁IP（1小时）；

- 防篡改：SPL输入过滤（基于正则表达式过滤drop、truncate、delete等危险关键字，特殊字符转义）；执行SQL前校验“表操作权限”（如业务侧仅允许select，禁止update/delete）；

- 防否认：所有API调用日志含“用户ID、IP、task_id、操作时间、SPL语句”，日志保留180天，支持审计查询；

- 防信息泄露：敏感字段脱敏（配置文件定义敏感字段列表，如password、id_card，返回时替换为***）；结果数据传输采用HTTPS加密；

- 防DoS：设置SPL复杂度阈值（如嵌套子查询≤3层、表关联≤2张），超出则拒绝执行；并发任务数限流（默认500，可配置），超出则返回1005（“系统繁忙，请稍后重试”）；

- 防权限提升：数据库连接采用“最小权限账号”（如仅select权限），禁止使用root账号；连接信息加密存储（配置文件中密码AES加密）。

参考安全需求清单（[https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=190536710](https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=190536710)）。

##### [4.6.1.4](4.6.1.4) 预使用组件合规性

使用Mend扫描器扫描所有开源组件（如JSON解析库cJSON、HTTP服务库libcurl），禁止使用“严重高危漏洞组件”（CVSS评分≥9.0）、“开源协议风险分数≥65分组件”（如GPLv3协议组件）。若存在合规风险组件，需按[4.6.1.1](4.6.1.1)流程备案，扫描指导参考（[https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=461416029](https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=461416029)）。

#### 4.6.2 可靠性设计

##### [4.6.2.1](4.6.2.1) 承载载体可靠

- 进程服务：模块进程纳入系统公共监控机制（如systemd），监控指标包括“进程存活状态、CPU使用率、内存占用”；进程故障（如崩溃、无响应）时，监控机制10s内自动重启，重启后加载备份的TaskContext（定时10s持久化至/var/lib/dsl/task_backup/）；

- 服务冗余：数据库连接池采用“主备冗余”设计，主连接池（默认使用）故障时，5s内切换至备连接池；连接池大小动态调整（最小10，最大200，根据并发任务数自动扩容/缩容）；

- 数据/文件：配置文件（/etc/dsl/config.json）每日凌晨3点自动备份至/var/backup/dsl/，保留30天；TaskContext每10s增量备份，避免任务数据丢失；

- 开源组件：选用成熟稳定的开源组件（如cJSON V1.7.15，无高危漏洞），组件版本锁定，避免兼容性问题。

##### [4.6.2.2](4.6.2.2) 周边无影响

- 资源隔离：通过系统cgroup机制限制模块资源：CPU使用率≤80%（单核心）、内存占用≤500MB、磁盘IO速率≤10MB/s（日志写入）；超出限制时触发告警（调用基础告警接口send_alert），不影响其他模块（如业务模块、基础连接池模块）；

- 异常隔离：子模块异常（如语句转换子模块崩溃）时，采用“故障隔离”机制，仅终止当前任务，不影响其他任务执行；异常子模块自动重启（5s内），重启后重新加载配置。

##### [4.6.2.3](4.6.2.3) 业务流程可靠（FMEA分析）

|子业务功能|失效模式|失效原因|影响程度（1-5）|发生概率（1-5）|风险优先级（影响×概率）|改进措施|
|---|---|---|---|---|---|---|
|SPL解析|解析超时（>100ms）|SPL语句复杂（如嵌套子查询≥5层）|3|2|6|设置解析超时时间（100ms），超时则返回1006（“SPL语句过复杂”）；限制嵌套子查询≤3层|
|语句转换|转换错误（生成无效SQL）|转换规则未覆盖特殊语法（如Clickhouse数组函数）|4|2|8|新增“规则校验”步骤，转换后校验SQL语法（调用数据库explain语句）；定期更新转换规则|
|数据库路由|路由错误（路由至错误数据库）|路由规则配置错误（如“user表→clickhouse”）|5|1|5|路由前增加“规则校验”（如检测user表是否存在于目标数据库）；配置变更需二次确认|
|SQL执行|执行失败（数据库离线）|数据库宕机、网络中断|5|2|10|执行前校验数据库状态；增加重试机制（3次，间隔1s）；配置备用数据库|
|结果处理|敏感字段未脱敏|敏感字段列表未更新（如新增phone字段未添加）|4|1|4|定期同步业务侧敏感字段列表；结果返回前增加“脱敏校验”（基于正则表达式匹配）|
参考指导链接（[https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=67890123](https://docs.atrust.sangfor.com/pages/viewpage.action?pageId=67890123)）及FMEA模板（《SFRD-FMEA-1.2_业务流程可靠性分析.xlsx》）。

#### 4.6.3 可测试性设计

- 高难度测试流程解耦：并发性能测试（如1000用户同时执行SPL）依赖大流量环境，解耦方法为搭建“模拟多库测试环境”（基于Docker部署Mysql、Clickhouse等数据库，模拟10万级数据量），提供流量生成工具（dsl_load_test）模拟并发请求；

- 自动化测试覆盖：开发自动化测试用例（基于JUnit/PyTest），覆盖100%核心功能（SPL解析、转换、路由、执行、结果返回），测试用例与代码提交挂钩（CI流程自动执行测试，通过率≥95%方可提交）；

- 测试数据准备：提供测试数据生成脚本（[generate_test_data.sh](generate_test_data.sh)），可生成不同场景的SPL语句（简单查询、关联查询、复杂子查询）和数据库表数据，避免手动准备数据的繁琐性。

参考《软件可测试性技术规范》（[http://docs.sangfor.org/pages/viewpage.action?pageId=42148775](http://docs.sangfor.org/pages/viewpage.action?pageId=42148775)）。

#### 4.6.4 可调试性设计

- 高难度调试流程解决方案：针对“高并发下任务状态异常”（如部分任务卡在“执行中”），设计“全流程日志追踪”：日志中包含task_id，可通过task_id查询该任务在各子模块的处理日志（如“grep 'task_123' /var/log/dsl/dsl.log”）；支持DEBUG模式，打印AST节点详情、SQL执行过程；

- 调试接口支持：提供调试接口dsl_debug_task，参数为task_id，返回该任务的全流程数据（SPL、AST、SQL、结果），便于定位问题；

- 异常数据保留：任务失败时，自动保留失败时的TaskContext（存储至/var/log/dsl/error_task/），包含错误码、错误位置、原始数据，便于复现问题。

#### 4.6.5 可运维性设计

- 安装部署：提供一键部署脚本（[deploy_dsl.sh](deploy_dsl.sh)），自动安装依赖组件（如libcurl、cJSON）、配置系统服务（systemd）、初始化配置文件；支持版本升级（[upgrade_dsl.sh](upgrade_dsl.sh)），升级时自动备份配置文件，避免数据丢失；

- 日志管理：日志存储路径为/var/log/dsl/，按天切割（如dsl_20240501.log），保留15天；日志级别可通过配置文件动态调整（DEBUG/INFO/ERROR），INFO级别包含task_id、操作结果，ERROR级别包含错误码、错误堆栈；

- 故障定位：提供故障定位工具（[dsl_troubleshoot.sh](dsl_troubleshoot.sh)），输入task_id或错误码，自动分析日志并输出问题原因（如“错误码1004：Mysql数据库离线，建议检查数据库状态”）；支持远程日志查询（通过SSH或Web接口）；

- 监控告警：提供监控接口dsl_monitor，返回模块状态（如“running”）、并发任务数、失败率、资源占用（CPU/内存）；对接系统监控平台（如Zabbix），设置告警阈值（如失败率≥5%、内存≥450MB），触发告警时通过邮件/短信通知运维人员。

#### 4.6.6 可扩展性设计

- 新增数据库适配：设计标准化的“语句转换插件接口”（含init()、convert()、destroy()方法），新增数据库（如Hive）时，仅需开发插件并放置于/plugins目录，模块启动时自动加载，无需修改核心代码；路由规则支持动态新增（通过API dsl_add_route_rule），无需重启服务；

- 功能扩展：预留“扩展钩子”（如SPL解析后钩子、结果返回前钩子），后续新增功能（如SPL语句缓存、结果导出至Excel）时，可通过钩子接入，不侵入核心流程；

- 性能扩展：并发任务数支持配置扩展（最大10000），数据库连接池大小动态调整，支持多实例部署（通过负载均衡器分发请求），满足高并发场景需求。

#### 4.6.7 可复用性设计

- 可使用的公用代码：复用系统公共组件，如日志工具（sys_log）、配置解析工具（sys_config）、HTTP服务工具（sys_http）、加密工具（sys_crypto），避免重复开发；复用基础连接池模块的连接管理逻辑，无需重新实现多库连接；

- 可产出的公用代码：将SPL解析子模块封装为独立SDK（支持C/Java/Python），供其他数据查询场景复用；将数据库路由逻辑封装为公用组件（route_util），供其他需多库路由的模块使用；

- 接口标准化：对外API遵循RESTful规范，参数和返回值格式统一，便于业务侧复用调用逻辑；内部子模块接口（如TaskContext传递）标准化，便于后续子模块重构或替换。

#### 4.6.8 系统隐私设计

- 敏感信息收集：仅收集业务侧传入的SPL语句和必要的任务数据（如task_id），不收集无关信息（如用户行为数据）；收集敏感信息（如数据库账号密码）时，在配置文档中明确告知用途（“用于数据库连接”）和保存方式（“加密存储，不对外泄露”）；

- 敏感信息存储：数据库账号密码在配置文件中采用AES加密存储，密钥通过环境变量传入，避免硬编码；TaskContext中的敏感数据（如密码）在持久化时脱敏，日志中不打印敏感信息；

- 敏感信息传输：API调用采用HTTPS加密，避免数据在传输过程中被窃取；进程间消息（如msg_dsl_result）中的敏感结果字段脱敏后传输；

- 隐私合规：遵循《个人信息保护法》《数据安全法》，明确数据保留期限（任务数据保留7天，日志保留15天），到期自动删除；支持数据删除请求（通过API dsl_delete_task_data），删除后不可恢复。

#### 4.6.9 跨平台设计

- 支持平台：明确支持x86_64和ARM64架构的Linux系统，包括CentOS 7+/8+、Ubuntu 18.04+/20.04+、Debian 10+；

- 差异处理：

    - 代码层面：平台无关代码（如SPL解析、语句转换）放置于/common目录，平台相关代码（如系统调用、硬件资源监控）放置于/platform/x86_64和/platform/arm64目录，通过条件编译（#ifdef **x86_64**）切换；

    - 数据结构：采用固定大小的数据类型（如uint32_t、int64_t），避免不同平台下数据类型长度差异；数据传输时统一使用网络字节序（大端序），解决大小端问题；

- 编译部署：提供跨平台编译脚本（[build_cross_platform.sh](build_cross_platform.sh)），支持在x86_64环境下编译ARM64版本；部署脚本自动识别平台架构，加载对应版本的依赖组件。

### 4.7 方案风险分析

#### 4.7.1 关键环节风险及解决方法

- 风险1：自研SPL解析引擎初期bug率高，导致解析失败率≥5%；

解决方法：1. 初期仅支持ANSI SPL核心语法（覆盖80%业务场景），复杂语法后续迭代；2. 单元测试覆盖率≥90%，新增语法需补充测试用例；3. 灰度发布（先部署至测试环境，验证1周无问题后发布至生产）；4. 提供“降级开关”，解析失败时自动切换至Antlr引擎（方案1备用）。

- 风险2：高并发（≥1000任务）下数据库连接池耗尽，导致执行失败率上升；

解决方法：1. 连接池动态扩容（最大200），基于并发任务数自动调整；2. 增加连接复用机制（同一数据库的任务共享连接）；3. 限流保护（并发任务数≥1000时，返回1005“系统繁忙”）；4. 配置备用连接池，主池耗尽时切换至备池。

- 风险3：语句转换准确率未达99.5%，导致生成无效SQL；

解决方法：1. 建立“转换规则库”，定期更新（每月1次），覆盖新语法；2. 转换后增加SQL语法校验（调用数据库explain语句，不执行SQL）；3. 收集生产环境转换失败案例，优化规则；4. 提供“手动修正”接口，允许业务侧修正转换后的SQL。

#### 4.7.2 现阶段无法消除的风险

|风险描述|跟进责任人|计划跟进时间|风险影响范围|缓解措施|
|---|---|---|---|---|
|自研SPL解析引擎长期维护成本高（需持续适配SPL新语法）|张XX（SPL解析子模块负责人）|每季度评估1次，持续跟进|SPL解析子模块|1. 建立语法迭代计划（每季度新增1-2个语法特性）；2. 文档化语法规则，降低维护难度；3. 调研开源SPL解析库新版本，评估替换可行性|
|新增小众数据库（如Greenplum）适配时，转换规则开发周期长（≥1周）|李XX（语句转换子模块负责人）|新增数据库时触发评估|语句转换子模块|1. 设计“通用转换模板”，小众数据库可基于模板快速修改；2. 提前调研主流小众数据库语法，预置基础规则；3. 与数据库厂商合作，获取语法文档|



## 5. 数据结构设计

### 5.1 配置文件定义

|名称|作用|默认值|取值范围|
|---|---|---|---|
|spl_max_nesting|SPL语句最大嵌套子查询层数|3|[1, 5]|
|concurrent_task_limit|最大并发任务数|1000|[100, 10000]|
|task_timeout|单任务超时时间（秒）|30|[10, 300]|
|retry_count|SQL执行重试次数|3|[1, 5]|
|retry_interval|重试间隔时间（秒）|1|[0.5, 5]|
|log_level|日志级别|INFO|[DEBUG, INFO, ERROR]|
|log_retention_days|日志保留天数|15|[7, 30]|
|sensitive_fields|敏感字段列表（逗号分隔）|password,id_card|字符串列表|
|db_connection_pool_size|数据库连接池大小|50|[10, 200]|
|route_rules_path|路由规则配置文件路径|/etc/dsl/route_rules.json|合法文件路径|
|convert_plugins_path|语句转换插件路径|/usr/lib/dsl/plugins/|合法目录路径|

**配置文件升级流程**：  
当配置文件从低版本（如V1.0）升级至V2.0时，系统将自动执行以下操作：  
1. 读取旧配置文件中`max_concurrent_tasks`字段，映射至新配置的`concurrent_task_limit`；  
2. 新增字段（如`sensitive_fields`）采用默认值初始化；  
3. 备份旧配置文件至`/var/backup/dsl/config_v1.0.json`，生成新配置文件`/etc/dsl/config.json`；  
4. 升级完成后，通过`dsl_config_validate`接口校验新配置合法性，若校验失败则回滚至旧配置。

### 5.2 全局数据结构定义

|结构名|说明|结构定义（类C风格）|字段说明|
|---|---|---|---|
|TaskContext|任务全流程上下文，贯穿所有子模块，存储任务状态及数据|```ctypedef struct {    char task_id[64];          // 任务唯一标识    int status;                // 任务状态：0-未开始，1-解析中，2-路由中，3-转换中，4-执行中，5-处理中，6-已完成    int error_code;            // 错误码：0-成功，1001-语法错误，1002-路由错误等    char error_msg[256];       // 错误描述信息    char spl_content[4096];    // 原始SPL语句    void* ast;                 // 抽象语法树（AST）指针    DBInfo target_db;          // 目标数据库信息    char converted_sql[8192];  // 转换后的目标SQL    RawResult raw_result;      // 数据库原始执行结果    FmtResult fmt_result;      // 格式化后的结果    long long create_time;     // 任务创建时间戳（ms）    long long update_time;     // 任务最后更新时间戳（ms）    int retry_count;           // 当前重试次数} TaskContext;```|task_id：[64字节] 由UUID生成，唯一标识任务；<br>status：[0-6] 表示任务所处阶段；<br>error_code：[0, 1001-1999] 0表示成功，1001-1999为模块错误码；<br>spl_content：[≤4096字节] 存储用户输入的SPL语句；<br>ast：指向AST结构体的指针，仅在解析后有效；<br>target_db：目标数据库信息，路由子模块填充；<br>converted_sql：[≤8192字节] 语句转换子模块生成的SQL；<br>raw_result：数据库返回的原始结果；<br>fmt_result：结果处理子模块生成的格式化结果；<br>create_time/update_time：[毫秒级时间戳] 用于任务超时判断；<br>retry_count：[0-3] 记录当前执行重试次数，超过阈值则终止|
|DBInfo|数据库连接信息及状态|```ctypedef struct {    char db_type[16];          // 数据库类型：mysql、clickhouse、pg、starrocks    char ip[32];               // 数据库IP地址    int port;                  // 数据库端口号    char db_name[64];          // 数据库名称    char username[64];         // 登录用户名    char password[128];        // 登录密码（加密存储）    int status;                // 状态：0-离线，1-在线    char charset[32];          // 字符集：如utf8mb4    int connection_timeout;    // 连接超时时间（秒）} DBInfo;```|db_type：[字符串] 支持的数据库类型；<br>ip：[IPv4/IPv6字符串] 数据库服务器地址；<br>port：[1-65535] 数据库服务端口；<br>db_name：[字符串] 目标数据库名称；<br>username/password：[字符串] 登录凭证，password采用AES加密；<br>status：[0-1] 由监控模块实时更新；<br>charset：[字符串] 数据交互字符集；<br>connection_timeout：[1-30] 连接建立超时阈值|
|RawResult|数据库执行返回的原始结果|```ctypedef struct {    int row_count;             // 结果行数    int col_count;             // 结果列数    char**col_names;           // 列名数组（长度=col_count）    char***data;               // 二维数据数组（row_count×col_count）    long long execute_time;    // 执行耗时（ms）    int has_more;              // 是否有更多数据：0-无，1-有} RawResult;```|row_count：[≥0] 实际返回的记录数；<br>col_count：[≥1] 结果列数；<br>col_names：动态分配的字符串数组，存储列名；<br>data：动态分配的二维字符串数组，存储原始数据；<br>execute_time：[≥0] SQL执行耗时；<br>has_more：用于分页场景，标识是否有未返回数据|
|FmtResult|格式化后的输出结果|```ctypedef struct {    int code;                  // 结果码：0-成功，非0-失败    char msg[256];             // 结果描述    int total;                 // 总记录数（分页场景）    int page_num;              // 当前页码    int page_size;             // 每页记录数    char* data;                // JSON格式的结果数据（动态分配）} FmtResult;```|code：与TaskContext.error_code一致；<br>msg：用户可读的结果描述（支持中英文）；<br>total：[≥0] 符合条件的总记录数；<br>page_num：[≥1] 当前页码；<br>page_size：[1-1000] 每页记录数；<br>data：JSON字符串，格式为`{"columns":[],"rows":[]}`|
|RouteRule|数据库路由规则|```ctypedef struct {    char table_name[64];       // 表名（支持通配符*）    char db_type[16];          // 目标数据库类型    char priority[16];         // 优先级：primary、secondary    char condition[256];       // 路由条件（如"date>2024-01-01"）} RouteRule;```|table_name：[字符串] 匹配的表名，如"user"或"log_*"；<br>db_type：对应DBInfo.db_type；<br>priority：主备路由优先级；<br>condition：额外路由条件，支持简单表达式|


## 6. 流程设计

### 6.1 SPL解析子模块

#### 6.1.1 静态结构
- **核心职责**：负责SPL语句的语法校验、注入风险过滤及抽象语法树（AST）生成，为后续转换和路由提供结构化输入。
- **细化软件单元**：
  - 语法分析单元：基于ANSI SPL核心规范，实现语法规则校验及错误定位；
  - 注入检测单元：通过正则匹配和语义分析，识别并过滤恶意注入语句；
  - AST构建单元：将合法的SPL语句转换为结构化AST，包含查询类型、表名、条件、字段等信息。

#### 6.1.2 处理流程
1. 接收业务侧传入的SPL语句及TaskContext，初始化任务状态为“解析中”（status=1）；
2. 注入检测单元执行预处理：
   - 过滤`drop`、`truncate`、`delete`等危险关键字（区分关键字与字段名）；
   - 转义特殊字符（如`;`、`--`、`'`），防止SQL注入；
   - 若检测到高风险注入，设置error_code=1005，跳转至步骤6；
3. 语法分析单元逐行校验SPL语法：
   - 校验关键字合法性（如`select`后必须跟字段或`*`）；
   - 校验语句结构完整性（如`where`后需有条件表达式）；
   - 若语法错误，定位错误位置（行号、列号），设置error_code=1001及错误描述，跳转至步骤6；
4. AST构建单元生成AST：
   - 解析SPL中的表名、字段、过滤条件、排序规则等元素；
   - 以树形结构存储元素间关系（如`join`关联的表及关联条件）；
   - 将AST指针存入TaskContext；
5. 更新TaskContext状态为“解析完成”（status=2），传递至路由子模块；
6. 标记任务状态为“已完成”（status=6），返回错误信息至业务侧。

#### 6.1.3 关键算法描述
- **SPL语法解析算法**：采用递归下降分析法，基于预设的SPL语法规则（BNF范式）构建语法树：
  - **常见场景性能**：单条简单SPL（如`select id from user where status=1`）解析耗时≤50ms；
  - **最差场景**：包含3层嵌套子查询的复杂SPL解析耗时≤100ms（符合需求1.1的性能指标）；
  - **影响**：通过语法规则预编译和关键字哈希表优化，最差场景出现概率≤0.5%，不影响整体并发性能。
- **注入检测算法**：结合特征匹配与语义分析：
  - 特征库包含200+常见注入模式（如`1=1--`、`union select`）；
  - 语义分析排除“字段名包含关键字”的误判（如`select drop_status from task`）；
  - 检测准确率≥99.9%，误判率≤0.01%。

#### 6.1.4 数据结构定义
|结构名|说明|结构定义（类C风格）|字段说明|
|---|---|---|---|
|ASTNode|AST节点，存储SPL语句的语法元素|```ctypedef struct ASTNode {    enum NodeType {        SELECT_NODE, JOIN_NODE, WHERE_NODE, LIMIT_NODE, ...    } type;                   // 节点类型    char value[256];          // 节点值（如字段名、条件值）    struct ASTNode* parent;   // 父节点指针    struct ASTNode* children; // 子节点链表    int child_count;          // 子节点数量} ASTNode;```|type：节点类型，覆盖SPL所有语法元素；<br>value：节点对应的具体值；<br>parent/children：构建树形结构；<br>child_count：子节点数量，用于遍历控制|
|InjectPattern|注入检测特征模式|```ctypedef struct {    char pattern[128];        // 注入特征正则表达式    int risk_level;           // 风险等级：1-低，2-中，3-高} InjectPattern;```|pattern：用于匹配注入语句的正则表达式；<br>risk_level：风险等级，3级需直接拦截|

#### 6.1.5 函数列表
|函数名|功能|参数及返回值|前后置约束|
|---|---|---|---|
|dsl_parse_spl|驱动SPL解析全流程，协调注入检测、语法分析和AST构建|参数：[in] const char* spl_content（SPL语句），[in/out] TaskContext* ctx（任务上下文）<br>返回值：int（0-成功，非0-失败，对应error_code）|前置：spl_content非空，ctx已初始化；<br>后置：成功时ctx->ast非空，失败时ctx->error_code已设置|
|check_injection|检测SPL语句中的注入风险|参数：[in] const char* spl_content，[out] char* err_msg（错误描述）<br>返回值：int（0-无风险，1-有风险）|前置：spl_content非空，err_msg缓冲区大小≥256字节；<br>后置：有风险时err_msg包含风险描述|
|validate_syntax|校验SPL语法合法性|参数：[in] const char* spl_content，[out] int* line（错误行号），[out] int* col（错误列号），[out] char* err_msg<br>返回值：int（0-合法，1-不合法）|前置：spl_content非空，line、col、err_msg已分配；<br>后置：不合法时line、col、err_msg均有效|
|build_ast|基于合法SPL语句构建AST|参数：[in] const char* spl_content，[out] ASTNode**ast（输出AST根节点）<br>返回值：int（0-成功，1-失败）|前置：spl_content已通过语法校验；<br>后置：成功时*ast为有效指针，需调用destroy_ast释放|
|destroy_ast|释放AST节点占用的内存|参数：[in] ASTNode* ast<br>返回值：void|前置：ast为build_ast生成的有效指针；<br>后置：所有节点内存已释放，ast置NULL|

#### 6.1.6 设计要点检视
- **调试难题解决方案**：DEBUG模式下，通过`print_ast`函数打印AST节点树（含类型、值、父子关系），辅助定位解析逻辑错误；
- **测试难题解决方案**：构建包含1000+用例的SPL测试集（覆盖语法正确/错误、注入攻击等场景），通过自动化测试验证解析准确性；
- **自动化测试支持**：提供`test_parse_spl`接口，输入SPL语句返回解析结果（AST结构或错误信息），支持CI流程集成；
- **扩展特性支持**：预留SPL语法扩展接口，可通过配置文件新增关键字（如`with cube`），无需修改核心解析逻辑；
- **异常检测恢复**：解析超时（>100ms）时，自动终止当前解析并返回1006错误，释放内存资源；
- **工作量估算**：12人天（语法规则设计3天，算法实现5天，测试4天）。

### 6.2 语句转换子模块

#### 6.2.1 静态结构
- **核心职责**：基于AST和目标数据库类型，将SPL语句转换为目标数据库兼容的SQL，支持Mysql、Clickhouse、PG、Starrocks等多类型数据库。
- **细化软件单元**：
  - 转换规则管理单元：加载、更新数据库专属转换规则（如函数映射、语法差异）；
  - AST映射单元：遍历AST节点，根据目标数据库类型替换语法元素（如`limit m,n`在PG中转为`limit n offset m`）；
  - SQL构建单元：将映射后的节点重组为完整SQL，优化语句性能（如添加索引提示）。

#### 6.2.2 处理流程
1. 接收路由子模块传递的TaskContext（含AST和target_db），设置任务状态为“转换中”（status=3）；
2. 转换规则管理单元加载目标数据库的转换规则：
   - 从`convert_plugins_path`路径加载对应数据库的插件（如`mysql_plugin.so`）；
   - 规则包含函数映射（如SPL的`date_format`→Clickhouse的`formatDateTime`）、语法差异（如字符串拼接符`||`在Mysql中需启用`PIPES_AS_CONCAT`模式）；
   - 若插件不存在或加载失败，设置error_code=1007，跳转至步骤6；
3. AST映射单元遍历AST节点进行转换：
   - 替换数据库专属函数（如聚合函数、日期函数）；
   - 调整语法结构（如分页、关联查询的写法）；
   - 处理数据类型差异（如字符串长度、时间格式）；
4. SQL构建单元生成目标SQL：
   - 按目标数据库语法规则拼接映射后的节点；
   - 针对大表查询添加优化提示（如Mysql的`force index`）；
   - 将生成的SQL存入TaskContext->converted_sql；
5. 调用目标数据库的`explain`接口验证SQL合法性，若无效则设置error_code=1003，跳转至步骤6；
6. 转换成功则更新任务状态为“转换完成”（status=4），传递至执行调度子模块；失败则标记为“已完成”（status=6），返回错误信息。

#### 6.2.3 关键算法描述
- **AST节点映射算法**：采用“规则匹配+模板填充”的混合策略：
  - 规则匹配：基于JSON配置的函数映射表（如`{"sum":"SUM","count_distinct":"COUNT(DISTINCT)"}`）；
  - 模板填充：针对复杂语法结构（如分页），预定义数据库专属模板（如Mysql模板：`limit {{offset}}, {{row_count}}`）；
  - **性能**：单条AST转换耗时≤100ms，转换准确率≥99.5%（核心语法场景），满足需求1.2；
  - **最差场景**：包含10个以上函数的复杂SPL转换耗时≤150ms，通过规则缓存优化（热点规则缓存至内存）。

#### 6.2.4 数据结构定义
|结构名|说明|结构定义（类C风格）|字段说明|
|---|---|---|---|
|ConvertRule|数据库转换规则|```ctypedef struct {    char db_type[16];          // 数据库类型    char spl_pattern[256];     // SPL语法模式（正则）    char sql_template[512];    // 目标SQL模板    int priority;              // 规则优先级（1-高，2-中，3-低）} ConvertRule;```|db_type：规则适用的数据库类型；<br>spl_pattern：匹配SPL元素的正则表达式；<br>sql_template：替换后的SQL模板，支持变量（如{{field}}）；<br>priority：解决规则冲突，高优先级先匹配|
|PluginInfo|转换插件信息|```ctypedef struct {    char name[64];             // 插件名称（如"mysql_plugin"）    char path[256];            // 插件路径    void* handle;              // 动态库句柄（dlopen返回值）    int (*init)();             // 插件初始化函数    int (*convert)(ASTNode*, char*); // 转换函数    void (*destroy)();         // 插件销毁函数} PluginInfo;```|name/path：插件标识及存储路径；<br>handle：动态库加载句柄；<br>init/convert/destroy：插件导出的标准接口函数|

#### 6.2.5 函数列表
|函数名|功能|参数及返回值|前后置约束|
|---|---|---|---|
|dsl_convert_ast_to_sql|将AST转换为目标数据库SQL|参数：[in] ASTNode* ast，[in] DBInfo* db_info，[in/out] TaskContext* ctx<br>返回值：int（0-成功，非0-失败）|前置：ast非空，db_info->status=1（在线）；<br>后置：成功时ctx->converted_sql非空|
|load_convert_rules|加载目标数据库的转换规则|参数：[in] const char* db_type，[out] ConvertRule**rules，[out] int* count<br>返回值：int（0-成功，1-失败）|前置：db_type为支持的类型；<br>后置：成功时rules为规则数组，count为规则数量|
|map_ast_node|转换单个AST节点|参数：[in] ASTNode* node，[in] ConvertRule* rules，[in] int rule_count，[out] char* mapped_str<br>返回值：int（0-成功，1-失败）|前置：node非空，rules已加载；<br>后置：mapped_str为转换后的节点字符串|
|validate_converted_sql|验证转换后SQL的合法性|参数：[in] const char* sql，[in] DBInfo* db_info，[out] char* err_msg<br>返回值：int（0-合法，1-非法）|前置：sql非空，db_info可连接；<br>后置：非法时err_msg包含数据库返回的错误信息|
|load_plugin|加载数据库转换插件|参数：[in] const char* db_type，[out] PluginInfo* plugin<br>返回值：int（0-成功，1-失败）|前置：插件文件存在且有执行权限；<br>后置：成功时plugin->handle及函数指针有效|

#### 6.2.6 设计要点检视
- **调试难题解决方案**：转换过程中记录“AST节点→转换后字符串”的映射日志，DEBUG模式下可通过task_id查询，辅助定位转换偏差；
- **测试难题解决方案**：构建跨数据库转换测试矩阵（4种数据库×500+SPL用例），自动对比转换后的SQL与预期结果；
- **自动化测试支持**：提供`test_dsl_convert`接口（见3.1.2），支持批量验证转换准确性；
- **扩展特性支持**：采用插件化架构，新增数据库（如Hive）时仅需开发符合接口规范的插件，核心代码零改动；
- **异常检测恢复**：转换超时（>150ms）或插件崩溃时，自动切换至备用转换规则集（纯配置方式），保障基本功能可用；
- **工作量估算**：16人天（规则设计4天，插件框架3天，4种数据库适配各2天，测试3天）。

### 6.3 路由子模块

#### 6.3.1 静态结构
- **核心职责**：根据SPL语句中的表名、数据类型及数据库状态，匹配路由规则，确定目标数据库，实现SPL语句到具体数据库的路由决策。
- **细化软件单元**：
  - 规则解析单元：解析JSON格式的路由规则配置文件，构建规则索引；
  - 数据库状态校验单元：调用基础监控接口，实时获取数据库在线状态、负载等信息；
  - 目标库选择单元：结合路由规则和数据库状态，选择最优目标数据库（如主库优先，主库离线则选备库）。

#### 6.3.2 处理流程
1. 接收SPL解析子模块传递的TaskContext（含SPL和AST），设置任务状态为“路由中”（status=2）；
2. 规则解析单元提取SPL中的表名：
   - 从AST中解析出所有涉及的表名（如`from user join log`中提取`user`和`log`）；
   - 若未解析到表名（如`select 1+1`），使用默认路由规则（配置文件中的`default_db_type`）；
3. 加载路由规则：
   - 从`route_rules_path`读取路由规则（如`{"table_route":{"user":"mysql","log_*":"clickhouse"}}`）；
   - 支持通配符匹配（如`log_*`匹配所有前缀为`log_`的表）；
   - 若规则文件不存在或解析失败，设置error_code=1008，跳转至步骤7；
4. 数据库状态校验单元查询状态：
   - 调用基础监控接口`monitor_db_status`获取候选数据库的在线状态（0-离线，1-在线）、负载（CPU/内存使用率）；
   - 过滤离线数据库，若候选库全离线，设置error_code=1004，跳转至步骤7；
5. 目标库选择单元确定最终目标：
   - 单表查询：直接匹配表名对应的数据库；
   - 多表查询：若表路由至同一数据库，选择该库；若路由至不同库，触发多库协同查询逻辑（暂不支持，返回1009错误）；
   - 选择负载最低的在线数据库（负载>80%的库降级为低优先级）；
6. 将目标数据库信息（DBInfo）存入TaskContext，更新状态为“路由完成”（status=3），传递至语句转换子模块；
7. 路由失败则标记任务状态为“已完成”（status=6），返回错误信息。

#### 6.3.3 关键算法描述
- **路由规则匹配算法**：采用“精确匹配→通配符匹配→默认匹配”的三级匹配策略：
  1. 精确匹配：表名与规则完全一致（如`user`→`mysql`）；
  2. 通配符匹配：表名匹配带`*`的规则（如`log_2024`匹配`log_*`）；
  3. 默认匹配：未匹配到规则时使用`default_db_type`；
  - **性能**：单表路由决策耗时≤30ms，多表（≤5张）路由耗时≤50ms，满足需求1.3；
  - **冲突解决**：当表名匹配多个规则时，按“最长前缀优先”原则选择（如`log_detail`优先匹配`log_detail`而非`log_*`）。
- **数据库负载均衡算法**：基于加权轮询，权重与负载成反比：
  - 负载=（CPU使用率×0.6 + 内存使用率×0.4）/100；
  - 权重=1 - 负载（负载>0.8时权重为0，不参与选择）；
  - 确保高可用的同时，避免数据库过载。

#### 6.3.4 数据结构定义
|结构名|说明|结构定义（类C风格）|字段说明|
|---|---|---|---|
|DBStatus|数据库实时状态|```ctypedef struct {    char db_type[16];          // 数据库类型    char ip[32];               // IP地址    int port;                  // 端口号    int status;                // 0-离线，1-在线    float cpu_usage;           // CPU使用率（0-100）    float mem_usage;           // 内存使用率（0-100）    long long last_update;     // 最后更新时间戳（ms）} DBStatus;```|cpu_usage/mem_usage：[0-100] 百分比；<br>last_update：用于判断状态是否过期（超过30s视为过期）|
|RouteResult|路由决策结果|```ctypedef struct {    int db_count;              // 候选数据库数量    DBInfo* dbs;               // 候选数据库列表（动态分配）    int selected_idx;          // 选中的数据库索引（-1表示未选中）} RouteResult;```|db_count：[≥0] 有效候选库数量；<br>dbs：候选库信息数组；<br>selected_idx：选中库在数组中的索引|

#### 6.3.5 函数列表
|函数名|功能|参数及返回值|前后置约束|
|---|---|---|---|
|dsl_route_spl|根据SPL语句和AST确定目标数据库|参数：[in] TaskContext* ctx，[out] DBInfo* target_db<br>返回值：int（0-成功，非0-失败）|前置：ctx->ast非空，路由规则已加载；<br>后置：成功时target_db有效|
|extract_table_names|从AST中提取表名|参数：[in] ASTNode* ast，[out] char***tables，[out] int* count<br>返回值：int（0-成功，1-失败）|前置：ast为SELECT_NODE类型；<br>后置：成功时tables为表名数组，count为数量|
|match_route_rules|匹配表名对应的路由规则|参数：[in] const char* table_name，[in] RouteRule* rules，[in] int rule_count，[out] RouteRule* matched_rule<br>返回值：int（0-匹配成功，1-匹配失败）|前置：table_name非空，rules已加载；<br>后置：匹配成功时matched_rule为有效规则|
|get_db_status|获取数据库实时状态|参数：[in] const char* db_type，[out] DBStatus**status_list，[out] int* count<br>返回值：int（0-成功，1-失败）|前置：db_type为支持的类型；<br>后置：成功时status_list包含所有该类型数据库状态|
|select_best_db|从候选数据库中选择最优目标|参数：[in] DBStatus* candidates，[in] int count，[out] DBInfo* selected<br>返回值：int（0-成功，1-失败）|前置：count≥1，且存在在线数据库；<br>后置：selected为负载最低的在线数据库|

#### 6.3.6 设计要点检视
- **调试难题解决方案**：路由过程日志记录“表名→匹配规则→候选库→选中库”全链路信息，支持通过`dsl_debug_route`接口查询；
- **测试难题解决方案**：开发路由规则模拟工具，可临时注入测试规则（如强制将`user`表路由至PG），验证路由逻辑；
- **自动化测试支持**：提供`test_dsl_route`接口（见3.1.2），覆盖单表、多表、通配符匹配等场景；
- **扩展特性支持**：支持动态添加路由规则（通过`dsl_add_route_rule`接口），无需重启服务，规则变更实时生效；
- **异常检测恢复**：数据库状态获取失败时，使用最近30s内的缓存状态，避免监控接口异常导致路由中断；
- **工作量估算**：10人天（规则解析2天，状态校验2天，选择算法3天，测试3天）。

### 6.4 执行调度子模块

#### 6.4.1 静态结构
- **核心职责**：从基础连接池获取目标数据库连接，执行转换后的SQL，处理执行过程中的异常（如连接超时、执行失败），并将原始结果传递至结果处理子模块。
- **细化软件单元**：
  - 连接管理单元：与基础连接池模块交互，获取/释放数据库连接，处理连接超时；
  - SQL执行单元：执行目标SQL，获取原始结果，记录执行耗时；
  - 异常处理单元：针对执行失败（如连接中断、SQL错误）触发重试机制，超过阈值则终止。

#### 6.4.2 处理流程
1. 接收语句转换子模块传递的TaskContext（含converted_sql和target_db），设置任务状态为“执行中”（status=4）；
2. 连接管理单元获取数据库连接：
   - 调用基础连接池接口`get_db_connection`，传入DBInfo信息；
   - 若连接失败（如网络超时），触发重试（最多3次，间隔1s）；
   - 重试失败则设置error_code=1010，跳转至步骤7；
3. SQL执行单元执行SQL：
   - 通过获取的连接执行`converted_sql`；
   - 记录执行开始/结束时间，计算耗时（execute_time）；
   - 若执行超时（>task_timeout），中断执行并设置error_code=1011，跳转至步骤7；
4. 获取数据库返回的原始结果：
   - 解析结果集的行数、列数、列名及数据；
   - 存储至TaskContext->raw_result；
5. 连接管理单元释放连接：
   - 调用`release_db_connection`接口归还连接至连接池；
   - 若释放失败，记录警告日志（不影响任务流程）；
6. 更新任务状态为“执行完成”（status=5），传递至结果处理子模块；
7. 执行失败则标记任务状态为“已完成”（status=6），返回错误信息。

#### 6.4.3 关键算法描述
- **连接重试算法**：采用指数退避策略，重试间隔随次数递增（1s→2s→4s），避免短时间内频繁重试导致数据库压力过大：
  - **成功概率**：网络瞬断场景下，3次重试成功率≥90%；
  - **性能影响**：重试总耗时≤7s（1+2+4），包含在task_timeout（默认30s）内，不影响用户体验。
- **执行超时控制算法**：通过线程超时机制实现：
  - 执行SQL时启动独立线程，主线程等待指定超时时间；
  - 超时未返回则发送中断信号终止执行线程，释放连接资源；
  - 准确率≥99.9%，无虚假超时或超时未中断情况。

#### 6.4.4 数据结构定义
|结构名|说明|结构定义（类C风格）|字段说明|
|---|---|---|---|
|DBConnection|数据库连接句柄及元信息|```ctypedef struct {    void* handle;              // 数据库连接句柄（如MYSQL*、PGconn*）    char db_type[16];          // 数据库类型    char ip[32];               // 连接的数据库IP    int port;                  // 端口    long long create_time;     // 连接创建时间戳（ms）    int in_use;                // 是否正在使用：0-否，1-是} DBConnection;```|handle：具体数据库驱动的连接句柄；<br>create_time：用于连接池的 idle 时间计算；<br>in_use：连接状态标记，防止重复使用|
|ExecuteParams|SQL执行参数|```ctypedef struct {    char* sql;                 // 待执行的SQL语句    int timeout;               // 执行超时时间（秒）    int retry_count;           // 重试次数    int fetch_size;            // 每次获取的行数（分页场景）} ExecuteParams;```|sql：非空字符串；<br>timeout：≤task_timeout；<br>fetch_size：[1-1000] 控制分页获取大小|

#### 6.4.5 函数列表
|函数名|功能|参数及返回值|前后置约束|
|---|---|---|---|
|dsl_execute_sql|执行转换后的SQL并获取原始结果|参数：[in] TaskContext* ctx，[in] DBConnection* conn<br>返回值：int（0-成功，非0-失败）|前置：ctx->converted_sql非空，conn有效；<br>后置：成功时ctx->raw_result有效|
|get_db_connection|从连接池获取数据库连接|参数：[in] DBInfo* db_info，[in] int timeout，[out] DBConnection* conn<br>返回值：int（0-成功，1-失败）|前置：db_info->status=1（在线）；<br>后置：成功时conn->in_use=1|
|release_db_connection|释放连接至连接池|参数：[in/out] DBConnection* conn<br>返回值：int（0-成功，1-失败）|前置：conn为get_db_connection获取的有效连接；<br>后置：成功时conn->in_use=0|
|execute_with_retry|带重试机制的SQL执行|参数：[in] DBConnection* conn，[in] ExecuteParams* params，[out] RawResult* result<br>返回值：int（0-成功，非0-失败）|前置：conn有效，params->sql非空；<br>后置：成功时result包含执行结果|
|cancel_execution|取消正在执行的SQL|参数：[in] DBConnection* conn<br>返回值：int（0-成功，1-失败）|前置：conn正在执行SQL（in_use=1）；<br>后置：成功时SQL执行被中断|

#### 6.4.6 设计要点检视
- **调试难题解决方案**：执行过程记录“连接信息→SQL→执行耗时→结果行数”日志，支持通过`dsl_debug_exec`接口查询完整执行详情；
- **测试难题解决方案**：开发数据库异常模拟工具，可注入“连接超时”“执行失败”等异常，验证重试和错误处理逻辑；
- **自动化测试支持**：自动化测试用例覆盖正常执行、连接失败、执行超时等场景，验证执行成功率和耗时；
- **扩展特性支持**：支持动态调整重试次数和间隔（通过配置文件），适配不同数据库的稳定性需求；
- **异常检测恢复**：连接泄露检测（超过30s未释放的连接强制回收），避免连接池耗尽；
- **工作量估算**：14人天（连接管理3天，执行逻辑4天，异常处理3天，测试4天）。

### 6.5 结果处理子模块

#### 6.5.1 静态结构
- **核心职责**：将数据库返回的原始结果（RawResult）转换为统一JSON格式，对敏感字段进行脱敏处理，支持分页，并将最终结果返回给业务侧。
- **细化软件单元**：
  - 结果解析单元：解析RawResult中的数据，处理数据类型转换（如字符串→数字、日期）；
  - 脱敏处理单元：根据配置的敏感字段列表，对结果中的敏感信息进行替换（如`password`→`***`）；
  - 格式转换单元：将处理后的数据封装为标准JSON格式（含code、msg、data字段），支持分页信息。

#### 6.5.2 处理流程
1. 接收执行调度子模块传递的TaskContext（含raw_result），设置任务状态为“处理中”（status=5）；
2. 结果解析单元处理原始结果：
   - 转换数据类型（如数据库返回的字符串日期→ISO格式日期）；
   - 处理NULL值（转换为JSON的`null`）；
   - 若解析失败（如类型转换错误），设置error_code=1012，跳转至步骤6；
3. 脱敏处理单元处理敏感字段：
   - 遍历结果列名，匹配配置的`sensitive_fields`（如`password`、`id_card`）；
   - 对匹配列的所有值进行脱敏（替换为`***`或部分隐藏，如身份证号显示前6后4位）；
4. 格式转换单元生成统一JSON：
   - 封装分页信息（total、page_num、page_size）；
   - 组织columns（列名）和rows（数据行）；
   - 生成包含code、msg、data的JSON字符串，存入TaskContext->fmt_result；
5. 更新任务状态为“已完成”（status=6），将fmt_result返回至业务侧；
6. 处理失败则标记任务状态为“已完成”，返回错误信息。

#### 6.5.3 关键算法描述
- **结果格式化算法**：基于流式JSON构建，避免内存占用过高：
  - 逐行处理原始数据，实时写入JSON字符串，而非一次性加载所有数据；
  - **性能**：1万行数据格式化耗时≤30ms，满足需求1.4的“结果格式化耗时≤50ms”；
  - **内存优化**：内存占用与单行数据大小成正比，而非总数据量，支持大数据量分页处理。
- **敏感字段脱敏算法**：基于列名匹配的精准脱敏：
  - 预编译敏感字段哈希表，提高匹配效率（O(1)时间复杂度）；
  - 支持自定义脱敏规则（如手机号隐藏中间4位：`138****5678`）；
  - 脱敏准确率100%（配置正确场景），无漏脱敏或误脱敏。

#### 6.5.4 数据结构定义
|结构名|说明|结构定义（类C风格）|字段说明|
|---|---|---|---|
|SensitiveField|敏感字段及脱敏规则|```ctypedef struct {    char name[64];             // 敏感字段名（如"password"）    char rule[128];            // 脱敏规则（如"replace_all"、"mask_middle:3,4"）} SensitiveField;```|name：需脱敏的列名；<br>rule：脱敏方式，如"replace_all"替换为***，"mask_middle:3,4"表示保留前3后4位|
|PageInfo|分页信息|```ctypedef struct {    int total;                 // 总记录数    int page_num;              // 当前页码    int page_size;             // 每页记录数    int has_more;              // 是否有更多数据} PageInfo;```|total：通过`select count(*)`预查询或数据库返回；<br>has_more：`(page_num * page_size) < total`则为1|

#### 6.5.5 函数列表
|函数名|功能|参数及返回值|前后置约束|
|---|---|---|---|
|dsl_process_result|处理原始结果并生成统一格式输出|参数：[in/out] TaskContext* ctx，[in] PageInfo* page_info<br>返回值：int（0-成功，非0-失败）|前置：ctx->raw_result有效，page_info非空；<br>后置：成功时ctx->fmt_result.data非空|
|parse_raw_result|解析原始结果，处理数据类型转换|参数：[in] RawResult* raw，[out] void* parsed_data（内部数据结构）<br>返回值：int（0-成功，1-失败）|前置：raw->row_count≥0，raw->col_count≥1；<br>后置：parsed_data为结构化数据，需调用free_parsed_data释放|
|desensitize_data|对解析后的数据进行敏感字段脱敏|参数：[in/out] void* parsed_data，[in] SensitiveField* fields，[in] int field_count<br>返回值：int（0-成功，1-失败）|前置：parsed_data为parse_raw_result的输出；<br>后置：敏感字段已按规则脱敏|
|format_to_json|将处理后的数据转换为标准JSON格式|参数：[in] void* parsed_data，[in] PageInfo* page_info，[in] int code，[in] const char* msg，[out] char**json_str<br>返回值：int（0-成功，1-失败）|前置：parsed_data已脱敏；<br>后置：json_str为动态分配的JSON字符串，需手动释放|
|free_parsed_data|释放解析后的数据占用的内存|参数：[in] void* parsed_data<br>返回值：void|前置：parsed_data为parse_raw_result生成的有效指针；<br>后置：内存已释放，指针置NULL|

#### 6.5.6 设计要点检视
- **调试难题解决方案**：DEBUG模式下输出脱敏前后的结果对比日志，辅助验证脱敏规则有效性；
- **测试难题解决方案**：构建包含敏感字段的测试数据集，验证脱敏准确性和JSON格式正确性；
- **自动化测试支持**：提供`test_dsl_result`接口，输入原始结果返回格式化后的JSON，验证处理逻辑；
- **扩展特性支持**：支持动态更新敏感字段列表（通过`dsl_update_sensitive_fields`接口），实时生效；
- **异常检测恢复**：结果过大（>100MB）时自动启用压缩传输，避免内存溢出；
- **工作量估算**：8人天（解析逻辑2天，脱敏处理2天，JSON格式化2天，测试2天）。


## 7. 总结

### 7.1 关联分析
本模块基于SFRD-TS-02-3.4模板设计，与现有系统架构的关联影响如下：
1. **对业务模块的影响**：业务系统需将原直连数据库的SQL调用改为调用本模块的`dsl_execute_spl`接口，涉及“用户中心”“报表中心”“日志分析”3个业务模块，需调整约50处接口调用点；
2. **对基础连接池模块的影响**：需基础连接池模块新增`get_db_connection_by_type`接口，支持按数据库类型获取连接，该改动已与基础模块负责人确认；
3. **对监控模块的影响**：需监控模块新增数据库负载监控指标（CPU/内存使用率），已与监控模块团队达成共识；
4. **兼容性**：模块支持的数据库版本与业务系统现有数据库版本完全兼容（如Mysql 5.7+、Clickhouse 22.0+），无需业务侧升级数据库；

上述影响已与架构师张XX、业务模块负责人李XX、基础模块负责人王XX交流确认，一致同意该设计方案。

### 7.2 遗留问题解决
1.** 总体设计未解决问题 **：
   - 问题：多库协同查询场景（如跨Mysql和Clickhouse关联查询）未明确实现方案；
   - 解决方案：本版本暂不支持多库协同查询，返回1009错误码；下一版本通过分布式查询引擎（如Presto）实现，当前预留接口扩展点。
2.** 潜在技术风险 **：
   - 问题：自研SPL解析引擎对复杂语法（如窗口函数、CTE）的支持可能不足；
   - 解决方案：优先支持核心语法（select、join、where等），复杂语法通过“用户反馈+迭代优化”逐步完善，首版本覆盖80%业务场景。
3.** 性能瓶颈 **：
   - 问题：高并发（1000任务）场景下，AST构建和SQL转换可能成为瓶颈；
   - 解决方案：引入缓存机制（缓存SPL→AST→SQL的映射关系，有效期5分钟），重复SPL可直接复用结果，降低计算开销。


## 8. 业务逻辑相关的测试用例

|用例编号|测试场景|测试步骤|验证点|
|---|---|---|---|
|TC-001|SPL语法校验（正确语句）|1. 调用dsl_validate_spl接口，传入SPL：`select id, name from user where status=1`；2. 查看返回结果|1. 返回code=0；2. msg包含“语法正确”|
|TC-002|SPL语法校验（错误语句）|1. 调用dsl_validate_spl接口，传入SPL：`select id, name from user where`（缺少条件）；2. 查看返回结果|1. 返回code=1001；2. msg包含错误位置（如“第1行第28列：条件表达式不完整”）|
|TC-003|注入检测（恶意语句）|1. 调用dsl_execute_spl接口，传入SPL：`select * from user where id=1; drop table user;`；2. 查看返回结果|1. 返回code=1005；2. msg包含“检测到注入风险”|
|TC-004|Mysql路由（单表）|1. 配置路由规则：`{"user":"mysql"}`；2. 调用test_dsl_route接口，传入SPL：`select * from user`，预期数据库类型“mysql”；3. 查看返回结果|1. 返回code=0；2. actual_db_type=“mysql”|
|TC-005|Clickhouse转换（聚合函数）|1. 调用test_dsl_convert接口，传入SPL：`select count(distinct id) from log`，目标库类型“clickhouse”；2. 查看返回结果|1. 返回code=0；2. converted_sql=“select count(distinct id) from log”（Clickhouse兼容）|
|TC-006|执行超时处理|1. 调用dsl_execute_spl接口，传入耗时SQL（如`select * from big_table`），设置task_timeout=1；2. 查看返回结果|1. 返回code=1011；2. msg包含“执行超时”|
|TC-007|敏感字段脱敏|1. 配置sensitive_fields=“password”；2. 执行SPL：`select username, password from user`；3. 查看返回结果|1. data.rows中password字段值为“***”|
|TC-008|并发性能测试|1. 使用dsl_load_test工具模拟500用户并发执行SPL；2. 监控模块性能指标|1. 并发任务数≥500；2. 单任务响应时间≤500ms；3. CPU使用率≤80%|
|TC-009|数据库离线容错|1. 手动停止Mysql服务；2. 执行路由至Mysql的SPL；3. 查看返回结果|1. 返回code=1004；2. msg包含“目标数据库离线”|
|TC-010|结果分页处理|1. 执行SPL：`select * from user`，设置page_num=2，page_size=10；2. 查看返回结果|1. data.total为总记录数；2. data.rows包含第11-20条数据；3. data.page_num=2|

（注：完整测试用例集包含100+用例，覆盖功能、性能、安全、可靠性等维度，本章节列出核心用例）


## 9. 变更控制

### 9.1 变更列表

|变更章节|变更内容|变更原因|对老功能/原有设计的影响|
|---|---|---|---|
|4.2 方案选型|将路由方案由“Drools规则引擎”改为“JSON配置化规则”|原方案性能不满足需求（路由耗时>50ms），且学习成本高|需重新开发路由规则解析模块，不影响老功能（无历史版本）|
|6.1.3 关键算法|优化SPL解析算法，支持嵌套子查询层数由2层扩展至3层|业务侧反馈需要支持更复杂的SPL语句|解析逻辑复杂度增加，需补充3层嵌套的测试用例，兼容原有解析功能|
|4.6.2 可靠性设计|进程故障自动恢复时间由15s调整为10s|用户需求提升模块可用性，缩短故障恢复时间|需修改系统监控机制的配置参数，不影响模块核心功能|
|5.2 全局数据结构|TaskContext中新增retry_count字段|用于记录执行重试次数，便于问题定位|需调整所有子模块对TaskContext的处理逻辑，增加兼容性代码|

（注：所有变更已完成部门内部评审，评审记录见附件《设计变更评审表.xlsx》）

