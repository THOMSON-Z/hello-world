# GaussDB轻量化部署形态部署配置手册 V3.1-20241011
## 1 概述
本文介绍了GaussDB实例（轻量化）的部署安装步骤，包括：初始化DB，主机纳管，实例安装，GUC参数/数据库的客户化以及DRS-Node/Monitor Agent的安装和回退步骤。

## 2 DB 初始化
!!! danger  
	- 以下动作需以root用户进行操作，操作完成后请及时登出root用户，避免误操作。  
	- <font color="blue">**该节操作需要在所有实例节点进行。**</font>  
### 2.1 查看系统配置
该版本的GaussDB最低要求系统配置为8C64G，可通过以下命令分别查看CPU核数以及内存大小：  
```
free -g
lscpu
```
如果系统配置不满足该要求，需要增加CPU核数以及内存扩容。  

### 2.2 查看磁盘空间
GaussDB支持使用SSD盘作为数据库的主存储设备，支持SAS接口和NVME协议的SSD盘。  
<font color="blue">**数据盘要求独立未被使用且至少为300GB**</font>，若小于可能导致实例创建失败。  
<font color="blue">**lsblk & vgs查看是否有空间大于300GB且未被使用的独立块设备；**</font>  
※：若存在与上述块设备关联的逻辑卷rootvg，需要解绑，以块设备/dev/sdb为例：  
```
vgreduce rootvg /dev/vdb
pvremove /dev/vdb
dd if=/dev/zero of=/dev/vdb bs=512k count=20
```

### 2.3 执行自动化脚本
执行以下shell脚本，完成 2.4\~2.13 的检查项配置：  
【脚本】  
<font color="blue">**执行完成并确认无问题后，DB初始化完成；若有问题，参照 2.4~2.13 中对应章节进行处理。**</font>  

### 2.4 配置操作系统防火墙
需在防火墙关闭的状态下进行安装，关闭防火墙操作步骤如下。  
**步骤 1**&emsp;执行以下命令，检查防火墙是否关闭。  
```
systemctl status firewalld
```
- 若防火墙状态显示为active (running)，则表示防火墙未关闭，请执行步骤2。
- 若防火墙状态显示为inactive (dead)，则表示防火墙已关闭，无需再关闭防火墙。

**步骤 2**&emsp;执行以下命令，关闭防火墙并禁止开机启动。  
```
systemctl stop firewalld.service
systemctl disable firewalld.service
```
**步骤 3**&emsp;修改/etc/selinux/config文件中的“SELINUX”值为“permissive”  
1. 使用vi打开config文件
```
 vi /etc/selinux/config 
```
2. 修改“SELINUX”的值“permissive”。
```
SELINUX=permissive
```
3. 按“Esc”键后执行:wq!保存并退出修改。

### 2.5 配置iptables服务
在数据库实例安装的各节点上，执行以下命令打开iptables服务：  
```
systemctl start iptables.service
```

### 2.6 设置字符集参数
将各主机的字符集设置为相同的字符集。  
**步骤 1**&emsp;vi打开profile文件。  
```
vi /etc/profile
```
**步骤 2**&emsp;在/etc/profile文件中添加"export LANG=en_US.UTF-8"。  
**步骤 3**&emsp;执行:wq保存并退出修改。  
**步骤 4**&emsp;设置/etc/profile生效。  
```
source /etc/profile
```
**步骤 5**&emsp;vi打开/etc/locale.conf文件。
```
vi /etc/locale.conf
```
**步骤 6**&emsp;修改参数“LANG”的值为en_US.UTF-8。  
**步骤 7**&emsp;执行以下命令，确保安装的字符集生效：
```
source /etc/locale.conf
```

### 2.7 设置时钟源
该步骤要保证各时间点的时钟源同步，可以配置chrony或者ntpd时间同步。  
使用Chrony配置时间同步：  
**步骤 1**&emsp;以root用户登录到待配置时间同步的所有服务器节点。  
**步骤 2**&emsp;键入“chrony”并连按两次“Tab”键观察，检查是否安装了chrony。  
- 若显示chronyc和chronyd，则表示已经安装了chrony。继续执行后续步骤。
- 若未显示则表示当前未安装chrony，执行以下命令进行安装。   
```
yum install chrony -y
```
**步骤 3**&emsp;执行以下命令，修改客户端配置。  
1. 使用vi命令编辑客户端的/etc/chrony.conf文件：
```
vi /etc/chrony.conf
```
2. 参照如下图示，添加“#”注释掉配置文件最前面原有的pool行，并新增“server 时间同步服务器域名/IP地址 iburst”。这里server后面填写：22.12.75.23
3. 按“Esc”键后执行:wq!命令，保存并退出。
4. 执行以下命令，重启客户端chrony服务使配置生效。
```
systemctl restart chronyd
```
**步骤 4**&emsp;执行以下命令，配置后检查。  
```
chronyc sources -v
```
查看时钟源列表，列表中有配置的时钟源服务器IP即可。  
关闭swap交换内存  
**步骤 5**&emsp;执行以下步骤，临时关闭交换内存。  
```
swapoff -a
```
**步骤 6**&emsp;验证swap是否关闭：  
```
free -g
```
回显结果如下：
```
             total       used        free       shared     buff/cache   available
Mem:         30          8           2          12         19           5
Swap:        0           0           0
```

### 2.8 设置网卡MTU值
**步骤 1**&emsp;执行命令ifconfig，查看IP地址绑定的网卡，如eth0。  
**步骤 2**&emsp;执行以下命令，查看与IP地址绑定的网卡后显示的MTU的值。  
```
ifconfig eth0 | grep mtu 
```
回显如下。 
```
eth0: flags=4163 mtu 1500
```
**步骤 3**&emsp;如回显MTU值为1500，本项操作到此结束；否则继续步骤4  
**步骤 4**&emsp;执行以下命令，打开文件ifcfg-*。  
```
vi /etc/sysconfig/network-scripts/ifcfg-*
```
**步骤 5**&emsp;按“i”进入编辑模式，添加如下语句，设置网卡MTU值，以设置MTU为1500为例。  
```
MTU=1500
```
**步骤 6**&emsp;保存并关闭文件。  
```
:wq!
```
**步骤 7**&emsp;执行以下命令，重启网络。  
```
service network restart
```

