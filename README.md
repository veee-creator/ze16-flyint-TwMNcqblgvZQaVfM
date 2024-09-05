
## 1\.问题背景


GaussDB轻量化分布式集群安装完成后，进行openssh和openssl升级，现有环境openssh\-8\.2p1\-9\.p03\.ky10\.x86\_64和openssl\-1\.1\.1f\-2\.ky10\.x86\_64版本，可以安装数据库，然后升级这两个版本到openssh\-8\.2p1\-9\.p15\.ky10\.x86\_64和openssl\-1\.1\.1f\-4\.p17\.ky10\.x86\_64。


对集群安装完成后的命令测试，启停机群节点都没问题，然后但是被协调节点被剔除以后，修复出现了这个故障，出现了报错，跟第一次安装的集群出现了一样的问题，报错截图如下：


![](https://img2024.cnblogs.com/blog/1538923/202407/1538923-20240719153506251-1011546396.png)


![](https://img2024.cnblogs.com/blog/1538923/202407/1538923-20240719153529770-1227508278.png)


 


集群状态如下，有一个CN节点显示被剔除，集群状态变为降级，DN正常，集群仍为可用状态


 


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904161538939-1270076363.png)


 


## 2\.进行openssh和openssl版本规避


 


修改说明：




```
1. 修改GaussDB(DWS) 的环境变量文件/opt/huawei/Bigdata/mppdb/.mppdbgs_profile， 调整LD_LIBRARY_PATH变量执行
修改前：
[omm@redhat-4 ~]$ cat  /opt/huawei/Bigdata/mppdb/.mppdbgs_profile  | grep -in LD_LIBRARY_PATH
5:export LD_LIBRARY_PATH=$GPHOME/lib:$LD_LIBRARY_PATH
7:export LD_LIBRARY_PATH=$GPHOME/lib/libsimsearch:$LD_LIBRARY_PATH
11:export LD_LIBRARY_PATH=$GAUSSHOME/lib:$LD_LIBRARY_PATH
12:export LD_LIBRARY_PATH=$GAUSSHOME/lib/libsimsearch:$LD_LIBRARY_PATH
```


修改后:




```
[omm@redhat-4 ~]$ cat  /opt/huawei/Bigdata/mppdb/.mppdbgs_profile  | grep -in LD_LIBRARY_PATH
5:export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GPHOME/lib
7:export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GPHOME/lib/libsimsearch
11:export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GAUSSHOME/lib
12:export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GAUSSHOME/lib/libsimsearch
增加内容如下：
export LD_LIBRARY_PATH=/lib64:$LD_LIBRARY_PATH
2. 在/etc/profile中增加LD_LIBRARY_PATH变量。其中/lib64为ssh二进制工具的依赖库路径。
增加内容如下：
export LD_LIBRARY_PATH=/lib64:$LD_LIBRARY_PATH
```


## 3\.重新修复CN


### 3\.1重新进行gs\_replace修复协调节点，但是有其他报错


 




```
[omm@DN01 ~]$ gs_replace -t config -h DN02
Checking all the cm_agent instances.
There are [0] cm_agents need to be repaired in cluster.
Fixing all the CMAgents instances.
Checking and restoring the secondary standby instance.
The secondary standby instance does not need to be restored.
Configuring
Waiting for promote peer instances.
.
Successfully upgraded standby instances.
Configuring replacement instances.
Successfully configured replacement instances.
Deleting abnormal CN from pgxc_node on the normal CN.
No abnormal CN needs to be deleted.
Unlocking cluster.
Successfully unlocked cluster.
Locking cluster.
Successfully locked cluster.
Unlocking cluster.
Successfully unlocked cluster.
Creating all fixed CN on the normal CN.
No CN needs to be created.
Warning: failed to turn off O&M management. Please re-execute "cm_ctl set --maintenance=off" once again.
[GAUSS-51400] : Failed to execute the command: source /opt/huawei/Bigdata/mppdb/.mppdbgs_profile ; cm_ctl set --maintenance=on  -n 2. Error:
cm_ctl: Starting to enable the maintenance mode.
cm_ctl: Close maintenance mode on cm instances.
cm_ctl: Close maintenance mode on cm instances failed.
```


 


### 3\.2 执行如上面报错提示




```
[omm@DN01 ~]$ source /opt/huawei/Bigdata/mppdb/.mppdbgs_profile
[omm@DN01 ~]$
[omm@DN01 ~]$ cm_ctl set --maintenance=on  -n 2
cm_ctl: Starting to enable the maintenance mode.
cm_ctl: Close maintenance mode on cm instances.
cm_ctl: Close maintenance mode on cm instances failed.
```


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162000371-1225801976.png)


### 3\.3 查看日志




```
[omm@DN01 ~]$ cd $GAUSSLOG/bin/cm_ctl
[omm@DN01 cm_ctl]$ less cm_ctl-2024-07-13_191612-current.log

报错截图如下：
```


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162030229-1399084712.png)


 


