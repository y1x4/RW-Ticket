# RW-Ticket

RW-Ticket 是一个基于Java、SSM框架，采用前后端分离技术，通过使用Redis进行多种缓存、RabbitMQ异步下单，并实现了接口隐藏、验证码、接口限流等功能的校园票务系统，用于购买一些限量、优惠的讲座、演出等票务。

- 完成基本功能（登录、浏览、下单）后，还使用不同层级和粒度的缓存对系统进行了优化，构造5000个虚拟用户、使用JMeter进行压力测试，结果显示QPS提升到之前的近2倍
- 通过预减库存减少透穿到DB的请求，通过异步处理和排队机制缓解数据库的压力，降低应用接口负载
- 实现接口隐藏、设置验证码、接口限流防刷等功能，提升了安全性，同时也减轻了系统压力

![票务列表页面](/resources/list.png)
![票务详情页面](/resources/detail.png)


### 一、如何启动项目

1. `git clone`或直接下载到本地  
2. 导入到`Intellij IDEA`，加载`pom.xml`中的依赖  
3. 在`MySQL`中新建`miaosha`库，进入库中执行`miaosha.sql`  
4. 部署和启动`Redis`和`RabbitMQ`  
5. 将`MySQL`、`Redis`和`RabbitMQ`的账号密码等信息在`src/main/resources/application.properties`中进行修改配置  
6. 运行`src/main/com.rwticket.miaosha.MainApplication`，访问`http://localhost:8080/login/to_login`，使用账号（13366061234，123456）进行登录即可进行商品浏览和秒杀。  
7. 可以在`miaosha_goods`表中修改秒杀时间段`start_time`和`end_time`。

PS: Redis 修改配置
```$conf
# 允许所有计算机访问
bind 0.0.0.0
# 允许后台执行
daemonize yes
# 设置客户端访问需要密码
requirepass admin
```

### 二、项目迭代