### 2.9 安装Expect
**步骤 1**&emsp;执行以下命令，检查是否安装了Expect。  
```
expect -v 
```
回显如下： 
```
expect version 5.45.4
```
**步骤 2**&emsp;若没有出现Expect版本，或者提示命令不存在，需执行以下命令进行安装。  
```
yum install expect -y
```
**步骤 3**&emsp;检查软件是否安装成功。  
```
expect -v
```
### 2.10 检查OpenSSH
**步骤 1**&emsp;执行以下命令，检查是否安装了OpenSSH工具。  
```
ssh -V 
```
回显如下： 
```
OpenSSH_8.2p1, OpenSSL 1.1.1f 31 Mar 2020
```
**步骤 2**&emsp;若没有出现OpenSSH版本，或者提示命令不存在，需执行以下命令进行安装。  
```
yum -y install openssh-server
```
**步骤 3**&emsp;安装完成后，执行以下命令启动OpenSSH服务。  
```
systemctl start sshd
```
### 2.11 配置 sshd_config
**步骤 1**&emsp;使用vi打开/etc/ssh/sshd_config文件。  
```
vi /etc/ssh/sshd_config
```
**步骤 2**&emsp;设置GSSAPIAuthentication参数为no。  
```
GSSAPIAuthentication no
```
**步骤 3**&emsp;保存并关闭文件。  
```
:wq!
```
**步骤 4**&emsp;重启SSH服务。  
```
systemctl restart sshd.service
```
### 2.12 设置 umask
**步骤 1**&emsp;执行如下指令，查看umask回显值。  
```
umask
```
若回显小于等于0022，表示umask设置正确。  
若回显大于0022，请执行后续步骤，设置umask。  
**步骤 2**&emsp;执行如下命令，打开bashrc文件。  
```
vi /etc/bashrc
```
**步骤 3**&emsp;bashrc文件的最下方增加一行，使umask的值等于0022。  
```
umask 0022
```
**步骤 4**&emsp;保存并关闭文件。  
```
:wq!
```
**步骤 5**&emsp;执行如下命令，使umask修改生效。  
```
source /etc/bashrc
```
**步骤 6**&emsp;再次执行umask命令，回显等于0022表示设置umask成功。  

## 3 主机纳管
### 3.1 添加机房
**步骤 1**&emsp;登录云数据库GaussDB管理平台（TPOPS）。  
**步骤 2**&emsp;单击“平台管理 > 数据中心管理”，进入“数据中心管理 > 机房管理”页面。  
**步骤 3**&emsp;单击页面右上方“添加机房”按钮。  
**步骤 4**&emsp;单击“确定”，添加机房。   
### 3.2 删除机房
**步骤 1**&emsp;登录云数据库GaussDB管理平台（TPOPS）。  
**步骤 2**&emsp;单击“平台管理 > 数据中心管理”，进入“数据中心管理 > 机房管理”页面。  
**步骤 3**&emsp;选择所需删除的机房，单击“删除”按钮。  
**步骤 4**&emsp;输入“delete”，单击“确定”，删除机房。  
### 3.3 添加主机
**步骤 1**&emsp;登录云数据库GaussDB管理平台（TPOPS）。  
**步骤 2**&emsp;单击“平台管理 > 数据中心管理”，进入“数据中心管理”页面。  
**步骤 3**&emsp;单击需要查询主机所在的“机房别名/ID”，这里选择AZ1。  
**步骤 4**&emsp;单击“添加主机”。  
&emsp;&emsp;主机别名，用户密码根据实际需要填写；
&emsp;&emsp;操作系统：麒麟；  
&emsp;&emsp;CPU厂商：鲲鹏；  
&emsp;&emsp;管理IP，数据IP，业务IP：实际分配的机器IP地址；  
&emsp;&emsp;授权类型：选择“用户名/密码”；  
&emsp;&emsp;用户密码：输入root用户密码；  
**步骤 5**&emsp;单击下方的“确定”，添加主机。等待3~5分钟，主机添加成功。  
**步骤 6**&emsp;单击”任务中心”，查看添加主机的进展情况。  
### 3.4 删除主机
**步骤 1**&emsp;登录云数据库GaussDB管理平台（TPOPS）。  
**步骤 2**&emsp;单击“平台管理 > 数据中心管理”，进入“数据中心管理 > 机房管理”页面。  
**步骤 3**&emsp;单击需要删除的主机所在的“机房别名/ID”。  
**步骤 4**&emsp;单击需要删除的主机的“操作 > 删除”。  
**步骤 5**&emsp;填写主机连接的授权类型，SSH私钥或者用户名/密码。  
**步骤 6**&emsp;输入“delete”，单击“确定”，删除主机。也可单击“取消”，取消删除主机。  

## 4 实例管理
### 4.1 创建数据库参数模板
**步骤 1**&emsp;登录云数据库GaussDB管理平台（TPOPS）。  
**步骤 2**&emsp;单击“参数模板管理”，进入“参数模板管理”页面。  
**步骤 3**&emsp;单击“创建参数模板”，选择数据库引擎，输入参数模板名和相关描述信息。  
&emsp;&emsp;可以创建100个参数模板。当前项目下所有GaussDB引擎共享参数模板配额。  
&emsp;&emsp;根据安装的数据库部署形态，选择对应的数据库引擎版本：  
&emsp;&emsp;模板名以bocnet开头，比如主备版可以命名为bocnet_centralized。  
**步骤 4**&emsp;填写完成后，单击“确定”。  
&emsp;&emsp;创建的参数模板显示在“自定义”页签：  
**步骤 5**&emsp;点击模板”bocnet”，参考以下excel文档，进行编辑修改。  
**步骤 6**&emsp;在右上角的查询框输入要修改的参数名，修改参数值；之后点击保存。  
### 4.2 删除数据库参数模板
**步骤 1**&emsp;登录云数据库GaussDB管理平台（TPOPS）。  
**步骤 2**&emsp;选择“参数模板管理 > 自定义”，进入“自定义”页签。  
**步骤 3**&emsp;选择待删除的参数模板，单击“操作 > 更多 > 删除”。  
删除操作无法恢复，请谨慎操作。  
**步骤 4**&emsp;确定删除后单击“是”。  

