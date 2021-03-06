## 后端高可用
数据库性能瓶颈：代码每次给数据库发送SQL执行时，都需要创建一个数据库连接池通道，建立这个通道极为昂贵。对于每一个Connection，需要开辟不少缓冲区，用来读取表中的数据，进行join、sort操作等，既费时又费力。

有限的connection对于应用程序来说是一个大障碍，在局域网时代不明显，可是到了互联网时代，数据库成了瓶颈。

参考CPU的三级缓存，于是在应用程序和数据库之间增加一个抽象层-缓存，缓存直接放在内存中。

初期可以把缓存和应用程序放在一台机器上，否则还需要跨网络访问，还得对数据做序列化和反序列化，十分麻烦。

### 分家
所有的东西都在一台机器上，而一起机器毕竟能力有限，用户量大的时候处理起来还是力不从心，更重要的是内存、硬盘、网卡等偶尔罢工就会导致整个系统停摆，因此分家势在必行。但分家必然会导致很多的问题。一起来看看如何解决吧。

第一期配置如下
* Nginx 独占一台服务器
* 系统的核心和骨干Tomcat，分配多台服务器，每台服务器部署相同的应用代码，这样即使一台停摆了，其他服务器可以工作，不会受制于人
* 缓存一台单独服务器
* 数据库一台单独服务器

这样一来，tomcat和缓存不在一台服务器上，就得通过网络访问，就需要使用烦人的序列化和反序列化了。但分家是非常必要的，将来缓存也需要做集群。此时redis就可以排上用处了。

redis最大优点是能够快速的存储海量的key-value字符串。你可以把对象序列化，然后作为二进制数据存放在redis中。但目前更常用的是JSON，直接把JSON字符串存储在redis中，前端需要时，直接取出来就可以返回，十分方便。同时redis也支持最常用的数据结构
* List：列表，可以当做队列或者栈使用
* Set：集合，包含不重复的字符串，没有次序
* SortedSet：有序集合，有次序的不重复字符串
* Hash：包含键值对的无序散列表，和Java中的HashMap很像

有了redis后，Tomcat不需要关心网络访问的细节，只要关注把对象转换为JSON字符串就可以，剩下的事情交给redis处理。

越来越多的数据放入缓存服务器，一台缓存服务器内存快用完了，那就只能增加几台缓存服务器，做redis集群，但随之而来会带来几个问题
* 多缓存服务器后，每个redis存储的数据不一样，应用程序如何选择哪个服务器呢
* 取数据的时候必须知道之前存储在那台服务器，否则缓存就会找不到
* 数据存储要尽可能均匀

余数算法
* 对于用户要存储的k-v，计算出v的一个整数hash值，然后用这个hash值对服务器数目求余数
* 取数据的时候，用同样的办法，计算hash值，求余数，找到服务器，从那台服务器取出数据

余数算法有个致命缺点：如果动态增加一台新的缓存服务器，很明显，之前的缓存会大规模失效

一致性Hash算法：能极大的缓解缓存失效问题，它能把失效的缓存控制在特定的区间，不像余数算法那样会天下大乱
* 在圈上顺时针标注2的32次方个点
* 根据服务器的IP或hostname，计算hash，得到hashcode，通过hashcode对应到圆圈的某个位置
* 存储数据时同样对k做hash，得到code1
* code1映射到圆圈某个位置，然后从这个位置开始，顺时针找到第一台服务器，就近存储，取数据同样的原理
* 如果增加一台新的服务器4，这样一来受影响的只会服务器4和某服务器之间数据

这里还有个小问题，如果服务器经过hash之后，在圆圈上不均匀，挤在了一起，就会发生某些服务器负载过高，而某些负载过低的情况。解决办法就是虚拟服务器。把一台服务器看做多台虚拟服务器，让它们分布在圆圈上，增加均匀性。此时后计算hash的时候就需要在ip或hostname后追加一个编号，比如ip#001

> 一致性hash，当增删服务器时，只会影响相邻节点的缓存数据，其他部分的数据与原来保持一致

一致性hash很不错，但有个致命问题没有解决，就是还是会部分失效

hash插槽
* 共16384槽，每台服务器分管其中一部分
* k-v过来保存时，计算key的hash值，然后对16384取余，看余数在哪个槽里，就放到对应的服务器中
* 新增加服务器需要对key进行迁移，从原有服务器中负责的槽中各自分出一些交给新增服务器处理，对应数据也要搬过去
* 客户端取数据时，可以向任一节点发出请求，如果该服务器没有会进行跳转，也就是说各个服务器之间可以重定向

故障转移：Redis Cluster 提供了 master-slave 功能，假设每组插槽有A、B、C负责，假设 A 是 master，则其余两台是 slave，slave 节点数据和 master 节点数据一样。同时 slave 节点虽然是备份，但时刻准备着替换 master，如果 master 节点挂掉，则从剩下的 slave 中选取一个当作新的 master。


### 高可用的 Nginx
由于 Nginx 需要对外提供统一的入口服务，因此目前只有一台服务器，但是一旦出现故障，就收不到请求，也就无法转发请求，就会出现单点失败。

解决办法：Nginx 同样提供多台服务器，通过一个Keepalived的家伙，能够把多个Nginx服务器形成一种master-slave结构，同一时候只有一个工作，另一个原地待命，如果工作的那个挂掉了，则待命的就接管。同时多个 Nginx 服务器对外只提供一个 ip，在人类看来就只有一台服务器

> nginx为了给用户提供唯一的入口，迫不得已只能同时让一台服务器工作，其余待命。同时nginx只会处理简单的静态资源，对于动态资源请求，nginx都会只能转发给Tomcat处理

