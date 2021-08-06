# **NDN**

### 背景

IP最初的设计是为了在两个终端间进行通信。互联网通信逐渐以内容分发和检索为主导。涉及内容为中心（content-centric）的应用例如社交网络、电子商务、youtube、Netfilx、Amazon等等已经成为全球一半互联网流量的来源。通过端到端通信协议来解决分布式网络中的分布问题是容易出错和复杂的。大部分的应用数据传送模型关心的是需要什么数据而不再在意它们的地址。

与IP网络的不同：

1. NDN的包携带的是独一无二的内容名字（content name）以代替IP包中的源地址、目的地址。
2. 所有的数据包都会被其产出者签名和被消费者辨认出。
3. 此外NDN路由器支持多路径转发，即NDN路由器可以同时向多个接口转发一个用户请求。

### NDN网络结构

NDN网络结构继承了IP网络的分层细腰状结构，使用数据检索模型代替了IP网络中的端到端数据传送。在协议栈中新增了安全层和策略层，同时IP网络中的传输层的功能全面嵌入到NDN转发层面中。

![image](https://user-images.githubusercontent.com/49645739/128478221-23e23385-8ffa-4eec-b9ca-c2e277a33205.png)

## NDN通信

#### name 名字

1. 用于获取全局数据的名称必须是全局唯一的。
2. 搜索应用程序会依据CN生成请求。

<img src="C:\Users\刘博\AppData\Roaming\Typora\typora-user-images\image-20210806102808210.png" alt="image-20210806102808210" style="zoom:50%;" />



#### 通信机制

NDN通信是由消费者以兴趣包的形式发起的。当兴趣包到达内容发布者或者到达一个有有效请求内容的节点时，对应该兴趣包的数据包将被发出。数据包沿着兴趣包的路径反向到达请求者。这里，返回请求数据副本的被成为提供者 provider，真正产出（create or generated）请求内容的称为生产者 producer。

兴趣包和数据包都包含content name。

##### 兴趣包

<img src="C:\Users\刘博\AppData\Roaming\Typora\typora-user-images\image-20210806120454848.png" alt="image-20210806120454848" style="zoom: 67%;" />

- 兴趣包的名字作用相当于IP包中的目的地址。
- 用于返回请求数据的兴趣包不携带请求消费者的身份（地址或名字）。
- 对于每个被转发的兴趣包，NDN路由器记录其传入的接口。
- nonce字段是由消费者生成的随机数。PIT条目接收到的每个兴趣包的名字和nonce都被记录在路由器中。通过这些信息，路由器可以检测相同的兴趣包是否来源于不同的消费者。因此网络包不会陷入循环并且路由器可以尝试多条可选择的不同路径。 

##### 数据包

<img src="C:\Users\刘博\AppData\Roaming\Typora\typora-user-images\image-20210806120926728.png" alt="image-20210806120926728" style="zoom: 67%;" />

​	每一个数据包都是

- self-identifying：content name标识每个用户想要的数据。
- self-authenticating：包含生产者的签名。

**在NDN中，如果兴趣包在数据仓库或者中间的路由器中遇到了请求数据的副本后则直接返回。但是在IP网络中，IP包总会被传送到目的地。**

#### CS、PIT、FIB

每个NDN路由器都保持3个主要的数据结构：FIB、PIT、CS。

![image-20210806152907473](C:\Users\刘博\AppData\Roaming\Typora\typora-user-images\image-20210806152907473.png)

- **Content Store**

  每一个数据包可以被多个消费者使用，例如多个用户可能阅读同一份报纸。NDN路由器缓存传送给它们的数据包，直到被新的内容所代替（因为缓存大小有限）。通过匹配名字（逐字符匹配）来搜索CS中的条目。

- **Pending Interest Table**

  PIT为每个到来的兴趣包维持一个条目，直到与其相关的数据包到来或者该条目的生命周期到期，二者谁更早则执行哪一个。PIT存储所有未满足转发的兴趣包。每个PIT的条目都记录了兴趣包的名字、传入接口和传出接口。

- **Forwarding Information Base**

  FIB维持下一跳和其他可到达的每一个目的地名字前缀。FIB由路由协议填充，用于向上游转发兴趣包。

  

#### 转发过程

<img src="C:\Users\刘博\AppData\Roaming\Typora\typora-user-images\image-20210806150221236.png" alt="image-20210806150221236"  />

##### 向上转发兴趣包

1. 无论何时当一个兴趣包到达一个NDN路由器后，将会在CS中搜索content name以找到匹配的内容。如果成功匹配，则NDN路由器通过数据包转发给消费者内容，并被生产者签名。否则将在PIT中进一步匹配名字。

2. 如果在PIT的条目中匹配到，兴趣包传入的接口将被添加到被看作兴趣包聚合（Interest aggregation）的接口列表。因此当相关数据包到来后，所有数据请求者都会收到这个数据包的副本。

3. 如果在PIT中没有匹配到，则在FIB中进行最长前缀匹配（LPM)。如果匹配到了一条FIB条目，则兴趣包会被转发到相关的下一跳（可能多个）并创建一条新的PIT条目，将该兴趣包的传入接口添加到其中。
<img src="C:\Users\刘博\AppData\Roaming\Typora\typora-user-images\image-20210806152645318.png" alt="image-20210806152645318" style="zoom:67%;" />

4. 否则，当不满足以上所有要求时，由路由器转发政策决定要么将兴趣包洪发给所有的传出接口，要么删除该兴趣包。

##### 向下转发数据包

当数据包返回到NDN路由器时，检索所有的PIT条目以匹配content name。当匹配到后，该数据包会转发到在传入接口列表中的所有接口。之后，删除该条PIT条目，基于本地缓存政策，将内容存储在CS中。

当在PIT中未匹配到内容后（可能由于其生命周期已过期），则丢弃该数据包。