### 4.3 安装实例
**步骤 1**&emsp;登录TPOPS管理界面，选择“实例管理”：  
**步骤 2**&emsp;选择“安装实例”后，进入实例配置选项界面。  
&emsp;&emsp;实例类型：“主备版”  
&emsp;&emsp;部署形态：“3节点”  
&emsp;&emsp;副本一致性：“Quarum”  
&emsp;&emsp;参数模板：“Default Enterprise Edition GaussDB-a.102.CNTL”  
&emsp;&emsp;主可用区：“AZ1”  
&emsp;&emsp;管理IP：选中AZ1机房下的三台主机  
&emsp;&emsp;存储设备：置空即可  
&emsp;&emsp;管理员密码/确认密码：自定义输入，须一致。  
**步骤 3**&emsp;点击“立即申请”，开始创建数据库实例。  
**步骤 4**&emsp;查看创建数据库实例进度：  
&emsp;&emsp;点击“任务中心”，找到创建数据库的任务，点击“任务详情”；  
&emsp;&emsp;查看数据库创建的进度：
&emsp;&emsp;查看中途是否有任务失败，如果失败，点击子任务中的“+”查看报错信息，找到错误原因并修复后，可以点击“操作”中的“继续任务”。  
**步骤 5**&emsp;选择“参数模板管理 > 自定义”，进入“自定义”页签。  
**步骤 6**&emsp;选择bocnet参数模板，单击“操作 > 更多 > 应用”。  
### 4.4 GUC参数客户化
在任一数据库节点执行以下操作：  
**步骤 1**&emsp;以root用户登录任意一台数据库服务器。  
**步骤 2**&emsp;切换到Ruby用户。  
```
su - Ruby
```
**步骤 3**&emsp;进入沙箱。  
执行命令：chroot /var/chroot，进入沙箱；  
依次执行以下命令：  
```
source /etc/profile
source /home/Ruby/.bashrc
source /home/Ruby/gauss_env_file
```
**步骤 4**&emsp;执行以下附件中gs_guc set命令，修改GUC参数值：  
**步骤 5**&emsp;登录TPOPS管理界面，选择“实例管理”：  
**步骤 6**&emsp;单击左侧目录“实例管理”，进入“实例列表”页面。  
**步骤 7**&emsp;选择本次新建的实例，单击“重启实例”。  
**步骤 8**&emsp;在弹出的窗口中输入“confirm”字样并勾选确认框。  
**步骤 9**&emsp;单击“是”，重启实例。  
重启过程中，实例将不可用。重启后实例会自动释放内存中的缓存，请在业务低峰期 进行重启，避免对高峰期业务造成影响。  
**步骤 10**&emsp;验证修改后的参数值。  
- 以Ruby用户登录任意一台数据库服务器  
```
su – Ruby
```
- gsql连接数据库  
```
source gauss_env_file
gsql -d postgres -p 8000 -U root -W dk@e7rJa
```
- 在gsql命令行中输入步骤4附件中gs_guc check的命令：  

查看命令输出，对照附件gs_guc set命令中设置的参数值，验证GUC参数已经客户化。  
### 4.5 GUC参数客户化-回退
在任一数据库节点执行以下操作：  
**步骤 1**&emsp;以root用户登录任意一台数据库服务器。  
**步骤 2**&emsp;切换到Ruby用户。  
```
su - Ruby
```
**步骤 3**&emsp;进入沙箱。  
执行命令：chroot /var/chroot，进入沙箱；  
依次执行以下命令：
```
source /etc/profile
source /home/Ruby/.bashrc
source /home/Ruby/gauss_env_file
```
**步骤 4**&emsp;参考以下gs_guc命令，修改GUC参数值：  
集中式：gs_guc set -Z datanode -N all -I all -c "参数名=参数值"  
分布式：gs_guc set -Z datanode -Z coordinator -N all -I all -c "参数名=参数值"  
**步骤 5**&emsp;登录TPOPS管理界面，选择“实例管理”：  
**步骤 6**&emsp;单击左侧目录“实例管理”，进入“实例列表”页面。  
**步骤 7**&emsp;选择本次新建的实例，单击“重启实例”。  
**步骤 8**&emsp;在弹出的窗口中输入“confirm”字样并勾选确认框。  
**步骤 9**&emsp;单击“是”，重启实例。   
重启过程中，实例将不可用。重启后实例会自动释放内存中的缓存，请在业务低峰期 进行重启，避免对高峰期业务造成影响。  
**步骤 10**&emsp;验证修改后的参数值  
- 以root用户登录任意一台数据库服务器。   
- 切换到Ruby用户。
```
su – Ruby
```
- gsql连接数据库
```
source gauss_env_file
gsql -d postgres -p 8000 -U root -W dk@e7rJa
```
- 参考以下命令格式，在gsql命令行输入：
```
show 参数名;
``` 
查看命令输出，验证GUC参数已经回退到初始默认值。
### 4.6 删除实例
<font color="blue">注意事项：
- 实例删除后，不可恢复，请谨慎操作。
- 执行操作中的实例不能手动删除，只有在实例操作完成后，才可删除实例。
- 异常实例删除时，若出现删除异常的情况，可参考“任务中心 > 跳过任务”跳过执行失败的步骤，并联系客服删除存储设备上的备份数据。
</font>

**步骤1**&emsp;登录TPOPS管理界面，选择“实例管理”：
**步骤2**&emsp;单击左侧目录“实例管理”，进入“实例列表”页面。
**步骤3**&emsp;选择待删除的实例，单击“更多 > 删除实例”。
**步骤4**&emsp;输入“delete”字样并勾选“已确认”。
**步骤5**&emsp;单击“是”，删除实例。

