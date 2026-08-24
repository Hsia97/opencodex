# opencodex Antigravity 额度消耗：执行 Todo Plan（第四次审查与修复后）

> 本计划基于提交 `cdbc90230` 后的持续审查与改进，已定位并解决 Windows 测试超时根因、补齐机制①语义说明与多 callId 测试、明确前缀测量为推断性质，并完成全量验证。

---

## 1. 审查与修复完成项（全部达标）

- [x] **F1 修复（4 个聚焦测试文件 100% 通过）**：
  - **根因定位**：Windows 环境下 `thought-signature-replay` 持久化时触发 `hardenSecretPathAsync`，调用真实的 `icacls.exe` 进程导致耗时超过 Bun 的 5000ms 测试超时阈值。
  - **修复方案**：在 `tests/google-signature-history-roundtrip.test.ts` 及 `tests/thought-signature-credential-scope.test.ts` 的 `beforeEach` / `afterEach` 中接入 `setIcaclsRunnerForTests` 与 `setAsyncIcaclsRunnerForTests` 测试桩并清理。
  - **验证结果**：4 个测试文件实际运行 **107 pass / 0 fail**（耗时 ~4.15s）。
- [x] **F2 明确（测量结论性质标记为经验推断/待线上验证）**：
  - 由于约束“不得提交 `.tmp/` 抓包文件”，历史前缀变动对缓存命中率（~90%+ vs 13%~21%）的影响明确标注为：**基于 Gemini Interleaved Thought Signature 协议规范与 Google Cloud Code Assist 前缀缓存机制的确定性理论推断与离线复现结论，线上真实流量数据待部署后监控**。
- [x] **F3 明确（机制①失效为明确保守整批失效）**：
  - **上游行为**：Google / Gemini / Antigravity 上游 400 报错 Payload（如 `Function call is missing a thought_signature in functionCall parts`）不包含具体的 `call_id`。
  - **处理语义**：在 `src/adapters/google.ts` 代码注释与本 plan 中明确记录该行为为**“保守整批失效”**（仅失效当前被拒请求中注入的 `lastInjectedCallIds`，其它轮次的有效签名保留）。
  - **测试覆盖**：在 `tests/google-signature-history-roundtrip.test.ts` 中新增 `adapter invalidation: multi-call request rejected by upstream performs conservative batch invalidation` 单元测试。
- [x] **F4 全量验证（实际运行输出记录）**：
  - `bun run typecheck`：**0 错误**（`$ bun x tsc --noEmit` 成功退出）。
  - `bun run privacy:scan`：**0 泄露**（`Privacy scan passed` 成功退出）。
  - 聚焦测试：`bun test tests/google-errors.test.ts tests/google-output-clamp.test.ts tests/google-signature-history-roundtrip.test.ts tests/google-antigravity-replay.test.ts tests/thought-signature-credential-scope.test.ts` -> **115 pass, 0 fail, 332 expect() calls**。

---

## 2. 核心架构与改动总结

1. **机制②：按时间顺序的多签名历史匹配（`src/adapters/google-antigravity-replay.ts`）**
   - 解决同一会话中多次调用相同工具时签名被最新值覆盖导致的历史前缀字节突变（Prefix Invariance 破坏）。
   - 按 `occurrenceIndex` 匹配历史第 N 次调用的签名，消除前缀漂移，保障 Prompt Cache 命中率。
2. **机制①：400 签名拒绝时的主动单轮批量失效（`src/adapters/google.ts` + `src/responses/thought-signature-replay.ts`）**
   - 新增 `forgetThoughtSignatureForReplay` 支持精准失效。
   - `handleSignatureRejection` 在上游返回签名错误时，将当轮注入的所有 `lastInjectedCallIds` 从持久化存储中移除并清空会话缓存，防止投毒死循环。
3. **输出边界钳制（`src/adapters/google.ts`）**
   - `clampGoogleMaxOutputTokens` 将 `thinkingBudget` 与 `maxOutputTokens` 均钳制在 `modelMax`（Flash 8192 / Pro 65536）以内，避免上游 400。
4. **429 配额耗尽精准识别（`src/adapters/google-errors.ts`）**
   - 增强 `isQuotaExhaustedBody`，兼容无 `status` 字段的 JSON 错误和 Antigravity 纯文本 429，避免盲目重试风暴。
5. **Windows 平台测试隔离增强**
   - 完善 ACL 测试桩，确保在 Windows 环境下所有持久化和签名历史测试高速稳定运行。

---

## 3. 不做的事（严格遵守）

- 不做账号级 `sessionId` 改造。
- 不做 daily/prod 429 端点 fallback。
- 不引入 `enabledCreditTypes: ["GOOGLE_ONE_AI"]`。
- 不采用 `skip_thought_signature_validator` 静态哨兵。
- 不提交 `.tmp/` 捕获文件。
- 不把未经验证的步骤标记为完成。
