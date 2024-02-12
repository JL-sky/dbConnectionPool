  # connectionPool
  基于c++的MySQL连接池

  # 技术栈

  MySQL数据库编程、单例模式、queue队列容器、C++11多线程编程、线程互斥、线程同步通信和 unique_lock、基于CAS的原子整形、智能指针shared_ptr、lambda表达式、生产者-消费者线程模型



  # 项目背景

  为了提高MySQL数据库（基于C/S设计）的访问瓶颈，除了在服务器端增加缓存服务器缓存常用的数据 之外（例如redis），还可以增加连接池，来提高MySQL Server的访问效率，在高并发情况下，大量的 TCP三次握手、MySQL Server连接认证、MySQL Server关闭连接回收资源和TCP四次挥手所耗费的 性能时间也是很明显的，增加连接池就是为了减少这一部分的性能损耗。

  # 功能介绍

  连接池一般包含了数据库连接所用的ip地址、port端口号、用户名和密码以及其它的性能参数，例如初 始连接量，最大连接量，最大空闲时间、连接超时时间等，该项目是基于C++语言实现的连接池，主要 也是实现以上几个所有连接池都支持的通用基础功能

  **初始连接量（initSize）**：表示连接池事先会和MySQL Server创建initSize个数的connection连接，当 应用发起MySQL访问时，不用再创建和MySQL Server新的连接，直接从连接池中获取一个可用的连接 就可以，使用完成后，并不去释放connection，而是把当前connection再归还到连接池当中。

  **最大连接量（maxSize）**：当并发访问MySQL Server的请求增多时，初始连接量已经不够使用了，此 时会根据新的请求数量去创建更多的连接给应用去使用，但是新创建的连接数量上限是maxSize，不能 无限制的创建连接，因为每一个连接都会占用一个socket资源，一般连接池和服务器程序是部署在一台 主机上的，如果连接池占用过多的socket资源，那么服务器就不能接收太多的客户端请求了。当这些>  连 接使用完成后，再次归还到连接池当中来维护。

  **最大空闲时间（maxIdleTime）**：当访问MySQL的并发请求多了以后，连接池里面的连接数量会动态 增加，上限是maxSize个，当这些连接用完再次归还到连接池当中。如果在指定的maxIdleTime里面， 这些新增加的连接都没有被再次使用过，那么新增加的这些连接资源就要被回收掉，只需要保持初始连 接量initSize个连接就可以了。

  **连接超时时间（connectionTimeout）**：当MySQL的并发请求量过大，连接池中的连接数量已经到达 maxSize了，而此时没有空闲的连接可供使用，那么此时应用从连接池获取连接无法成功，它通过阻塞 的方式获取连接的时间如果超过connectionTimeout时间，那么获取连接失败，无法访问数据库。 该项目主要实现上述的连接池四大功能，其余连接池更多的扩展功能，可以自行实现。

  > **上述所有配置均可在src/dbconnection目录下的mysql.cnf文件中进行修改**

  # 功能实现设计

  ConnectionPool.cpp和ConnectionPool.h：连接池代码实现
  
  Connection.cpp和Connection.h：数据库操作代码、增删改查代码实现

  连接池主要包含了以下功能点：
  1.连接池只需要一个实例，所以ConnectionPool以单例模式进行设计
  2.从ConnectionPool中可以获取和MySQL的连接Connection
  3.空闲连接Connection全部维护在一个线程安全的Connection队列中，使用线程互斥锁保证队列的线 程安全
  4.如果Connection队列为空，还需要再获取连接，此时需要动态创建连接，上限数量是maxSize
  5.队列中空闲连接时间超过maxIdleTime的就要被释放掉，只保留初始的initSize个连接就可以了，这个 功能点肯定需要放在独立的线程中去做
  6.如果Connection队列为空，而此时连接的数量已达上限maxSize，那么等待connectionTimeout时间 如果还获取不到空闲的连接，那么获取连接失败，此处从Connection队列获取空闲连接，可以使用带 超时时间的mutex互斥锁来实现连接超时时间
  7.用户获取的连接用shared_ptr智能指针来管理，用lambda表达式定制连接释放的功能（不真正释放 连接，而是把连接归还到连接池中）  8.连接的生产和连接的消费采用生产者-消费者线程模型来设计，使用了线程间的同步通信机制条件变量 和互斥锁

  # 配置

  MySql版本:5.7
  
  操作系统版本:linux

  # 运行
  1.进入build文件夹下，执行以下代码

  =======
  # 配置
  MySql版本:5.7
  操作系统版本:linux

  =======
  # 配置
  MySql版本:5.7
  操作系统版本:linux

  # 编译运行
  1.进入build文件夹下，执行以下代码编译项目
  ```bash
  cmake .. && make
  ```

  2.进入bin文件夹下，执行

  ```bash
  ./connectionPool
  ```



  # 压力测试

  | 数据量 |         未使用连接池花费时间         |          使用连接池花费时间           |
  | :----: | :----------------------------------: | :-----------------------------------: |
  |  1000  | 单线程：10.3022s    四线程：2.48859s | 单线程：0.071227    四线程：0.569368s |
  |  5000  | 单线程：35.1183s   四线程：20.1149s  |  单线程：4.45264    四线程：4.81965s  |
  | 10000  | 单线程：60.1086s    四线程：40.711s  |  单线程：3.37099    四线程：1.72819s  |