## 5 客户端工具安装
### 5.1 安装说明
安装的客户端工具包含：
**gs_dump/gs_dumpall,gs_ktool,gs_loader,gs_restore以及gsql**
### 5.2 安装步骤
**步骤1**&emsp;客户端解压缩到GSQL中
FTP://22.122.18.86:/patch/gaussdb/831/
**GaussDB-Kernel_505.1.0_Kylin_64bit_Gsql_Ditributed.tar.gz     分布式GSQL包**
**GaussDB-Kernel_505.1.0_Kylin_64bit_Gsql_Centralized.tar.gz    集中式GSQL包**
**传送到对应目录**
```
mkdir /用户家目录/DBS-GaussDB-driver
cd /用户家目录/DBS-GaussDB-driver

cd /用户家目录/DBS-GaussDB-driver
tar -xvf  /用户家目录/DBS-GaussDB-driver/GaussDB-Kernel_505.1.0_Kylin_64bit_Gsql_Centralized.tar.gz

chmod -R 755 /用户家目录/DBS-GaussDB-driver/

cd /用户家目录/DBS-GaussDB-driver/
vi gsql_env.sh
```
**修改第六行内容**
**原文为**
```
LOCAL_PATH=${0}
```
**修改为**
```
LOCAL_PATH=/用户家目录/DBS-GaussDB-driver
```
```
cd /应用用户家目录/
echo "source /用户家目录/DBS-GaussDB-driver/gsql_env.sh" >> .bash_profile
```
**步骤2**&emsp;验证客户端版本信息 
```
gsql -V
```
**步骤3**&emsp;验证数据库版本信息，确认与客户端工具版本保持一致
```
gsql -d postgres -p XXXX -U 用户名 -W 密码 -ra
```
连接进入会话后，查看版本：
```
select version();
```

## 6 客户化
### 6.1 创建数据库
集中式
**步骤1**&emsp;登录云数据库GaussDB管理平台（TPOPS）
**步骤2**&emsp;单击具体实例名称： 
进入“实例管理”详情页：
**步骤3**&emsp;选择“数据库管理 > 库管理”，当前页面即显示当前实例下所有非模板数据库。
**步骤4**&emsp;单击“新建数据库”可进行数据库创建：
- 数据库名称：数据库名称由1~63个字符组成，可以包含英文字母（a-z、A-Z）、数字（0-9）、下划线（_），不能以'pg'和数字开头，不能和模板数据库重名（模板数据库包括postgres、template0、template1、templatem）。
- 数据库字符集：当前数据库字符集可选UTF8、SQL_ASCII、GBK、Latin1、GB18030、EUT_TW、EUT_KR、EUT_CN、EUT_JP。
- 属主：当前数据库仅可选择默认属主root。
- 数据库模板：仅支持template0。
- 数据库排序集：默认设置为C，即使用标准ASCII字符集进行比较。
- 数据库分类集：默认设置为C，即使用标准ASCII字符集进行分类。
分布式
在主数据库节点执行以下操作：
<font color="blue">※：主数据库节点的确认，参考第10章附录中10.2.3 cm_ctl中“确认主节点”。</font>
**步骤1**&emsp;以root用户登录主节点服务器
**步骤2**&emsp;切换到Ruby用户
```
su – Ruby
```
**步骤3**&emsp;初始换GaussDB环境
```
source gauss_env_file
```
**步骤4**&emsp;以root用户登录默认数据库postgres，创建数据库
```
gsql -d postgres -p 8000 -r -U root -W XXXXXXXX
create database 新建数据库名 owner=root dbcompatibility=’ORA’;
```
### 6.2 删除数据库
**步骤1**&emsp;登录云数据库GaussDB管理平台（TPOPS）
**步骤2**&emsp;单击具体实例名称： 
进入“实例管理”详情页：
**步骤3**&emsp;选择“数据库管理 > 库管理”，当前页面即显示当前实例下所有非模板数据库。
**步骤4**&emsp;单击“删除”，输入“delete”。
**步骤5**&emsp;单击“是”，可将新建的非模板数据库删除。
### 6.3 创建用户
**步骤1**&emsp;登录云数据库GaussDB管理平台（TPOPS）
**步骤2**&emsp;单击具体实例名称： 
进入“实例管理”详情页：
**步骤3**&emsp;选择“数据库管理 > 用户管理”，前页面即显示当前实例下所有数据库用户（不包括root以外的系统用户）。
**步骤4**&emsp;单击“创建用户账号”，填写用户名称、密码、确认密码，选择要赋予用户的权限类型，给用户授权数据库（可选）后，可创建新数据库用户。

### 6.4 删除用户
**步骤1**&emsp;登录云数据库GaussDB管理平台（TPOPS）
**步骤2**&emsp;单击具体实例名称： 
进入“实例管理”详情页：
**步骤3**&emsp;选择“数据库管理 > 用户管理”，前页面即显示当前实例下所有数据库用户（不包括root以外的系统用户）。
**步骤4**&emsp;单击“更多 > 删除”，可删除数据库用户。输入“delete”确认后即可删除数据库用户。
 
### 6.5 用户授权
在主数据库节点执行以下操作：
<font color="blue">※：主数据库节点的确认，参考第10章附录中10.2.3 cm_ctl中“确认主节点”。</font>
**步骤1**&emsp;以root用户登录任意一台数据库服务器
**步骤2**&emsp;切换到Ruby用户
```
su – Ruby
```
**步骤3**&emsp;初始化GaussDB环境
```
source gauss_env_file
```
**步骤4**&emsp;以root用户登录默认数据库postgres，给用户赋CREATE权限
```
gsql -d postgres -p 8000 -r -U root -W XXXXXXXX
```
登录后，在命令行输入以下SQL语句，给用户赋CREATE权限：
```
GRANT CREATE ON database 新建数据库名 to 新建用户名;
```
以root用户登录新建的业务数据库，给业务用户赋权限：
```
gsql -d 新建数据库名 -p 8000 -r -U root -W XXXXXXXX
CREATE SCHEMA 新建用户名;
GRANT CREATE ON SCHEMA PUBLIC TO 新建用户名;
```
### 6.6 撤销用户授权
在主数据库节点执行以下操作：
※：主数据库节点的确认，参考第10章附录中10.2.3 cm_ctl中“确认主节点”。
**步骤1**&emsp;以root用户登录任意一台数据库服务器
**步骤2**&emsp;切换到Ruby用户
```
su – Ruby
```
**步骤3**&emsp;初始换GaussDB环境
```
source gauss_env_file
```
**步骤4**&emsp;以root用户登录默认数据库postgres，给用户赋CREATE权限
```
gsql -d postgres -p 8000 -r -U root -W XXXXXXXX
```
登录后，在命令行输入以下SQL语句，给用户赋CREATE权限：
```
REVOKE CREATE ON database 新建数据库名 from 新建用户名;
```
**步骤5**&emsp;以root用户登录新建的业务数据库，给业务用户赋权限：
```
gsql -d 新建数据库名 -p 8000 -r -U root -W XXXXXXXX
REVOKE CREATE ON SCHEMA PUBLIC from 新建用户名;
```

