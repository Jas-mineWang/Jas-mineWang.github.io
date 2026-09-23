---
layout: post
title: "Ubuntu 24.04 加入 FreeIPA 后 SSSD 失联：从 Invalid user 到 socket activation 冲突"
date: 2026-09-22 18:30:00 +0800
categories: [freeipa, sssd, ubuntu, incident-response]
---

这篇文章记录一次 Ubuntu 24.04 虚拟机加入 FreeIPA 后，域用户一度无法通过 SSH 登录的排查过程。

故障表面上同时出现了 Kerberos 预认证失败、SSH 公钥权限错误、SSSD socket 启动失败和 `Invalid user`，但它们并不是同一个问题。真正导致域用户无法登录的直接原因，是 SSSD 在短时间内被多次重启，最终命中 systemd 的启动频率限制并停止运行。

更深一层的隐患，是 Ubuntu 24.04 为 SSSD responder 默认启用了 systemd socket activation，而 `ipa-client-install` 生成的 `sssd.conf` 又使用了传统的 `services = ...` 启动方式，导致两套机制同时存在。

> 文中主机名、域名、用户名和 IP 地址均已泛化。修改 SSSD 或 PAM 前，请保留一个已登录的 root 会话或控制台，不要在验证新配置前关闭唯一的管理通道。

## 环境与入域方法

现场环境的核心版本为：

- Ubuntu 24.04 LTS；
- SSSD 2.9.4；
- FreeIPA Client 4.11.1；
- 身份域示例：`example.com` / `EXAMPLE.COM`。

虚拟机使用的入域流程如下：

```bash
sudo apt update
sudo apt install freeipa-client -y

sudo hostnamectl set-hostname client01.example.com

sudo ipa-client-install \
  --mkhomedir \
  --domain=example.com \
  --realm=EXAMPLE.COM \
  --enable-dns-updates \
  --ntp-server=ntp.example.com \
  --no-krb5-offline-passwords
```

这个流程本身没有明显错误。各参数的作用分别是：

- `--mkhomedir`：配置 PAM，在用户首次登录时创建 home 目录；
- `--domain` / `--realm`：指定 IPA DNS 域与 Kerberos realm；
- `--enable-dns-updates`：允许 SSSD 使用 GSS-TSIG 更新 DNS；
- `--ntp-server`：配置时间源，Kerberos 对时钟偏差非常敏感；
- `--no-krb5-offline-passwords`：不在离线时保存用户密码。

可以用 `--hostname=client01.example.com` 把 FQDN 显式传给 `ipa-client-install`，但提前通过 `hostnamectl` 设置静态 FQDN 同样是正确做法，与本次故障无关。

## 现象：域用户被 SSH 识别为无效用户

SSH 日志一度显示：

```text
Invalid user user1 from 10.0.0.10
pam_sss(sshd:auth): Request to sssd failed. Connection refused
PAM: Authentication failure for illegal user user1
```

`Invalid user` 不一定表示 IPA 中没有这个用户。SSH 在认证前会先通过 NSS 查询用户；当 SSSD 没有运行时，`nss_sss` 无法返回域用户，sshd 便会把该用户标记为无效用户。

启动 SSSD 后，同一用户能够被查到：

```bash
getent passwd user1
id user1
```

后续日志也显示了成功认证：

```text
pam_sss(sshd:auth): authentication success
Accepted keyboard-interactive/pam for user1
pam_unix(sshd:session): session opened for user user1
```

这证明 IPA 用户查询、Kerberos 密码验证、PAM 和 HBAC 授权链路已经恢复。

## 先验证 keytab、时间和主机凭据

检查主机 keytab：

```bash
klist -kte /etc/krb5.keytab
```

如果同一个 host principal 出现多行，通常只是不同加密类型，不是重复故障。`-e` 参数可显示每条记录的 enctype。

使用 keytab 获取主机票据：

```bash
kinit -k
echo $?
klist
```

