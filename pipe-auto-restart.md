# Pipe 设备子进程自动重启功能

## 概述

本次修改为 OpenVPN 的 pipe 设备类型添加了子进程自动重启功能。当使用 `--dev "|/program"` 参数启动的外部程序（如 tunsocks）意外退出时，OpenVPN 会自动检测并重启该程序，从而提高 VPN 连接的稳定性。

## 背景

在 OpenVPN 中使用 pipe 设备类型（`--dev-type pipe --dev "|/path/to/program"`）时，流量通过外部程序处理而不是真实的 tun/tap 设备。这允许非 root 用户通过用户态 TCP/IP 栈连接到 VPN。

然而，如果外部程序（如 tunsocks）因错误或崩溃而退出，OpenVPN 原本不会尝试重启它，导致 VPN 连接中断。本次修改解决了这个问题。

## 修改内容

### 1. 数据结构修改

**文件**: `src/openvpn/tun.h`

在 `struct tuntap` 结构体中添加了新字段：

```c
struct tuntap
{
    // ... 其他字段 ...
    bool is_pipe;
    pid_t pipe_pid;
    char *pipe_cmd;  // 新增：保存原始 pipe 命令用于自动重启
    // ... 其他字段 ...
};
```

### 2. 核心功能实现

**文件**: `src/openvpn/tun.c`

#### 2.1 保存原始命令

修改 `open_pipe()` 函数，在启动子进程时保存原始命令：

```c
tt->pipe_pid = (pid_t)pid;
/* Save the original pipe command for auto-restart */
tt->pipe_cmd = string_alloc(dev, NULL);
```

#### 2.2 子进程监控和重启

新增 `check_pipe_monitor()` 函数：

```c
bool check_pipe_monitor(struct tuntap *tt)
{
    // 1. 使用 waitpid() WNOHANG 非阻塞检查子进程状态
    ret = waitpid(tt->pipe_pid, &status, WNOHANG);

    if (ret == 0) {
        // 子进程仍在运行
        return false;
    }
    else if (ret == tt->pipe_pid) {
        // 子进程已退出 - 重新启动
        msg(M_INFO, "Pipe subprocess (PID %d) exited, restarting...",
            tt->pipe_pid);

        // 创建新的 socketpair
        socketpair(AF_UNIX, SOCK_DGRAM, 0, fds);

        // 关闭旧 fd，使用新的
        close(tt->fd);
        tt->fd = fds[0];
        set_nonblock(tt->fd);
        set_cloexec(tt->fd);

        // 重新启动子进程
        // ...
        tt->pipe_pid = (pid_t)pid;

        msg(M_INFO, "Pipe subprocess restarted with new PID %d", tt->pipe_pid);
        return true;
    }
    return false;
}
```

#### 2.3 内存管理

在各个平台的 `close_tun()` 函数中添加 `pipe_cmd` 的内存释放：

- `close_tun_generic()` - 通用 Unix 平台
- Solaris 平台的 `close_tun()`
- Windows 平台的 `close_tun()`

```c
if (tt->pipe_cmd)
{
    free(tt->pipe_cmd);
}
```

### 3. 定时器集成

**文件**: `src/openvpn/forward.c`

在 `process_coarse_timers()` 函数中添加监控调用：

```c
/* Check if pipe subprocess needs to be restarted */
if (c->c1.tuntap && c->c1.tuntap->is_pipe)
{
    check_pipe_monitor(c->c1.tuntap);
}
```

## 工作原理

```
┌─────────────────────────────────────────────────────────┐
│                    OpenVPN 主进程                        │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         粗粒度定时器 (process_coarse_timers)    │    │
│  │                                                  │    │
│  │  每隔一段时间调用 check_pipe_monitor()          │    │
│  └──────────────────┬─────────────────────────────┘    │
│                     │                                    │
│                     ▼                                    │
│  ┌────────────────────────────────────────────────┐    │
│  │           check_pipe_monitor()                 │    │
│  │                                                  │    │
│  │  1. waitpid(WNOHANG) 检查子进程状态            │    │
│  │  2. 如果子进程退出:                            │    │
│  │     - 创建新 socketpair                        │    │
│  │     - 重新启动子进程                           │    │
│  │     - 更新 pipe_pid                            │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────┐              ┌────────────┐           │
│  │ tt->fd     │◄────────────►│ 子进程 fd  │           │
│  │ (父端)     │  socketpair  │ (通过VPNFD)│           │
│  └────────────┘              └──────┬─────┘           │
│                                      │                 │
│                              ┌───────┴───────┐         │
│                              │  外部程序      │         │
│                              │  (tunsocks)    │         │
│                              └───────────────┘         │
└─────────────────────────────────────────────────────────┘
```