## 7 安装DRS-Node
**步骤1**&emsp;以root用户登录待安装DRS的服务器。
**步骤2**&emsp;挂载/data磁盘
安装DRS-Node的/data目录需要至少100GB磁盘空间，考虑文件头额外占用空间，需挂载150GB空间：
```
lvcreate -L 150G -n lv_drsdata rootvg
mkfs.ext4 -F /dev/mapper/rootvg-lv_drsdata
mkdir /data
echo -e "/dev/rootvg/lv_drsdata\t/data\text4\tdefaults\t0\t0" >> /etc/fstab
mount -a
df -h 
```
**步骤3**&emsp;挂载/drs磁盘
安装DRS-Node的/drs目录需要100GB磁盘空间：
```
lvcreate -L 100G -n lv_drsdrs rootvg
mkfs.ext4 -F /dev/mapper/rootvg-lv_drsdrs
mkdir /drs
echo -e "/dev/rootvg/lv_drsdrs\t/data\text4\tdefaults\t0\t0" >> /etc/fstab
mount -a
df -h 
```
**步骤4**&emsp;将DRS-Node软件包上传到/data文件夹中。
**步骤5**&emsp;执行以下命令，进入/data
```
cd /data
```
**步骤6**&emsp;执行以下命令，解压并进入DRS-Node文件夹，其中DRS-Node-*版本号以实际为准。
```
tar -xvf DRS-Node-*.tar.gz
cd DRS-Node-*
```
**步骤7**&emsp;执行如下命令，配置install.conf文件。
修改蓝色字体部分的内容：
metaDB_address：填写实际元数据库节点的IP地址(通常与DRS-Service在一个节点，30100为默认端口，保持不变)；
metaDB_drs_password：元数据库drs用户的密码；
node_ip：当前节点的IP地址
vim install.conf
```
[meta db] 
metaDB_engine = gaussdb 
metaDB_address = 192.168.0.60:30100,192.168.0.130:30100,192.168.0.245:30100    # 元数据库IP地址，多IP地址可用逗号分隔 
metaDB_drs_user = drs                                  # 元数据库DRS用户名 
metaDB_drs_password = huawei@123Pwd                 # 元数据库DRS密码 
metaDB_database = drs                                  # 元数据库database 
 
[drs] 
node_ip = 192.168.0.60                                 # 节点IP地址 
 
[system] 
swap_space = 2G                                        # swap分区大小
```
**步骤8**&emsp;安装前环境预检查，执行以下命令，检查安装环境。
sh precheck_env.sh
```
================== START TO PRECHECK ==================== 
Item1: User Check 
User is root 
User Check Success! 
Item2: Hardware Check 
Cpu Check 
cpu_num is 8U 
Warning:Cpu Recommended over 16U 
Memory Check 
Warning:Memory Recommended over 32G 
memory_capacity is 30.6G 
Hardware Check Complete! 
Item3: Diskspace Check 
Warning:Disk Recommended: /drs: 100.0G /data: 2.5T 
Warning:Diskspace Check Fail! 
Diskspace Check Complete! 
=================== PRECHECK RESULT===================== 
Item                                    Actual/Recommand 
1.User Check:                                  Pass/Pass    #检查项1：安装用户检查，非root用户退出 
2.Hardware Check:                                           #检查项2：机器规格检查，建议16U32G                
  CPU Check:                                      8U/16U 
  Memory Check:                                30.6G/32G 
3.Diskspace Check:                           NoPass/Pass    #检查项3：/drs /data目录磁盘空间检查，建议目录可用共2.5T，若机器不存在上述目录，检查根目录可用磁盘空间 
3.1 Diskspace Check: /drs:                  86.4G/100.0G 
3.2 Diskspace Check: /data:                    1.8T/2.5T 
=================== PRECHECK END ======================= 
```
**步骤9**&emsp;执行以下命令，运行安装脚本。
sh install.sh
```
Precheck environment begin. 
================== START TO PRECHECK ==================== 
Item1: User Check 
User is root 
User Check Success! 
Item2: Hardware Check 
Cpu Check 
cpu_num is 8U 
Memory Check 
memory_capacity is 14.4G 
Hardware Check Complete! 
Item3: Diskspace Check 
Diskspace Check Success! 
Diskspace Check Complete! 
=================== PRECHECK RESULT===================== 
Item                                    Actual/Recommand 
1.User Check:                                  Pass/Pass 
2.Hardware Check:                   
  CPU Check:                                   Pass/Pass 
  Memory Check:                                Pass/Pass 
3.Diskspace Check:                             Pass/Pass 
=================== PRECHECK END ======================= 
Precheck environment success. 
================= 0 / 6 START TO DEPLOY DRS NODE ================= 
[INFO] --- install node begin 
[INFO] --- node id is 39ad4fb9-87e0-4037-a1b2-082414786976 
======================= 1 / 6 CREATE USER ======================== 
[INFO] --- check Ruby exist begin 
[INFO] --- Ruby exist. Return 1 
User Ruby already existed, continue will recreate user. If not, existed user Ruby will be used 
Create new Ruby user?[Y/N] n                                           # Ruby用户已经存在，输入n使用已存在用户，输入Y重新创建Ruby用户 
[INFO] --- No 
[INFO] --- No need create Ruby 
====================== 2 / 6 INIT 3RD_PARTY ====================== 
[INFO] --- init 3rd_party begin 
[INFO] --- init 3rd_party success 
[INFO] --- prepare environment cost time: 0m2s 
[INFO] --- Test meta db connect start. 
[INFO] --- primary meta db is: 192.168.44.121:30100 
[INFO] --- Connect drs service db success. address:192.168.44.121:30100,192.168.48.76:30100 user:drs 
[INFO] --- Test meta db connect end. 
===================== 3 / 6 CHECKING SWAP ======================== 
[INFO] --- check swap start 
[INFO] --- clean up existed swap file 
[INFO] --- try to create swap 2G 
[INFO] --- create swap file.edit 
[INFO] --- format swap file 
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes) 
no label, UUID=429fe24e-5873-4a56-a55f-c41fcdc2afa4 
[INFO] --- startup swap 
[INFO] --- check swap finished 
=============== 4 / 6 PREPARING SOFTWARE PACKAGES ================ 
[INFO] --- begin to tar tungsten.tar.gz 
[WARNING] --- warnings occurs in step install sdv 
[INFO] --- check cgroup_sudo file 
[INFO] --- check nodeAgentSudo files 
[INFO] --- crypter jar found in /drs/lib/cipher-2.24.01.000-r1.jar 
[WARNING] --- warnings occurs in encrypt 
[WARNING] --- warnings occurs in encrypt 
[WARNING] --- warnings occurs in encrypt 
[WARNING] --- warnings occurs in encrypt 
[INFO] --- begin to render node config 
[INFO] --- render node config finished 
[INFO] --- max number of job supported in this node is 2 
[INFO] --- begin to change owner of drs and data 
touch: cannot touch '/data/drsTouchLog': Permission denied 
[WARNING] --- Ruby does not have privilege to touch file in the path /data, please check ! 
[INFO] --- install node finished 
[INFO] --- add host name 
==================== 5 / 6 STARTING DRS NODE ===================== 
[INFO] --- begin to generate replay ca cert 
[INFO] --- begin to config iptables and optimize net params for replay 
[INFO] --- Begin to startup node-agent: su - Ruby -c 'sh /drs/start_cluster_node_agent.sh' 
[INFO] --- node agent added to crontab 
[INFO] --- node-agent started 
================= 6 / 6 RESET INSTALL CONF FILE ================== 
[INFO] --- deploy drs node finished 
deploy drs node success 
==================== DEPLOY DRS NODE FINISHED ====================
```
**步骤10**&emsp;补熵操作
操作系统应包含Haveged用于补充系统熵值，防止加密等操作因熵值不足导致系统不可用：检查系统是否安装Haveged软件
```
Haveged –help
```
如果报错说明未安装，安装命令如下：
```
yum install -y Haveged
```
检查系统是否启动haveged服务：
```
systemctl status haveged
```
启动服务：
```
systemctl start haveged
```

