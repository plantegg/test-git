## SSH 你没见过的花式玩法


> ssh是指的openSSH 命令工具，程序员每天基本都需要敲很多次。这篇文章提供一些基本的技巧和方法来提升工作效率。比如我自己碰到莫名其妙的问题就会看 ssh debug日志 。
> 比如多机房总是要走跳板机，太麻烦了； 动态密码也很浪费时间 ……

### 公司内部跳板机每次ssh登录都需要 密码+动态码，太复杂了，怎么解？
    
    ren@ren-VirtualBox:~$ cat ~/.ssh/config 
    
    #reuse the same connection
    ControlMaster auto
    ControlPath ~/tmp/ssh_mux_%h_%p_%r
    
    #keep one connection in 72hour
    ControlPersist 72h

在你的ssh配置文件增加上述参数，意味着72小时内登录同一台跳板机只有第一次需要输入密码，以后都是重用之前的连接，所以也就不再需要输入密码了。

加了如上参数后的登录过程就有这样的东东：
    
    debug1: setting up multiplex master socket
    debug3: muxserver_listen: temporary control path /home/ren/tmp/ssh_mux_10.16.*.*_22_alibaba.86g3C34vy36tvCtn
    debug2: fd 3 setting O_NONBLOCK
    debug3: fd 3 is O_NONBLOCK
    debug3: fd 3 is O_NONBLOCK
    debug1: channel 0: new [/home/ren/tmp/ssh_mux_10.16.*.*_22_alibaba]
    debug3: muxserver_listen: mux listener channel 0 fd 3
    debug1: control_persist_detach: backgrounding master process
    debug2: control_persist_detach: background process is 15154
    debug2: fd 3 setting O_NONBLOCK
    debug1: forking to background
    debug1: Entering interactive session.
    debug2: set_control_persist_exit_time: schedule exit in 259200 seconds
    debug1: multiplexing control connection
   
 /home/ren/tmp/ssh_mux_10.16.*.*_22_alibaba 这个就是保存好的socket，下次可以重用，免密码。 in 259200 seconds 对应 72小时

### 我有很多不同机房（或者说不同客户）的机器都需要跳板机来登录，能一次直接ssh上去吗？

比如有一批客户机房的机器IP都是192.168.*.*, 然后需要走跳板机100.10.1.2才能访问到，那么我希望以后在笔记本上直接 ssh 192.168.1.5 就能连上

    $ cat /etc/ssh/ssh_config

	Host 192.168.*.*
    ProxyCommand ssh -l ali-renxijun 100.10.1.2 exec /usr/bin/nc %h %p

上面配置的意思是执行 ssh 192.168.1.5的时候命中规则 Host 192.168.*.* 所以执行 ProxyCommand 先连上跳板机再通过跳板机连向192.168.1.5 。这样在你的笔记本上就跟192.168.*.* 的机器仿佛在一起

比如我的跳板配置：


    #到美国的机器用美国的跳板机速度更快
    Host 10.74.*
    ProxyCommand ssh -l user us.jump exec /bin/nc %h %p 2>/dev/null
   
    Host 192.168.0.*
    ProxyCommand ssh -l user 1.1.1.1 exec /usr/bin/nc %h %p


来看一个例子 
    
    ren@ren-VirtualBox:~$ ssh -l alibaba 10.16.1.* -vvv
    OpenSSH_6.7p1 Ubuntu-5ubuntu1, OpenSSL 1.0.1f 6 Jan 2014
    debug1: Reading configuration data /home/ren/.ssh/config
    debug1: Reading configuration data /etc/ssh/ssh_config
    debug1: /etc/ssh/ssh_config line 28: Applying options for *
    debug1: /etc/ssh/ssh_config line 44: Applying options for 10.16.*.*
    debug1: /etc/ssh/ssh_config line 68: Applying options for *
    debug1: auto-mux: Trying existing master
    debug1: Control socket "/home/ren/tmp/ssh_mux_10.16.1.*_22_alibaba" does not exist
    debug1: Executing proxy command: exec ssh -l alibaba 139.*.*.* exec /usr/bin/nc 10.16.1.* 22
    
本来我的笔记本跟 10.16.1.* 是不通的，ssh登录过程中自动走跳板机139.*.*.* 就连上了

### 为什么有时候ssh 比较慢，比如总是需要30秒钟后才能正常登录

先了解如下知识点，在 ~/.ssh/config 配置文件中：

    GSSAPIAuthentication=no

禁掉 GSSAPI认证，GSSAPIAuthentication是个什么鬼东西请自行 Google。 这里要理解ssh登录的时候有很多种认证方式（公钥、密码等等），具体怎么调试请记住强大的命令参数 ssh -vvv 上面讲到的技巧都能通过 -vvv 看到具体过程。

比如我第一次碰到ssh 比较慢总是需要30秒后才登录，不能忍受，于是登录的时候加上 -vvv明显看到控制台停在了：GSSAPIAuthentication 然后Google了一下，禁掉就好了

### 批量打通所有机器之间的ssh登录免密码

ssh免密码的原理是将本机的pub key复制到目标机器的 ~/.ssh/authorized_keys 里面。可以手工复制粘贴，也可以 ssh-copy-id 等

如果有100台机器，互相两两打通还是比较费事（大概需要100*99次copy key）。 下面通过 expect 来解决输入密码，然后配合shell脚本来批量解决这个问题。

![](http://i.imgur.com/S9jLW7B.png)

这个脚本需要四个参数：目标IP、用户名、密码、home目录，也就是ssh到一台机器的时候帮我们自动填上yes，和密码，这样就不需要人肉一个个输入了。

再在外面写一个循环对每个IP执行如下操作：

![](http://i.imgur.com/4SZcnvc.png)

if代码部分检查本机~/.ssh/下有没有id_rsa.pub，也就是是否以前生成过秘钥对，没生成的话就帮忙生成一次。

for循环部分一次把生成的密钥对和authorized_keys复制到所有机器上，这样所有机器之间都不需要输入密码就能互相登陆了（当然本机也不需要输入密码登录所有机器）

最后一行代码： 

    ssh $user@$n "hostname -i"

验证一下没有输密码是否能成功ssh上去。

**思考一下，为什么这么做就可以打通两两之间的免密码登录，这里没有把所有机器的pub key复制到其他所有机器上去啊**

#### 留个作业：第一次ssh某台机器的时候总是出来一个警告，需要yes确认才能往下走，怎么干掉他？

----------
**这里只是帮大家入门了解ssh，掌握好这些配置文件和-vvv后有好多好玩的可以去挖掘，同时也请在留言中说出你的黑技能**