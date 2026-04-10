## Mikrotik Routeros Container 安装Freeswitch
1. 第一步：开启 RouterOS 容器功能 /system/device-mode/update container=yes
2. 第二步：配置网络（VETH） 
   # 创建虚拟网口
   /interface/veth/add name=veth_fs address=172.17.0.2/24 gateway=172.17.0.1
   # 将其加入你的常用网桥（例如 bridge）
   /interface/bridge/port add bridge=bridge interface=veth_fs
3. 设置环境变量 根据你的需求定义音频文件和采样率：
   /container/envs/add name=fs_env key=SOUND_RATES value="8000:16000"
   /container/envs/add name=fs_env key=SOUND_TYPES value="music:zh-cn-sinmei"
4. 准备挂载目录（可选但建议）
    FreeSwitch 需要存储配置和音频。在 ROS 的磁盘（如 slot1 或 flash）上创建目录：
   /container mounts
   add dst=/etc/freeswitch name=freeswitch src=/data/freeswitch
   add dst=/usr/share/freeswitch/sounds name=freeswitch-sounds src=/data/freeswitch-sounds
5. 配置防火墙端口转发 (NAT)
   /ip/firewall/nat/add chain=dstnat dst-port=5060 protocol=udp action=dst-nat to-addresses=172.17.0.2
   /ip/firewall/nat/add chain=dstnat dst-port=16384-32768 protocol=udp action=dst-nat to-addresses=172.17.0.2
6. 启动容器 /container/start [find where remote-image~"freeswitch"]

- 特别提示
- 控制台访问：在 ROS 中，你可以使用 /container/shell 0 进入容器命令行，但由于此镜像极度精简，可能只有最基础的 shell。
- fs_cli 访问：建议在容器启动后，使用 docker exec 式的操作或通过 ROS 的命令行进入

在 FreeSwitch 中，编辑分机号码、用户和密码通常涉及修改 XML 配置文件。由于你使用的是 safarov/freeswitch 这个极简版镜像，操作方式与标准 Linux 环境略有不同。

由于该镜像删除了大部分工具，建议你在宿主机（RouterOS 或 PC）上编辑好文件，再挂载/上传到容器中。

1. 核心配置文件路径
FreeSwitch 的用户配置文件通常位于：
/etc/freeswitch/directory/default/

在该目录下，每一个 .xml 文件通常代表一个分机号（例如 1000.xml）。

2. 修改默认分机和密码
如果你想修改现有的分机（如 1000）或添加新号码：

第一步：找到配置文件
如果你按照之前的步骤挂载了目录，请在你的存储设备（如 /data/freeswitch/directory/default/）中找到：
directory/default/1000.xml 修改后在 fs_cli 中输入reloadxml
~~~
<include>
  <user id="1001">
    <params>
      <param name="password" value="$${default_password}"/>
      <param name="vm-password" value="1001"/>
    </params>
    <variables>
      <variable name="toll_allow" value="domestic,international,local"/>
      <variable name="accountcode" value="1001"/>
      <variable name="user_context" value="default"/>
      <variable name="effective_caller_id_name" value="Extension 1001"/>
      <variable name="effective_caller_id_number" value="1001"/>
      <variable name="outbound_caller_id_name" value="$${outbound_caller_name}"/>
      <variable name="outbound_caller_id_number" value="$${outbound_caller_id}"/>
      <variable name="callgroup" value="techsupport"/>
    </variables>
  </user>
</include>
~~~

# 配置铃声
- 打开 /etc/freeswitch/dialplan/default.xml，找到你处理分机的段落（通常叫 Local_Extension），按照下面这样修改：
  <extension name="Local_Extension">
  <condition field="destination_number" expression="^(10[01][0-9])$">
    <action application="set" data="ringback=/usr/share/freeswitch/sounds/安妮的仙境-纯音乐.wav"/>
    <action application="bridge" data="user/$1"/>
  </condition>
</extension>

# 设置全局密码
~~~
如果你只想修改全局默认密码（影响所有未单独设密的分机），可以修改 vars.xml 中的 default_password 变量：

文件位置：/etc/freeswitch/vars.xml

找到：<X-PRE-PROCESS cmd="set" data="default_password=1234"/>

在 fs_cli 中输入reloadxml
~~~