## 8 卸载DRS-Node服务
**步骤1**&emsp;以root用户登录待安装DRS的服务器。
**步骤2**&emsp;执行以下命令，进入/data
```
cd /data
```
**步骤3**&emsp;进入DRS-Node文件夹，其中DRS-Node-*版本号以实际为准。
```
cd DRS-Node-*
```
**步骤4**&emsp;执行以下命令，运行卸载脚本。
```
sh uninstall.sh
```
回显信息如下所示：
```
# sh uninstall.sh  
[INFO] --- node agent crontab cleaned 
[INFO] --- start to stop node-agent process 
[INFO] --- stop node-agent process success 
[INFO] --- start to stop all drs process 
[INFO] --- stop all drs process success 
[INFO] --- start to remove files 
[INFO] --- remove files success 
[INFO] --- uninstall drs finished
```
- DRS-Node服务卸载后，Node心跳两分钟后会自动过期，如果需要重新安装，建议在卸载两分钟后再进行重新安装。
- 若存在多Node节点部署，请在每个节点上执行相同的卸载操作。
----结束

## 9 安装Monitor-Agent
**步骤1**&emsp;以root用户登录待安装DRS-Node的服务器。
**步骤2**&emsp;将DRS-Node软件包上传到/data文件夹中。
Monitor-Agent-*.tar.gz
**步骤3**&emsp;执行以下命令，进入/data
```
cd /data
```
**步骤4**&emsp;执行以下命令，解压并进入Monitor-Agent文件夹，其中Monitor-Agent-*版本号以实际为准。
```
tar -xvf Monitor-Agent-*.tar.gz
cd Monitor-Agent-*
```
**步骤5**&emsp;执行以下命令，运行install.sh脚本。
安装脚本能够自动识别当前机器已部署的组件：
以下部分需要手动输入信息：
please input database ip:port [default: 127.0.0.1:30100]:(输入三台元数据库IP地址:端口号，使用英文逗号隔开)
please input platform drs user[default:drs]:  (输入回车，选择默认用户drs)；
please input platform drs password:     (输入元数据库drs用户登录密码，即安装DRS-Node时install.conf配置的metaDB_drs_password)        
Retype password:    (再次输入元数据库drs用户登录密码) 
```
sh install.sh
```
回显信息如下所示
```
[root@host-172-16-11-98 Monitor-Agent-2.23.07.222]# sh install.sh  
<13>Oct 30 19:18:50 DRS-Monitor-agent[533308]: [DRSS] execute install DRS-Monitor-agent script. 
============= 0 / 4 START TO DEPLOY MONITOR AGENT ================ 
2023-10-30 19:18:50 --- gaussdb_process_num: 1 
2023-10-30 19:18:50 --- drs_application_process_num: 1 
2023-10-30 19:18:50 --- node_agent_process_num: 1 
#元数据库IP:端口，如果是三副本节点，则IP:port使用英文逗号隔开，例如[10.8.2.1:30100,10.8.2.2:30100,10.8.2.3:30100] 
please input database ip:port [default: 127.0.0.1:30100]: 
please input platform drs user[default:drs]:    #元数据库DRS用户 
please input platform drs password:             #元数据库DRS用户密码 
Retype password:                                #二次确认密码           
2023-10-30 19:18:57 --- create mkdir opt begin 
2023-10-30 19:18:57 --- create mkdir opt success 
======================== 1 / 4 CREATE USER ======================= 
2023-10-30 19:18:57 --- create Ruby user begin 
2023-10-30 19:18:57 --- check Ruby exist begin 
2023-10-30 19:18:57 --- Ruby exist. Return 1 
User Ruby already existed, continue 
2023-10-30 19:18:57 --- create Ruby user success 
====================== 2 / 4 INIT 3RD_PARTY ====================== 
2023-10-30 19:18:57 --- install java/gsql begin 
2023-10-30 19:18:57 --- x86 OS_ARCH is x86_64 
2023-10-30 19:18:57 --- java_pkg is jre-8u372-linux-x64.tar.gz 
2023-10-30 19:18:59 --- modify /home/Ruby/.bashrc for java 
2023-10-30 19:18:59 --- install java for Ruby finish. 
2023-10-30 19:18:59 --- gsql_pkg is x86_gsql_Kylin.tar.gz 
2023-10-30 19:19:00 --- modify /home/Ruby/.bashrc for gsql 
2023-10-30 19:19:00 --- install gsql for Ruby finish. 
2023-10-30 19:19:00 --- install java/gsql success 
=============== 3 / 4 PREPARING SOFTWARE PACKAGES ================ 
2023-10-30 19:19:00 --- decompress monitor agent begin 
drs-monitor-agent/ 
drs-monitor-agent/conf/ 
drs-monitor-agent/conf/application.properties 
drs-monitor-agent/bin/ 
drs-monitor-agent/bin/dep_manager.sh 
drs-monitor-agent/bin/process-monitor.sh 
drs-monitor-agent/bin/db_monitor_init.sh 
drs-monitor-agent/bin/stop.sh 
drs-monitor-agent/bin/start.sh 
drs-monitor-agent/monitor-agent-2.16.1.jar 
2023-10-30 19:19:00 --- tar MonitorAgent.tar.gz success 
2023-10-30 19:19:00 --- decompress monitor agent success 
2023-10-30 19:19:00 --- deploy monitor agent begin 
2023-10-30 19:19:01 --- database and drs_service on this node. 
2023-10-30 19:19:01 --- This node type is database 
2023-10-30 19:19:01 --- deploy database system status begin 
2023-10-30 19:19:02 --- drs_system_status exist 
java -jar /opt/drs-monitor-agent/monitor-agent-*.jar -command db_delete_by_ip -param '172.16.11.98,gaussdb' -conf /opt/drs-monitor-agent/conf/application.properties 
java -jar /opt/drs-monitor-agent/monitor-agent-*.jar -command db_insert -param '172.16.11.98,gaussdb,primary,e3089620-afbc-419f-b410-b78b374a0488,normal,database' -conf /opt/drs-monitor-agent/conf/application.properties 
Oct 30, 2023 7:19:03 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: [cc1ed06c-09fd-4c49-b4bc-4b6b71b94240] Try to connect. IP: 127.0.0.1:30100 
Oct 30, 2023 7:19:04 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: [127.0.0.1:42072/127.0.0.1:30100] Connection is established. ID: cc1ed06c-09fd-4c49-b4bc-4b6b71b94240 
Oct 30, 2023 7:19:04 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: Known status of host 127.0.0.1:30100 is Master 
Oct 30, 2023 7:19:04 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: Connect complete. ID: cc1ed06c-09fd-4c49-b4bc-4b6b71b94240 
Oct 30, 2023 7:19:05 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: [781ea611-d6d3-4b25-bf9f-7f9c1bc646b6] Try to connect. IP: 127.0.0.1:30100 
Oct 30, 2023 7:19:05 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: [127.0.0.1:42090/127.0.0.1:30100] Connection is established. ID: 781ea611-d6d3-4b25-bf9f-7f9c1bc646b6 
Oct 30, 2023 7:19:05 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: Known status of host 127.0.0.1:30100 is Master 
Oct 30, 2023 7:19:05 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: Connect complete. ID: 781ea611-d6d3-4b25-bf9f-7f9c1bc646b6 
2023-10-30 19:19:05 --- Save system status success. 
2023-10-30 19:19:05 --- deploy database system status end 
2023-10-30 19:19:05 --- This node contain drs service 
Oct 30, 2023 7:19:06 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: [6d6a85ea-b889-4cb5-89f0-5444ef8944f5] Try to connect. IP: 127.0.0.1:30100 
Oct 30, 2023 7:19:06 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: [127.0.0.1:42104/127.0.0.1:30100] Connection is established. ID: 6d6a85ea-b889-4cb5-89f0-5444ef8944f5 
Oct 30, 2023 7:19:06 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: Known status of host 127.0.0.1:30100 is Master 
Oct 30, 2023 7:19:06 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: Connect complete. ID: 6d6a85ea-b889-4cb5-89f0-5444ef8944f5 
Oct 30, 2023 7:19:07 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: [f245e849-44c2-4cee-908e-30316649a276] Try to connect. IP: 127.0.0.1:30100 
Oct 30, 2023 7:19:08 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: [127.0.0.1:42118/127.0.0.1:30100] Connection is established. ID: f245e849-44c2-4cee-908e-30316649a276 
Oct 30, 2023 7:19:08 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: Known status of host 127.0.0.1:30100 is Master 
Oct 30, 2023 7:19:08 PM com.huawei.opengauss.jdbc.core.v3.ConnectionFactoryImpl openConnectionImpl 
INFO: Connect complete. ID: f245e849-44c2-4cee-908e-30316649a276 
2023-10-30 19:19:08 --- deploy monitor agent success 
================== 4 / 4 STARTING MONITOR AGENT ================== 
2023-10-30 19:19:08 --- install crontab 
2023-10-30 19:19:08 --- crontab now: 
* * * * *   source/etc/profile;source ~/.bashrc;sh /opt/drs/process-monitor.sh >> /opt/drs/logs/process-monitor.log 2>&1 & 
40 3 * * *  source/etc/profile;source ~/.bashrc;sh /opt/drs/backup/control/backup_crontab.sh >> /opt/drs/logs/backup.log 2>&1 & 
40 4 * * *  source/etc/profile;source ~/.bashrc;sh /opt/drs/backup/db/backup_db_crontab.sh >> /opt/drs/logs/backup_db.log 2>&1 & 
* * * * * source /etc/profile;source ~/.bashrc;/usr/bin/sh /drs/start_cluster_node_agent.sh 
* * * * * source /etc/profile; source ~/.bashrc; sh /opt/drs-monitor-agent/bin/process-monitor.sh >> /opt/drs-monitor-agent/logs/process-monitor.log 2>&1 & 
2023-10-30 19:19:08 --- install monitor successful 
2023-10-30 19:19:08 --- chmod log file start 
2023-10-30 19:19:08 --- chmod log file end 
============================= END ================================ 
[root@host-172-16-11-98 Monitor-Agent-2.23.07.222]# 
```
----结束

