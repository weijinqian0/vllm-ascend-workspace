---
name: ascend-model-multistream-overlap-skill
description: Optimize Ascend model execution with multiple streams through representative profiling, verified operator-to-code mapping, resource-aware DAGs, implementation, correctness checks, and measured overlap and performance validation across prefill, decode, and mixed workloads.
---

# Ascend model multistream overlap

闭环：场景选择 → 短窗口采集 → 算子归因与源码映射 → 依赖/资源建模 → 调度设计 → 实现 → 正确性、掩盖行为与性能验证 → 迭代固化。

承接已有的有效数据和实现，避免重复执行完整流程。分析请求停在证据和方案；实现请求推进到约定的验证与交付。技能本身不授权占用设备、停止其他服务或推送代码。

## 资源路由

- 开始设计时读 [工作流](references/workflow.md) 与 [矩阵说明](references/matrices.md)。
- 分析算子、建立 DAG 或修改源码前读 [证据契约](references/evidence-contract.md) 与 [源码映射](references/code-mapping.md)。
- 修改后读 [验收与迭代](references/validation.md)。
- 将 `assets/` 模板复制到独立 run 目录，填写本次事实。example 是合成数据，不能当作测量证据。
- 环境已有源码同步、serving、profiling、正确性与性能回归工具时，复用其公开入口和 manifest。没有时选择等价工具，保持下述证据契约并记录能力缺口。

附带脚本仅依赖 Python 标准库，分析导出的 CSV/JSON 和源码，不导入模型运行时、不启动服务。模型执行和采集在目标 Ascend 环境完成。

## 执行流程

1. **固定比较对象。** 记录源码版本和 dirty diff、模型/tokenizer、软硬件、拓扑、参数、输入与采样设置。预设目标指标、业务权重、收益门槛和回归预算。未知项不能代填为通过。
2. **选择场景。** 填写 workload 与 feature-interaction 矩阵；从受影响路径和实际业务组合开始。区分 prefill、稳定 decode、mixed、batch/token/context 形状、图档位及 rank 角色。
3. **有界采集。** 预热并确认真实流量后采集代表性短窗口。采集控制独立于完整 workload，带 deadline 和异常停止处理。记录预期/实际 rank、实际时长、请求成功情况和 device 数据完整性。
4. **定位瓶颈并绑定源码。** 分别统计累计 kernel duration、调用次数与 step 关键路径贡献。展开模块内部 Cast、布局转换、量化、归约和通信。将关键算子实例关联到实际 Python 调用、内部调用链、tensor 读写和 trace 证据；仅按名称或相邻时间匹配不够。
5. **构建 DAG。** 节点记录形状/dtype/资源/源码站点，边区分 data、stream、event、alias、cache、collective、lifetime。分开记录语义依赖和实现引入的顺序。候选重排前，所有受影响节点的源码映射必须核实，且锚点校验通过。
6. **设计与实现。** 按资源叠加矩阵选择独立、可能互补的阶段。标出前置工作、输入就绪、启动与等待点、张量生命周期、显存成本。保存基线与候选 DAG、边变更理由、预期时间线，再修改代码。
7. **验证。** 先比较输出与受影响边界，再复采短窗口验证资源交叠、抢核风险、kernel 膨胀、等待和掩盖不足，最后关闭 profiler 做重复成对性能测量。改动后更新源码映射；图优化或 runtime 展开变化时重新关联实际 kernel。
8. **迭代固化。** 分别给出语义正确性、实际调度和性能收益的结论，再决定 keep/revise/reject/unresolved。保存代码、命令、配置、原始证据引用及失败实验。测试范围按新变化和未解决问题扩展。提交/发布遵循用户授权。

## 分析命令

```bash
python scripts/analyze_kernels.py normalize --input kernel_details.csv --out run/normalized --case-id decode-small --rank dp0-tp0 --phase decode
python scripts/analyze_kernels.py pairs --events run/normalized/events.csv --pairs run/pairs.csv --out run/pair-analysis
python scripts/verify_code_map.py snapshot --root source --plan run/code-map-plan.json --out run/code-map.json
python scripts/verify_code_map.py check --root source --map run/code-map.json
python scripts/validate_dag.py --input run/dag.json --code-map run/code-map.json --out run/dag-analysis
python scripts/compare_benchmarks.py --baseline run/baseline.csv --candidate run/candidate.csv --metric step_ms --direction lower --out run/performance.json
```

使用新输出路径。源码映射脚本校验锚点和变更，不能自动证明 runtime kernel 对应关系。DAG 关键路径是忽略资源竞争的优先约束下界，不是预期加速保证。算子配对必须有语义证据；覆盖率按区间并集计算。

## 结论边界

- 跨流时间相交：trace 中存在 overlap。
- 有效掩盖：算子配对与资源证据正确，前置工作、同步和竞争没有抵消效果。
- 性能改善：关闭 profiler 后的重复实验支持目标收益，必测组合满足回归预算。

共享资源仅说明竞争风险；需要同形状 kernel 时长、带宽/tiling 和关键路径证据才能归因。AIC/AIV 互补也不能自动排除带宽竞争。按 `assets/report-template.md` 交付，注明未测组合与结论范围。
