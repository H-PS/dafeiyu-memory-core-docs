# CodeArts「API 密钥无效」修复补丁

## 这是什么

修复 DSH 内置 `dsh-providers-hub` 里 CodeArts（华为云）渠道从 2026-11-04 起
**所有模型都报「API 密钥无效」**的问题。

## 根因（一句话）

华为 snap-access 网关的 WAF 按 **User-Agent 指纹**封禁了小写
`deepseek-harness/<版本号>`，命中即返回 `403 / TM.00000001 网络繁忙，请稍后重试`。
DSH 主程序 `attributionHeaders()` 自带的 UA 恰好命中该规则；这个 403 被 DSH
归类为 `AUTH`，客户端再替换成固定文案「API 密钥无效」，真实原因被吞掉。

**与 API 密钥、账号权限、签名算法、代理、并发、子代理数量全都无关。**

实验证据（同一时刻、同一凭据、同一请求内容，只改 UA）：

| UA | 结果 |
|---|---|
| `deepseek-harness/0.2.1-rc.5 (+https://github.com/deepseek-ai/deepseek-harness)` （DSH 原 UA） | ❌ 403 `TM.00000001` |
| `deepseek-harness/1.0.0` | ❌ 403 |
| `DeepSeekHarness/0.2.1-rc.5` （首字母大写，补丁值） | ✅ 200 |
| `deepseek-harness` （无版本号） | ✅ 200 |
| 无 UA / `CodeArts-Agent/1.0` | ✅ 200 |

## 补丁做了什么

改 `dsh-providers-hub/lib/llm-adapter.js` 两处，均只影响 codearts 渠道：

1. **必须**：在 `new Headers(attributionHeaders())` 之后覆盖 UA 为
   `DeepSeekHarness/0.2.1-rc.5`。UA 不参与 SDK-HMAC-SHA256 签名，改它不影响验签。
2. **建议**：让报错模板带上华为上游真实响应体（`| body: ...`），以后再出问题
   能直接看到真实原因，而不是「API 密钥无效」。

## 用法

```powershell
# 检查当前是否已打补丁（不写入）
node E:\[PROJECT]\codearts-ua-fix\apply-fix.mjs --check

# 打补丁（幂等，重复运行安全）
node E:\[PROJECT]\codearts-ua-fix\apply-fix.mjs
```

打完**必须完全退出 DSH**（托盘右键退出，确认任务管理器无 `DeepSeek Harness.exe`）再打开，
否则进程仍加载旧模块。

默认目标文件：
```
C:\Users\[USER]\.dsh\profiles\desktop\node_modules\dsh-providers-hub\lib\llm-adapter.js
```
也可传入自定义路径：`node apply-fix.mjs <path>`。

## ⚠️ 何时需要重跑

**每次 `dsh-providers-hub` 插件升级之后**——升级会覆盖 `llm-adapter.js`。
症状会原样复发（所有 codearts 模型报「API 密钥无效」）。
此时重跑一次 `apply-fix.mjs` 即可，无需重新排查。

脚本是幂等的：已打过就跳过，锚点异常（插件结构变了）会拒绝写入并提示人工确认，
不会改坏文件。首次写入会生成 `llm-adapter.js.orig` 备份。

## 排查要点备忘

- DSH 的「API 密钥无效」= `AUTH` 类错误的统一替身文案，401/403 任何原因都会显示它，
  **不等于密钥有问题**。
- `TM.00001041 并发会话数已达上限(3个)` 是华为每账号 3 会话的真实限流，
  约 30–60 秒自动释放；短时间连续多发会撞上，容易误判成主因。
- `APIG.0301 verify ak sk signature fail` 是凭据失效时的错误，与本次故障无关。
