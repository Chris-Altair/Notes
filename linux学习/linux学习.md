[TOC]

## 一、基础命令

### 1.文件及目录操作

```bash
#以列表形式显示所有文件，并大小以k,m,g单位显示
#-a表示输出所有内容；-l表示以列表显示；-h会以k,m,g单位显示文件大小
ls -alh
#以树的形式展示文件
tree <dirpath>
#创建文件
touch <filename>
#查看文件
cat <filename>  #查看文件内容，-n:显示行号
stat <filename> #查看文件详细信息
#删除文件
rm -rf <filename/dirname>
#创建目录
mkdir <dirname>

#搜索文件夹下的文件
find <path> -name <文件名> | -cmin <+|-分钟> | -ctime <+|-天数> | -type <d表示文件夹，f表示文件>
#例子，搜索当前路径下文件名带test1的文件
	find ./ -name test1 -type f

#实时输出文件
# head查看文件前几行，tail查看文件后几行
tail -f <filename>
```

### 2. 压缩及解压操作

```bash
#tar压缩文件
tar -czvf <目标文件>.tar.gz <源文件或文件夹>
#tar解压文件
tar -xzvf <目标文件>.tar.gz
#查看tar内文件
tar -tzvf <目标文件>.tar.gz

-c：压缩包
-x：解压包
-t或--list 列出备份文件的内容

-z或--gzip或--ungzip 通过gzip指令处理备份文件，经处理的备份文件会以.gz后缀结尾
-v或--verbose：显示打包文件过程
-f：指定压缩包的文件名；
------------------------------------------------------
#zip unzip
#压缩zip：
zip -r xxx.zip ./* #（压缩当前路径下的所有文件及目录）
#解压zip到指定目录：
unzip -o -d <path> xxx.zip
```

### 3.文件内容操作

```bash
#向文件输入内容，-e：支持转义字符
echo -e '<文件内容>' >> <filename>
#例子
	echo -e 'a\tb\tc\na1\tb1\tc1' >> test
#输出
	a	b	c
	a1	b1	c1
---------------------------------------------
#按列拼接文件
paste <filename1> <filename2>
#例子
	echo -e 'a\tb\tc\na1\tb1\tc1' >> test1
	echo -e 'a b c\na1 b1 c1' >> test2
	paste test1 test2
#输出
	a	b	c	a b c
	a1	b1	c1	a1 b1 c1
---------------------------------------------
#grep：行内容匹配，可将满足条件的行输出，支持正则
grep <rule> <filename>
#例子，注意：正则表达式内的{}等规则需要用\转义
	echo -ne 'Amadeus@163.com'|grep '[a-zA-Z0-9]\{2,\}@[a-zA-Z0-9]*\(.com\)'
---------------------------------------------
#awk

---------------------------------------------
#sed 学习参考：https://coolshell.cn/articles/9104.html
```

### 4.文件及目录权限操作

linux用户分为：当前用户u，组内用户g，组外用户o
每种的权限可用三位二进制表示：000～111，从左到右每位分别表示，可读r，可写w，可执行x

```bash
#递归修改文件夹下所有文件的权限
chmod -R 777 <dirpath>
#非数字的方式赋予权限
chmod u=rwx,g+r+w,o-x <filename>
```

### 5.进程查看及操作

```bash
#top命令，查看系统进程资源占用情况
#查看进程内线程占用资源情况
top -H -p <pid>
#pidstat

--------------------------------------------
#不挂断地运行命令，会在当前目录生成nohup.out日志（run可省）
nohup <执行文件路径> run &
#查看占用端口的线程
netstat -apn|grep <port>
#按名字查看线程
ps -ef|grep <name>
#直接终止进程
kill -9 <pid>
#正常终止进程，15可省
kill -15 <pid>
#pkill 按命令名批量杀死进程，但可能误杀
```



### *.目录栈操作

linux目录栈操作，可以用于快速切换多个目录

```bash
cd - #前往上一目录
cd ~ #前往当前用户工作空间

#1.查看/清除当前目录栈: 
dirs -v|c
#2.目录入栈 
pushd {filepath}
#3.切换栈，从0(当前目录)开始向右向左移动N: 
pushd +|-{N}
#4.出栈,默认从当前路径出栈，向右向左删除第N个目录: 
popd +|-{N}
```

