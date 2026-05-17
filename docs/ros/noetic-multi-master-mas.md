# ROS1 Noetic：两个 master 互相发现并同步 rostopic（一步一步）

> 目标：机器 A 与机器 B 各自运行一个 `roscore`（multi-master），最终在两边都能看到并通信对方的 `rostopic`。

> 前提：你已经安装了 `fkie_mas_discovery` 和 `fkie_mas_sync`（可通过 fkie 提供的 deb 安装脚本安装，或在 ROS workspace 中源码编译后 `source devel/setup.bash`）。本文不展开安装细节，只给最小可用操作流程。

---

## 1) 场景说明

- 机器 A：运行自己的 `roscore`（例如 `http://A:11311`）
- 机器 B：运行自己的 `roscore`（例如 `http://B:11311`）
- 双方都运行：
  - `fkie_mas_discovery`（负责发现对方 master）
  - `fkie_mas_sync`（负责把对方 topic/service 注册信息同步到本地 master）

---

## 2) 术语解释：什么是“对外可达的地址”

**对外可达的地址** = 对端机器可以直接访问到你的地址（IP 或可解析 hostname）。

在 ROS1 中，节点会把自己的地址注册到 master。  
如果你设置了一个对端不可达地址，就会出现：
- `rostopic list` 能看到（注册信息同步了）
- 但 `rostopic echo` 连不上（真实数据链路失败）

### 常见错误示例
- `ROS_IP=127.0.0.1`（仅本机可达）
- `ROS_IP=172.x.x.x`（Docker 网卡地址，对端机器通常不可达）
- `ROS_IP=10.x.x.x`（VPN 虚拟网卡地址，局域网对端通常不可达）
- `ROS_HOSTNAME` 指向未被对端解析的主机名

---

## 3) 前提检查清单（开始前逐项确认）

- [ ] A 和 B 网络互通（`ping` 对方 IP）
- [ ] hostname 可解析（`getent hosts <hostname>`）
- [ ] 端口放通：
  - [ ] `11311/tcp`（ROS master XMLRPC）
  - [ ] `11511/udp` 到 `226.0.0.0`（组播 discovery 默认）
  - [ ] `11611`（discovery rpc，默认）
- [ ] 防火墙未拦截上述端口
- [ ] 时间同步建议：两机时间大致一致（`chrony` / `ntp`），便于日志排查

---

## 4) 最小可用步骤（可复制命令）

下面按“先做 A，再做 B，再验证”。

## Step 1：在 A/B 分别确认你要用的 IPv4 地址

在 A、B 各执行：

```bash
ip -4 addr
```

从输出中选一块**对端可达**网卡（例如同一网段的 `192.168.1.x`）。

示例假设：
- A_IP=`192.168.1.10`
- B_IP=`192.168.1.11`

## Step 2：在 A 上临时设置环境变量（仅当前终端）

> 不写入 `~/.bashrc`，避免污染系统默认环境。

```bash
export ROS_MASTER_URI=http://192.168.1.10:11311
export ROS_IP=192.168.1.10
# 如果你更想用主机名，也可改为 ROS_HOSTNAME（二选一）：
# export ROS_HOSTNAME=a-hostname
```

## Step 3：在 B 上临时设置环境变量（仅当前终端）

```bash
export ROS_MASTER_URI=http://192.168.1.11:11311
export ROS_IP=192.168.1.11
# 或 ROS_HOSTNAME（二选一）
```

## Step 4：A/B 各自启动自己的 roscore

A 上：

```bash
roscore
```

B 上：

```bash
roscore
```

## Step 5：A/B 各自启动 discovery

A 上：

```bash
roslaunch fkie_mas_discovery mas_discovery.launch
```

B 上：

```bash
roslaunch fkie_mas_discovery mas_discovery.launch
```

## Step 6：A/B 各自启动 sync

A 上：

```bash
roslaunch fkie_mas_sync mas_sync.launch
```

B 上：

```bash
roslaunch fkie_mas_sync mas_sync.launch
```

## Step 7：验证（区分“看得到”和“连得上”）

在 A、B 都做以下检查：

```bash
rostopic list
```

- 如果出现了对端 topic：说明“看得到”（注册同步基本成功）

```bash
rostopic info <一个来自对端的topic>
```

- 看发布者/订阅者地址是否是可达 IP/hostname

```bash
rostopic echo <一个来自对端的topic>
```

- 能持续收到数据才是“连得上”（通信链路真正成功）

---

## 5) 组播不通时的替代方案：`robot_hosts` 单播 discovery

有些网络（企业网、跨网段、VPN）会屏蔽组播。此时改用单播发现。

### A 机器示例：`mas_discovery_unicast.launch`

```xml
<launch>
  <include file="$(find fkie_mas_discovery)/launch/mas_discovery.launch">
    <arg name="send_mcast" value="False"/>
    <arg name="listen_mcast" value="False"/>
    <arg name="robot_hosts" value="['192.168.1.11']"/>
  </include>
</launch>
```

### B 机器示例：`mas_discovery_unicast.launch`

```xml
<launch>
  <include file="$(find fkie_mas_discovery)/launch/mas_discovery.launch">
    <arg name="send_mcast" value="False"/>
    <arg name="listen_mcast" value="False"/>
    <arg name="robot_hosts" value="['192.168.1.10']"/>
  </include>
</launch>
```

然后把 Step 5 的命令替换为：

```bash
roslaunch <your_pkg> mas_discovery_unicast.launch
```

---

## 6) 常见故障排查

### 6.1 `rostopic list` 有但 `rostopic echo` 不通

通常是 `ROS_IP` / `ROS_HOSTNAME` 选错（不可达网卡）。

先查本机：

```bash
echo "$ROS_MASTER_URI"
echo "$ROS_IP $ROS_HOSTNAME"
```

从对端验证连通性（替换为实际地址）：

```bash
ping -c 3 <对端ROS_IP>
nc -vz <对端ROS_IP> 11311
```

### 6.2 两个 master 互相发现不到

- 先怀疑组播被屏蔽、`11511/udp` 被挡
- 检查防火墙策略
- 直接切换到上面的 `robot_hosts` 单播模式

### 6.3 同步日志出现 topic 类型 / md5 不一致警告

说明两端同名 topic 的消息定义不一致。  
处理方式：统一消息包版本，并重新 `catkin build` / `source` 后再测。

---

## 7) 最佳实践（避免影响其他 ROS 任务）

- 不要把本场景变量长期写死到 `~/.bashrc`
- 用“单独终端 + 临时 `export`”或启动脚本隔离环境

示例脚本（A 机器）`run_on_master_a.sh`：

```bash
#!/usr/bin/env bash
set -e
export ROS_MASTER_URI=http://192.168.1.10:11311
export ROS_IP=192.168.1.10
exec "$@"
```

用法：

```bash
chmod +x ./run_on_master_a.sh
./run_on_master_a.sh roscore
./run_on_master_a.sh roslaunch fkie_mas_discovery mas_discovery.launch
./run_on_master_a.sh roslaunch fkie_mas_sync mas_sync.launch
```

---

如果你愿意，我可以按你的真实 A/B IP（或 hostname）把上面命令全部替换成“可直接粘贴执行”的版本。
