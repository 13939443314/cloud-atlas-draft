> Red Hat Enterprise Linux的《安全性指南》写𢔶非常详尽，本文是阅读学习过程基本只做了搬运，详细案例请参考实践：

* [找出瞬间消失的TCP网络连接进程](find_short_lived_tcp_connections_owner_process)

# 系统审计`audit`软件包

系统审计软件包 `audit` 和 `audit-libs` 是默认安装在RHEL系统中的。

# 配置`audit`服务

审核守护程序可以在 `/etc/audit/auditd.conf` 配置文件中进行配置。这个文件包括修改审核守护进程特性的配置参数。

所有配置参数的列表以及它们的解释都可以在 `audit.conf(5)` 手册页中找到。

# 为了 在 `CAPP` 环境配置 `auditd`

默认 `auditd` 配置应该对大多数环境都适合。但是可控制存取保护档案（`CAPP`）所建立的标准是公共标准认证的一部分，审核守护程序必须用以下设定配置：

* 保存审核日志文件的目录（`/var/log/audit/`）经常应该在另一个分区。这将防止其他进程耗费此目录中的空间，并且为剩余的审核守护程序提供准确的检测。

* `max_log_file` 参数详细说明了每个审核日志文件最少的占用空间，参数必须设定为充分利用保存审核日志文件分区所在的可用空间。

* `max_log_file_action` 参数决定采取何种行动，一旦到达在`max_log_file`中所设定的极限，则应该设定为 `keep_logs` 防止审核日志文件被重写。

* `space_left` 参数明确说明磁盘中可用空间的数量，这样的话在`space_left_action` 参数中所设定的行动会被触发。此参数必须被设定为一个数字它会给予管理者足够的时间来回应和刷新磁盘空间。 `space_left` 价值取决于审核日志文件生成的速度。

> 推荐采用合适的通知方法把 `space_left_action` 参数设定为`email` 或者 `exec`。

* `admin_space_left` 参数明确说明自由空间的绝对最小数量，为了在 `admin_space_left_action` 参数中所设定的行动会被触发，必须设定一个会给予管理者的日志行动总够空间的值。

* `admin_space_left_action` 参数必须设定 `single` 使系统属于单一用户模式，并且允许管理者开放一些磁盘空间。

* `disk_full_action` 参数明确说明当保存审核日志文件的分区没有可用空间时，应该触发行动，并且必须设定为 `halt` 或者 `single`。这保障了当审核不再记录事件时，系统也能在单一用户模式下关闭或者运行。

* `disk_error_action`，明确说明如果保存在审核日志文件的分区检测到错误时，应该采取行动，必须设定 `syslog`、`single` 或者 `halt`，这取决于当地的安全政策有关硬件故障的处理。

* `flush` 配置参数必须设定为 `sync` 或者 `data`。这些参数保证所有的审核事件数据能与磁盘中的日志文件同步。

# 开始`audit`服务

* 启动`auditd`服务

```
service auditd start
```

> `auditd`服务启动动作中还支持`reload`（重新加载配置）和`rotate`（轮转`/var/log/audit/`目录日志文件）。

* 激活系统启动时启动`auditd`

```
chkconfig auditd on
```

# 定义审核规则

审核系统根据一组规则运行，这组规则定义了日志文件中所获取的内容。有三种类型的审核规则可以详细说明：

* 控制规则 — 允许审核系统的行为和它的一些被修改的配置。
* 文件系统规则 — 也被称为文件监视，允许审核进入特定文件或者目录。
* 系统调用规则 — 允许记录任何指定程序所做的系统调用。

审核规则可以在命令行上使用 `auditctl` 实用程序进行详细说明（请注意这些规则并不是在重新启动时一直有效），或者写在 `/etc/audit/audit.rules` 文件中。

# 使用 auditctl 实用程序来定义审核规则

## 定义控制规则

* `-b` 在 Kernel 中设定最大数量的已存在的审核缓冲区

```
auditctl -b 8192
```

* `-f` 当追踪重要错误时设定所要完成的行动，例如：

```
auditctl -f 2
```

以上配置触发 kernel 恐慌以防重要错误。(?)

* `-e` 启动或者禁用审核系统或者锁定它的配置，例如：