## 二、进阶命令

### 1.定时任务

```bash
#查看系统定时任务
crontab -l
#修改系统定时任务
crontab -e
#注意crontab内的cron表达式第1位表示m，而不是s
* 7 * * * sh /usr/XXX.sh >/dev/null 2&>1
#修改完定时任务后需要重启
sudo service cron restart #重启定时服务,其余操作：service cron start|stop|status
---------------------------------------	
#查看定时任务启动的pid
pgrep cron
```

默认情况下，ubuntu没有开启cron日志，这样你可能无法查看定时任务的执行情况。
通过更改设置，我们可以开启它：

```bash
#1.修改rsyslog文件，将/etc/rsyslog.d/50-default.conf 文件中的#cron.*前的#删掉；
#2.重启rsyslog服务
service rsyslog restart
#3.重启cron服务
service cron restart
#查看Linux系统定时任务的日志
tail -f /var/log/cron 
```

### 2. ssh及sftp操作

```bash
1.ssh和sftp一般Linux系统都会默认安装,公用一个端口，默认端口22，而ftp需要手动安装
ssh远程登录：ssh -p <port> <username>@<ip>
sftp远程登录：sftp -P <port> <username>@<ip>
#sftp命令上传put 下载get 操作本地文件命令前+l
--------------------用法------------------------------------------
#主机C第一次远程ssh登录主机S，会将S的公钥写入~/.ssh/known_hosts文件中,S的密钥信息可在/etc/ssh内查看
ssh-keyscan -t ECDSA -p <port> <ip>  #可查看服务器公钥
ssh-keygen -E sha256 -lf ~/.ssh/known_hosts #可根据本地known_hosts文件查看公钥指纹
--------------------认证流程简易版------------------------------------------
1.服务端将公钥pub及会话id发给客户端（如果客户端的known_hosts中有该公钥说明曾经连接过）
2.客户端生成会话密钥key，并计算res= id^key，并将res发给服务端
3.服务端计算key=res^id，得到会话密钥
至此，双方都知道会话密钥及id，后续通信通过对称算法加密通信内容

------------------认证流程详细版------------------------------------

1. 客户端发起一个TCP连接，默认端口号为22.

2. 服务端收到连接请求后，将自己的一些关键信息发给客户端。这些信息包括：

- 服务端的公钥：客户端在收到这个公钥后，会在自己的“known_hosts”文件进行搜索。如果找到了相同的公钥，则说明此前连接过该服务器。如果没有找到，则会在终端上显示一段警告信息，由用户来决定是否继续连接。

coderunner@geekyshacklebolt:~$ ssh geekyshacklebolt The authenticity of host 'geekyshacklebolt (192.168.42.222)' can't be established. ECDSA key fingerprint is SHA256:Ql/KnGlolY9eCGuYK3OX3opnSyJQzsbtM3DW/UZIxms. Are you sure you want to continue connecting (yes/no)?

- 服务器所支持的加密算法列表：客户端根据此列表来决定采用哪种加密算法。

3. 生成会话密钥。此时，客户端已经拥有了服务端的公钥。接下来，客户端和服务端需要协商出一个双方都认可的密钥，并以此来对双方后续的通信内容进行加密。

密钥协商是通过Diffie - Hellman算法来实现的。具体过程是：

    1）服务端和客户端共同选定一个大素数，叫做种子值；

    2）服务端和客户端各自独立地选择另外一个只有自己才知道的素数；

    3）双方使用相同的加密算法（如AES），由种子值和各自的私有素数生成一个密钥值，并将这个值发送给对方；

    4）在收到密钥值后，服务端和客户端根据种子值和自己的私有素数，计算出一个最终的密钥。这一步由双方分别独立进行，但是得到的结果应该是相同的。

    5）双方使用上一步得到的结果作为密钥来加密和解密通信内容。

4. 接下来，客户端将自己的公钥id发送给服务端，服务端需要对客户端的合法性进行验证：

    1）服务端在自己的“authorized_keys”文件中搜索与客户端匹配的公钥。

    2）如果找到了，服务端用这个公钥加密一个随机数，并把加密后的结果发送给客户端。

    3）如果客户端持有正确的私钥，那么它就可以对消息进行解密从而获得这个随机数。

    4）客户端由这个随机数和当前的会话密钥共同生成一个MD5值。

    5）客户端把MD5值发给服务端。

    6）服务端同样用会话密钥和原始的随机数计算MD5值，并与客户端发过来的值进行对比。如果相等，则验证通过。

至此，通信双方完成了加密信道的建立，可以开始正常的通信了。
```



