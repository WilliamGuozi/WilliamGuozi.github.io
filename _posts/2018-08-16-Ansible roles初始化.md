# Ansible：Roles初始化系统

## 简介
>本文介绍ansible的roles，通过roles来实现系统的初始化，其相当于将ansible的playbook拆分。本文通过Jenkins，传参，调用playbook来初始化系统。

## Github地址
>ansible的roles将功能、变量等抽象为一个个子模块，并在主配置文件中按顺序调用模块，形成一个功能集合，这样做的好处是可以复用各个模块，组合为新的playbook，项目代码地址<https://github.com/WilliamGuozi/ansible-init>，欢迎使用交流，也可以丰富它。


## ansible-init仓库介绍
>该初始化内容应放在/etc/ansible/目录下  

+ 其中inventory目录中放置主机host（可参考文章<https://www.cnblogs.com/William-Guozi/p/ansible_hosts.html>）   
+ key目录中放置deploy的公钥，使用中需要替换为你们的deploy的公钥  
+ roles中可放置多种需要批量运行的roles，比如说系统初始化，安装软件等  

![img-w500](/images/201808211032.png)

+ 其中文件目录ansible-init/roles/init_sys/site.yaml中，指定了各个角色内容。
+ 目前包含:keys:同步deploy公钥到authorized_key
+ hostname:设定服务器的hostname
+ yum:安装一些常用的软件
+ jdk:安装JDK环境
+ optimize:优化系统参数
+ zabbixaggent:安装zabbix-agent并自动加入监控（可参考<https://www.cnblogs.com/William-Guozi/p/zabbix-active.html>）
+ reboot:重启计算机  

>以上角色可根据需求调整更改，并可通过主控文件 **ansible-init/roles/init_sys/site.yaml** 控制是否调用个别角色。

![img-w500](/images/201808211012.png)
![img-w500](/images/201808211054.png)

## Jenkins Job创建
### 参数化构建
>需要传递两个参数，一个是host，为inventory中的主机，即需要初始化的目标主机；另一个是hostname，即需要将目标主机的主机名设置为什么  

![img-w500](/images/201808211121.png)
### Invoke Ansible Playbook
>将上述两个参数传递给ansible  

![img-w500](/images/201808211123.png)


## Jenkins构建后输出结果