```
auditctl -e 2
```

以上命令锁定审核配置。

* `-r` 设定每秒生成信息的速率，例如：

```
auditctl -r 0
```

以上配置在生成信息方面不设定限制速率。

* `-s` 报告审核系统状态，例如：

```
auditctl -s
```

输出举例

```
AUDIT_STATUS: enabled=1 flag=2 pid=0 rate_limit=0 backlog_limit=8192 lost=259 backlog=0
```

* `-l` 列出所有当前装载的审核规则，例如：

```
auditctl -l
```

输出举例

```
LIST_RULES: exit,always watch=/etc/localtime perm=wa key=time-change
LIST_RULES: exit,always watch=/etc/group perm=wa key=identity
LIST_RULES: exit,always watch=/etc/passwd perm=wa key=identity
LIST_RULES: exit,always watch=/etc/gshadow perm=wa key=identity
⋮
```

* `-D` 删除所有当前装载的审核规则，例如：

```
auditctl -D
```

## 定义文件系统规则

定义文件系统规则：

```
auditctl -w path_to_file -p permissions -k key_name
```

* `path_to_file` 是审核过的文件或者目录
* `permissions` 是被记录的权限：
  * `r` — 读取文件或者目录。
  * `w` — 写入文件或者目录。
  * `x` — 运行文件或者目录。
  * `a` — 改变在文件或者目录中的属性。
* `key_name` 是可选字符串，可帮助您判定哪个规则或者哪组规则生成特定的日志项。
⁠
### 定义文件系统规则举例

* 定义所有的输写访问权限以及在 /etc/passwd 文件中每个属性更改的规则，执行以下命令：

```
auditctl -w /etc/passwd -p wa -k passwd_changes
```

> 字符串 `-k` 选项是任意的

* 记录所有输写访问权限，以及在 `/etc/selinux/` 目录中所有文件属性更改的规则，执行以下命令：

```
auditctl -w /etc/selinux/ -p wa -k selinux_changes
```

* 定义可以记录执行 /sbin/insmod 命令的规则，在 Linux Kernel 中插入模块，执行以下命令：

```
auditctl -w /sbin/insmod -p x -k module_insertion
```

## 定义系统调用规则

为了定义系统调用规则，使用以下语法：

```
auditctl -a action,filter -S system_call -F field=value -k key_name
```

* `action` 以及 `filter` 详细说明某个事件何时被记录。 `action` 可能是 `always`（经常是）或者`never`（从不是）其中之一。 `filter` 详细说明哪个 Kernel 规则匹配过滤器应用在事件中。以下是其中之一的与规则匹配的过滤器： `task`、`exit`、`user` 以及 `exclude`。

* `system_call` 通过它的名字详细说明系统调用。所有的系统调用都可以在`/usr/include/asm/unistd_64.h` 文件中找到。许多系统调用都能形成一个规则，每个都在 `-S` 选项之后详细说明。

* `field=value` 详细说明其他选项，进一步修改规则来与以特定架构、组 ID、进程 ID和其他内容为基础的事件相匹配。为了列出完整可用的输入栏类型和它们的数值，请参考 `auditctl(8)` 手册页。

* `key_name` 是可选字符串，可帮助判定哪个规则或者哪组规则生成特定的日志项。

### 定义系统调用规则举例

* 为了定义创造日志项 的规则，每次通过程序使用系统调用 adjtimex 或者 settimeofday。当系统使用 64 位架构，请执行以下命令：

```
auditctl -a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time_change
```

* 为了定义创造日志项的规则，每次由 ID 是 500 或更大的系统用户删除或者重命名文件时，使用（`-F auid!=4294967295` 选项排除没有设定登录 UID的用户），执行以下命令：

```
auditctl -a always,exit -S unlink -S unlinkat -S rename -S renameat -F auid>=500 -F auid!=4294967295 -k delete
```

* 使用系统调用语法来定义文件系统也是有可能的。对于 `-w /etc/shadow -p wa`文件系统规则来说，以下命令为模拟的系统调用创造了规则：

```
auditctl -a always,exit -F path=/etc/shadow -F perm=wa
```

