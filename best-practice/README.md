# 1. 变更

## 1.1. 遵循窗口

所有可能影响到生产 DB 的操作都遵循发布窗口。

即便是一个只有自己使用的内部工具，如果涉及到生产数据写操作也必须遵循发布窗口。

包括给别人加权限这种看似无风险的操作，也可能触发某个系统的 Bug 从而影响全局。

比如给别人加权限时，由于对方的用户名内有一个系统不能处理的特殊字符，加了权限后触发系统 Bug 导致全局瘫痪。


## 1.2. 充分测试

所有变更都必须在非生产环境测试通过后再同步到生产。

你的代码也许没有问题，但是很可能在构建时由于依赖更新导致发布的版本跑不起来。

不仅测试环境需要测试，在上线完成后还需要做生产环境的功能验证，有问题第一时间回滚。

大流量的业务除了基本的功能测试外还需要做压测后才可以在生产环境放量。

如果涉及到多活，还需要做多活验证。


## 1.3. 灰度操作

所有变更尽可能地灰度进行。

所谓的「变更」包括但不仅限于代码、开关、数据等。

比如代码发布必须先发单机房一台，观察稳定后在再逐步扩大范围直至全量。

也许你的变更只是一行文案这种看似没有任何风险的东西，但你的代码可能在发布阶段出问题。

比如修改文案后导致单个文件大小超出发布系统限制而发布失败，影响整个项目。


## 1.4. 不可逆确认

所有不可逆的操作都必须落地 SOP 文档，列出风险点，经过多重 Cross Review，最后严格遵循。

如果可以，应该尽可能地避免此类，让操作尽量可回滚。或者即便不可回滚也应该考虑设计挽救措施，比如要刷数据之前应该考虑备份数据库。

以上这些实在都做不到的场景，必须要 3 人以上混合的 Cross Review 确认整个方案没问题之后，遵循 SOP 操作。


# 2. 开发

## 2.1. 设置预警

所有资源使用要有「满」的概念，并且在达到一定阈值时发出预警。

系统资源如 CPU、内存等，本身有系统监控，只要设置好正确的阈值即可。

而资源的概念还包括一些逻辑上的，比如连接池、线程池、队列等。对于逻辑上的资源，应该人为地设置他们的使用上限，并且在资源使用达到一定阈值时发出预警，而不是等到系统资源被打爆。

特别要注意的是此处说的是预警而不是告警，不要等到资源真正枯竭后才发出死亡告警。


## 2.2. 设置超时

所有不可控的异步调用都要设置超时。

所谓「不可控的异步调用」是指你不知道这个调用的对方会不会因为故障而永远不返回结果，比如前端 HTTP 调用、后端 RPC 调用、DB/Redis 基础设施调用等。

程序不能无限制的等待，否则一旦某个依赖异常，即便它是一个非关键的部分，也可能导致整个服务被拖垮。

比如一个非关键的 RPC 调用，对方因为故障而无法响应，导致你的服务线程池被打爆，最终影响到同服务上的关键业务。


## 2.3. 防御空指针

所有非静态定义的数据访问都采取防御性措施。

比如接口响应的 JSON 内容很可能是一个 null，直接访问很可能会抛出空指针异常。

访问这类数据前应该先做类型校验，或者整个操作在 try-catch 中执行，确保异常能够被程序处理掉。

空指针可能导致整个调用堆栈被 throw，比如前端页面因为某个接口异常返回了 null，导致整个页面白屏。如果代码能够处理空指针，可能只是某个模块加载失败，不会导致整个页面级别的问题。


## 2.4. 防御环状结构

所有环状结构都要考虑最坏跳出情况。

有些循环（包括递归）在数据正常时可以工作，但是遇上脏数据就永远不会结束。

所以当你写出一个依赖于数据作为跳出条件的环状结构时，那就必须做最大次数限制，即便你认为数据非常安全也应该做防御。

比如遍历一个递归表，也许这个递归表的 insert 和 update 都是你的服务在操作，看似非常安全。但即便如此也可能因为其它人的 Bug 或误操作导致 parent_id 等于 id 的脏数据产生造成死循环。

即便不谈存储，内存里的数据都不一定是安全的，你不能确保另一个线程不会去改你的数据。即便你自己现在可以确保，那也不代表将来维护你代码的人不会这么干。

## 2.5. 考虑一致性

所有异步不安全的操作都考虑加锁或避免此类。

异步不安全包括多线程操作内存、多进程操作文件、多实例操作存储等。

当你写下任何一个函数，都要考虑这个函数是否会被两个协程或线程同时调用，是否会访问到同一块内存空间。如果涉及的是外部存储，还得考虑是否会在两台机器上被同时调用，影响数据的一致性。

比如多个线程同时操作一块内存可能造成致命错误。
又或者从 DB 中读出一个数据计算后再写回去，如果没有锁可能就会产生脏数据。

甚至是前端的一个按钮点击，如果点击后不加锁，可能会因为用户重复点击造成事件重复触发。

## 2.6. 考虑补偿

所有不可丢的消息订阅都要考虑补偿。

任何主动数据推送都是不可靠的，如果要确保消息不丢就必须有轮询机制作为补偿。

比如从消息队列监听消息时，要考虑消息队列服务异常的情况，增加一个轮询作为补偿。

又或者是基于 WebSocket 订阅服务器端的数据变更，那就得考虑 Socket 堵塞或者长连服务挂掉的情况，增加一个轮询作为补偿。