当 `kinit -k` 没有报错、退出码为 0，并且 `klist` 能看到类似下面的 TGT 时，说明主机 keytab 基本可用：

```text
Default principal: host/client01.example.com@EXAMPLE.COM
krbtgt/EXAMPLE.COM@EXAMPLE.COM
```

同时检查：

```bash
timedatectl
ls -l /etc/krb5.keytab
```

期望时钟已同步，且 keytab 权限为 `0600 root:root`。服务器使用 UTC 不会导致 Kerberos 失败，只要实际时间与 KDC 一致即可。

## 日志时间线：SSSD 为什么停了

`journalctl -u sssd` 显示 SSSD 在一分钟内被多次停止和启动：

```text
Stopping sssd.service...
Shutting down (status = 0)
Deactivated successfully
Starting sssd.service...
```

`status = 0` 和 `Deactivated successfully` 表明这不是进程崩溃、段错误或 OOM，而是 systemd 发起的正常停止。

同一时间窗口中，`apt-daily-upgrade.service` 正在运行，并多次触发 systemd reload/reexec：

```text
Reexecuting requested from client ... (unit apt-daily-upgrade.service)
Reloading requested from client ... (unit apt-daily-upgrade.service)
```

最后 SSSD 触发了 systemd 的启动频率限制：

```text
sssd.service: Start request repeated too quickly
sssd.service: Failed with result 'start-limit-hit'
Failed to start sssd.service
```

因此完整链路是：

```text
apt-daily-upgrade 运行期间发生多次 systemd reexec/服务切换
    ↓
SSSD 在短时间内被多次停止和启动
    ↓
systemd 触发 start-limit-hit
    ↓
SSSD 保持停止
    ↓
NSS 无法查到 IPA 用户
    ↓
sshd 报 Invalid user
```

仅凭 journal 可以确认这些事件处于同一个 `apt-daily-upgrade` 上下文，但不足以确定究竟是哪个具体包的 maintainer script 发起了每次服务切换。可继续对照：

```bash
less /var/log/apt/history.log
less /var/log/dpkg.log
less /var/log/unattended-upgrades/unattended-upgrades.log
```

## 根本隐患：两套 SSSD responder 启动方式同时存在

Ubuntu 24.04 的 `sssd-common` 包安装并默认启用多个 socket 单元：

```bash
systemctl list-unit-files --type=socket | grep -E '^sssd-'
```

现场结果为：

```text
sssd-autofs.socket       enabled
sssd-nss.socket          enabled
sssd-pac.socket          enabled
sssd-pam-priv.socket     enabled
sssd-pam.socket          enabled
sssd-ssh.socket          enabled
sssd-sudo.socket         enabled
```

但 `ipa-client-install` 同时在 `/etc/sssd/sssd.conf` 中生成了：

```ini
[sssd]
services = nss, pam, ssh, sudo
```

这代表两套模式并存：

| 模式 | 启动方式 | 当前状态 |
| --- | --- | --- |
| 传统 monitor 模式 | `sssd.service` 按 `services = ...` 启动 responder | 已配置 |
| systemd socket activation | `sssd-*.socket` 在收到请求时启动 responder | 已启用 |

SSSD 的检查程序会主动拒绝重复启动：

```text
Misconfiguration found for the nss responder
The nss responder has been configured to be socket-activated
but it's still mentioned in the services' line
sssd-nss.socket: Failed with result 'exit-code'
```

因此，socket 的 `status=17` 本质上是保护检查生效，防止两个 responder 同时占用同一个 Unix socket。主 `sssd.service` 在这种情况下仍可能正常运行，但 systemd 会持续留下失败的 socket 单元，并在重载、升级或重启时增加复杂度。

这不是入域命令参数使用错误，而是 Ubuntu 的 SSSD 打包默认值与 `ipa-client-install` 生成的传统配置没有完全对齐。Ubuntu 24.04 上已有用户报告相同现象，并通过删除 `services = nss, pam, ssh, sudo` 恢复 socket activation。

