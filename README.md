# Agent Skills — 实战沉淀版

一批**从真实生产系统里沉淀出来**的 agent skills。不是模板批量生成，每个 skill
背后的坑都是实际踩过的。

```bash
npx skills add breezesamuel/agent-skills
```

## 立刻可用的在线端点（都已实测）

| 端点 | 用途 | 实测 |
|---|---|---|
| `https://mcp.highkingflower.com/mcp` | MCP（streamable HTTP）→ Claude / Cursor / OpenCode 直接挂 | initialize → tools/list → tools/call 全通 |
| `https://agentauditor.highkingflower.com` | x402 按次付费的审计端点（$0.0025/次） | 无支付头 → 402（付费墙生效） |
| `https://pay.highkingflower.com` | 零点击收银台（弹窗支付 + 自动回跳 + 自动确认） | 真实 PayPal 建单 + approve 页可达 |

MCP 客户端配置（Claude Desktop / Cursor / OpenCode 通用）：

```json
{
  "mcpServers": {
    "approval-risk-auditor": {
      "url": "https://mcp.highkingflower.com/mcp"
    }
  }
}
```

工具：`audit_approvals(wallet, {stale_after_days?, top_tokens?})` → `{summary, approvals, risk_flags, revoke_tx_data}`

## 包含的 skills

| Skill | 解决什么 | 状态 |
|---|---|---|
| [`onchain-approval-audit`](skills/onchain-approval-audit/SKILL.md) | 审计 EVM 钱包的 unlimited / stale 授权，产出可直接签名的撤销 calldata | 实盘验证 |
| [`zero-click-checkout`](skills/zero-click-checkout/SKILL.md) | 零点击收款：弹窗支付 + 自动回跳 + 自动确认 + 自动开通 + 自动收据 + 自动核账 | 生产在用 |
| [`seo-bulk-page-enrichment`](skills/seo-bulk-page-enrichment/SKILL.md) | 批量把上千个薄页面改成可索引 / 可内链 / 可变现 / 可观测 | 跑过 1515 页 |
| [`x402-paid-api-endpoints`](skills/x402-paid-api-endpoints/SKILL.md) | 把 HTTP 端点变成按次付费的 API（x402 + facilitator） | 线上计费中 |
| [`alipay-wechat-cn-checkout`](skills/alipay-wechat-cn-checkout/SKILL.md) | 支付宝 / 微信支付接入：签名、notify 验签、异步对账、退款 | 国内通道 |

## 为什么这几个 skill 不一样

大部分公开 skill 是"祈使句列表"。这几个 skill 里写的是**具体的、会静默出错的坑**：

- `eth_getLogs` 的 topic0 写错时**不报错、只返回 0 条** → 审计报告显示"干净无风险"，
  实则全漏（附运行时链上自检的做法）
- 撤销 calldata 的地址补到 40 位而不是 64 位 → **交易字段错位**，会打错地址
- 手写 `aggregate3` 的 ABI 编码漏掉数组偏移量 → 节点一律 `execution reverted`
- 注入页面的 `<script>` 少一个 `catch` → **整段脚本不解析**，页面照常渲染、统计全死
- 推荐折扣阶梯封顶 100% → 平台实收 ¥0、推荐人佣金也是 ¥0（双输）
- Vercel 免费额度 100 次/天 → 内容改造全部堵在部署这一步

每个坑都给出**可执行的验证方式**，不是"注意一下"。

## 来源

这些 skill 来自一套自托管的在线运营系统（工具站 + 收银台 + x402 agent +
自动核账 + 自动告警）的生产实践。相关实现：

- Approval Risk Auditor（Apify Actor + MCP server）：https://github.com/breezesamuel/approval-risk-auditor
- 线上工具站：https://tools.highkingflower.com/

## License

MIT
