# login和non-login

```
linux 有两种登录shell：login和non-login：
login shell：登录shell时需要完整的登录流程，称为 login shell。何为完整：输入用户名和密码。例如：走tty1-tty6控制终端，或走ssh等伪终端远程登入。
non-login shell：登入shell时不需要输入帐号信息。例如在X11下，打开伪终端，或者在shell下，进入shell子进程。

这两种登入shell的区别是：在登入shell时，读取的配置文件不同。
login shell(bash)在登入时，会读取的配置文件：
1. /etc/profile
2. ~/.bash_profile 或~/.bash_login 或 ~/.profile
3. ~/.bashrc
第二步之所以有三个文件，是因为不同的shell有可能命名不同，只会按顺序读取其中的一个。
non-login shell(bash)在登入时，只会读取 ~/.bashrc
```



# 新建nologin用户

请注意要在root用户下进行

要修改一个已经存在的用户，执行这个命令：

```
usermod -s /sbin/nologin <username >
```

对新用户，可以使用这个命令：

```
useradd -s /sbin/nologin <new username>
```

使用nologin账号来运行程序，su -s /bin/bash -c "ls" www

su -s 是指定shell，这里www用户是nologin用户，是没有默认的shell的，这里指定使用/bin/bash, -c 后面接需要运行的命令， 后面www是用www用户来运行

# wget -qO- 命令详解

原文链接：https://blog.csdn.net/qq_32331073/article/details/79239323
wget(选项)(参数)

```
选项
-a<日志文件>：在指定的日志文件中记录资料的执行过程；
-A<后缀名>：指定要下载文件的后缀名，多个后缀名之间使用逗号进行分隔；
-b：进行后台的方式运行wget；
-B<连接地址>：设置参考的连接地址的基地地址；
-c：继续执行上次终端的任务；
-C<标志>：设置服务器数据块功能标志on为激活，off为关闭，默认值为on；
-d：调试模式运行指令；
-D<域名列表>：设置顺着的域名列表，域名之间用“，”分隔；
-e<指令>：作为文件“.wgetrc”中的一部分执行指定的指令；
-h：显示指令帮助信息；
-i<文件>：从指定文件获取要下载的URL地址；
-l<目录列表>：设置顺着的目录列表，多个目录用“，”分隔；
-L：仅顺着关联的连接；
-r：递归下载方式；
-nc：文件存在时，下载文件不覆盖原有文件；
-nv：下载时只显示更新和出错信息，不显示指令的详细执行过程；
-q：不显示指令执行过程；
-O：下载并以指定的文件名保存；
-nh：不查询主机名称；
-v：显示详细执行过程；
-V：显示版本信息；
--passive-ftp：使用被动模式PASV连接FTP服务器；
--follow-ftp：从HTML文件中下载FTP连接文件。
```

参数
URL：下载指定的URL地址。

其中 -O：下载并以指定的文件名保存

wget -O wordpress.zip http://www.linuxde.net/download.aspx?id=1080
wget默认会以最后一个符合/的后面的字符来命名，对于动态链接的下载通常文件名会不正确。

*错误：下面的例子会下载一个文件并以名称download.aspx?id=1080保存:

wget http://www.linuxde.net/download?id=1
即使下载的文件是zip格式，它仍然以download.php?id=1080命名。

*正确：为了解决这个问题，我们可以使用参数-O来指定一个文件名：

wget -O wordpress.zip http://www.linuxde.net/download.aspx?id=1080
*特殊的：

-O file（--output-document=file）
     The documents will not be written to the appropriate files, but all will be concatenated together and written to file.  If - is used as file, documents will be  printed to standard output, disabling link conversion.  (Use ./- to print to a file literally named -.)

表示：wget 会把url中获取的数据统一写入 '-O' 指定的file中。
        **wget -O-以'-'作为file参数，那么数据将会被打印到标准输出，通常为控制台。**
        **wget -O ./-以'./-'作为file参数，那么数据才会被输出到名为'-'的file中。**

```
案例：票付通环境
nodes="$(wget -qO- http://172.16.0.133:8088/cluster/apps|grep "<a href='/cluster/app/application")"
echo "$nodes" | awk -F"," '{print $1,$2}' OFS="\t" | head -n 4
#awk逗号分隔，以制表符分隔，输出1，2列，前4行
```

# awk使用参考：

awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理
awk工作流程是这样的：读入有'\n'换行符分割的一条记录，然后将记录按指定的域分隔符划分域，填充域，\$0 则表示所有域，\$1表示第一个域,\$n表示第n个域。默认域分隔符是"空白键" 或 "[tab]键"

https://www.cnblogs.com/hepeilinnow/p/10331095.html

# 管道命令

原文参考： https://www.jianshu.com/p/9c0c2b57cb73

## grep

分析一行信息，如果其中有我们需要的信息，就将该行拿出来

```
grep [-acinv] [--color=auto] '查找字符串' filename`
 `[参数]`
 `-a : 将binary文件以text文件的方式查找数据`
 `-c : 计算找到 '查找字符串'的次数`
 `-i : 忽略大小写的不同`
 `-n : 输出行号`
 `-v : 反向选择，显示没有查找内容的行`
 `--color=auto : 将找到的关键字部分加上颜色显示
```

```bash
admindeMacBook-Pro-2:keepLearn admin$ cat /etc/passwd |grep -ic 'admin'
6
admindeMacBook-Pro-2:keepLearn admin$ cat /etc/passwd |grep -i 'admin'
root:*:0:0:System Administrator:/var/root:/bin/sh
_cyrus:*:77:6:Cyrus Administrator:/var/imap:/usr/bin/false
_dovecot:*:214:6:Dovecot Administrator:/var/empty:/usr/bin/false
_kadmin_admin:*:218:-2:Kerberos Admin Service:/var/empty:/usr/bin/false
_kadmin_changepw:*:219:-2:Kerberos Change Password Service:/var/empty:/usr/bin/false
_krb_kadmin:*:231:-2:Open Directory Kerberos Admin Service:/var/empty:/usr/bin/false
```

# ps

ps -aux | grep -v grep | grep engine  查看engine服务器的engine进程



# df du查看磁盘空间使用情况

1. 如何记忆这两个命令
du   -Disk Usage
df   -Disk Free
2. df 和du 的工作原理
2.1 du的工作原理
du命令会对待统计文件逐个调用fstat这个系统调用，获取文件大小。它的数据是基于文件获取的，所以有很大的灵活性，不一定非要针对一个分区，可以跨越多个分区操作。如果针对的目录中文件很多，du速度就会很慢了。
2.2 df的工作原理
df命令使用的事statfs这个系统调用，直接读取分区的超级块信息获取分区使用情况。它的数据是基于分区元数据的，所以只能针对整个分区。由于df直接读取超级块，所以运行速度不受文件多少影响。
3 du和df不一致情况模拟
常见的df和du不一致情况就是文件删除的问题。当一个文件被删除后，在文件系统目录中已经不可见了，所以du就不会再统计它了。然而如果此时还有运行的进程持有这个已经被删除了的文件的句柄，那么这个文件就不会真正在磁盘中被删除，分区超级块中的信息也就不会更改。这样df仍旧会统计这个被删除了的文件。

# jmap

jmap -dump:format=b,file=168engine.hprof 3956