### 3\.4三节点移除pssh文件




```
[omm@DN01 cm_ctl]$ sudo mv /usr/bin/pssh /usr/bin/pssh.bak
[omm@DN02 cm_ctl]$ sudo mv /usr/bin/pssh /usr/bin/pssh.bak
[omm@DN03 cm_ctl]$ sudo mv /usr/bin/pssh /usr/bin/pssh.bak
```


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162049978-768660257.png)


 


### 3\.5重新调用提示命令




```
[omm@DN01 cm_ctl]$ cm_ctl set --maintenance=on  -n 2
cm_ctl: Starting to enable the maintenance mode.
cm_ctl: Close maintenance mode on cm instances.
cm_ctl: Close maintenance mode on cm instances successfully.
cm_ctl: Generate and distribute the maintenance white-list file.
cm_ctl: Generate and distribute the maintenance white-list file successfully.
cm_ctl: Set maintenance mode on related cm instances.
cm_ctl: Set maintenance mode on related cm instances successfully.
cm_ctl: Reload configuration on related cm instances.
cm_ctl: Reload configuration on related cm instances successfully.
cm_ctl: Query the maintenance mode from the primary cm server.
cm_ctl: Enable the maintenance mode successfully.

The following nodes enter the maintenance mode:
node_2
```


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162114535-165546548.png)


 


### 3\.6 重新调用gs\_replace




```
[omm@DN01 cm_ctl]$ gs_replace -t config -h DN02
Checking all the cm_agent instances.
There are [0] cm_agents need to be repaired in cluster.
Fixing all the CMAgents instances.
Checking and restoring the secondary standby instance.
The secondary standby instance does not need to be restored.
Configuring
Waiting for promote peer instances.
.
Successfully upgraded standby instances.
Configuring replacement instances.
Successfully configured replacement instances.
Deleting abnormal CN from pgxc_node on the normal CN.
No abnormal CN needs to be deleted.
Unlocking cluster.
Successfully unlocked cluster.
Locking cluster.
Successfully locked cluster.
Incremental building CN from the Normal CN.
Successfully incremental built CN from the Normal CN.
Creating fixed CN on the normal CN.
Successfully created fixed CN on the normal CN.
Starting the fixed cns.
Successfully started the fixed cns.
Creating fixed CN on the fixed CN.
Successfully created fixed CN on the fixed CN.
Unlocking cluster.
Successfully unlocked cluster.
Creating unfixed CN on the fixed and normal CN.
No CN needs to be created.
Configuration succeeded.
```


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162144054-202451902.png)


 


### 3\.7 gs\_replace启动CN




```
[omm@DN01 cm_ctl]$ gs_replace -t start -h DN02
Starting.
======================================================================
.
Successfully started instance process. Waiting to become Normal.
======================================================================

======================================================================
Start succeeded.
```


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162214441-624880862.png)


 


### 3\.8集群balanced操作




```
[omm@DN01 cm_ctl]$ gs_om -t switch --reset
Operating: Switch reset.
cm_ctl: cmserver is rebalancing the cluster automatically.
.......
cm_ctl: switchover successfully.
Operation succeeded: Switch reset.
```


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162235481-1642980058.png)


 


### 3\.9集群状态


集群修复