```bash
初始化Started by user williamguozi
[EnvInject] - Loading node environment variables.
Building on master in workspace /var/lib/jenkins/workspace/ops-demo-ansible-roles
[ops-demo-ansible-roles] $ ansible-playbook /etc/ansible/roles/init_sys/site.yaml -f 5 -e host=192.168.30.100 -e hostname=ops-demo-100

PLAY [192.168.30.100] **********************************************************

TASK [hostname : centos 7 permanent modify hostname] ***************************
Friday 17 August 2018  16:43:49 +0800 (0:00:00.074)       0:00:00.074 ********* 
changed: [192.168.30.100]

TASK [yum : setup epel-release] ************************************************
Friday 17 August 2018  16:43:50 +0800 (0:00:01.144)       0:00:01.218 ********* 
ok: [192.168.30.100]

TASK [yum : install basic software] ********************************************
Friday 17 August 2018  16:43:51 +0800 (0:00:01.217)       0:00:02.436 ********* 
ok: [192.168.30.100] => (item=[u'gcc', u'gcc-c++', u'gdb', u'python2-pip', u'iotop', u'telnet', u'ntpdate', u'mutt', u'msmtp', u'wget', u'vim', u'htop', u'docker', u'rsync', u'lrzsz', u'psmisc', u'net-tools', u'curl', u'jq', u'lsof', u'nginx', u'tree'])

TASK [yum : enabled service] ***************************************************
Friday 17 August 2018  16:44:06 +0800 (0:00:14.725)       0:00:17.162 ********* 
ok: [192.168.30.100] => (item=docker)
ok: [192.168.30.100] => (item=nginx)

TASK [yum : Time synchronization] **********************************************
Friday 17 August 2018  16:44:08 +0800 (0:00:02.491)       0:00:19.653 ********* 
changed: [192.168.30.100]

TASK [jdk : check jdk version] *************************************************
Friday 17 August 2018  16:44:17 +0800 (0:00:09.038)       0:00:28.692 ********* 
changed: [192.168.30.100]

TASK [jdk : debug check jdk] ***************************************************
Friday 17 August 2018  16:44:18 +0800 (0:00:01.171)       0:00:29.864 ********* 
ok: [192.168.30.100] => {
    "resultt.stdout": "1.8.0_144"
}

TASK [jdk : Checking /data/soft directory] *************************************
Friday 17 August 2018  16:44:18 +0800 (0:00:00.043)       0:00:29.908 ********* 
skipping: [192.168.30.100]

TASK [jdk : Download jdk file] *************************************************
Friday 17 August 2018  16:44:18 +0800 (0:00:00.021)       0:00:29.929 ********* 
skipping: [192.168.30.100]

TASK [jdk : Checking directory] ************************************************
Friday 17 August 2018  16:44:19 +0800 (0:00:00.020)       0:00:29.949 ********* 
skipping: [192.168.30.100]

TASK [jdk : Extract archive] ***************************************************
Friday 17 August 2018  16:44:19 +0800 (0:00:00.021)       0:00:29.971 ********* 
skipping: [192.168.30.100]

TASK [jdk : create java link] **************************************************
Friday 17 August 2018  16:44:19 +0800 (0:00:00.021)       0:00:29.992 ********* 
skipping: [192.168.30.100]

TASK [jdk : create java command link] ******************************************
Friday 17 August 2018  16:44:19 +0800 (0:00:00.020)       0:00:30.012 ********* 
skipping: [192.168.30.100]

TASK [jdk : java_profile config] ***********************************************
Friday 17 August 2018  16:44:19 +0800 (0:00:00.021)       0:00:30.034 ********* 
skipping: [192.168.30.100] => (item=export JAVA_HOME=/usr/local/jdk) 
skipping: [192.168.30.100] => (item=export PATH=\$JAVA_HOME/bin:\$PATH) 

TASK [optimize : disable selinux] **********************************************
Friday 17 August 2018  16:44:19 +0800 (0:00:00.034)       0:00:30.069 ********* 
ok: [192.168.30.100]

TASK [optimize : init timezone] ************************************************
Friday 17 August 2018  16:44:20 +0800 (0:00:01.280)       0:00:31.349 ********* 
ok: [192.168.30.100]

TASK [optimize : ulimit configure temporary] ***********************************
Friday 17 August 2018  16:44:21 +0800 (0:00:00.904)       0:00:32.254 ********* 
changed: [192.168.30.100]

TASK [optimize : modify configure of ulimit for ever] **************************
Friday 17 August 2018  16:44:22 +0800 (0:00:00.924)       0:00:33.178 ********* 
ok: [192.168.30.100]

TASK [optimize : shutdown mail notify] *****************************************
Friday 17 August 2018  16:44:25 +0800 (0:00:03.030)       0:00:36.209 ********* 
changed: [192.168.30.100]

TASK [optimize : shutdown firewalld service] ***********************************
Friday 17 August 2018  16:44:26 +0800 (0:00:00.991)       0:00:37.200 ********* 
ok: [192.168.30.100]

TASK [optimize : history add config] *******************************************
Friday 17 August 2018  16:44:27 +0800 (0:00:01.148)       0:00:38.349 ********* 
ok: [192.168.30.100] => (item=HISTFILESIZE=3000)
ok: [192.168.30.100] => (item=HISTTIMEFORMAT='%F %T   ')
ok: [192.168.30.100] => (item=HISTIGNORE="history:which")
ok: [192.168.30.100] => (item=shopt -s histappend)
ok: [192.168.30.100] => (item=PROMPT_COMMAND="history -a")
ok: [192.168.30.100] => (item=export HISTTIMEFORMAT)

TASK [optimize : history change config] ****************************************
Friday 17 August 2018  16:44:31 +0800 (0:00:04.394)       0:00:42.744 ********* 
ok: [192.168.30.100]

TASK [optimize : Time synchronization] *****************************************
Friday 17 August 2018  16:44:32 +0800 (0:00:00.748)       0:00:43.493 ********* 
changed: [192.168.30.100]

TASK [zabbixagent : cheching zabbix-agent] *************************************
Friday 17 August 2018  16:44:41 +0800 (0:00:09.245)       0:00:52.738 ********* 
changed: [192.168.30.100]

TASK [zabbixagent : debug zabbix-agent] ****************************************
Friday 17 August 2018  16:44:42 +0800 (0:00:00.959)       0:00:53.698 ********* 
ok: [192.168.30.100] => {
    "result.stdout": "3.4.12"
}

TASK [zabbixagent : Install Repository with zabbix] ****************************
Friday 17 August 2018  16:44:42 +0800 (0:00:00.044)       0:00:53.742 ********* 
ok: [192.168.30.100]

TASK [zabbixagent : install zabbix-agent] **************************************
Friday 17 August 2018  16:44:44 +0800 (0:00:02.056)       0:00:55.799 ********* 
ok: [192.168.30.100]

TASK [zabbixagent : upload file aoubt process cpu & memory monitor] ************
Friday 17 August 2018  16:44:46 +0800 (0:00:01.397)       0:00:57.197 ********* 
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/scripts/', u'src': u'discovery_process.sh'})
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/scripts/', u'src': u'process_check.sh'})
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/zabbix_agentd.d/', u'src': u'userparameter_script.conf'})

TASK [zabbixagent : modify configure of percona] *******************************
Friday 17 August 2018  16:44:54 +0800 (0:00:08.736)       0:01:05.933 ********* 
ok: [192.168.30.100] => (item={u'dest': u'/var/lib/zabbix/percona/scripts/', u'src': u'get_mysql_stats_wrapper.sh'})
ok: [192.168.30.100] => (item={u'dest': u'/var/lib/zabbix/percona/scripts/', u'src': u'ss_get_mysql_stats.php'})
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/zabbix_agentd.d/', u'src': u'userparameter_percona_mysql.conf'})

TASK [zabbixagent : copy zabbix_agentd.conf status_tcp status_disk] ************
Friday 17 August 2018  16:45:03 +0800 (0:00:08.813)       0:01:14.746 ********* 
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/', u'src': u'zabbix_agentd.conf'})
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/zabbix_agentd.d/', u'src': u'status_TCP.conf'})
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/scripts/', u'src': u'tcp_status.sh'})
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/zabbix_agentd.d/', u'src': u'userparameter_diskstats.conf'})
ok: [192.168.30.100] => (item={u'dest': u'/etc/zabbix/scripts/', u'src': u'lld-disks.py'})

TASK [zabbixagent : server start] **********************************************
Friday 17 August 2018  16:45:18 +0800 (0:00:14.456)       0:01:29.203 ********* 
ok: [192.168.30.100]

PLAY RECAP *********************************************************************
192.168.30.100             : ok=24   changed=7    unreachable=0    failed=0   

Friday 17 August 2018  16:45:20 +0800 (0:00:02.636)       0:01:31.840 ********* 
=============================================================================== 
yum : install basic software ------------------------------------------- 14.73s
zabbixagent : copy zabbix_agentd.conf status_tcp status_disk ----------- 14.46s
optimize : Time synchronization ----------------------------------------- 9.25s
yum : Time synchronization ---------------------------------------------- 9.04s
zabbixagent : modify configure of percona ------------------------------- 8.81s
zabbixagent : upload file aoubt process cpu & memory monitor ------------ 8.74s
optimize : history add config ------------------------------------------- 4.39s
optimize : modify configure of ulimit for ever -------------------------- 3.03s
zabbixagent : server start ---------------------------------------------- 2.64s
yum : enabled service --------------------------------------------------- 2.49s
zabbixagent : Install Repository with zabbix ---------------------------- 2.06s
zabbixagent : install zabbix-agent -------------------------------------- 1.40s
optimize : disable selinux ---------------------------------------------- 1.28s
yum : setup epel-release ------------------------------------------------ 1.22s
jdk : check jdk version ------------------------------------------------- 1.17s
optimize : shutdown firewalld service ----------------------------------- 1.15s
hostname : centos 7 permanent modify hostname --------------------------- 1.14s
optimize : shutdown mail notify ----------------------------------------- 0.99s
zabbixagent : cheching zabbix-agent ------------------------------------- 0.96s
optimize : ulimit configure temporary ----------------------------------- 0.92s
Finished: SUCCESS

```
完毕

## 参考文档

+ 官方参考文档: <https://ansible-tran.readthedocs.io/en/latest/docs/playbooks_intro.html>

