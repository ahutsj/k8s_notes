# kubernates入门
## kubernates定义
1. 是一个全新的基于容器技术的分布式架构领先方案
2. 是一个开放的开发平台
3. 是一个完备的分布式系统支撑平台
## k8s的优点
1. 可以轻松开发复杂系统
2. 全面拥抱微服务架构
3. 系统可以随时整体搬迁到公有云上
4. k8s系统架构具备了超强的横向扩容能力
## k8s基本概念和术语
Node、Pod、Replication Controller和Servive等概念都可以看作一种资源对象，通过k8s提供的Kubectl工具或者API调用进行操作，并保存在etcd中。
### Node（节点）
Node是K8s集群相对于Master而言的工作主机，在较早的版本中也被成为Minion。在每个Node上运行用于启动和管理Pid的服务Kubelet，并能够被Master管理。在Node上运行的服务包括Kubelet、kube-proxy、docker daemon。    
Node信息如下：
1. Node的地址：主机的Ip地址，或者Node ID
2. Node的运行状态：包括Pending、Running、Terminated三种状态
3. Node Condition：描述Running状态Node的运行条件，目前只有一种条件--Ready。Ready表示Node处于健康状态，可以接受Master发来的创建Pod的指定
4. Node系统容量：描述Node可用的系统资源
5. 其他信息
#### Node的管理
Node通常是物理机、虚拟机或者云服务提供的资源，并不是由Kubernates创建的，所说的kubernates创建了一个Node，仅仅表示kubernates在系统内部创建了一个Node对象，创建后就会对其进行一系列检查，包括是否可以连通、服务是否正常启动、是否可以创建Pod等。如果检查未通过，则该Node会在集群内被标记为不可用（Not ready）
#### 使用Node Controller对Node进行管理
Node Controller是kubernates Master中的一个组件，用于管理Node对象，他的两个主要功能包括：集群范围内的Node信息同步，以及单个Node的生命周期管理。   
Node信息同步可以通过kube-controller-manager的启动参数--node-sync-period设置同步的时间周期。
#### Node的自我注册
当Kubelet的--register-node参数被设置为true时，Kubelet会向apiserver注册自己，这也是kubernates推荐的Node管理方式。
Kubelet进行自我注册的启动参数如下：
1. --apiserver=:apiserver地址
2. --kubeconfig=:登录apiserver所需凭据/证书的目录
3. --cloud_provider=:云服务商地址，用于获取自身的metadata
4. --register-node=:设置为true表示自动注册到apiserver
#### 手动管理Node
kubernates集群管理员可以手动创建和修改Node对象，当需要这样操作时，先要将Kubelet启动参数中的--register-node参数的值设置为false，这样在Node上的Kubelet就不会把自己注册到apiserver中去了
另外，Kubernates提供了一种运行时加入或者隔离某些Node的方法，见后续
### Pod
Pod是Kubernates最基本的操作单元，包含一个或多个紧密相关的容器，类似于豌豆荚的概念，一个Pod可以被一个容器化的环境看成应用层的逻辑宿主机，一个Pod的多个容器应用通常是紧密耦合的，Pod在Node上被创建。启动和销毁
为什么Kubernates使用Pod在容器之上再封装一层呢？一个很重要的原因是：Docker容器之间的通信受到Docker网络机制的限制，在Docker的世界，一个容器需要link的方式才能访问另一个容器提供的服务（端口）。大量容器之间的link是一个繁重的工作，通过Pod的概念将多个容器组合在一个虚拟的主机内，可以实现容器之间仅需通过localhost就能相互通信。
一个Pod中的应用容器共享同一组资源，如下描述：
1. PID命名空间：Pod内的不同应用程序可以看到其他程序的进程ID
2. 网络命名空间：Pod内的多个容器能够访问同一个IP和端口范围
3. IPC命名空间：Pod中的多个容器能够使用System IPC或者POSIX消息队列进行通信
4. UTS命名空间：Pod中的多个容器共享一个主机名
5. Volumes（共享数据卷）：Pod中的各个容器可以访问在Pod级别定义的Volumes
#### 对Pod的定义
对Pod的定义通过Yaml或者json格式的配置文件来完成，下面的配置文件将定义一个名为redis-slave的Pod，其中kind为Pod，在spec中主要包含了Containers的定义，可以定义多个container
```yaml
apiVersion: v1
kind: Pod
metadata:
name: redis-slave
spec:
containers:
- name: slave
image: kubeguide/guestbook-redis-slave
env:
- name: GET_HOSTS_FROM
value: env
ports:
- containerPort: 6379
```
Pod的生命周期是通过Replication Controller来管理的，Pod的生命周期过程包括：通过模板进行定义，然后分配到一个Node上运行，在Pod所含容器运行结束后Pod也结束，整个过程中，Pod处于以下4种状态之一：
1. Pending：Pod定义正确，提交到master,但是其包含的容器镜像还未完成创建，通常Master对Pod进行调度需要一些时间，之后Node对镜像进行下载也需要一定时间
2. Running：Pod已被分配到某个Node上，且其包含的所有容器镜像都已经创建完成，并成功运行起来
3. Succeeded：Pod中所有容器都成功结束，并且不会被重启，这是Pod的一种最终状态
4. Failed：Pod中所有容器都结束了，但至少一个容器是以失败状态结束的，这也是Pod的一种最终状态   