## 10 卸载Monitor-Agent
**步骤1**&emsp;以root用户登录待安装DRS-Node的服务器。
**步骤2**&emsp;执行以下命令，进入/data
```
cd /data
```
**步骤3**&emsp;执行以下命令，运行install.sh脚本。
```
sh uninstall.sh
```
回显信息如下所示：
```
[root@host-172-16-110-234 Monitor-Agent]# sh uninstall.sh  
2023-03-28 15:09:07 --- uninstall crontab 
2023-03-28 15:09:07 --- stop agent 
kill monitor Ruby     2312432       1  0 Mar17 ?        00:23:29 java -jar monitor-agent-2.16.1.jar 
2023-03-28 15:09:07 --- uninstall agent
```
----结束

## 11 附录
### 11.1 GUC参数全量列表

### 11.2 常用工具&运维命令
#### 11.2.1 gsql
连接数据库：
```
gsql -d <database-name> -p <port-number> -U <user-name> -W <password> -r
```
查看gsql内核版本：
```
gsql -V
```
进入gsql连接数据库后常用命令：
\l :列出所有数据库的名称、所有者、字符集编码以及使用权限；
\dt：列出当前数据库的所有用户表；命令后面加表名可以确认该表是否存在；
\d+：列出当前数据库的所有表，视图，序列；命令后面加对象名可以确认该表是否存在；
\i sql语句文件：执行一个SQL脚本文件
#### 11.2.2 gs_om
数据库维护工具；
启动整个集群数据库实例：gs_om -t start
停止整个集群数据库实例：gs_om -t stop
强制停止整个集群数据库实例：gs_om stop -mi
启动集群中的单个数据库:gs_om -t start -h <hostname or hostip>
停止集群中的单个数据库实例:gs_om -t stop -h <hostname or hostip>
重启数据库实例：gs_om -t restart
查询数据库实例状态：gs_om -t status -h <hostname or hostip>
#### 11.2.3 cm_ctl
统一集群管理工具；
查询集群状态：
```
cm_ctl query -Cvipd
```
确认主节点：
集中式：
查看” [  Datanode State   ]”部分的输出，State为”Primary”的Node即为主节点
 
