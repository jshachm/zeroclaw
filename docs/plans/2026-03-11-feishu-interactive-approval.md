# 飞书交互式审批功能实现计划

> **For Claude:** REQUIRED SUB-SKILL: 使用 superpowers:executing-plans 来逐任务实现此计划

**目标：** 在飞书中实现交互式审批——当 ZeroClaw 需要审批时，发送带按钮的消息卡片，等待用户点击，然后执行或拒绝。

**架构：** 在飞书 WebSocket 监听器中添加审批回调处理，将 ApprovalManager 连接到通道，实现异步等待机制。

**技术栈：** Rust, tokio 异步, 飞书卡片消息与交互按钮

---

## 任务  导出 ApprovalManager 供飞书访问

**文件：1:**
- 修改: `src/approval/mod.rs:80` - 添加 pub 或创建访问方法

**步骤 1: 添加访问方法到 ApprovalManager**

```rust
impl ApprovalManager {
    /// 根据 ID 获取待审批请求
    pub fn get_pending(&self, id: &str) -> Option<PendingApproval> {
        self.pending_approvals.lock().get(id).map(|p| PendingApproval {
            id: p.id.clone(),
            tool_name: p.tool_name.clone(),
            arguments: p.arguments.clone(),
            channel: p.channel.clone(),
            response_tx: None,
        })
    }
}
```

**步骤 2: 提交**

```bash
git add src/approval/mod.rs
git commit -m "feat: 添加待审批访问器"
```

---

## 任务 2: 在飞书中添加审批回调处理

**文件：**
- 修改: `src/channels/lark.rs:736` - 在 WS 处理器中添加按钮回调解析

**步骤 1: 添加回调事件解析**

在 WebSocket 消息处理器中（约第 736 行），在现有 message_type 处理后添加：

```rust
// 在现有的 message_type 处理后添加：
"interactive" => {
    // 解析按钮回调
    if let Ok(v) = serde_json::from_str::<serde_json::Value>(&lark_msg.content) {
        let action = v.get("action").and_then(|a| a.as_str());
        let value = v.get("value");
        // 转发到审批管理器
    }
}
```

**步骤 2: 提交**

```bash
git add src/channels/lark.rs
git commit -m "feat: 添加飞书按钮回调解析"
```

---

## 任务 3: 连接 ApprovalManager 到 Daemon 实现跨组件访问

**文件：**
- 修改: `src/agent/loop_.rs` - 传递审批管理器到通道以便回调

**步骤 1: 添加创建待审批并异步等待的方法**

在 `src/agent/loop_.rs` 约第 2454 行审批处理处：

```rust
// 对于飞书，创建待审批并发送卡片
if channel_name == "feishu" {
    let tool_args_str = serde_json::to_string(&tool_args).unwrap_or_default();
    
    // 发送审批卡片给用户
    // 需要通道引用
    
    // 创建带异步等待的待审批
    let (approval_id, rx) = mgr.create_pending_approval(
        tool_name.clone(),
        tool_args.clone(),
        channel_name.to_string(),
    );
    
    // 等待响应（带超时）
    match tokio::time::timeout(Duration::from_secs(120), rx).await {
        Ok(Ok(response)) => response,
        _ => {
            // 超时或错误 - 拒绝
            ApprovalResponse::No
        }
    }
}
```

**步骤 2: 提交**

```bash
git add src/agent/loop_.rs
git commit -m "feat: 实现飞书异步审批等待"
```

---

## 任务 4: 测试完整流程

**步骤 1: 编译并部署**

```bash
cd /Users/haomintsai/workspace/apps/linux/zeroclaw
cargo build --release --features channel-lark
# 部署到服务器
```

**步骤 2: 测试**

1. 让 ZeroClaw 运行 opencode 命令
2. 验证卡片消息是否带按钮发送
3. 点击按钮验证执行/拒绝

**步骤 3: 提交修复**

---

## 总结

本计划实现：
1. 导出 ApprovalManager 访问器
2. 在 WS 处理器中解析飞书按钮回调
3. 连接异步等待用户响应
4. 端到端测试

预计修改：约 100-150 行