Kubernates为Pod设计了一套独特的网络配置，包括：为每个Pod分配一个IP地址，使用Pod名作为容器间通信的主机名等。
另外不建议在Kubernates的一个Pod内运行相同应用的多个实例
### Label（标签）
Label是Kubernates系统的核心概念，Label以键值对的形式附加到各种对象上，如Pod、Service、RC、Node等，Label定义了这些对象的可识别属性，用来对它们进行管理和选择，Label可以在创建时附加到对象上，也可以在对象创建后通过API进行管理
在为对象定义好Label后，其他对象就可以使用Label Selector来定义其作用的对象了
Label Selector的定义由多个逗号分隔的条件组成：
```
"label":{
    "key1":"value1",
    "key2":"value2",
}
```
当前有两种Label Selector：基于等式的和基于集合的，在使用时可以将多个Label进行组合来选择
基于等式的Label Selector使用等式类的表达式来进行选择：
1. name = redis-slave:选择所有包含Label中key="name"且value="redis-slave"的对象
2. env != production:选择所有包含Label中的key="env"且value不等于"production"的对象

基于集合的Label Selector使用集合操作的表达式来进行选择：
1. name in (redis-master,redis-slave):选择所有包含Label中的key = “name”且value= “redis-master”或者“redis-slave”的对象
2. name not in (php-frontend):选择所有包含Label中的key=“name”且value不等于"php-frontend"的对象

在某些对象需要对另一些对象进行选择时，可以将多个Label Selector进行组合，使用逗号进行隔开，基于等式的Lable Selector和基于集合的Label Selector可以任意组合：
```
name = redis-slave,env != production
name not in (php-frontend),env != production
```
### Replication Controller（RC）
Replication Controller是Kubernates系统的核心概念，用于定义Pod副本的数量，在Master内，Controller Manager进程通过RC的定义来完成Pod的创建、监控、启动和停止等操作
根据Replication Controller的定义，Kubernates能够确保在任意时刻都能运行用于指定的Pod副本（Replica）数量。如果有过多的Pod副本在运行，系统会停掉一些Pod。如果运行的Pod副本太少，就会再启动一些Pod。总之，通过RC的定义，Kubernates总是保证集群中运行着用户期望的副本数量。
同时Kubernates会对全部运行的Pod进行监控和管理，如果有需要，例如某个Pod停止运行，就会将Pod重启指令提交给Node上的某个程序来完成（比如Kubelet或者Docker）
可以说，通过对Replication Controller的使用，Kubernates实现了应用集群的高可用性，并大大减少了系统管理员在传统IT环境中需要完成的许多手工运维成本
对Replication Controller的定义使用Yaml或者Json格式的配置文件来完成，以redis-slave为例，在配置文件中通过spec.template来定义Pod的属性（这部分定义与Pod的定义一致），设置spec.replicas=2来定义Pod副本的数量
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
name: redis-slave
label: redis-slave
name: redis-slave
spec:
replicas: 2
selector:
name: redis-slave
template:
metadata;
labels:
name: redis-slave
spec:
contianer:
- name: slave
image: kubeguide/guestbook-redis-slave
env:
- name: GET_HOSTS_FROM
value: env
ports:
- containerPort: 6379
```
通常，Kubernates集群中不止一个Node，假设一个集群内有3个Node，根据RC的定义，系统将可能在其中的两个Node上创建Pod
### Service（服务）
在Kubernates的世界，虽然每个Pod都会被分配一个单独的IP地址，这个IP地址会随着Pod的销毁而消失，这就引出一个问题：如果有一组Pod组成一个集群来提供服务，那么如何来访问它们呢
Kubernates的Service就是用来解决这个问题的核心概念
一个Service可以看做一组提供相同服务的Pod的对外访问接口，Service作用于哪些Pod是通过Label Selector来定义的
#### 1. 对Service的定义
对Service的定义同样使用Yaml或者Json格式的配置文件来完成。以redis-slave服务的定义为例：
```yaml
apiVersion: v1
kind: Service
metadata:
name: redis-slave
labels:
name: redis-slave
spec:
ports:
- port: 6379
selector:
name: redis-slave
```
通过该定义，Kubernates会创建一个名为”redis-slave“的服务，并在6379端口监听





