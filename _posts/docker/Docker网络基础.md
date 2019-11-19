### Docker网络基础

​        Docker技术依赖linux内核的虚拟化技术，这些技术包括网络命名空间、veth设备对、网桥、iptables及路由。

#### 网络命名空间

​        网络命令空间将网络协议栈独立隔离到不同的命名空间中，彼此之间无法通信。通过对网络资源的隔离，就能在一个宿主机上虚拟化出不同的网络环境。Docker利用网络命名空间特性，实现了不同容器之间的隔离。

​        在每个网络命名空间有独立的路由表及iptables设置来提供包转发，NAT及IP包过滤等功能。为隔离出独立的协议栈，需要纳入进程，套接字，网络设备等元素。

1. 网络命令空间实现

   ​        网络命令空间的实现比较复杂，笼统地说就是每个在代码中将网络命令空间相关的元素变量都设为每个网络命名空间私有，其他网络空间不可访问相互并不冲突。

   ​        新生成的网络命名空间只有一个回环设备（“名为lo”，并处于停止状态），如需要其他设备则需手动建立。

2. 网络命名空间操作

   * 创建网络命名空间

     ``` shell
     ip netns add <name> 
     ```

   * 进入网络命名空间，如进入“net1”空间bash:“ip netns exec bash”

     ``` shell
     ip netns exec <name> <command>
     ```

3. 使用技巧

   ​		可以通过ethtool查看网络命名空间里的设备是否能进行转移。其中"netns-local"参数为"on"的话就表明此设备不能转移到其他命名空间中，若为“off”则表示可以转移。

   ``` 
   ethtool -k <device>
   ```

   ​		将设备转移到其他明明空间中

   ``` 
   ip link set <device> netns <netns>Veth设备对
   ```

#### Veth设备对

​		新创建的网络命令空间是无法与其他网络命令空间通信的，利用veth设备可以将两个网络命名空间连接起来。Veth设备总是成对出现的。Veth设备对像是成对出现的以太网卡，其中有一条直连的网线。Veth设备的一端发送数据，另一端peer就能接受到数据。

1. 操作命令

   * 创建设备对

     ``` shell
     ip link add veth0 type veth peer name veth1
     ```

   * 查看设备对，此时可看见会新增veth0和veth1两个veth设备

     ``` shell
     ip link show
     ```

   * 将其中一个设置到网络命名空间"netns1"，此时进入"netns1"网络命名空间会发现“veth1”设备

     ``` shell
     ip link set veth1 netns netns1
     ```

   * 为两个设备分配网络地址

     ``` shell
     ip netns exec netns1 ip addr add 10.1.1.1/24 dev veth1
     ip netns add 10.1.1.1/24 dev veth0
     ```

   * 启动两个veth网络设备

     ``` shell
     ip netns exec netnes1 ip link set dev veth1 up
     ip link set dev veth0 up 
     ```

   * 此时两个网络设备就可以进行通信操作

     ``` shell
     ip netns exec netns1 ping 10.1.1.1
     ping 10.1.1.2
     ```

2. 查找peer对端

   * 首先在命令空间查询对端设备在设备列表中的序列号，其中”peer_ifindex“就为对端序列号，如对端序列号14

     ``` shell
     ip netns exec netns1 ethtool -S veth1
     ```

   * 在设备列表中找到序列号为5的设备，其中序列号为5的就为对端设备

     ``` she
     ip link | grep 14
     ```

     