nginx出了转发请求外，还需要实现负载均衡，不能一股脑的发给某几台服务器，要尽可能的分配到每个人。

问题来了：nginx只是转发请求，不保存状态，实现高可用很容易，但是 Tomcat 有 session 需要管理。会出现的问题就是
* 用户访问系统，在tomcat#1上创建了购物车，然后tomcat#1挂掉了，nginx会进行失效转移，把请求转移到其他的tomcat上，此时用户会发现购物车不见了
* 甚至，用户明明登录了系统，由于产生了失效转移或者负载均衡请求被发到了其他服务器上，其他服务器发现用户没有登录，就会让用户再次登录

处理不好 Tomcat 的状态问题，Tomcat集群的威力就会大打折扣，无法完成真正的失效转移，甚至无法使用。

nginx有一种叫做session sticky技术，可以保证来自同一个客户端的请求一直被转发到同一台tomcat服务器上，这样session就不用复制了。但是依旧无法解决失效转移导致的问题。

绝佳解决办法：把 session 放在 Redis 集群中保存。

### MySQL
Redis，Nginx，Tomcat都集群化了，就剩下数据库了，比如常用的 MySQL，如何让 MySQL 实现高可用呢？

数据库不同于前三者，实现集群会十分麻烦，而且很难保证数据的一致性。

> 在一个分布式环境中，保持数据的强一致性非常难

但可以实现读写分离，来提高数据库的性能，同样采用master-slave的方式，先设置一个master数据库，在设置一个或多个slave数据库，master数据库可读可写（以写为主），slave数据库只能读、不能写。最后还要讲master库的数据尽快的复制到slave库，让数据保持一致。

读写分离的好处
* 一个web应用读操作是远远多于写操作的，用专门的数据库负责读，能提高效率
* 数据库事务，为了实现不同的隔离级别，有X锁（排他锁，写数据使用）和S锁（共享锁，读数据使用），实现读写分离，能极大的缓解程序对X锁和S锁的争用
* 同时如果master库挂掉了，可以指定一个slave库成为master库，实现高可用

数据库实现读写分离后，增加了tomcat的工作，因为需要判断哪些是写操作，哪些是读操作，然后发到对应的数据库，增加业务层面的复杂度显然是不合适的，因此可以添加一个中间层MySQL Proxy。只管向他发送请求，不用区分读和写，由这个MySQL Proxy把脏活累活都干了。

> 数据库很难做集群，但是可以做业务拆分

## 函数调用
本地过程调用：所有的调用都发生在本机内的一个进程中，这种调用方式飞快。

因为在同一个进程中，每一个函数的地址对大家而言都是可见的，想要调用，直接找到地址执行代码即可。跨域网络访问的调用需要使用socket建立连接，在连接上按双方商量好的格式，次序来发送数据，接受数据。

远程调用RPC：客户端代理 + 服务端代理，这两个代理把复杂的网络细节都隐藏起来，看你们看来和本地调用一样，有些代理可能使用socket通信，有些可能使用HTTP通信

传递的数据需要进行序列化，不然无法通过网络发送，把内存中的值和对象变成二进制数据流，这样才能发送出去。到了服务端代理后，需要进行反序列化，把二进制数据在转换为对象，然后才能调用真正的函数。

> 有些人在使用HTTP作为通信协议的时候，喜欢把对象变成文本，如XML/JSON，可读性比较好，虽然在应用层的HTTP中看起来是文本，但是到底层通道（如TCP）发送出去的时候，还得变成二进制数据流，到了目的地再把他们转换成文本

## SOA到微服务
面向服务的体系结构（service-oriented architecture，SOA）：异构系统包装成粗粒度服务，还是web服务，可以通过HTTP访问，大家之间可以相互调用，甚至可以把这些服务进行编排，形成一个大的业务流程，完成更高层次的业务

微服务：把系统拆分成一个个小组件，小组件都是独立的，每个组件都是一个进程，通过轻量级的基于HTTP的RESTful来对外提供接口，这里面可能涉及到数据库拆分

> docker：代码和环境在一起可以成为一个镜像，把这个镜像放到服务端的docker运行

> 猴子测试：通过脚本随机的停掉一些实例，看系统运行的怎么样

微服务缺点
* 数据做了分区，一致性如何保证
* 分布式事务非常麻烦，不得不选择最终一致性妥协
* 基于HTTP的调用远远没有在一个进程内的方式效率高
* 监控麻烦：运行实例多，相互之间调用，出错难以追踪

## HTTP Server
* HTTP Server 1.0：阻塞等待
* HTTP Server 2.0：多进程
  * 为每个新的socket创建子进程来接管，主进程不会阻塞，可以继续接受新的连接
  * 每个进程都要消耗大量资源，进程切换对于操作系统而言开销也很大
* HTTP Server 3.0：select模型
  * 一个socket链接就是一个文件描述符（File Descriptor），fd背后是一个简单的数据结构
  * 操作系统保存好socket编号，如果发现socket可以读写了，就把对应socket做一个标记
  * server遍历socket，处理数据，处理完了再把fd告诉操作系统
* HTTP Server 4.0：epoll模型
  * 3.0有缺陷，1.每次最多告诉1024个 socket fd，2.反复遍历socket fd，看有没有标志位需要处理，而实际情况很多socket并不活跃，在一段时间内浏览器并没有数据发过来，这1000多个socket可能只有那么几十个需要处理，却不得不查看所有的socket
  * epoll模型就只直接把发生了变化的socket告诉HTPP Server