## 在 `/etc/audit/audit.rules` 文件中定义持久的审核规则和控制

### 定义控制规则

文件可以只包括以下的控制规则，修改审核系统的行为： -b、-D、-e、-f、或者 -r。

* 在 audit.rules中控制规则

```
# Delete all previous rules
-D

# Set buffer size
-b 8192

# Make the configuration immutable -- reboot is required to change audit rules
-e 2

# Panic when a failure occurs
-f 2

# Generate at most 100 audit messages per second
-r 100
```

### ⁠定义文件系统和系统调用规则

使用 auditctl 语法定义文件系统和系统调用原则。在上述的例子可以用以下规则文件来表示：

```
-w /etc/passwd -p wa -k passwd_changes
-w /etc/selinux/ -p wa -k selinux_changes
-w /sbin/insmod -p x -k module_insertion

-a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time_change
-a always,exit -S unlink -S unlinkat -S rename -S renameat -F auid>=500 -F auid!=4294967295 -k delete
```

## 预配置规则文件

在 `/usr/share/doc/audit-version/` 目录中, 根据不同的认证标准 audit 软件包提供一组预配置规则文件：

* nispom.rules — 审核规则配置符合《国家行业安全程序操作运行指南》的第八章中详细说明的要求。
* capp.rules — 审核规则配置满足由 CAPP 设定的要求，是公共标准认定的一部分。
* lspp.rules —审核规则配置满足由 LSPP 设定的要求是公共标准认定的一部分。
* stig.rules — 审核规则配置满足由 STIG 所设定的要求。

为了使用这些配置文件，需要原始文件的备份 `/etc/audit/audit.rules `并且复制选择的有关 `/etc/audit/audit.rules` 文件的配置文件:

```
cp /etc/audit/audit.rules /etc/audit/audit.rules_backup
cp /usr/share/doc/audit-version/stig.rules /etc/audit/audit.rules
```

# 理解审核日志文件

默认情况下，在 `/var/log/audit/audit.log` 文件中的审核系统储存日志项；如果启用日志旋转，就可以旋转储存在同一目录中的 `audit.log` 文件。

以下的审核规则记录了每次读取或者修改 `/etc/ssh/sshd_config` 文件的尝试：

```
-w /etc/ssh/sshd_config -p warx -k sshd_config
```

如果 auditd 守护程序在运行，就需在审核日志文件中运行以下命令创造新事件：

```
~]# cat /etc/ssh/sshd_config
```

在 `audit.log` 文件中的事件如下所示：

```
type=SYSCALL msg=audit(1364481363.243:24287): arch=c000003e syscall=2 success=no exit=-13 a0=7fffd19c5592 a1=0 a2=7fffd19c4b50 a3=a items=1 ppid=2686 pid=3538 auid=500 uid=500 gid=500 euid=500 suid=500 fsuid=500 egid=500 sgid=500 fsgid=500 tty=pts0 ses=1 comm="cat" exe="/bin/cat" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="sshd_config"
type=CWD msg=audit(1364481363.243:24287):  cwd="/home/shadowman"
type=PATH msg=audit(1364481363.243:24287): item=0 name="/etc/ssh/sshd_config" inode=409248 dev=fd:00 mode=0100600 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:etc_t:s0
```

以上事件由三个记录组成（每个以 `type=`作为开始），共享相同的时间戳和编号。每个记录包含好几对 `name=value `，由空格或者逗号分开。以下是关于以上事件的详细分析：

## 第一个记录

* `type=SYSCALL`

type 输入栏包含这类记录。在这个例子中， SYSCALL 数值详细说明连接到 Kernel 的系统调用触发了这个记录。

> 所有可能的类型值和它们的解释，请参考 [审核记录类型](https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Audit_Record_Types.html)。

* `msg=audit(1364481363.243:24287):`
  * audit(time_stamp:ID) 表格中记录的时间戳和特殊 ID。如果多种记录生成为 **相同审核事件的一部分，那么它们可以共享相同的时间戳和ID**。
  * Kernel 或者用户空间应用提供不同的事件特定 `name=value` 组。

# 参考

* [红帽企业版Linux 7: 安全性指南 - 系统审核](https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/7/html/Security_Guide/chap-system_auditing.html)