## 修复原则：两种模式二选一

两种方案都可以工作，关键是不要混用。

### 方案 A：使用 Ubuntu 默认的 socket activation（推荐）

对 Ubuntu 24.04，更符合发行版默认的做法是保留 socket，删除 `services` 行。

先备份配置：

```bash
cp -a /etc/sssd/sssd.conf \
  /etc/sssd/sssd.conf.bak-$(date +%Y%m%d-%H%M%S)
```

编辑 `/etc/sssd/sssd.conf`，删除：

```ini
services = nss, pam, ssh, sudo
```

然后执行：

```bash
sssctl config-check

systemctl enable \
  sssd-nss.socket \
  sssd-pam.socket \
  sssd-pam-priv.socket \
  sssd-ssh.socket \
  sssd-sudo.socket

systemctl reset-failed \
  sssd.service \
  sssd-nss.socket \
  sssd-pam.socket \
  sssd-pam-priv.socket \
  sssd-ssh.socket \
  sssd-sudo.socket

systemctl restart sssd
systemctl restart \
  sssd-nss.socket \
  sssd-pam-priv.socket \
  sssd-pam.socket \
  sssd-ssh.socket \
  sssd-sudo.socket
```

socket activation 下，某个 responder 可能只有在收到第一个请求后才出现对应进程，这是正常现象。

### 方案 B：保留传统 `services` 模式

如果环境需要保留：

```ini
services = nss, pam, ssh, sudo
```

则应禁用相同 responder 的 socket：

```bash
systemctl disable --now \
  sssd-nss.socket \
  sssd-pam.socket \
  sssd-pam-priv.socket \
  sssd-ssh.socket \
  sssd-sudo.socket

systemctl reset-failed \
  sssd.service \
  sssd-nss.socket \
  sssd-pam.socket \
  sssd-pam-priv.socket \
  sssd-ssh.socket \
  sssd-sudo.socket
sssctl config-check
systemctl restart sssd
```

`sssd-pac.socket` 和 `sssd-autofs.socket` 未必与当前 `services` 列表冲突，应根据实际使用情况处理，不要在没有验证的情况下一次性 mask 所有 SSSD socket。

## 修复后的验证清单

先检查服务和域状态：

```bash
systemctl is-active sssd
systemctl --failed
sssctl config-check
sssctl domain-status example.com
```

再验证身份查询和授权：

```bash
getent passwd user1
id user1
sssctl user-checks -s sshd user1
```

最后从另一个终端实际登录，并在服务端观察：

```bash
journalctl -u ssh -u sssd -f
```

期望看到：

```text
pam_sss(sshd:auth): authentication success
Accepted keyboard-interactive/pam for user1
session opened for user user1
```

## 几条容易误判的日志

### `Preauthentication failed`

```text
krb5_child: Preauthentication failed
```

常见于用户密码错误、密码过期、账户锁定或旧密钥缓存。当 `kinit -k` 能够获取主机 TGT 时，不应立即把这条日志归因于主机 keytab 损坏。应先通过时间戳把它与具体 SSH/PAM 登录尝试对齐。

如需在不覆盖 root 当前票据缓存的情况下测试用户密码，可使用独立缓存：

```bash
KRB5CCNAME=FILE:/tmp/krb5cc_user1 \
  kinit user1@EXAMPLE.COM

KRB5CCNAME=FILE:/tmp/krb5cc_user1 \
  klist
```

### `pam_unix` 失败，但 `pam_sss` 成功

```text
pam_unix(sshd:auth): authentication failure
pam_sss(sshd:auth): authentication success
```

域用户通常不存在于本地 `/etc/shadow`，所以 `pam_unix` 失败并不代表整个 PAM 认证失败。只要后续 `pam_sss` 成功并出现 `Accepted ...`，整体认证就是成功的。