### 3. 防火墙操作

ubuntu

```bash
#1.查看防火墙状态： 
sudo ufw status
#2.开启\关闭\重启防火墙：
sudo ufw enable|disable|reload
#3.建立开放|禁止指定端口的规则：
sudo ufw allow|deny <port>
#4.删除开放|禁止指定端口的规则：
sudo ufw delete allow|deny <port>
```

centos

```bash
操作防火墙
1. 查看已打开的端口  # netstat -anp
2. 查看想开的端口是否已开 # firewall-cmd --query-port=666/tcp
   若此提示 FirewallD is not running
   表示为不可知的防火墙 需要查看状态并开启防火墙
3. 查看防火墙状态  # systemctl status firewalld
   running 状态即防火墙已经开启
   dead 状态即防火墙未开启
4. 开启防火墙，# systemctl start firewalld  没有任何提示即开启成功
5. 开启防火墙 # service firewalld start  
   关闭防火墙 # systemctl stop firewalld
   centos7.3 上述方式可能无法开启，可以先#systemctl unmask firewalld.service 然后 # systemctl start firewalld.service
6. 查看想开的端口是否已开 # firewall-cmd --query-port=666/tcp    提示no表示未开
7. 开永久端口号 firewall-cmd --add-port=666/tcp --permanent   提示    success 表示成功
8. 重新载入配置（重启防火墙）  # firewall-cmd --reload    比如添加规则之后，需要执行此命令
9. 再次查看想开的端口是否已开  # firewall-cmd --query-port=666/tcp  提示yes表示成功
10. 若移除端口 # firewall-cmd --permanent --remove-port=666/tcp
```

## 三、目录结构

### 1. /etc

#### ① /etc/hosts

hosts —— the static table lookup for host name（**主机名查询静态表**）。

​	hosts文件是Linux系统上一个负责ip地址与域名快速解析的文件，以ascii格式保存在/etc/目录下。hosts文件包含了ip地址与主机名之间的映射，还包括主机的别名。在没有域名解析服务器的情况下，系统上的所有网络程序都通过查询该文件来解析对应于某个主机名的ip地址，否则就需要使用dns服务程序来解决。通过可以将常用的域名和ip地址映射加入到hosts文件中，实现快速方便的访问。
优先级 ： **dns缓存 > hosts > dns服务**

```bash
# 例子：修改/etc/hosts如下，我原先需要访问192.168.1.10，现在只需访问fanjc.com即可
127.0.0.1	localhost server1 server2 server3
127.0.1.1	Amadeus
192.168.1.10 fanjc.com
```

#### ② /etc/init.d

系统启动时执行的脚本路径，service就是调的这个

```bash
#下面两者等价
service XXX start|stop|restart|reload|status
/etc/init.d/XXX start|stop|restart|reload|status
```

注：比service更高级的命令systemctl，**service命令本身不支持开机启动和禁用，但systemctl支持**

#### ③ /etc/profile.d

登录及切换用户时执行/etc/profile，profile会循环执行profile.d内的脚本

```bash
#profile脚本内容
if [ "${PS1-}" ]; then
  if [ "${BASH-}" ] && [ "$BASH" != "/bin/sh" ]; then
    # The file bash.bashrc already sets the default PS1.
    # PS1='\h:\w\$ '
    if [ -f /etc/bash.bashrc ]; then
      . /etc/bash.bashrc
    fi
  else
    if [ "`id -u`" -eq 0 ]; then
      PS1='# '
    else
      PS1='$ '
    fi
  fi
fi
#可见profile会执行/etc/profile.d内的所有脚本
if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi
```

#### ④ /etc/passwd

查看系统所有的用户

```bash
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
```

#### ⑤ /etc/shadow

查看系统所有用户对应的密码信息（**当然是加密后的**）

### 2. /proc/<pid>

伪文件系统，可查看系统执行的进程信息

## 四、shell相关

```bash
#通过 cat 命令来查看当前 Linux 系统的可用 Shell
cat /etc/shells
#查看系统默认shell
echo $SHELL
```