```
[omm@DN01 cm_ctl]$ gs_om -t status --detail
[  CMServer State   ]

node    node_ip         instance                                    state
---------------------------------------------------------------------------
1  DN01 10.254.21.75    1    /opt/huawei/Bigdata/mppdb/cm/cm_server Primary
3  DN03 10.254.21.77    2    /opt/huawei/Bigdata/mppdb/cm/cm_server Standby

[   Cluster State   ]

cluster_state   : Normal
redistributing  : No
balanced        : Yes

[ Coordinator State ]

node    node_ip         instance                                   state
--------------------------------------------------------------------------
1  DN01 10.254.21.75    5001 /srv/BigData/mppdb/data1/coordinator Normal
2  DN02 10.254.21.76    5002 /srv/BigData/mppdb/data1/coordinator Normal
3  DN03 10.254.21.77    5003 /srv/BigData/mppdb/data1/coordinator Normal

[ Central Coordinator State ]

node    node_ip         instance                                  state
-------------------------------------------------------------------------
3  DN03 10.254.21.77    5003 /srv/BigData/mppdb/data1/coordinator Normal

[     GTM State     ]

node    node_ip         instance                           state                    sync_state
---------------------------------------------------------------
3  DN03 10.254.21.77    1001 /opt/huawei/Bigdata/mppdb/gtm P Primary Connection ok  Sync
1  DN01 10.254.21.75    1002 /opt/huawei/Bigdata/mppdb/gtm S Standby Connection ok  Sync

[  Datanode State   ]

node    node_ip         instance                                  state            | node    node_ip         instance                                  state            | node    node_ip         instance                                  state
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1  DN01 10.254.21.75    6001 /srv/BigData/mppdb/data1/master1     P Primary Normal | 2  DN02 10.254.21.76    6002 /srv/BigData/mppdb/data1/slave1      S Standby Normal | 3  DN03 10.254.21.77    3002 /srv/BigData/mppdb/data1/dummyslave1 R Secondary Normal
1  DN01 10.254.21.75    6003 /srv/BigData/mppdb/data2/master2     P Primary Normal | 3  DN03 10.254.21.77    6004 /srv/BigData/mppdb/data1/slave2      S Standby Normal | 2  DN02 10.254.21.76    3003 /srv/BigData/mppdb/data1/dummyslave2 R Secondary Normal
2  DN02 10.254.21.76    6005 /srv/BigData/mppdb/data1/master1     P Primary Normal | 3  DN03 10.254.21.77    6006 /srv/BigData/mppdb/data2/slave1      S Standby Normal | 1  DN01 10.254.21.75    3004 /srv/BigData/mppdb/data1/dummyslave1 R Secondary Normal
2  DN02 10.254.21.76    6007 /srv/BigData/mppdb/data2/master2     P Primary Normal | 1  DN01 10.254.21.75    6008 /srv/BigData/mppdb/data1/slave2      S Standby Normal | 3  DN03 10.254.21.77    3005 /srv/BigData/mppdb/data2/dummyslave2 R Secondary Normal
3  DN03 10.254.21.77    6009 /srv/BigData/mppdb/data1/master1     P Primary Normal | 1  DN01 10.254.21.75    6010 /srv/BigData/mppdb/data2/slave1      S Standby Normal | 2  DN02 10.254.21.76    3006 /srv/BigData/mppdb/data2/dummyslave1 R Secondary Normal
3  DN03 10.254.21.77    6011 /srv/BigData/mppdb/data2/master2     P Primary Normal | 2  DN02 10.254.21.76    6012 /srv/BigData/mppdb/data2/slave2      S Standby Normal | 1  DN01 10.254.21.75    3007 /srv/BigData/mppdb/data2/dummyslave2 R Secondary Normal
```


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162314248-1934941658.png)


 


![](https://img2024.cnblogs.com/blog/1538923/202409/1538923-20240904162318386-883477514.png)


 


### 3\.10正常状态数据库环境变量




```
[root@DN01 ~]# tail -5f /etc/profile
fi
#TMOUT=600
export TMOUT=0
#LD_LIBRARY_PATH=/usr/local/lib/
export LD_LIBRARY_PATH=/lib64:$LD_LIBRARY_PATH
```




```
[omm@DN01 ~]$ cat .bash_profile
# Source /root/.bashrc if user has one
[ -f ~/.bashrc ] && . ~/.bashrc
source /home/omm/.profile

LD_LIBRARY_PATH=/usr/local/lib/
export LD_LIBRARY_PATH=/lib64:$LD_LIBRARY_PATH
```




```
[omm@DN01 ~]$ cat /opt/huawei/Bigdata/mppdb/.mppdbgs_profile
#LD_LIBRARY_PATH=/usr/local/lib
export MPPDB_ENV_SEPARATE_PATH=/opt/huawei/Bigdata/mppdb/.mppdbgs_profile
export LDAPCONF=/opt/huawei/Bigdata/mppdb/ldap.conf
export GPHOME=/opt/huawei/Bigdata/mppdb/wisequery
export PATH=$PATH:$GPHOME/script/gspylib/pssh/bin:$GPHOME/script
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GPHOME/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GPHOME/lib/libsimsearch
export PYTHONPATH=$GPHOME/lib
export GAUSS_WARNING_TYPE=1
export GAUSSHOME=/opt/huawei/Bigdata/mppdb/core
export PATH=$GAUSSHOME/bin:$PATH
export S3_CLIENT_CRT_FILE=$GAUSSHOME/lib/client.crt
export GAUSS_VERSION=8.2.1
export PGHOST=/opt/huawei/Bigdata/mppdb/mppdb_tmp
export GS_CLUSTER_NAME=FI-MPPDB
export GAUSSLOG=/var/log/Bigdata/mpp/omm
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GAUSSHOME/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GAUSSHOME/lib/libsimsearch
export ETCD_UNSUPPORTED_ARCH=386
if [ -f '/opt/huawei/Bigdata/mppdb/core/utilslib/env_ec' ] && [ `id -u` -ne 0 ]; then source '/opt/huawei/Bigdata/mppdb/core/utilslib/env_ec'; fi
export GAUSS_ENV=2
export LD_LIBRARY_PATH=/lib64:$LD_LIBRARY_PATH
```


 


 


 


 


 本博客参考[veee加速器](https://liuyunzhuge.com)。转载请注明出处！
