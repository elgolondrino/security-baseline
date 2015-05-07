
[sth0r@shawn-fortress]$ uname -a
Linux shawn-fortress 3.7-trunk-686-pae #1 SMP Debian 3.7.2-0+kali8 i686 GNU/Linux

|=-----------------------------------------------------------------=|
|=-----=[ D O   N O T   F U C K   W I T H   A   H A C K E R ]=-----=|
|=-----------------------------------------------------------------=|
|=------------------------[ #4 File 0x02 ]-------------------------=|
|=-----------------------------------------------------------------=|
|=----------------=[ GNU/Linux安全基线与加固-0.3 ]=----------------=|
|=-----------------------------------------------------------------=|
|=-------------------=[ By Shawn the R0ck   ]=---------------------=|
|=-----------------------------------------------------------------=|
|=--------------------=[ 初稿:July 22 2014  ]=---------------------=|
|=-----------------------------------------------------------------=|
|=-------------------=[ Update: Jan 7 2015 ]=---------------------=|
|=-----------------------------------------------------------------=|

--[ CONTENTS

0. About this doc

1. Routine security baseline

   1.1 Security fix update

   1.2 Password policy

   1.3 SSH secure config

   1.4 audit framework

       1.4.1 file permission audit

       1.4.2 Prevent *UNSET* bash history

   1.5 sudoers

2. Kernel security baseline

3. Hardening

   3.1 Kernel hardening - Grsecurity/PaX

   3.2 PHP

4. SSL/TLS Checklist

5. Weirdo audit

6. Reference



--[ 0. 关于这份文档

随着GNU/Linux在各个行业的IT基础架构中的普及，安全问题也成为了关注的焦点，
GNU/Linux主要是由GNU核心组建( 编译器GCC, C库Glibc等)和Linux内核组合而成，
在自由开源软件统治着基础平台的大环境下，不少人认为开源一定是安全的，这
是一种不完全正确的观念，Coverity的报告只是说明了开源比闭源更安全，这并
不代表自由开源软件就是牢不可破的，自由开源软件在一定程度上具有一些安全
的特性，这些特性不一定在GNU/Linux发行版中是默认开启，这些特性中有一些是
必须在安全基线中去部署的，有一些可以根据具体需求来定制，这篇文档主要是
介绍一些关于安全基线建设和加固的基本内容。

在实际的安全咨询工作中，很多普通个人用户和企业用户并不是安全领域的黑客，
大多客户都会要求给出一份简单易懂的部署文档，也就是所谓的安全基线，基线
和加固是很大的话题，我会尽力不断更新这篇文档的内容，也希望有社区的朋友
参与，本文所使用的GNU/Linux发行版是Debian。


--[ 1. 安全基线

在遵循最小安装和最小权限的部署原则下，还有一些地方是需要注意的，我们把
这些部分称为安全基线。

----[ 1.1 安全修复更新

作为系统管理员，每天干的第1件事情就应该是查看生产环境的机器是否有安全修
复的更新，甚至应该除了生产环境再另外跑一套模拟环境，用于测试升级后是否
影响业务系统。


Debian检查需要安全修复包：

sudo apt-get upgrade -s | grep -i security

OpenSuSE发行版检查需要安全修复的包：

sudo zypper lp | awk '{ if ($7=="security"){ if ($11=="update") {print $13} else{ print $11 }}}' | sed 's/:$//' | grep -v "^$" | sort | uniq

RHEL/CentOS( 6.4)检查需要安全修复的包和明细：
sudo yum list-security | grep RHSA
sudo yum info-security RHSA-NUMBER


----[ 1.2 密码策略

root密码策略至少应该考虑以下几点：

1，密码强度，至少是大小写字母+数字+符号一共9位以上，PAM例子：
password requisite pam_cracklib.so minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1

2，不同的系统密码不能一样

3，更换密码策略(每90天更换一次）

4，密码分发管理，管理不同业务服务器的系统管理员掌握不同的密码


PAM例子：

分别代表最短密码长度为8,其中至少包含一个大写字母，一个小写字母，一个数
字和一个字符, 编辑/etc/pam.d/passwd，增加：

password  requisite pam_cracklib.so minlen=8 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1


编辑/etc/login.defs，设置：

PASS_MAX_DAYS 90	#密码最长生命周期90天
PASS_WARN_AGE 7		#密码过期之前7天报警信息


登录失败3次死锁，编辑/etc/pam.d/system-auth，设置：

auth	required	pam_tally2.so onerr=fail even_deny_root deny=3 unlock_time=3600


------[ 1.2.1 业务分离

生产环境中,不同的业务可以做水平分离,比如把不同的服务运行到不同的虚拟机
中,不需要远程访问的服务器可以绑定到 localhost( 比如只需要访问本机业务的
mysql) 上。


----[ 1.3 SSH安全配置

openssh目前的默认配置文件相比以前虽然要安全的多，但还是有必要对生产系统
中的ssh服务器进行基线检查。

配置文件：/etc/ssh/ssh_config
--------------------------------------------------------------
1，known_hosts保存相关服务器的签名，所以必须把主机名hash:

HashKnownHosts yes

2，SSH协议v1不安全：

Protocol 2

3，如果没用X11转发的情况：

X11Forwarding no

4，关闭rhosts:

IgnoreRhosts yes

5，关闭允许空密码登录：

PermitEmptyPasswords no

6，最多登录尝试次数：

MaxAuthTries 5

7，禁止root登录

PermitRootLogin no
--------------------------------------------------------------

(可选)
--------------------------------------------------------------
1，关闭密码认证，启用公钥认证：

PubkeyAuthentication yes

PasswordAuthentication no


2，允许或者禁止用户/组登录:

AllowGroups, AllowUsers, DenyUsers, DenyGroups
--------------------------------------------------------------

密钥交换( Kex)
--------------------------------------------------------------
OpenSSH支持8种密钥交换协议：
1. curve25519-sha256: ECDH over Curve25519 with SHA2
2. diffie-hellman-group1-sha1: 1024 bit DH with SHA1
3. diffie-hellman-group14-sha1: 2048 bit DH with SHA1
4. diffie-hellman-group-exchange-sha1: Custom DH with SHA1
5. diffie-hellman-group-exchange-sha256: Custom DH with SHA2
6. ecdh-sha2-nistp256: ECDH over NIST P-256 with SHA2
7. ecdh-sha2-nistp384: ECDH over NIST P-384 with SHA2
8. ecdh-sha2-nistp521: ECDH over NIST P-521 with SHA2

按照以下3个标准：

* ECDH曲线：安标NIST已经受到污染所以排除6--8
* DH的位数：排除1024-bit的
* 安全的Hash：排除2--4的SHA1

*** 只剩下1和5，建议配置：

/etc/ssh/sshd_config:

KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256

/etc/ssh/ssh_config snippet:

Host *
    KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256


*** 建议删除/etc/ssh/moduli低于2048位长度的，或者直接创建：
ssh-keygen -G /tmp/moduli -b 4096
ssh-keygen -T /etc/ssh/moduli -f /tmp/moduli
--------------------------------------------------------------

认证( Authentication)
--------------------------------------------------------------
1. DSA
2. ECDSA
3. Ed25519
4. RSA

2是属于被污染的NIST里的，这里不能通过配置来屏蔽ECDSA，但可以通过坏链接
来实现：
cd /etc/ssh
rm ssh_host_ecdsa_key*
rm ssh_host_key*
ln -s ssh_host_ecdsa_key ssh_host_ecdsa_key
ln -s ssh_host_key ssh_host_key

DSA虽然没有被污染，但是必须要求是1024-bit，长度过低，所以关掉最好的情况是只使用：
Protocol 2
HostKey /etc/ssh/ssh_host_ed25519_key

如果需要使用RSA，生成更高强度的密钥：
cd /etc/ssh
rm ssh_host_rsa_key*
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key < /dev/null

生成客户端的密钥：
ssh-keygen -t ed25519
ssh-keygen -t rsa -b 4096
--------------------------------------------------------------

对称算法( Symmetric ciphers)
--------------------------------------------------------------
1.  3des-cbc
2.  aes128-cbc
3.  aes192-cbc
4.  aes256-cbc
5.  aes128-ctr
6.  aes192-ctr
7.  aes256-ctr
8.  aes128-gcm
9.  aes256-gcm
10. arcfour
11. arcfour128
12. arcfour256
13. blowfish-cbc
14. ast128-cbc
15. chacha20-poly1305

对称算法选型是最后意思的，根据以下标准：

* 算法本身的安全性，RC4和DES已经存在风险，所以排除1和10--12
* 密钥强度：至少128-bit
* 块大小：块加密的情况下至少128 bits
* cipher mode：建议使用AE：
https://en.wikipedia.org/wiki/Authenticated_encryption

这样只剩下了5--9和15，/etc/ssh/sshd_config建议配置：

Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

建议配置/etc/ssh/ssh_config：

Host *
    Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
--------------------------------------------------------------

MAC（Message authentication codes)：
--------------------------------------------------------------
为了抗侧信道攻击，所以必须采用先加密再MAC，OpenSSH提供了以下MAC支持：

1.  hmac-md5
2.  hmac-md5-96
3.  hmac-ripemd160
4.  hmac-sha1
5.  hmac-sha1-96
6.  hmac-sha2-256
7.  hmac-sha2-512
8.  umac-64
9.  umac-128
10. hmac-md5-etm
11. hmac-md5-96-etm
12. hmac-ripemd160-etm
13. hmac-sha1-etm
14. hmac-sha1-96-etm
15. hmac-sha2-256-etm
16. hmac-sha2-512-etm
17. umac-64-etm
18. umac-128-etm

按照以下标准：

* HASH算法的安全性，MD5和SHA1被排除
* 必须先加密再MAC，所有不带-etm的被排除
* Tag大小：最少256 bits，UMAC和RIPEMD160被排除
* 密钥长度：至少128-bit

建议配置/etc/ssh/sshd_config:

MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

建议配置/etc/ssh/ssh_config

Host *
    MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

为了兼容github，可以加上：
Host github.com
    MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512
--------------------------------------------------------------


----[ 1.4 auditd审计框架

auditd是接收内核审计模块关于系统调用信息的一个用户态程序，可以通过一些
规则来对一些系统调用或者文件目录进行监控。

安装auditd:
sudo apt-get install auditd

配置文件:/etc/audit/auditd.conf

存储地址log_file = /var/log/audit/audit.log

审计规则的配置文件：/etc/audit/audit.rules，这里给出一个例子：
--------------------------------------------------------------
# First rule - delete all
-D

# Increase the buffers to survive stress events.
# Make this bigger for busy systems
-b 320
-a always,exit -S adjtimex -S settimeofday -S stime -k time-change
-a always,exit -S clock_settime -k time-change
-a always,exit -S sethostname -S setdomainname -k system-locale
# Monitor binaries ps & ls and saved the log when being executed....
-a always,exit -F path=/bin/ps -F path=/bin/ls -F perm=x -k binaries
-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k identity
-w /var/run/utmp -p wa -k session
-w /var/log/wtmp -p wa -k session
-w /var/log/btmp -p wa -k session
-w /etc/apparmor -p wa -k MAC-policy
-w /etc/apparmor.d -p wa -k MAC-policy
--------------------------------------------------------------

上面的例子对一系列关于时间的系统调用进行了监控，一旦时间出现改变就会记
录进如日志，之后对几个跟创建/删除用户和组的文件也进行了监控，最后是对
apparmor的配置文件和规则目录进行监控。在/etc/apparmor下的shell中输入：

sudo touch hello

用工具ausearch来进行查询：

--------------------------------------------------------------
sudo ausearch -i -k MAC-policy
----
type=CONFIG_CHANGE msg=audit(07/20/2014 20:36:48.397:58) : auid=root ses=2 op="add rule" key=MAC-policy list=exit res=1 
----
type=CONFIG_CHANGE msg=audit(07/20/2014 20:36:48.445:59) : auid=root ses=2 op="add rule" key=MAC-policy list=exit res=1 
----
type=PATH msg=audit(07/20/2014 20:38:42.717:61) : item=1 name=hello inode=799889 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 nametype=CREATE 
type=PATH msg=audit(07/20/2014 20:38:42.717:61) : item=0 name=/etc/apparmor inode=783766 dev=08:01 mode=dir,755 ouid=root ogid=root rdev=00:00 nametype=PARENT 
type=CWD msg=audit(07/20/2014 20:38:42.717:61) :  cwd=/etc/apparmor 
type=SYSCALL msg=audit(07/20/2014 20:38:42.717:61) : arch=i386 syscall=open success=yes exit=3 a0=bfeca8ab a1=8941 a2=1b6 a3=1 items=2 ppid=17704 pid=17876 auid=root uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=2 comm=touch exe=/bin/touch key=MAC-policy 
----
type=PATH msg=audit(07/20/2014 20:38:56.017:62) : item=1 name=hello inode=799889 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 nametype=DELETE 
type=PATH msg=audit(07/20/2014 20:38:56.017:62) : item=0 name=/etc/apparmor inode=783766 dev=08:01 mode=dir,755 ouid=root ogid=root rdev=00:00 nametype=PARENT 
type=CWD msg=audit(07/20/2014 20:38:56.017:62) :  cwd=/etc/apparmor 
type=SYSCALL msg=audit(07/20/2014 20:38:56.017:62) : arch=i386 syscall=unlinkat success=yes exit=0 a0=ffffff9c a1=89eb8c0 a2=0 a3=0 items=2 ppid=17704 pid=17889 auid=root uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=2 comm=rm exe=/bin/rm key=MAC-policy 
--------------------------------------------------------------

也可以使用aureport来生成报告。

记录tty的场景:

编辑/etc/pam.d/password-auth-ac：

session     required      pam_tty_audit.so enable=*

查看相关信息：
#tail -n 20 /var/log/audit/audit.log
#aureport --tty -ts today

------[ 1.4.1 针对文件的审计 

1, Leo Juranic的详细的分析[]了异常的通配符威胁有多大，：
--------------------------------------------------------------
find / -path /proc -prune  -name "-*"
--------------------------------------------------------------

2, 所谓的world-writable权限的文件是不太合理的，所以这种文件我们必须得提防：
--------------------------------------------------------------
find / -path /proc -prune -o -perm -2 ! -type l -ls

针对日志的world-readable也需要注意：
find /var/log -perm -o=r ! -type l

正确：
chmod 640 /var/log/
--------------------------------------------------------------

3, 一个没有owner的文件是存在潜在威胁的，因为你永远也不知道未来某个时候她的
uid/gid成为了你的敌人：
--------------------------------------------------------------
find / -path /proc -prune -o -nouser -o -nogroup
--------------------------------------------------------------

4, 作为"自主可控"的自由软件用户，你得知道你的生产环境中哪些用户是可用的：
--------------------------------------------------------------
egrep -v '.*:\*|:\!' /etc/shadow | awk -F: '{print $1}'
--------------------------------------------------------------

5, 你要删除一个用户前，应该先了解一些有哪些文件是他拥有的：
--------------------------------------------------------------x
find / -path /proc -prune -o -user account -ls

然后，安全的删除：

userdel -r account
--------------------------------------------------------------

6, 如果不带':x:'的用户肯定是无法正常使用的：
--------------------------------------------------------------
grep -v ':x:' /etc/passwd
--------------------------------------------------------------

7, 找到没有密码的用户：
--------------------------------------------------------------
cat /etc/shadow | cut -d: -f 1,2 | grep '!'
--------------------------------------------------------------

8，找到被锁定的用户：
--------------------------------------------------------------
cat /etc/shadow | cut -d: -f 1,2 | grep '*'
--------------------------------------------------------------

9, 找到带有过期密码的用户：
--------------------------------------------------------------
cat /etc/shadow | cut -d: -f 1,2 | grep '!!'

passwd -u -l可以操作
--------------------------------------------------------------

10, /boot目录权限下至少是644，甚至是600:
--------------------------------------------------------------
ls -l /boot
--------------------------------------------------------------

11，找到suid和sgid相关的可执行文件：
--------------------------------------------------------------
find / -xdev -user root \( -perm -4000 -o -perm -2000 \)
--------------------------------------------------------------

(可选) 

1, GNU/Linux的访问控制列表(ACL)也是不错的文件权限管理的途径，获得文件
ACL的信息：
--------------------------------------------------------------
getfacl file

设置哪些用户对哪些文件有什么样的权限：
setfacl -m u:user:r file
--------------------------------------------------------------


----[ 1.4.2 Prevent *UNSET* bash history

针对history_bash不允许unset:
#Prevent unset of histfile, /etc/profile
export HISTSIZE=1500
readonly HISTFILE
readonly HISTFILESIZE
readonly HISTSIZE

#Set .bash_history as attr +a
find / -maxdepth 3|grep -i bash_history|while read line; do chattr +a "$line"; done


----[ 1.5 sudoers

执行visudo，让普通用户shawn有所有权限：
shawn ALL=(ALL) ALL

让普通用户shawn只能以root权限执行ls和cat：
shawn ALL=/bin/ls,/bin/cat

让普通用户shawn只能在主机localhost上以root权限执行ls：
shawn localhost=/bin/ls


--[ 2. 内核安全基线


SYN cookies防护主要是为了防止SYN洪水攻击，开启设置为1：
--------------------------------------------------------------
net.ipv4.tcp_syncookies = 1
/proc/sys/net/ipv4/tcp_syncookies

(可选)，如果需要开启SYNPROXY可以直接：
iptables -t raw -A PREROUTING -i eth0 -p tcp --dport 80 --syn -j NOTRACK
iptables -A INPUT -i eth0 -p tcp --dport 80 -m state UNTRACKED,INVALID \
	 -j SYNPROXY --sack-perm --timestamp --mss 1480 --wscale 7 --ecn

echo 0 > /proc/sys/net/netfilter/nf_conntrack_tcp_loose

注意:SYNPROXY是在Linux内核3.13里加入的NETFILTER特性。
--------------------------------------------------------------


TCP FIN-WAIT-2状态保留时间，目前很多发行版的默认时间是60秒，这个值太大
会造成DOS攻击的风险，太小也有一定概率造成远端机器来不急主动关掉链接，建
议一般设置成15：
--------------------------------------------------------------
参考： 
http://benohead.com/tcp-about-fin_wait_2-time_wait-and-close_wait/

net.ipv4.tcp_fin_timeout = 15
/proc/sys/net/ipv4/tcp_fin_timeout
--------------------------------------------------------------


网络抗DOS相关：
--------------------------------------------------------------
SYN队列的长度，值越大能处理更多的链接数：
net.ipv4.tcp_max_syn_backlog = 8192
/proc/sys/net/ipv4/tcp_max_syn_backlog

设备队列的长度，这个值建议比syn队列大：
net.core.netdev_max_backlog = 16384
/proc/sys/net/core/netdev_max_backlog

listen()的backlog限制数，内存大可以提升到65535：
net.core.somaxconn = 4096
/proc/sys/net/core/somaxconn

TIME_WAIT状态的TCP链接最大数，超过这个值系统会自动清除，抗DOS攻击的选项：
net.ipv4.tcp_max_tw_buckets = 4096
/proc/sys/net/ipv4/tcp_max_tw_buckets

TIME-WAIT状态的链接重新使用于新的TCP链接，1表示允许：
net.ipv4.tcp_tw_reuse = 1
/proc/sys/net/ipv4/tcp_tw_reuse

TIME-WAIT状态的链接快速回收，1表示开启：
net.ipv4.tcp_tw_recycle = 1
/proc/sys/net/ipv4/tcp_tw_recycle

TCP KEEPALIVE探测频率，以秒为单位，建议设置在30分钟以内，300为5分钟：
net.ipv4.tcp_keepalive_time = 300
/proc/sys/net/ipv4/tcp_keepalive_time

TCP KEEPALIVE探测包的数量：
net.ipv4.tcp_keepalive_probes = 3
/proc/sys/net/ipv4/tcp_keepalive_probes

SYN和SYN+ACK的重传次数：
net.ipv4.tcp_syn_retries = 3
/proc/sys/net/ipv4/tcp_syn_retries

net.ipv4.tcp_synack_retries = 3 
/proc/sys/net/ipv4/tcp_synack_retries

TCP ORPHAN的值调大能防止简单DOS攻击，每个ORPHAN消耗大约64k的内存，
65535相当于4GB内存：
net.ipv4.tcp_max_orphans = 65536
/proc/sys/net/ipv4/tcp_max_orphans

TCP链接的内存大小，以PAGE为单位，x86下每个page为4KB大小：
net.ipv4.tcp_mem = 131072 196608 262144
/proc/sys/net/ipv4/tcp_mem

如果不是特殊场景的服务器或者网络设备，一般不要调整tcp_mem，超过最大的值
会报OOM的错误。


设置最大的发送和接收窗口,以10G NIC为例,设置64MB：
net.core.rmem_max = 67108864
/proc/sys/net/core/rmem_max

net.core.wmem_max = 67108864
/proc/sys/net/core/wmem_max

每个TCP链接的读，写缓冲区内存大小，单位为Byte：
net.ipv4.tcp_rmem = 4096 8192 16777216( 4096 87380 33554432)
/proc/sys/net/ipv4/tcp_rmem

net.ipv4.tcp_wmem = 4096 8192 16777216( 4096 65536 33554432)
/proc/sys/net/ipv4/tcp_wmem

如果按照缺省分配8KB * 2 = 16KB/链接，4GB内存能提供的链接数为：
(4 * 1024 * 1024) / 16 = 262144 

Oracle 10g的最佳建议：
http://www.dba-oracle.com/t_linux_networking_kernel_parameters.htm
--------------------------------------------------------------


源路由通常可以用于在IP包的OPTION里设置途经的部分或者全部路由器，这个
特性可以用于网络排错和优化，比如traceroute，攻击者也可以使用这个特性来
进行IP欺骗，关闭设置为0：
--------------------------------------------------------------
net.ipv4.conf.all.accept_source_route = 0
/proc/sys/net/ipv4/conf/all/accept_source_route
--------------------------------------------------------------


ICMP重定向，正常用于选择最优路径，攻击者可以利用开展中间人攻击，关闭设
置为0:
--------------------------------------------------------------
net.ipv4.conf.all.accept_redirects = 0
/proc/sys/net/ipv4/conf/all/accept_redirects
--------------------------------------------------------------

如果这台GNU/Linux是作为服务器使用而非网关设备，可以关掉：
--------------------------------------------------------------
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
--------------------------------------------------------------

IP欺骗防护，启动设置为1：
--------------------------------------------------------------
net.ipv4.conf.all.rp_filter = 1
/proc/sys/net/ipv4/conf/all/rp_filter
--------------------------------------------------------------


忽略ICMP请求( PING)，启动设置为1：
--------------------------------------------------------------
net.ipv4.icmp_echo_ignore_all = 1
/proc/sys/net/ipv4/icmp_echo_ignore_all
--------------------------------------------------------------


忽略ICMP广播请求，启动设置为1：
--------------------------------------------------------------
net.ipv4.icmp_echo_ignore_broadcasts = 1
/proc/sys/net/ipv4/icmp_echo_ignore_broadcasts
--------------------------------------------------------------


错误消息防护，会警告你关于网络中的ICMP异常，启动设置为1：
--------------------------------------------------------------
net.ipv4.icmp_ignore_bogus_error_responses = 1
/proc/sys/net/ipv4/icmp_ignore_bogus_error_responses
--------------------------------------------------------------


对特定packet(IP欺骗，源路由，重定向）进行审计，启动设置为1：
--------------------------------------------------------------
/proc/sys/net/ipv4/conf/all/log_martians
net.ipv4.conf.all.log_martians = 1
--------------------------------------------------------------


地址随机化，启动设置为2：
--------------------------------------------------------------
kernel.randomize_va_space=2
/proc/sys/kernel/randomize_va_space
--------------------------------------------------------------


内核符号限制访问，启动设置为1：
--------------------------------------------------------------
kernel.kptr_restrict=1
/proc/sys/kernel/kptr_restrict

类似CVE-2014-0196的exploit对于这一项就很难做到“通杀”;-)
--------------------------------------------------------------


内存映射最小地址，启动设置为65536：
--------------------------------------------------------------
vm.mmap_min_addr=65536
/proc/sys/vm/mmap_min_addr
--------------------------------------------------------------


防护进程被ptrace跟踪调试:
--------------------------------------------------------------
kernel.yama.ptrace_scope = 2
/proc/sys/kernel/yama/ptrace_scope

0: 所有进程都可以被调试
1: 只有一个父进程可以被调试
2: 只有系统管理员可以调试（CAP_SYS_PTRACE ）
3: 没有进程允许被调试

一般加固机用3，普通服务器用2/1。
--------------------------------------------------------------


Apparmor
--------------------------------------------------------------
安装Apparmor和社区规则：
sudo apt-get install -y apparmor-profiles apparmor

查看状态是否运行正常：
sudo aa-status
--------------------------------------------------------------


--[ 3. 加固

安全基线是在防御已知的威胁，而加固则是侧重于防御未知的威胁，加固的主要
目的是增加攻击者的成本。


----[ 3.1 内核加固 - Grsecurity/PaX

GNU/Linux平台从用户空间到内核空间都有一系列的加固措施，但是,对于真正面
临复杂安全环境,确实需要高级安全防护能力的机构,最极端的加固防御是:
Grsecurity/PaX，对于注重完美的“老派( old school )”黑客社区而言,没有
Grsecurity/PaX 的定制方案是不完美的 ,这里摘选几段old school社区的调侃：

"The "better than none" point of view is actually a nice way to false
sense of security for those who don't know better. We got
better-than-none apparmor, selinux, tomoyo, some poorly maintained and
crippled ports of grsec features or alikes, namespaces and containers,
rootkit-friendly LSM, the dumb and useless kernel version of SSP,
etc. What's the sum of all this shit for end users? False sense of
security..."

"without Grsecurity/PaX, linux security is like monkey can never
perform a regular masturbation cu'z lacking of giant pennis;-)"

作为个人完全赞同以上观点，太多的better-than-none可以算是"甜点"，用户使
用后会觉得自high的很爽，但是到了夜幕降临的时候，用户还是很难入睡，因为
"痛点"依然在那里，在安全领域，"甜点"就是"安全感"，安全感绝对不等同于安
全。

Anyway，对于商业客户而言，是否对Grsecurity/PaX定制是一种选择。

安装新的带Grsecurity/PaX补丁的内核，以3.14.1为例，先下载原生内核：
--------------------------------------------------------------
https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.14.1.tar.xz

下载Grsecurity/PaX补丁：
https://github.com/citypw/citypw-SCFE/raw/master/security/apparmor_test/grsecurity-3.0-3.14.11-201407072045.patch

解压内核然后打补丁：
Patch the kernel with grsecurity:
xz -d linux-3.14.1.tar.xz
tar xvf linux-3.14.1.tar
cd linux-3.14.1/
patch -p1 < ../grsecurity-3.0-3.14.3-201405121814.patch

你可以在"make menuconfig"里自己定制符合你口味的内核，也可以使用我测试用
的内核config文件：
https://raw.githubusercontent.com/citypw/citypw-SCFE/master/security/apparmor_test/debian-7.4-linux-3.14.1-grsec.config

编译内核(-jx, x通常==你的CPU核数+1):
make -j7 deb-pkg

安装编译后的内核:
dpkg -i ../*.deb
--------------------------------------------------------------

现在内核的部分结束，关于RBAC规则可以使用用户态工具gradm的Learning Mode
来实现，但不在本文的讨论范畴。


----[ 3.2 PHP加固

1, PHP是常用的WEB开发语言，在WEB生产环境部署的过程中，目录和文件的权限是需
要关注的，通常除了少数用途的目录（比如上传）外，都应该把写入权限禁用：

--------------------------------------------------------------
find -type f -name \*.php -exec chmod 444 {} \;
find -type d -exec chmod 555 {} \;
--------------------------------------------------------------

2, 开启 php 的安全模式,禁用 php 不安全的函数等加固，修改 php 配置文件
/etc/php5/apache2/php.ini :

--------------------------------------------------------------
// 设置模式为安全模式,此值直接影响 disable_functions 的命令是否生效;
[SQL]
; http://php.net/sql.safe-mode

sql.safe_mode = On

// 禁用不安全的函数
disable_functions = system, show_source, symlink, exec, dl, shell_exec,
passthru, phpinfo, escapeshellarg, escapeshellcmd

// 避免暴露 php 信息
expose_php = Off

// 关闭错误信息提示
display_errors = Off

// 不允许调用 dl
enable_dl = Off

// 避免远程调用文件
allow_url_include = Off
--------------------------------------------------------------


--[ 4. SSL/TLS基线检查

SSL/TLS的实现经历BEAST/CRIME/LUCKY-13/HEARTBLEED/POODLE一系列的安全事件
后，我们根据之前的NDAY漏洞的经验可以制定一些安全基线。这里给出的例子都
以HOST为"www.google.com"和PORT为443。

SSLv2早已经因为一系列安全漏洞被废弃，必须保证在生产环境当中去除，用以下
命令验证：
--------------------------------------------------------------
openssl s_client -ssl2 -connect www.google.com:443

由于OpenSSL 1.0开始已经不支持SSLv2，所以这条命令如果成功执行代表你有危
险，GnuTLS也可以用类似的命令来快速检测：

gnutls-cli -d 5 -p 443 --priority "NORMAL:-VERS-TLS1.2:-VERS-TLS1.1:-VERS-TLS1.0:-VERS-SSL3.0" www.google.com
--------------------------------------------------------------

SSLv3的漏洞也不少，POODLE攻击是其中的著名例子，如果你没有运行非常古老的
软件(比如IE6)，那你应该禁用掉SSLv3，检测对端是否支持SSLv3：
--------------------------------------------------------------
gnutls-cli -d 5 -p 443 --priority "NORMAL:-VERS-TLS1.2:-VERS-TLS1.1:-VERS-TLS1.0" www.google.com | grep -i version

如果能看到"- Version: SSL3.0"那你得当心了。
--------------------------------------------------------------

目前(2014年11月18日)，只有TLS 1.1/1.2是相对安全的，EFF早在2011年就建议
公众使用TLS 1.2的PFS，检查是否支持TLS 1.1/1.2：
--------------------------------------------------------------
gnutls-cli -p 443 --priority "NORMAL:-VERS-TLS1.2:-VERS-TLS1.0:-VERS-SSL3.0" www.google.com | grep -i version
显示“- Version: TLS1.1"就说明支持TLS 1.1。

gnutls-cli -p 443 --priority "NORMAL:-VERS-TLS1.1:-VERS-TLS1.0:-VERS-SSL3.0" www.google.com | grep -i version
显示”- Version: TLS1.2“就说明支持TLS 1.2。

使用OpenSSL工具也可以检测：
openssl s_client -tls1_1 -connect www.google.com:443 | grep -i protocol
openssl s_client -tls1_2 -connect www.google.com:443 | grep -i protocol
--------------------------------------------------------------

安全重协商支持：
--------------------------------------------------------------
gnutls-cli -d 5 -p 443  www.google.com
显示“Safe renegotiation succeeded“说明支持
--------------------------------------------------------------

客户端发起的重协商：
--------------------------------------------------------------
详情见：
http://www.solidot.org/story?sid=34026
--------------------------------------------------------------

公钥长度检测：
--------------------------------------------------------------
gnutls-cli  -p 443  www.google.com | grep -i key

如果公钥小于或者等于1024-bit，请根据业务场景重新评估安全性。
--------------------------------------------------------------

检测是否存在较弱的ciphersuites：
--------------------------------------------------------------
openssl s_client -cipher NULL,EXPORT,LOW,DES -connect www.google.com:443

如果执行成功代表有风险。
--------------------------------------------------------------

PFS( Perfect Forward Secrecy)，对性能有至少10%左右的损耗。
--------------------------------------------------------------
openssl s_client -cipher EDH,EECDH -connect www.google.com:443

如果成功说明支持PFS，这个特性是EFF强烈推荐的。
--------------------------------------------------------------

CRIME攻击是利用压缩算法，所以可以检测：
--------------------------------------------------------------
gnutls-cli www.google.com -p 443 | grep -i compress
如果显示“- Compression: NULL“就是安全的。
--------------------------------------------------------------

OpenSSL的Heartbeat特性：
--------------------------------------------------------------
openssl s_client -tlsextdebug -connect www.google.com:443 | grep -i heart
显示"TLS server extension heartbeat"说明此特性开启。
--------------------------------------------------------------

POODLE:
--------------------------------------------------------------
openssl s_client -ssl3 -fallback_scsv -connect www.google.com:443 | grep -i "alert ina"
显示"alert inappropriate fallback"则是安全的。
--------------------------------------------------------------

FREAK:
--------------------------------------------------------------
openssl s_client -cipher EXPORT -connect www.google.com:443

如果协商成功则说明有风险。
--------------------------------------------------------------

--[ 5. Weirdo audit

Grinch攻击虽然不是漏洞，但pkcon+wheel的组合场景还是需要注意，首先检查
wheel组是否存在：

grep -i wheel /etc/group

如果存在的话检查pkcon是否存在。


--[ 6. Reference

[1] 开源闭源项目代码质量对比
    http://www.solidot.org/story?sid=39173

[2] Back To The Future: Unix Wildcards Gone Wild
    http://www.defensecode.com/public/DefenseCode_Unix_WildCards_Gone_Wild.txt

[3] SYNPROXY：廉价的抗DoS攻击方案
    www.solidot.org/story?sid=38791

[4] INTERNET PROTOCOL
    http://tools.ietf.org/html/rfc791

[5] A simple TCP spoofing attack
    http://www.citi.umich.edu/u/provos/papers/secnet-spoof.txt

[6] ICMP Attacks Illustrated
    http://www.sans.org/reading-room/whitepapers/threats/icmp-attacks-illustrated-477

[7] SUSE Linux Enterprise Server 11 SP3 - Security and Hardening
    https://www.suse.com/documentation/sles11/singlehtml/book_hardening/book_hardening.html

[8] Securing Debian Manual 
    https://www.debian.org/doc/manuals/securing-debian-howto/

[9] A Brief Introduction to auditd
    http://security.blogoverflow.com/2013/01/a-brief-introduction-to-auditd/

[10] Apparmor RBAC
     http://wiki.apparmor.net/index.php/Pam_apparmor_example

[11] Hardening PHP from php.ini
     http://www.madirish.net/199

[12] CVE-2014-0196 exploit
http://bugfuzz.com/stuff/cve-2014-0196-md.c

[13] Secure Secure Shell