分布式：
查看“[ Central Coordinator State  ]”部分的输出，连接该台CN即可：
 
启动集群或者集群的单个节点：
```
cm_ctl start -z <AZ-NAME> -n <NODE_ID>
```
※：NODE_ID可以从cm_ctl query -Cvipd查询结果中得知
停止集群或者集群的单个节点：
```
cm_ctl stop -z <AZ-NAME> -n <NODE_ID>
```
※：NODE_ID可以从cm_ctl query -Cvipd查询结果中得知
#### 11.2.4 进入沙箱
执行命令：chroot /var/chroot，进入沙箱
依次执行以下命令：
```
source /etc/profile
source /home/Ruby/.bashrc
source /home/Ruby/gauss_env_file
````
#### 11.2.5 gs_guc
使用该工具，需要先进入沙箱。

查看GUC参数：

gs_guc check -Z datanode -N all -I all -c “参数名” –集中式
gs_guc check -Z datanode -Z coordinator -N all -I all -c “参数名” –分布式

修改GUC参数；

修改POSTMASTER类型参数：
gs_guc set -Z datanode -N all -I all -c “参数名=参数值” –集中式
gs_guc set -Z datanode -Z coordinator -N all -I all -c “参数名=参数值” –分布式
修改后需要重启数据库：
```
gs_om -t start
gs_om -t stop
```
修改SIGHUP等其它类型参数：
gs_guc reload -Z datanode -N all -I all -c “参数名=参数值” –集中式
gs_guc reload -Z datanode -Z coordinator -N all -I all -c “参数名=参数值” –分布式

修改hba.conf安全访问控制策略：
gs_guc set/reload -Z datanode -N all -I all -h “host/local <database-name>/all  <user-name>/all <IP/Port> < authmehod-options>” –集中式
gs_guc set/reload -Z coordinator -N all -I all -h “host/local <database-name>/all  <user-name>/all <IP/Port> < authmehod-options>” –分布式
authmehod-options可选值：trust/sh256/md5/cert/gss/reject
#### 11.2.6 gs_collector
使用该工具，需要先进入沙箱。

默认收集GaussDB的系统信息：
```
gs_collector --begin-time="20240617 10:00" --end-time="20240617 11:00"
```
此时会收集System,Database系统表/系统视图,Log,Config信息，并存放到$GAUSSLOG目录下。

收集Coredump,Gstack信息：
先配置config文件，内容如下：
```
{
    "Collect":
    [
        {"TypeName": "Gstack", "Content":"DataNode","Interval":"10", "Count":"10"},
        {"TypeName": "CoreDump", "Content" : "gaussdb,cm_server,cm_agent,gs_ctl,gs_rewind,gaussdb_stack,gs_rewind_stack,cm_server_stack,cm_agent_stack,gs_ctl_stack", "Interval":"0", "Count":"0"}
    ]
}
```
CoreDump类型，不需要指定Interval和Count
```
gs_collector --begin-time="20240617 10:00" --end-time="20240617 11:00" -o /home/Ruby/coredump -C /home/Ruby/coredump/collector.conf
```
注：这里-o和-C指定的路径是指沙箱里的目录，即进入沙箱后创建的目录。