项目的核心是一个秒杀系统，参考了[这门慕课网课程](https://coding.imooc.com/class/chapter/168.html)，利用缓存、异步来应对大并发。

##### 一、项目搭建
Result 类对返回的JSON结果进行封装,包含 int code、String msg、T data。
集成 Thymeleaf 前端模板；集成 MyBatis、Druid，可以在业务层用@Transactional注解进行事务管理。
安装、设置、集成 Redis，使用时从 JedisPool 连接池（单机版）中获取 Jedis对象，使用完放回。存取User对象到Redis时使用了 fastjson 进行 String 与 Bean 之间的解析。

Redis设置：
	bind 0.0.0.0 允许所有计算机访问
	daemonize yes	允许后台执行
	设置了客户端访问需要密码：admin

	cd /usr/local/redis-4.0.10
	启动服务端： ./redis-server ./redis.conf
	关闭redis： redis-cli -h 0.0.0.0 -p 6379 shutdown
	启动客户端： redis-cli
	查看运行状态： ps -ef | grep redis

为避免key重复，**通用缓存key封装**：KeyPrefix接口 -> BasePrefix抽象类 -> 实现类
	KeyPrefix： int expireSeconds()、String getPrefix();
	BasePrefix： expireSeconds 默认为0，getPrefix = className + ":" + prefix

##### 二、登录功能
/login/to_login -> login.html，前端对输入的账号密码进行简单的校验，然后 formPass = MD5(密码+固定salt) 后 POST 提交到 /login/do_login ，登录成功则跳转至 /goods/to_list 商品列表页。若存在手机号，且 dbPass == MD5(formPass+个人salt) 则验证成功，生成新的cookie放入response（若已有该用户的，相当于延长过期时间）。
两次 MD5 加密，第一次防止明文密码在网络上传输，第二次防止数据库被盗后将一次加密反向破解。

bootstrap 画页面、jQuery-validation做表单验证、js做弹框

JSR303参数校验
	在doLogin方法的LoginVo参数入参前加上 @Valid注解，LoginVo中的变量加 @NotNull、@Length 等注解，@IsMobile 需要自定义接口+实现类。

全局异常处理
	输入格式错误等异常信息，需要捕获才可以正常提示。新建 GlobalExceptionHandler 拦截所有的异常，进行GlobalException、BindException、其他异常的处理，返回 Result。

分布式session:
	不能简单的用服务器的session，因为有多台服务器，同步很麻烦；生成token和cookie，用户相关信息写入分布式缓存Redis中，客户端拿着token表示身份。

	GoodsController 中每个方法里都要获取user，重复；可以实现一个 ArgumentResolver ，通过传入的token或cookie里查找到的token获取 user 对象，直接在参数里将其注入。

##### 三、秒杀功能
miaosha_user、goods、miaosha_goods、miaosha_order、order_info 几张表。
商品列表页进入详情页，如果未登录（没有user），会提示登录，点击秒杀会跳转到登录页面。根据后端时间计算传入的状态码，有秒杀未开始（倒计时，不断回调 countdown()）、秒杀进行中（按钮可使用）、秒杀已结束三种情况。
秒杀：从数据库中读取判断商品库存、判断是否已有秒杀订单（user_id+goods_id），然后进行秒杀步骤（事务操作）：减库存、创建普通订单、创建秒杀订单。成功进入订单详情页，失败进入秒杀失败页面。

##### 四、JMeter 压力测试
在 0s 内启动（并发） 1000 个线程用设置的 HTTP请求访问指定路径，聚合报告页面中的Throughput表示吞吐量，可以简单的理解为QPS（Queries-per-second）。终端使用top命令监控CPU，Load Avg表示系统平均负载，增大线程数再运行可以看到负载增大。瓶颈在于数据库。
测试 /user/info 根据cookie获取用户信息，可以看到没了数据库的瓶颈，吞吐量大了很多。
还可导入文件配置，模拟多个用户的测试。

5000 个线程 * 10 次，QPS：700.5，Load Avg 17-22左右，CPU占用率最高的是 Java（JMeter是Java程序）、MySQL、kernel_task。
人为创建 5000 个用户和其登录cookie、token，进行秒杀，QPS：558。卖超了！

命令行测试：
	sh jmeter -n -t xxx.jmx -l result.jtl
	sz result.jtl，然后可以在聚合报告里打开

Redis压测工具：redis-benchmark

还讲了怎么打war包，用于放到Tomcat里运行。

一般说并发多少的时候QPS（Queries-per-second）是多少，还有TPS


##### 五、页面级高并发秒杀优化（Redis缓存+静态化分离）
页面缓存、URL缓存：适用于变化不大的页面，列表页、详情页，先从缓存里查找，若有则返回，若无则从数据库里查询、存入缓存、返回。过期时间很快，60s。
对象缓存：token - user 对象。改密码讲解了更新缓存的步骤：获取用户、更新数据库、更新缓存。

页面静态化（前后端分离）：
	之前需要通过 Thymeleaf 从后端获取数据、渲染成页面（或者从缓存中获取页面）；现在是 HTML+ajax，HTML可以缓存在客户端（浏览器）中，只需要从后端获取动态的数据。
	商品详情页、秒杀页
304、last_modified，与服务端交互后直接取浏览器缓存；进行静态资源配置（资源路径、缓存时间等）后直接取缓存。

GET、POST区别：GET是幂等的，无论请求多少次、结果一致、不会对服务端产生改变。

防止秒杀重复：创建唯一联合索引 ALTER TABLE `miaosha_order` 
ADD unique INDEX `u_uid_gid` USING BTREE (`user_id`, `goods_id`); 这样就不能生成重复的订单了。一点小优化：生成订单后存入缓存，判定秒杀重复时查缓存。

防止超卖：加唯一索引、SQL库存数量判断

静态资源优化：JS/CSS压缩（去除空白字符、注释等），减少流量；多个JS/CSS组合，减少连接数；CDN（内容分发网络）就近访问
	tengine.taobao.org

发起请求，从浏览器开始，可以页面静态化，将页面缓存在用户端；可以部署一些CDN，请求就近访问；Nginx里还可以加缓存；页面缓存，细粒度的用户缓存：通过一层层的缓存削减到达数据库的访问量，减轻压力。会有一些数据不一致的问题，需要平衡。

to_list: QPS 1935；do_miaosha: QPS 1510，快了不少，MySQL压力还是比Redis大。


##### 六、服务级高并发秒杀优化（RabbitMQ+接口优化）
思路：减少数据库访问
	1、系统初始化，把商品库存数量加载到Redis；
	2、收到请求，（内存标记：map记录无库存商品，商品少，返回结果快，减少redis访问），Redis预减库存，库存不足，直接返回，否则进入3；
	3、请求入队，立即返回排队中；
	4、请求出队，生成订单，减少库存；
	5、客户端轮询，是否秒杀成功。
更快响应用户请求，显示排队中而不是堵塞。

brew install rabbitmq 安装
cd /usr/local/sbin
./rabbitmq-server 启动，在 http://localhost:15672 可以看到管理页面
MQ安装路径：/usr/local/Cellar/rabbitmq/3.8.3

Direct模式：指定queue名直接发送消息
Topic模式：模糊匹配的广播模式，queue绑定一些topic
Fanout模式：广播模式，需要生产者消费者绑定相同的Exchange
Header模式：根据headers中的信息进行匹配

cd /Users/yixu/Downloads/apache-jmeter-3.3/bin 
sh jmeter
do_miaosha: QPS 1926，Java、redis排前面，MySQL开始时出现了，后来就下降、消失了。

Nginx横向扩展：通过配置**反向代理**到多台服务器，**负载均衡**。Nginx也可以缓存。
Nginx前还可配置LVS（Linux虚拟服务器）。

##### 七、安全优化
一、秒杀接口地址隐藏 
思路：秒杀开始之前，先去请求接口获取秒杀地址（即生成特定 path，并存入缓存）
1、接口改造，带上PathVariable参数
2、添加生成地址的接口
3、秒杀收到请求，先验证PathVariable

二、数学公式验证码
思路：点击秒杀之前，先输入验证码，分散用户的请求
1、添加生成验证码的接口
2、在获取秒杀路径的时候，验证验证码
3、ScriptEngine使用（计算执行JS代码 eval()，用来计算数学公式字符串）

三、接口限流防刷
思路：对接口做限流
缓存一个用户x秒钟内的访问次数，超过限制则返回登陆失败，过期时间x秒钟，之后重新计数。
不同接口限制次数不同，改进：@AccessLimit，通过拦截器拦截方法上的注解，进行限流操作。同时拦截了用户信息，存入ThreadLocal，UserArgumentResolver里直接获取就行了。


##### 八、Tomcat服务端优化（Tomcat/Ngnix/LVS/Keepalived）

to be continued...