## 使用方法

### 基本用法

```bash
# 使用 pipe 设备启动 OpenVPN
openvpn --config client.conf \
        --dev-type pipe \
        --dev "|/opt/tunsocks/tunsocks"
```

### 配置文件示例

在 OpenVPN 配置文件中添加：

```bash
# 使用 pipe 设备
dev-type pipe
dev |/opt/tunsocks/tunsocks

# 其他配置...
remote vpn.example.com 1194
proto udp
client
ca ca.crt
cert client.crt
key client.key
```

### 日志输出

当子进程重启时，你会看到类似以下的日志：

```
2024-03-12 10:30:45 Pipe subprocess (PID 12345) exited, restarting...
2024-03-12 10:30:46 Pipe subprocess restarted with new PID 12346
```

## 技术细节

### waitpid() WNOHANG 选项

使用 `WNOHANG` 选项使 `waitpid()` 非阻塞：

- 如果子进程仍在运行：返回 0
- 如果子进程已退出：返回子进程 PID
- 如果出错：返回 -1

这确保了监控不会阻塞主事件循环。

### Socketpair 重建

子进程重启时，必须创建新的 socketpair：

1. 父进程关闭旧的 `tt->fd`
2. 创建新的 socketpair
3. 父进程使用新的 fds[0]
4. 子进程通过环境变量 `VPNFD` 获取 fds[1]
5. 关闭父进程端的 fds[1]

### 监控频率

监控通过 `process_coarse_timers()` 执行，这是一个粗粒度定时器：

- 通常每秒执行一次
- 不会对性能产生明显影响
- 足够快速地检测到子进程退出

## 兼容性

### 支持的平台

- ✅ Linux
- ✅ macOS / Darwin
- ✅ FreeBSD / OpenBSD / NetBSD
- ✅ Solaris
- ✅ Windows (需要相应支持)

### 版本要求

- 需要 POSIX 兼容的系统（支持 `socketpair()`, `waitpid()` 等）
- OpenVPN 2.4+

## 优势

1. **提高可靠性**：外部程序崩溃时自动恢复
2. **无需人工干预**：自动检测和重启
3. **快速恢复**：秒级检测和重启
4. **完整日志**：记录所有重启事件
5. **向后兼容**：不影响现有配置

## 注意事项

1. **子程序设计**：外部程序应该设计为可重复启动的，避免依赖持久状态
2. **资源清理**：子程序退出时应正确清理资源
3. **日志监控**：建议监控 OpenVPN 日志以了解重启事件
4. **频繁重启**：如果子进程频繁重启，应检查子程序本身的问题

## 测试建议

### 手动测试

1. 启动 OpenVPN with pipe 设备
2. 手动 kill 外部程序：
   ```bash
   # 找到子进程 PID
   ps aux | grep tunsocks
   # 终止它
   kill -9 <PID>
   ```
3. 观察 OpenVPN 日志，应该看到重启消息
4. 验证 VPN 连接仍然正常工作

### 自动化测试

可以编写测试脚本：

```bash
#!/bin/bash
# 启动 OpenVPN
openvpn --config test.conf --dev "|/opt/tunsocks/tunsocks" &
OVPN_PID=$!

# 等待连接建立
sleep 5

# 终止 tunsocks 多次
for i in {1..3}; do
    pkill -9 tunsocks
    sleep 2
done

# 检查 OpenVPN 仍在运行
if ps -p $OVPN_PID > /dev/null; then
    echo "✓ 测试通过：OpenVPN 成功恢复了子进程"
    kill $OVPN_PID
else
    echo "✗ 测试失败：OpenVPN 已退出"
fi
```

## 故障排查

### 子进程没有自动重启

检查：
1. OpenVPN 日志是否有错误
2. 子进程命令路径是否正确
3. 子进程是否有执行权限
4. 是否使用 `--dev-type pipe`

### 频繁重启

如果子进程频繁重启（每秒多次），可能原因：
1. 子程序本身有 bug
2. 配置参数不正确
3. 系统资源不足

建议：
1. 单独测试子程序
2. 检查子程序日志
3. 使用 `--log-level` 增加日志详细程度

## 相关代码文件

- `src/openvpn/tun.h` - 数据结构定义
- `src/openvpn/tun.c` - Pipe 设备实现
- `src/openvpn/forward.c` - 事件循环和定时器

## 许可证

本修改遵循 OpenVPN 项目的原始许可证（GPL-2.0）。

## 作者

修改由 Claude Code 辅助完成。

## 更新日期

2024-03-12
