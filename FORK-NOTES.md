# 引擎治理备忘(2026-10-23)

决策:日常依赖 npm 上游 acp-kernel,不 fork;本仓(SeniorHan/acp-kernel)是保险仓。
基线:v0.0.101 = 4d38906,与运行时零漂移,测试基线 1233/1233。

三条触发器——每次升级内核版本或出现新需求时**人工对照**,不建自动监控:
- T1 停更:上游 ≥2 个月无提交 → 保险仓转正(我们接盘维护)
- T2 需求被拒:需要改内核默认行为且上游不接受(例:CCR 默认开)→ 转 fork
- T3 破坏性变更:内核仍在 0.x,minor 版本带破坏性变更 → 升级前对照变更日志评估,必要时锁版本

附注:CCR(ccr-store 节点,配置默认关)激活属适配层接线;若未来需要改其默认值,走 T2 程序。

P2.7 终审(2026-10-23):CCR 取消立项。实测全工具族均已有超大输出保护:keel-bash(fullOutputPath 截断+文件指针)、pi 内置 read/grep(有保护)、MCP(script-mode 与原生均有保护)。无缺口,CCR 收益为零,且与各工具自责机制双头管理。

## 2026-10-24: branch-guard 上游 PR(裁定:先上游 PR,后定迁移)

- **PR**: ranxianglei/acp-kernel#516 `feat: opt-in branch-aware block masking via persistedMessageIds`
- 内容:`ProcessTurnInput.persistedMessageIds?`(opt-in)→ /tree 回卷进块区间时,跨分支块本轮按 expanded 掩蔽(原文可见),原 expanded 旗标在结果态还原,不落盘。零传参零行为变化。1237/1237(基线 1233+4)。
- 分支:`feat/branch-aware-blocks`(fork main 保持纯上游镜像,特性走分支——T1 触发时合入即转正)
- 来源:lean 的 host 侧 branch-guard(weakmap 包装版,生产验证 ~80k msgs)移核,检测每轮从声明集重算,无跨轮状态。
- **T2 计时开始**:PR 被拒/2 周无回应 → fork 分支合入 main,lean 换传参形态;合并 → lean 删自身 wrapper 改传 persistedMessageIds。
- 遗留(B 类):degeneration 例外(#351,4655×「【」打死会话,内核无阀门)迁前复核未做。

## 2026-10-24: degeneration #351 复核裁决(B 类最后一项,队列清空)

**裁定:不迁核、不做观测层化、原样保留在 lean。** P2.6 的"同策略"分组过粗,细看后 degeneration 与 summary-sanitize 不同类:

1. **它不是拒绝门**。summary-sanitize 旧形态是 throw(拒摘要→区间不折叠),才需要观测层化;degeneration 是纯视图修复——`collapseAssistantDegeneration` 重写出站视图(≥200 码点游程折叠)+ 一次性恢复提示,持久历史不动,没有可误触发而"拒服务"的判决点。
2. **它是 #351 唯一阀门**。机理:模型退化出 4655×「【」,pi 每请求回放 thinking(reasoning_content),退化尾部经续写偏置自激,abort 循环杀死会话且无恢复路径。内核层看不见这问题(它发生在 host→provider 出站流量),内核无对应阀门,迁核属于层级错位。
3. **误触发风险有界**:minRun=200(实测正常漂移上限 ~60:分隔线/点线远低于阈值)、enabled kill-switch、位置自限(新回合使旧消息非最后,提示自动消失,#223 教训已吸收)。

B 类迁移队列终态:branch-guard 上游 PR #516(等裁决,T2 计时)/ summary-sanitize 观测层化(已落地)/ degeneration 保留 lean(本裁定)/ overflow-selfheal 同 degeneration 类属视图修复(顺带确认,同裁定)。**B 类收尾完成。**

## [2026-10-24] B-类迁移进度(上游化跟踪)
- branch-guard → 上游 PR #516(OPEN,等审查;feat 分支 857b39f)
- circuit-breaker → 上游 PR #515(OPEN)
- **degeneration 阀门 → 上游 issue #517 已挂**(#351 复核结论:内核无对应阀门,prune.ts 注释无关;事故真实;机构自限位无持久状态)。待上游表态:接收则交 PR(参考实现 keel-fold src/degeneration.ts 289 行),倾向 billion-context 则挪提。
- overflow-selfheal:维持观测层,不迁移(P2.6 裁定)。