### `Enumeration requested but not enabled`

```text
sssd_nss: Enumeration requested but not enabled
```

这通常是因为执行了不带用户名的 `getent passwd`，试图枚举整个域。SSSD 对 IPA/AD 默认不建议全量枚举，但指定用户查询仍可正常工作：

```bash
getent passwd user1
```

## SSH 公钥权限是另一个独立问题

本次日志还出现了：

```text
Authentication refused: bad ownership or modes for file
/home/user1/.ssh/authorized_keys
```

这条错误只表示 sshd 认为 home、`.ssh` 或 `authorized_keys` 的所有者/权限不安全，所以拒绝读取公钥。它不会导致 SSSD 停止。

检查完整路径：

```bash
namei -l /home/user1/.ssh/authorized_keys

stat -c '%U:%G %a %n' \
  /home/user1 \
  /home/user1/.ssh \
  /home/user1/.ssh/authorized_keys
```

常见修复方式：

```bash
chown user1:"$(id -gn user1)" \
  /home/user1/.ssh \
  /home/user1/.ssh/authorized_keys

chmod 700 /home/user1/.ssh
chmod 600 /home/user1/.ssh/authorized_keys
chmod go-w /home/user1
```

修复后应在 SSH 日志中看到：

```text
Accepted publickey for user1
```

## 建议的 Ubuntu 24.04 FreeIPA 入域 SOP

后续新建 Ubuntu 24.04 客户端时，可在原有入域流程后增加一个 SSSD 启动模式检查：

```bash
# 1. 确认基础环境
hostname -f
timedatectl
getent hosts ipa.example.com
dig +short _ldap._tcp.example.com SRV
dig +short _kerberos._udp.example.com SRV

# 2. 安装并入域
apt update
apt install -y freeipa-client

ipa-client-install \
  --hostname=client01.example.com \
  --mkhomedir \
  --domain=example.com \
  --realm=EXAMPLE.COM \
  --enable-dns-updates \
  --ntp-server=ntp.example.com \
  --no-krb5-offline-passwords

# 3. 检查是否同时存在 services 和已启用 socket
grep -E '^[[:space:]]*services[[:space:]]*=' /etc/sssd/sssd.conf
systemctl list-unit-files --type=socket | grep -E '^sssd-'

# 4. 统一为一种 responder 启动模式后验证
sssctl config-check
systemctl status sssd --no-pager
kinit -k
klist
getent passwd user1
sssctl user-checks -s sshd user1
```

最后再从远端测试密码和公钥登录，不要只根据 `systemctl status sssd` 一项判定入域成功。

## 总结

这次故障中同时存在三类问题：

1. SSSD 短时间频繁启停并命中 `start-limit-hit`，是域用户一度被判定为 `Invalid user` 的直接原因；
2. Ubuntu 默认启用 SSSD socket，而 `ipa-client-install` 生成了传统 `services` 列表，两种 responder 启动模式并存是需要消除的隐患；
3. `authorized_keys` 权限过宽只影响 SSH 公钥认证，不会导致 SSSD 进程异常。

排查这类问题时，最重要的是按时间戳将 SSH、PAM、SSSD、Kerberos 和 systemd 日志串起来，并明确区分“身份查询”、“密码认证”、“账户授权”和“SSH 公钥安全检查”四条不同的链路。

## 参考资料

- [SSSD: Systemd Activatable Responders](https://sssd.io/design-pages/systemd_activatable_responders.html)
- [Ubuntu 24.04 `sssd-common` 文件列表](https://packages.ubuntu.com/noble/all/sssd-common/filelist)
- [FreeIPA `ipa-client-install` 手册](https://github.com/freeipa/freeipa/blob/master/client/man/ipa-client-install.1)
- [Ubuntu SSSD Bug #1838680](https://bugs.launchpad.net/ubuntu/+source/sssd/+bug/1838680)
