# open-vm-tools 插件说明

## 1. 插件定位

`open-vm-tools` 在爱快虚拟机内部运行 VMware Tools 的开源实现 `vmtoolsd`，用于让 VMware 系虚拟化宿主与爱快虚拟机通信。

适用平台：

- VMware ESXi
- VMware Workstation
- VMware Fusion

不适用平台：

- PVE / QEMU / KVM：请使用 `qemu-guest-agent` 插件。
- Hyper-V：请使用 Hyper-V 对应集成服务能力。

## 2. 能力范围

启用后，VMware 宿主侧可获得以下能力：

- 显示 VMware Tools 状态为“正在运行”
- 读取虚拟机 IP 地址、主机名等 guest 信息
- 宿主侧触发优雅关机 / 重启
- 时间同步
- 快照或备份时的 guest 协调能力，例如 vmbackup 相关能力

具体能力取决于 VMware 宿主版本、虚拟机硬件版本、宿主侧功能开关以及爱快系统环境。

## 3. 工作原理

插件内置 open-vm-tools 的核心运行时：

- `vmtoolsd`
- vmsvc 插件
- guestInfo / powerOps / timeSync / vmbackup 等必要插件
- glib、libtirpc 等 musl 运行时依赖
- `setsid-exec` 静态包装器

启动流程：

1. 检测当前虚拟机是否运行在 VMware 宿主上。
2. 通过自带 musl loader 启动 `vmtoolsd`。
3. 使用 `setsid-exec` 让进程脱离 appmarket 启动会话，避免 start.sh 返回后进程被回收。
4. 写入 pidfile 与日志。

## 4. 为什么需要 setsid-exec

爱快插件由 appmarket 调用插件启动。直接后台运行 `vmtoolsd` 容易遇到两个问题：

1. `vmtoolsd --background` 会重新 exec 裸 ELF，丢失自带 musl loader，导致在爱快系统上运行失败。
2. BusyBox shell 直接 `cmd &` 后台化时，进程仍在 `start.sh` 的会话或进程组中，`start.sh` 结束后可能被回收或收到 SIGHUP。

因此插件自带静态 `setsid-exec`：

- fork
- setsid
- 忽略 SIGHUP
- 写 pidfile
- 再 exec musl loader + vmtoolsd

这样 `vmtoolsd` 能稳定常驻。

## 5. 前置条件

ESXi / Workstation / Fusion 通常会自动识别 guest 内的 open-vm-tools 状态。

安装启动后，宿主界面中的 VMware Tools 状态应变为运行中，并不再要求客户机（guest）安装VMware Tools，因此无需前置条件。

## 6. 插件配置项

![undefined](https://img.meituan.net/content/1902dd2ec4962d391d514562a149d00250932.png)

### `OVT_FORCE`

默认值：`0`

含义：是否跳过 VMware 宿主检测。

- `0`：正常检测，仅在 VMware 环境启动
- `1`：强制启动，跳过检测

仅在检测误判时使用。非 VMware 环境强制启动没有实际意义。

### `OVT_VERBOSE`

默认值：`0`

含义：是否开启详细日志。

- `0`：关闭详细日志
- `1`：开启详细日志，日志写入插件日志目录

### `OVT_EXTRA_ARGS`

默认值：空

含义：附加传给 `vmtoolsd` 的参数。普通用户不需要配置。

## 7. 验证方法

VMware 宿主侧验证，在 ESXi / Workstation / Fusion UI 中查看：

- VMware Tools 状态是否为“正在运行”
- 是否能读取 guest IP 地址
- 是否能执行“关闭客户机操作系统”一类的优雅关机动作

## 8. 常见问题

### 8.1  VMware UI 中 Tools 状态仍未运行

处理：

1. 确认虚拟机硬件兼容性与宿主 VMware 版本正常。
2. 确认不是安装到了非 VMware 虚拟机。
3. 临时开启 `OVT_VERBOSE=1` 收集详细日志，反馈到论坛。

### 8.2 控制台出现 Unknown ioctl 刷屏

`vmtoolsd` 访问 VMware backdoor 时，部分内核会在控制台输出 `Unknown ioctl`。插件启动时会尝试执行：

```sh
dmesg -n 1
```

降低控制台日志等级，停止插件时恢复：

```sh
dmesg -n 7
```

这是为了避免控制台被刷屏，不影响 VMware Tools 功能。

## 9. 架构支持

Apple Silicon 上 VMware Fusion 运行 arm64 爱快（暂不提供ARM免费版）时可使用 arm64 包。

## 10. 打包内容说明

插件只打包无 GUI 所需的核心组件，重点覆盖路由虚拟机常用能力：

- vmsvc 服务
- guestInfo
- powerOps
- timeSync
- vmbackup

不包含：

- GTK/桌面组件
- HGFS 文件共享组件
- VIX/图形桌面相关组件

如运行环境不是 VMware，请选择对应虚拟化平台的 Guest Agent 插件。
