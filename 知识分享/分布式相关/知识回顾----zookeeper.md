[toc]

# zk是啥？

zookeeper是分布式协调系统，本质上是一个实现了通知机制的分布式文件系统，我们可以在zk执行以下操作

（1）创建一个永久节点。

（2）创建一个暂时节点（创建节点的客户端关闭时节点销毁）。

（3）监听某个节点（该节点发生变化是会推送一条通知信息）。

（4）查询、修改某个节点信息。

# 什么场景我们可以使用zk？

（1）注册中心。服务提供者在zk上创建暂时节点，服务提供者查询并watch节点信息并缓存到本地缓存中，服务提供者创建的暂时节点销毁后zk推送信息给订阅该节点的服务提供者。

（2）配置中心。将配置信息写在zk上，节点通过zk拿到配置信息。

（2）分布式锁。多个服务获取锁时均在zk上创建一个有序节点，序号最小的被认为拿到锁，该有序节点注销后可以通知其他序号节点竞争锁。

# watch机制是如何实现的？


