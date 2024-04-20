---
title: Linux权能与PAM机制
date: 2018-05-01 08:09:12
updated: 2018-05-01 08:09:12
tags:
 Linux
 capability
categories: Linux
copyright: true
---

> 实验环境：`uname -a`
>
> ​      `Linux ubuntu 4.4.0-121-generic #145-Ubuntu SMP Fri Apr 13 13:47:23 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux`

------

### 权能对应的系统调用

使用 `sudo find / -name capability.h ` 查找到路径 `/usr/src/linux-headers-4.4.0-121/include/uapi/linux/capbility.h`  ，整理可得下表：

<!-- more -->

| 权能                 | 编号(相关系统调用） | 解释                                                        |
| -------------------- | ------------------- | ----------------------------------------------------------- |
| CAP_CHOWN            | 0(chown)            | 对文件的UIDs和GIDs做任意修改                                |
| CAP_DAC_OVERRIDE     | 1                   | 忽略对文件的DAC访问限制                                     |
| CAP_DAC_READ_SEARCH  | 2                   | 忽略DAC中对文件和目录的读、搜索权限                         |
| CAP_FOWNER           | 3                   | 忽略进程UID与文件UID的匹配检查                              |
| CAP_FSETID           | 4                   | 文件修改时不清除setuid和setgid位，不匹配时设置setgid位      |
| CAP_KILL             | 5(kill)             | 绕过发送信号时的权限检查                                    |
| CAP_SETGID           | 6(setgid)           | 设置或管理进程GID                                           |
| CAP_SETUID           | 7(setuid)           | 管理或设置进程UID                                           |
| CAP_SETPCAP          | 8(capset)           | 允许授予或删除其他进程的任何权能                            |
| CAP_LINUX_IMMUTABLE  | 9(chattr)           | 允许设置文件的不可修改位(IMMUTABLE)和只添加(APPND-ONLY)属性 |
| CAP_NET_BIND_SERVICE | 10                  | 允许绑定到小于1024的端口                                    |
| CAP_NET_BROADCAST    | 11                  | 允许socket发送监听组播                                      |
| CAP_NET_ADMIN        | 12                  | 允许执行网络管理任务                                        |
| CAP_NET_RAW          | 13(socket)          | 允许使用原始套接字                                          |
| CAP_IPC_LOCK         | 14(mlock)           | 允许锁定共享内存片段                                        |
| CAP_IPC_OWNER        | 15                  | 忽略IPC所有权检查                                           |
| CAP_SYS_MOUDLE       | 16(init_module)     | 插入和删除内核模块                                          |
| CAP_SYS_RAWIO        | 17                  | 允许对ioperm/iopl的访问                                     |
| CAP_SYS_CHROOT       | 18(chroot)          | 允许使用chroot()系统调用                                    |
| CAP_SYS_PTRACE       | 19(ptrace)          | 允许跟踪任何进程                                            |
| CAP_SYS_PACCT        | 20(acct)            | 允许配置进程记账                                            |
| CAP_SYS_ADMIN        | 21                  | 允许执行系统管理任务                                        |
| CAP_SYS_BOOT         | 22(reboot)          | 允许重新启动系统                                            |
| CAP_SYS_NICE         | 23(nice)            | 允许提升优先级，设置其他进程优先级                          |
| CAP_SYS_RESOURCE     | 24(setrlimit)       | 设置资源限制                                                |
| CAP_SYS_TIME         | 25(stime)           | 允许改变系统时钟                                            |
| CAP_SYS_TTY_CONFIG   | 26(vhangup)         | 允许配置TTY设备                                             |
| CAP_MKNOD            | 27(mknod)           | 允许使用mknod()系统调用，创建特殊文件                       |
| CAP_LEASE            | 28(fcntl)           | 为任意文件建立租约                                          |
| CAP_AUDIT_WRITE      | 29                  | 允许向内核审计日志写记录                                    |
| CAP_AUDIT_CONTROL    | 30                  | 启用或禁用内核审计，修改审计过滤器规则                      |
| CAP_SETFCAP          | 31                  | 设置文件权能                                                |
| CAP_MAC_OVERRIDE     | 32                  | 允许MAC配置或状态改变，为smack LSM实现                      |
| CAP_MAC_ADMIN        | 33                  | 覆盖强制访问控制                                            |
| CAP_SYSLOG           | 34(syslog)          | 执行特权syslog(2)操作                                       |
| CAP_WAKE_ALARM       | 35                  | 触发将唤醒系统的东西                                        |
| CAP_BLOCK_SUSPEND    | 36(epoll)           | 可以阻塞系统挂起的特性                                      |
| CAP_AUDIT_READ       | 37                  | 允许通过一个多播socket读取审计日志                          |

详情见[man capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html) 。

------

### 基于PAM用户权限设置系统

PAM 的全称为“可插拔认证模块（Pluggable Authentication Modules）”。设计的初衷是将不同的底层认证机制集中到一个高层次的API中，从而省去开发人员自己去设计和实现各种繁杂的认证机制的麻烦。

PAM认证一般遵循这样的顺序：Service(服务)→PAM(配置文件)→pam_*.so，PAM配置文件在`/etc/pam.d/` 目录下，原理图如下：

![pam](pam.png)

PAM中配置字段：

```text
moudle_type  control_flag  moudle_path moudle_option/moudle_args
```

1. `module_type` 为 相应服务指定模块类型

| 模块类型              | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| auth(验证模块)        | 用于验证用户或设置/销毁凭证                                  |
| account(帐户管理模块) | 将执行与访问、帐户及凭证有效期、密码限制/规则等有关的操作    |
| session(会话管理模块) | 定义用户登录前的,及用户退出后所要进行的操作.如:登录连接信息,用户数据的打开与关闭,挂载文件系统等 |
| passwd(密码管理模块)  | 将执行与密码更改/更新有关的操作                              |

2. control_flag 将指定模块的堆栈行为，配置文件中列出模块被调用的顺序

| 控制标记                              | 说明                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| required                              | 堆栈中的所有 required 模块必须看作一个成功的结果。如果一个或多个 required 模块失败，则实现堆栈中的所有 required 模块，但是将返回第一个错误 |
| requisite                             | 如果本模块没有被满足，那本次认证一定失败，而且整个栈立即中止并返回错误信号 |
| sufficient                            | 如果标记为 sufficient 的模块成功并且先前没有 required 或 sufficient 模块失败，则忽略堆栈中的所有其余模块并返回成功 |
| optional                              | 如果堆栈中没有一个模块是 required 并且没有任何一个 sufficient 模块成功，则服务/应用程序至少要有一个 optional 模块成功 |
| include                               | 包含其他规则（服务），文件嵌套，可以互相调用，如：login，auth，system-auth等 |
| [value1=action1  value2=action2 ....] | 六种动作：ok ，done，bad，die，ignore，reset。               |

​        Example：auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so

3. `module_path` 将指定实现模块的库对象的路径名称。默认情况下，它将被设为 `/lib/security`。        ![libpam](libpam.png)
4. module_options/module_args（可选字段）将指定可以传递给服务模块的选项或实参。

------

实验：配置用户`userping` ，设置`cap_net_raw` 权能。

- 添加用户`userping`  :  `sudo adduser userping` 
- 查看并清除`/bin/ping` 的权能：

![选区_057](%E9%80%89%E5%8C%BA_057.png)

- 切换到用户`userping` 再`ping` 无法`ping`   : `su userping` 
- `userping` 登录时给`/bin/ping` 添加权能：

```shell
#####################################################
# File Name: ping_cap.sh
# Author: xsyin
# mail: shouyinXu@163.com
# Created Time: 2018年04月30日 星期一 16时05分57秒
####################################################

#!/bin/sh
[ "$PAM_TYPE" == "open_session" ] || exit 0

if [ "$PAM_USER" == "userping" ]; then
	setcap cap_net_raw=eip /bin/ping
	echo "SUCCESS"
else
	echo "FAILURE"
fi

```

- 将`ping_cap.sh` 移动到 `/usr/local/bin` 路径下，并设置为可执行：

````text
sudo chmod u+x /usr/loca/bin/ping_cap.sh
````

- 所有登录都使用`common-session` 模块：

![选区_058](%E9%80%89%E5%8C%BA_058.png)

因此在`common-session` 模块中添加规则: 

```
session  optional  pam_exec.so debug log=/tmp/pam_exec.log seteuid /usr/local/bin/ping_cap.sh
```

必须模块，开启了`debug`模式，用户`userping` 每次登录会执行`ping_cap.sh`

- 切换到`userping` 用户：

![选区_059](%E9%80%89%E5%8C%BA_059.png)

出错多次，最终调试成功：

![选区_060](%E9%80%89%E5%8C%BA_061.png)

### 总结

优点：`userping` 用户无法执行特权命令，通过`pam` 可赋予一些特权操作。

缺点：本实验中修改了`ping` 的文件权能，影响了其他用户。退出`userping` 后，`ping` 命令变成有效了。



过渡C版本：

```c
/*************************************************************************
    > File Name: ping_cap.c
    > Author: xsyin
    > Mail: shouyinXu@163.com 
    > Created Time: 2018年04月30日 星期一 14时56分35秒
    > Make: gcc ping_cap.c -lcap -o ping_cap
 ************************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/capability.h>
#include <sys/prctl.h>

#undef _POSIX_SOURCE

extern int errno;

const char *path_p = "/bin/ping";

void listCaps(){
    cap_t caps = cap_get_file(path_p);
    ssize_t y = 0;
    printf("The file %s was give capabilities %s\n",path_p, cap_to_text(caps, &y));
    fflush(0);
    cap_free(caps);
}

int main(int argc, char const *argv[])
{


	uid_t uid = getuid();

	if(uid != 1000) 
		return 0;

	printf("uid:%d\n", uid);
	
	cap_value_t cap_values[] = {CAP_NET_RAW};
	cap_t caps = cap_init();
	cap_set_flag(caps,CAP_PERMITTED,1,cap_values,CAP_SET);
	cap_set_flag(caps,CAP_EFFECTIVE,1,cap_values,CAP_SET);
	cap_set_flag(caps,CAP_INHERITABLE,1,cap_values,CAP_SET);
	if(cap_set_file(path_p,caps))
		perror("cap_set_file() ERROR: ");
	else
		printf("success\n");
	cap_free(caps);
	listCaps();

	return 0;
}

```

缺陷：未搞清楚如何获取PAM正在授权的用户，无法判断切换至`userping` 用户时设置权能。