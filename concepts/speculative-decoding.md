---
title: "Speculative Decoding"
kind: concept
created: 2026-08-16
---

# Speculative Decoding

# Speculative Decoding 投机解码

## 面试定义（可直接背）
> "Speculative decoding accelerates autoregressive inference by using a small, fast draft model to propose several tokens ahead, which the large target model then verifies in a single parallel forward pass — accepted tokens are kept, so output is identical to the target model's, just faster."

## 解决的问题
LLM 逐 token 生成是**串行**的：生成第 N 个 token 必须等第 N-1 个完成。每步都做一次完整前向，GPU 算力大量闲置（瓶颈在显存带宽而非算力）。

## 工作流程
1. **Draft（起草）**：草稿器快速连猜 k 个 token
2. **Verify（验证）**：大模型**一次并行前向**同时给这 k 个位置打分
3. **Accept/Reject**：逐位比对，接受最长前缀；第一个被拒位置由大模型输出替换，之后的草稿丢弃

关键点：**输出分布与不用投机解码完全一致**——纯加速，不是近似。

```mermaid
sequenceDiagram
    participant D as Draft 草稿器(快)
    participant T as Target 模型(准)
    D->>D: 连猜 k 个 token
    D->>T: 提交草稿 [t1,t2,t3,t4]
    T->>T: 一次并行前向验证 4 个位置
    T-->>D: 接受 t1,t2,t3，拒绝 t4
    Note over T: t4 用大模型自己的采样替换<br/>本轮净产出 4 token / 1 次大模型前向
```

## 三种草稿器方案对比（2026 现状）
| | MTP | DFlash | DSpark |
|---|---|---|---|
| 草稿器 | 主模型自带预测头 | 独立 block-diffusion 模型 | 独立半自回归 + Markov head + confidence head |
| 起草方式 | 自回归逐个猜 | **一次前向并行出整块** | 并行出块 + 置信度调度验证长度 |
| 来源 | 模型厂商训练自带（Gemma4/Qwen 等） | z-lab（arXiv 2602.06036） | DeepSeek V4 生产方案（arXiv 2607.05147） |
| llama.cpp | `--spec-type draft-mtp` | `--spec-type draft-dflash -md <drafter.gguf>` | 代码已合并，GGUF 生态薄，主场 vLLM/SGLang |
| 草稿大小 | ~240MB | ~420MB(Q8)/835MB(BF16)，0.4B 参数 | — |

## 实战数据（Strix Halo APU + Gemma4-26B, llama.cpp build 10448, Vulkan）
**MTP（生产在用）**：61-78 t/s，接受率 58-88%，稳定 ✅

**DFlash 实测翻车（2026-08 验证）** ❌：
- `-fa on`（flash attention）与 DFlash 冲突 → 首个请求即静默崩溃（exit code 5，无错误日志）
- 去掉 `-fa` 能跑，但接受率仅 **19.5%**（MTP 是 80%），速度反降至 27 t/s
- 长生成（~500 tok）中途依然崩溃
- 换 BF16 草稿（排除量化因素）→ 崩得更快
- 与 llama.cpp issue #25792 症状完全吻合（官方草稿接受率卡 0.15、净减速）

**教训**：新 spec 方法落地前先用小请求 + 长生成两级测试验证；`timings.draft_n_accepted/draft_n` 是健康度核心指标。**接受率 <50% 的投机解码大概率是负优化**（草稿白算）。

## Trade-off
- 多花一份草稿的显存和计算，换主模型前向次数减少
- batch 大、带宽不是瓶颈时收益递减

## 相关
[[MoE (Mixture of Experts)]] [[llama.cpp Router 模式]] [[统一内存 Unified Memory]] [[Gemma4-26B 本地部署实战]]

## Related

- [[MoE (Mixture of Experts)]]
- [[llama.cpp Router 模式]]
- [[统一内存 Unified Memory]]
- [[Gemma4-26B 本地部署实战]]
