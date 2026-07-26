# CLAUDE.md · messauto（上游 fork + 自研钉钉支持）

> 上游 = [LeeeSe/MessAuto](https://github.com/LeeeSe/MessAuto)（Rust，macOS 自动提取短信/邮件验证码，v1.3.0）。
> 本 clone = **GitHub 真 fork**，`origin` → [`zengtianli/MessAuto`](https://github.com/zengtianli/MessAuto)（isFork=true，parent=LeeeSe/MessAuto），另配 `upstream` remote 指上游。
> **2026-07-25 从 `~/Dev` 根归位到 `~/Apps`**（A 档：自带完整依赖、不 import 总部代码）。

## 本 fork 的增强（`feat/dingtalk-support` 分支，领先 upstream/master 4 个提交）

上游只监听 信息 App（`chat.db`）+ 邮件 App（`.emlx`）。本 fork 加了**钉钉验证码**：

- `src/monitor/dingtalk.rs` — 读 macOS 通知中心库，按 `rec_id` 增量取 bundle id `com.alibaba.dingtalkmac` 的通知
- **定时轮询 `POLL_INTERVAL = 2s`，不是文件事件**——通知中心是高频 WAL 库，实测 `notify`(FSEvents) 对它不可靠，新通知写入不触发事件。**禁改回文件监听**（这是踩过的坑，代码注释已记）
- 启动时以当前最大 `rec_id` 为基线，避免把历史通知当新验证码
- `src/parser.rs` 已加「示例单位综合管理平台」验证码测试用例

## 构建 / 安装

```bash
cd ~/Apps/messauto
cargo build --release
cargo packager --release     # 产物: target/release/MessAuto.app
# 安装: 备份旧版进 ~/.Trash 后 ditto 到 /Applications/MessAuto.app
```

- `/Applications/MessAuto.app` 现 = 本 fork 自编译版（二进制 mtime 与 `target/release/MessAuto` 一致），**非上游 release**；升级上游 = `git fetch upstream && git rebase upstream/master feat/dingtalk-support` 后重编译
- 配置落 `~/.config/messauto/messauto.json`（其真源在 `~/Dev/tools/configs/_dotfiles/messauto/`，dotfiles 软链，改配置去那边）
- 需授予：全磁盘访问（读 chat.db / 通知中心库）+ 辅助功能（自动回车）；`src/permissions.rs` 有体检

## 上游 PR 现状

`LeeeSe/MessAuto#107`（钉钉验证码）状态 **CLOSED + draft，未合并** —— 本机验证不充分就抢跑提交，提完才转 draft。
**再提 PR 前必须**：本机构建 → 安装 → 真实场景端到端确认有效，才推上游（memory `feedback-verify-effective-before-pr`）。

## 坑

- 通知中心库路径随 macOS 版本变，`dingtalk.rs` 里取库路径失败会 warn 但不崩，日志级别 info 才看得到命中统计
- 钉钉「工作通知」与「聊天消息」落库行为不同，近似样本会骗人：验证方案必须用**目标类型的真实通知**测，别拿相邻类型推断（memory `feedback-verify-with-target-sample-not-proxy`）
- Rust edition 2024，需较新 toolchain
