---
title: VMWare 以用户systemd形式自动挂载共享文件夹
published: 2026-04-01
description: >-
  一篇关于 VMware 虚拟机共享文件夹自动挂载的实战指南。针对 Linux 宿主环境，通过简单的用户级 systemd
  配置，将共享文件夹自动挂载至用户家目录，解决传统方案权限配置复杂的问题。
draft: false
slug: 260401_vmware_share
tags:
  - vmware
---

# VMware 共享文件夹自动挂载指南

> - 本文由Codex完成，本人仅在Fedora 43系统（niri+dms）上测试成功，其它系统请自行测试

适用场景：

- 宿主机使用 VMware
- 已在 VMware 里开启 Shared Folders
- 希望在图形会话登录后自动挂载

这份方案默认使用“用户级 `systemd` 服务”来完成自动挂载。

优点：

- 不需要修改 `/etc/fstab`
- 不依赖桌面环境的自动挂载组件
- 不需要 root 权限
- 对 Wayland compositor 场景更稳
- 后续新增共享文件夹后，不需要改服务文件

## 1. 前提检查

确认系统已安装 VMware Tools：

```bash
rpm -q open-vm-tools open-vm-tools-desktop
```

确认相关命令存在：

```bash
which vmhgfs-fuse vmware-hgfsclient vmtoolsd
```

确认 VMware 已经把共享目录暴露给客户机：

```bash
vmware-hgfsclient
```

如果这里能列出共享目录名，例如：

```text
Share
Work
```

说明 VMware 共享文件夹功能已经正常。

## 2. 创建用户级 systemd 服务

创建目录：

```bash
mkdir -p ~/.config/systemd/user
```

写入服务文件 `~/.config/systemd/user/vmhgfs-mount.service`：

```ini
[Unit]
Description=Mount VMware shared folders in the user session
Documentation=https://github.com/vmware/open-vm-tools
ConditionVirtualization=vmware
ConditionPathExists=/usr/bin/vmhgfs-fuse

[Service]
Type=simple
ExecStartPre=/usr/bin/mkdir -p %h/hgfs
ExecStartPre=/usr/bin/bash -lc '/usr/bin/mountpoint -q %h/hgfs && /usr/bin/fusermount3 -u %h/hgfs || true'
ExecStart=/usr/bin/bash -lc '/usr/bin/vmhgfs-fuse .host:/ %h/hgfs -f -o uid=$(id -u),gid=$(id -g),auto_unmount'
ExecStop=/usr/bin/bash -lc '/usr/bin/mountpoint -q %h/hgfs && /usr/bin/fusermount3 -u %h/hgfs || true'
Restart=on-failure
RestartSec=2

[Install]
WantedBy=default.target
```

说明：

- 挂载点固定为 `~/hgfs`
- `.host:/` 表示挂载“所有共享文件夹”的根
- 这不是硬编码某一个共享目录
- `uid=$(id -u)` 和 `gid=$(id -g)` 会在服务启动时读取当前用户的实际编号
- 因此不需要假设当前用户一定是 `1000:1000`
- 后续你在 VMware 里新增共享目录后，重新登录或重启后会自动出现在 `~/hgfs/` 下

## 3. 启用服务

```bash
systemctl --user daemon-reload
systemctl --user enable --now vmhgfs-mount.service
```

检查状态：

```bash
systemctl --user status --no-pager vmhgfs-mount.service
```

`--no-pager` 的作用是禁止 `systemctl` 调用分页器（例如 `less`），让结果直接一次性输出完，适合脚本和非交互环境。

如果你修改过服务文件，记得重新加载：

```bash
systemctl --user daemon-reload
systemctl --user restart vmhgfs-mount.service
```

## 4. 验证挂载结果

查看挂载状态：

```bash
findmnt ~/hgfs
```

查看共享目录：

```bash
ls -la ~/hgfs
```

正常情况下会看到类似：

```text
~/hgfs/Share
~/hgfs/Work
```

## 5. 后续维护

如果你在 VMware 里新增了共享目录：

- 通常重新登录图形会话即可生效
- 或者手动刷新服务

```bash
systemctl --user restart vmhgfs-mount.service
```

如果你删除了某个共享目录：

- 重新登录或重启服务后，对应目录会从 `~/hgfs` 下消失

## 6. 卸载或回滚

停止并禁用服务：

```bash
systemctl --user disable --now vmhgfs-mount.service
```

删除服务文件：

```bash
rm -f ~/.config/systemd/user/vmhgfs-mount.service
systemctl --user daemon-reload
```

如果还需要移除挂载点目录：

```bash
rmdir ~/hgfs
```

## 7. 设计说明

这份配置选择的是“登录后自动挂载”，不是“系统开机但用户未登录时就挂载”。

这样做的原因：

- `vmhgfs-fuse` 本身非常适合在用户会话里运行
- 不需要 root 配置
- 对 `niri + dms` 这类轻量会话环境更直接
- 挂载目录在用户家目录下，权限模型更简单
- 服务启动时动态读取当前用户的 UID/GID，适合跨机器复用

如果你需要的是系统级挂载，例如固定挂到 `/mnt/hgfs`，可以另做一个 root 级的 `systemd` mount/service 方案，但那是另一套配置。
