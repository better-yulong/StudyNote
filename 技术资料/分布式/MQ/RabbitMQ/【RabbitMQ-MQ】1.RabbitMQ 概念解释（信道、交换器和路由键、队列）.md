转自：https://www.cnblogs.com/zhangxue/category/1099956.html
推荐：https://www.cnblogs.com/pjjlt/p/10741963.html
     https://www.cnblogs.com/xiaoxing/p/9250823.html
     https://blog.csdn.net/gbbqrglvir3dyi82/article/details/78663828
     https://www.cnblogs.com/vipstone/p/9368106.html
     https://www.cnblogs.com/wxgblogs/p/5618151.html
     https://yq.aliyun.com/articles/699886
### 一、 channel 信道：
- 　　概念：信道是生产消费者与rabbit通信的渠道，生产者publish或是消费者subscribe一个队列都是通过信道来通信的。信道是建立在TCP连接上的虚拟连接，什么意思呢？就是说rabbitmq在一条TCP上建立成百上千个信道来达到多个线程处理，这个TCP被多个线程共享，每个线程对应一个信道，信道在rabbit都有唯一的ID ,保证了信道私有性，对应上唯一的线程使用。
-     疑问：为什么不建立多个TCP连接呢？原因是rabbit保证性能，系统为每个线程开辟一个TCP是非常消耗性能，每秒成百上千的建立销毁TCP会严重消耗系统。所以rabbitmq选择建立多个信道（建立在tcp的虚拟连接）连接到rabbit上。
-     类似概念：TCP是电缆，信道就是里面的光纤，每个光纤都是独立的，互不影响。

###  二、exchange 交换机和绑定routing key
-  exchange的作用就是类似路由器，routing key 就是路由键，服务器会根据路由键将消息从交换器路由到队列上去。

-  exchange有多个种类：direct，fanout，topic，header（非路由键匹配，功能和direct类似，很少用）。前三种类似集合对应关系那样，（direct）1:1,（fanout）1：N,（topic）N:1
1. direct： 1:1类似完全匹配
2. fanout：1：N  可以把一个消息并行发布到多个队列上去，简单的说就是，当多个队列绑定到fanout的交换器,那么交换器一次性拷贝多个消息分别发送到绑定的队列上，每个队列有这个消息的副本。
　　1. ps：这个可以在业务上实现并行处理多个任务，比如，用户上传图片功能，当消息到达交换器上，它可以同时路由到积分增加队列和其它队列上，达到并行处理的目的，并且易扩展，以后有什么并行任务的时候，直接绑定到fanout交换器不需求改动之前的代码。
3. topic   N:1 ，多个交换器可以路由消息到同一个队列。根据模糊匹配，比如一个队列的routing key 为*.test ，那么凡是到达交换器的消息中的routing key 后缀.test都被路由到这个队列上。

### 三、结合信道、交换器和路由键到队列
- 总结几点重要知识：1.信道才是rabbit通信本质，生产者和消费者都是通过信道完成消息生产消费的。2.交换器本质是一张路由查询表（名称和队列id，类似于hash表），这是一个虚拟出来的东西，并不存在真实的交换器。
- 消息的生命周期：生产者生产消息A 交由信道，信道通过消息（消息由载体和标签）的标签（路由键）放到交换器发送到队列上（其实就是查询匹配，一旦匹配到了规则，信道就直接和队列产生连接，然后将消息发送过去）