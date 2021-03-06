## 消息推送系统 

> 案例分析

在`2020-02-30`日，运营同学圈选了一个`5000W`的人群选择在晚上8点发送一条短信，大致的情况就是告诉用户**文章更新了，不看血亏**。系统在`晚上8点`准时执行任务，读取该模板的模板信息下发。`5000W`人，系统能秒发吗？显然是不行的

所以，这`5000W`人肯定是需要一定的时间才能完全下发的，现在我们假设是`15分钟`完全下发完毕吧。在`8点2分`触发了一条验证码的短信，结果因为这个`5000W`的人群所导致验证码的消息延迟发送，这合理吗？显然不合理。

怎么导致的？原因是这`5000W`的消息和验证码的消息走的是同一个**通道**，导致验证码的消息被阻塞掉了。我们将不同的消息类型走不同的通道，就可以解决掉上面的问题。

小结：

- 我们把发送一条消息所**必要**的信息（文案、发送账号、传入的接收者Id类型、消息类型：通知、营销和验证码）、**平台性的信息**（业务规则：是否去重、屏蔽、展示逻辑等）和基本信息（业务方信息、消息名称）全都塞到模板中
- 由于使用场景，模板会分为运营模板和技术模板。运营模板主要的特点是需要填写人群信息和发送时间，运营模板由**消息管理平台自身**进行调度发送消息。

消息管理平台的系统架构链路图

![](https://gitee.com/wwxw/image/raw/master/blog/rO6ZW3iWYiLk.jpg) 

调用者调用我们的`send/batchSend`方法，会直接调用下游的API下发消息吗？**不会** 

直接调用下游的API下发消息风险太大了，接口`1W+QPS`都是很正常的事，所以我们接收到消息后只是做简单的**参数校验处理和信息补全**就把消息发到**消息队列**上。这样做的好处就是**接口接入层**十分轻量级，只要**Kafka抗得住，请求就没问题**。

![](https://gitee.com/wwxw/image/raw/master/blog/push/!gi*jpAUEqf8.jpg) 

发到消息队列时，会根据不同的消息类型发到不同的`topic`上，发送层监听`topic`进行消费就好了。架构大致如下：

![](https://gitee.com/wwxw/image/raw/master/blog/push/@sn1U01QVDpu.jpg) 

发送层消费`topic`后，会把消息放在**各自**的内存队列上，多个线程消费内存队列的消息来实现消息的下发。

可以看到的是：从接入层发到消息队列上我们就已经做了分`topic`来实现业务上的隔离，在消费时我们也是放到各自的内存队列中来进行消费。这就实现了：**不同渠道和同渠道的不同类型的消息都互不干扰**。

看到上面这张图，如果思考过的同学肯定会问：这要内存队列干啥啊？反正你在上层已经分了`topic`了，不用内存队列也可以实现你所讲的“业务隔离”啊。

也的确，这里使用内存队列的主要原因是为了提高**并发度**。提高了并发度，这意味着下发速度可以更快（在下发消息的过程中，最耗时的还是网络交互，像短信这种可以多开点线程进行消费）。

在前面所提到的业务规则就是在下发层这儿做的，包括夜间屏蔽、1小时去重和Id转换等

- 夜间屏蔽就是判断是否在晚上，如果勾选了夜间屏蔽并且在晚上，过滤掉就好了
- 1小时去重就是拿`userId+消息渠道`作为Key，看是否存在Redis上，假设存在，则过滤掉  *
- `id转换`这功能我们做成了个系统，这块我放在下面简单说一下吧，这就不在赘述了。

> 画外音：这种场景最好使用**Pipeline**来读写Redis

随后就是**适配**各个渠道的接口，调用`API`下发消息了，这块就跟你们单个的实现没什么大的区别了，调用个接口还能给你玩出花来？（代码风格会稍好一些，模板方法模式、责任链、生产者与消费者模式等在项目中都有对应的应用）

总结一下接口的实现：

1. 调用方调用接口时，接口不会**同步**直接调用下游的`API`发送消息，而是放入消息队列上（支持高并发）
2. 放入队列时，会根据不同渠道以及不同类型的消息进行分类，放到不同的topic（业务隔离）
3. 消费队列时，会在本地使用阻塞队列来提高并发度（加快消费的速度）

### 1.1 ID 转换

在前面也提到了，发不同类型的消息会需要有不同的`id`类型：微信类需要`openId`、短信需要手机号、push通知栏推送需要`did`。

在大多数情况下，一般调用者就传入`userId`给到我，我这边需要根据不同的消息类型对`userId`进行转换。

那在我们这边是怎么实现该系统的呢？主要的步骤和逻辑有以下：

1. 监听用户变更和微信公众号订阅/取关的`topic`，在`Flink`清洗出一个统一的数据模型，将清洗后的数据写到另一个的`topic`。
2. Id映射系统监听`Flink`清洗出的`topic`，实时写到数据源（这里我们用的是搜索引擎）

> 有没有想过一个问题，为什么要用一个Id映射系统去监听`Flink`洗出来的`topic`，而不是在`Flink`直接写到数据源呢？

其实通过Flink直接写到数据源也是完全没问题的，而封装了一个Id映射系统，就可以把这活做得更**细致**。

从描述可以发现的是：在上面只实现了**实时增量**。很多时候我们会担心增量存在问题，导致部分数据的不准确或者丢失，都会写一份全量，Id映射也是同样的。

那Id映射的全量是怎么做的呢？用户数据通过各种关联关系会在`Hive`形成一张表，而Id映射的全量就是基于这张`Hive`表来实现全量（每天**凌晨**会读取Hive表的信息，再写一遍数据源）。

基于上面这些逻辑，专门给Id映射做了个**后台管理**（可以手动触发全量、是否开启增量/全量、修改全量触发的时间）

![](https://gitee.com/wwxw/image/raw/master/blog/push/G6Dl^9**yU9d.jpg) 

### 1.2 数据统计

我觉得这块是消息管理平台**最最最精华**的一部分。

梦回我们当初的接口设计环节，我们就是因为有“数据统计”的需求，才引入了**模板**的概念。现在我们已经有了一个`模板Id`了，在我们这边是怎么实现数据的统计的呢？我们对消息的统计都是**基于模板的维度**来实现的。

在创建模板时就会有一个模板Id生成，基于这个模板Id，我们生成了一个叫做`umpId`的值：第一位分为技术/运营推送，最后八位是日期，中间六位是模板Id

![](https://gitee.com/wwxw/image/raw/master/blog/push/i3h7dzWPRylD.jpg) 

因为所有的消息都会经过**接入层**，只要消息带有链接，我们就会给链接后加上`umpid`参数，链接会一直下发**透传**，直至用户点击

每个系统在执行消息的时候都会可能导致这条消息发不出去（可能是消息去重了，可能是用户的手机号不正确，可能是用户太久没有登录了等等都有可能）。我们在这些『**关键位置**』都打上日志，方便我们去排查。

这些「关键位置」我们都给它**用简单的数字来命个名**。比如说：我们用「11」来代表这个用户没有绑定手机号，用「12」来代表这个用户10分钟前收到了一条一模一样的消息，用「13」来代表这个用户屏蔽了消息.....

「11」「12」「13」「14」「15」「16」这些就叫做「**点位**」，把这些点位在关键的位置中打上日志，这个就叫做「**埋点**」

有了埋点，我们要做的就是将这些**点位收集**起来，然后统一处理成我们的数据格式，输出到数据源中。

1. 收集日志
2. **清洗日志**
3. 输出到数据源

有logAgent帮我们收集日志到**Kafka**，实时清洗日志我们用的是**Flink**，清洗完我们输出到**Redis(实时)/Hive（离线）**。

Hive表的数据样例（主要用于离线报表统计）：

![](https://gitee.com/wwxw/image/raw/master/blog/push/*QupQMrcKZXm.jpg) 

Redis会以**多维度**来进行存储，以便支撑我们的业务需要。比如，要查一条消息为何发送失败，通过`userId`搜一下，直接完事（实时的都记录在Redis中，所以这里读取的是Redis的数据）

![](https://gitee.com/wwxw/image/raw/master/blog/push/4XkTeoLRN8rl.jpg)

比如，通过模板Id，查某条消息的整体下发情况：

![](https://gitee.com/wwxw/image/raw/master/blog/push/NmMwW*M3MXsm.jpg) 

为什么我说这是消息管理平台最最最精华的呢？**umpId贯穿了所有消息管理平台经过的系统**，只要是在消息管理平台发的消息，都会被记录下来发送，可以通过**点位**来**快速追踪**消息的下发情况。

![](https://gitee.com/wwxw/image/raw/master/blog/push/7kslqg8KMv8q.jpg) 

总结一下数据统计：

1. 设计出业务上的`umpid`，给所有的消息推送链接都加上`umpdId` 参数
2. 打通上下游，共同设计和维护关键点位，**统一日志格式**来实现跨平台的收集和清洗
3. 兼顾实时和离线需求写到不同的数据源，实时以多维度统计来快速定位问题

### 1.3 运营层面

> 运营的模板是需要圈选一批人群，然后下发消息的，那这群人从哪里来？

在很久之前，消息管理平台也把人群给做掉了，大致的思路就是可以支持`文件上传`和`hivesql`上传两种方式去圈选人群，圈出来上传到`hdfs`进行读取，支持对人群的**更新/切分/导出**等功能。

有了人群的概念，你会发现**你收到的消息其实都是跟你息息相关的**（不是瞎给你推送的，你在里面，才能圈到你）。可能是因为你看了几天的连衣裙，所以给你推送连衣裙的消息，吸引去你购买。

后来，由于公司内部`DMP`系统崛起，人群就都交由`DMP`给管理了。但实现的思路也都是类似的，只不过还是同样的：人家做的是平台，功能肯定比会自己写几个接口要完善不少。

做推送就免不了发错了消息，特别是在运营侧（分分钟就推送千万人），我们平台又做了什么措施去尽可能避免这种问题的发生呢？

在运营圈定人群后，我们会有单独的测试功能去「测试**单个**用户」是否能正常下发消息，文案链接是否存在问题。

这一个步骤是必须要做的，给用户发出的消息，首先要经过自己的校验。如果确认链接和文案都无问题后，则提交任务，**走工单审批**后才能发送。

针对于（技术方推送），我们在预发环境下配置了「**白名单**」才能收到消息。

线上消息有「去重」的逻辑：

- 在某段时间内，过滤掉重复消息
- 运营类消息推送（圈定人群的方式去下发消息）同一个用户需要相隔一段时间才能下发一次。

![](https://gitee.com/wwxw/image/raw/master/blog/push/YLsvuQ*MqR7k.jpg) 

> 相关文章

1. [消息管理平台的实现原理](https://mp.weixin.qq.com/s/4XcefSzHQewbZGQg34oeVA) 



##  消息推送管理平台 

「消息管理平台」可能在不同的公司会有不同的叫法，有的时候我会叫它「推送系统」，有的时候我会叫它「消息管理平台」，也有的同事叫它「触达平台」，甚至浮夸点我也可以叫它「消息中台」

但是不管怎么样，它的功能就是**给用户发消息**。在公司里它是怎么样的定位？只要**以官方名义**发送的消息，都走消息管理平台。

一般你注册一个`APP/网站`，你可以收到`该APP/网站`给你发什么消息呢？一般就以下吧？

- 站内信（IM）消息：其实就是**APP内**聊天的消息
- 通知栏（PUSH）消息：系统弹窗消息
- 邮件（Email）消息
- 短信（Sms）消息
- 微信服务号消息
- 微信小程序（服务通知）消息

### 推送的定义与价值

- **推送**：消息发送方将信息传递给接受者的行为。

结合到我们日常的场景，就是公司的运营同学或业务系统将营销消息或通知消息通过短信、push、微信等渠道发送给用户的行为。每天针对用户的推送消息可以引导用户参加活动、阅读资讯、查看账单等行为，是一块重要的流量入口，推送是推动业务目标的达成的重要手段。

> 那么问题来了，发消息困难吗？发消息复杂吗？

1. 发短信无非就是调用第三方短信的API
2. 发邮件无非就是调用邮件的API
3. 发微信类的消息（手Q/小程序/微信服务号）无非就是调用微信的API
4. 发通知栏消息（Push）无非就是调APNS/手机厂商的API、
5. 发IM消息也可以使用云服务，调云服务的API...

可能很多人的项目都是这么干的，**无非发条消息，自己实现也不是不可以**。

但这样会带来的问题就是在一个公司内部，会有很多个项目都会有「发送消息」的代码实现。假设发消息出了问题，还得去自己解决。

首先是系统不好维护，其次是没必要。**我一个搞广告的，虽然我要发消息，凭什么要我自己去实现**？

我们在写代码时，可能会把公用的代码抽成方法，供当前的项目重复调用。如果该公用的代码被多个项目使用，可能我们又会抽成**组件包**，供多个项目使用。只要该**公用的代码**被足够多的人去用，那它就很有可能从组件上升为一个**平台（系统**）级的东西。

- **本文的目的** 

  搭建一套较为完善的公司内部消息推送管理平台，对公司内部各业务线、产品线的消息推送进行统一管理，统一发送；这样既提高了公司的运营效率，又保证了用户体验。

本文的目的主要说明系统的产品设计思路，对于深入的短信、push、微信各渠道的发送机制说明在后续文章进行介绍。

### 推送系统流程

![img](http://image.woshipm.com/wp-files/2019/03/4V3w0cwAwuVBiRuhBCLt.jpg) 

一般来说，消息推送有2种发送方式：

1. 一种方式为运营活动批量定时投放，需提供系统功能方便运营筛选用户，然后编辑文案，经审核通过后进行发送。
2. 另一种是需要实时触发的消息，比如支付成功通知、验证码获取、满足某种条件触发的营销活动等消息，这类时效性要求较高且每个用户发送的消息内容中涉及到差异化的参数，需要业务应用实时触发。

触发的消息需要经过一定的过滤与拦截规则，针对于短期内已经覆盖过用户进行过滤，异常或者不合规的消息进行拦截，按照设定好的渠道进行推送。

### 数据准备

对于消息推送系统，需要获取投放的目标用户的账号数据，往往公司产品的customer ID和对应推送渠道的账号不一致，需要获取绑定关系，比如短信需要手机号，push需要SDK上报的token，微信需要使用OPEN ID，相关数据的采集在各个渠道的发送机制的文章里进行阐述。

![](http://image.woshipm.com/wp-files/2019/03/OvA8IbzDMPK9F0lUtpMi.png)

### 消息创建

#### 5.1 投放人群选择

日常的运营活动为了更加精准，提高活动转化率，运营同学会根据一些用户的特征进行筛选，比如北京地区用户，近3天内有登录过APP的用户等等，因此消息投放系统需与公司内部数据部门的标签系统进行对接，提供运营同学投放人群选择。 

接口实时触发的消息，一般需要**业务系统监控**到用户行为，将用户账号与需要的参数通过MQ或者接口传递至**消息推送系统**进行发送。

#### 5.2 消息类型与等级划分

消息的类型的应以消息内容的目的进行划分，大类可分为通知、营销、验证码等类型。

例如，短信行业内分为通知、营销、验证码类型的消息， 该类型的划分主要为方便路由短信至SP服务商不同通道，不同的通道触达率也不同，为了保证重要短信的触达率，需要将各个内容的短信路由至不同的通道发送

结合个人经验，公司内部可以根据实际情况进行更细粒度的划分，比如增加通知+营销类型，可能场景为用户支付成功后，在表述完用户支付成功信息后，结合适当场景增加领取优惠文案，引导用户向其他活动转化。

对于金融借贷类的机构，也可增加还款通知类型，主要为用户产生逾期行为需要提示还款的消息；原因为特殊期间，还款通知类短信可能会受特别的管制，单独出来可以进行较好的监控与处理。

对于通知类的消息，也应该按照等级进行划分，比如用户支付成功提示消息和优惠券到账通知消息，显然不应该是同一等级。支付消息涉及用户资金变动，通知等级较高；优惠券到账消息更偏营销类型，通知等级较低。为避免对用户产生更多干扰，需要分级进行控制，必要的时候降低等级较低的消息的推送频率。

#### 5.3 消息内容

对于通知类的消息，也应该按照等级进行划分，比如用户支付成功提示消息和优惠券到账通知消息，显然不应该是同一等级。支付消息涉及用户资金变动，通知等级较高；优惠券到账消息更偏营销类型，通知等级较低。为避免对用户产生更多干扰，需要分级进行控制，必要的时候降低等级较低的消息的推送频率。

在产品设计时，选择了对应的投放渠道后，应展示对应渠道所需的字段，且为必填项。

#### 5.4 消息跳转

消息触达到用户后，对于感兴趣的用户需要进一步了解信息，那么目前各类消息的载体不是有足够的空间来展示所有的信息，因此需要跳转到落地页进行详细信息获取。

短信类型的消息需要将长链转化成短链再进行发送，一是为了节省成本，因为短信是按照字符数进行收费的，二是为了用户体验，用户在手机上看到的不应该是一对长的乱码。

PUSH需要根据跳转的不同的页面设置不同的跳转类型，如H5页面和原生页面，跳转协议由客户端提供，消息系统只需要将其配置到系统上，运营同学可以选择就可以。

微信的消息内容一般模板消息条状到H5的活动页，图文消息跳转到文章详情，文本消息中也可以添加超链接，跳转到小程序。

#### 5.5  其它需记录信息

**消息发送部门**：此数据是用来作为后期短信费用结算的依据，按照消息发送部门扣减公司内部各业务线的费用，对于PUSH、微信消息等免费的资源，也可分析关系各个业务部门对消息资源的使用情况。

**转化行为口径**：消息点击后的一个环节一般是转化，为了更好地衡量消息发送的质量，应该记录下每条消息下发的目的，比如：订单、实名、激活、下载、通知等，将消息与转化行为匹配起来进行数据分析。

**产研负责人**：在消息发送之前应该记录好每个任务或模板，对应业务线的产品、研发实际消息的负责人，当消息发生客诉时，通过消息记录查询功能，便可迅速定位消息的产研负责人，紧急确认对应消息是否有异常并解决。

#### 5.6 推送时间设置

对于不同发送形式的消息，推送时间不同。创建的消息任务可以预定时间进行发送；对于已经固化下的营销场景，需设置周期性任务，设置初始执行时间与执行周期，降低运营操作成本。接口触发的时间一般为实时触发，触发时间由业务系统决定。

#### 5.7 在线测试

当消息任务设置好后，需要验证消息投放出去后展示的效果与相关跳转是否正常，避免造成线上推送事故。测试需要发送运营设置好的真实内容，推送对象为内部消息创建者。为避免出现消息误发，测试发送的文案前应添加“测试”，或设置测试白名单，不在白名单内的账号无法进行测试。

### 消息审核

当消息任务或者消息模板创建好，需要经过谨慎审核后才能发送，避免出现工作失误产生不良影响。

审核级别一般需要业务线内部负责人审核与公司平台或者对应职能部门审核。审核要点主要为：消息文案是否符合广告法、消息跳转是否正常、发送频率、时间是否合适等。

### 消息过滤与拦截

消息过滤主要针对营销类型消息，时段限制（早上9点至晚上8点之间可发送）、频率限制（用户7天内只能收到1条短信，针对于周期性任务，同一任务触达过的用户可以进一步扩大过滤周期）、黑名单限制（用户退订）。

消息拦截主要为限制发送量级，比如每个业务线针对同一用户每日最多发送5条短信；公司整体对同一个用户最多发送30条短信；短时间（时间可设置，如300S）内同一用户重复内容过滤；量级的控制只要为避免由于业务系统故障造成的对用户消息轰炸，产生不良影响。

关键词拦截，如包含违法、暴力等词汇。

不同的场景使用的过滤频率可做适当调整，比如用户对短信消息的容忍度比push的容忍度较低，因此短信频率应该更加严格。

### 消息发送

目前经过种种逻辑的处理，消息终于到了发送环节。发送环节主要后台逻辑，重点要优化消息发送的性能，提高消息发送的稳定性，避免业务损失。发送环节应该添加监控并且适当打印日志，以便及发现异常并定位问题。

### 消息路由

短信、安卓push均可接入多个渠道，搭建分发集群。可以根据业务业务逻辑指定通道发送，也可以根据下游通道状态自动路由。

### 数据分析

对于触达系统来说，数据分析一般按照消息的全流程进行分析，包括发送数量——触达数量——点击数量——转化数据。

![img](http://image.woshipm.com/wp-files/2019/03/4xYSRRjzB5OD2iYLcLrn.jpg)

如果涉及消息对APP进行导流，提高APP活跃，也许统计各消息为带来APP唤起次数。

对于短信来说，涉及到短信费用，需要针对渠道和成功触达条数进行计费，设计对账看板。

短信退订、PUSH关闭等等用户行为数据也需要进行分析，便于调整后续触达策略。

### 后台管理

#### 11.1 通道路由配置

对于短信类型的消息，涉及到签名与通道，不同的业务场景需要不同的短信签名，需要将某些账号、某些模板的消息路由至固定通道侧。以及系统需要根据下游通道性能或状态自动路由消息。

#### 11.2 消息发送记录查询

针对于近期发送出去的相关消息，需提供客服侧或运营侧一定的查询功能，以便用户来电咨询相关消息问题，比如未收到验证码消息、没有进行操作却收到消息等等情况。

#### 11.3 黑名单

黑名单功能主要应用于消息过滤，当用户投诉或退订后，避免再给用户发送消息，屏蔽的粒度需根据消息类型进行屏蔽，可适当根据内部业务划分。

#### 11.4 过滤与拦截规则配置

1. 需针对同一用户设置消息发送上限，避免由于业务系统异常导致对用户造成轰炸。
2. 重复内容拦截，需设置一定时间内，完全相同内容进行拦截，避免重复发送。
3. 关键词拦截，需针对于违规、违法的关键词进行拦截，避免出现运营事故。
4. 针对于营销消息，需根据不同的触达方式，控制触达频率，避免对用户造成干扰，反而让用户对品牌产生反感心理。

### 上行管理

上行管理主要应用于短信消息，用户回复退订或办理业务的关键词。由于从运营商到发送者的上行过程不能精确到用户回复的是哪条消息（也可能用户主动给某些号码发送短信），为了保证各场景不互相影响，需控制上行关键词唯一。

![](http://image.woshipm.com/wp-files/2019/03/nqiqOJlQlnTA1IqizILE.jpg) 

 

## APP PUSH 推送机制 

### 1.1 APP PUSH定义与价值

APP PUSH的定义为在手机终端锁屏状态下通知栏展示或在操作前台顶端弹出的消息通知，点击后可唤起对应的APP，并在APP内跳转到指定页面。

push消息是通知用户，引导用户进行参与活动、购买产品的重要手段，而且PUSH消息也可以引导用户查看消息，唤起APP提高日活，是一块重要的流量。

### 1.2 APP 推送分类

从应用的功能来划分，主要分为三类应用

- 第一类是IM类APP，如微信、QQ等
- 第二类是新闻资讯类，如华尔街见闻等；
- 其余暂归为为工具类，比如支付宝、美团等。

每种类型APP对PUSH的需求也不同

- IM类APP追求实时、稳定的触达，此类APP一般通过自己的长连接进行消息推送，保证用户在收到消息的时候能够实时地接收消息消息。另外，一些安卓厂商也会给予头部APP的进程一定保护，对相关的进程纳入白名单，在清理进程的时候予以忽略。
- 新闻资讯类的APP与工具类APP的PUSH推送机制基本一致，仅在频率控制上有差异，新闻资讯类由于新闻资讯较多，需要将突发新闻及时推送给用户。

由于目前工具类的APP占大多数，本文将主要讲解工具类APP的常见推送机制。

### 1.3 PUSH 流程

![img](http://image.woshipm.com/wp-files/2019/03/ZquIfFEMR6lkTudZt474.jpg) 

PUSH消息在消息系统创建好后进入发送阶段，服务端需要根据用户终端信息进行路由，如果是IOS系统，那么会调用苹果自身的推送通知服务（APNs）,如果用户的手机是安卓系统，那么根据不同的厂商去调用不同的厂商SDK。

对于不同的系统版本，支持的消息展示形式也是不同，比如IOS10之后，当APP在前台时，是否通知栏展示；此样式可以根据产品需求来选择，有服务端传输相应通知方式的值即可。如果用户的手机非五大厂商内的手机，可以通过自己搭建的长连接或者使用第三方服务进行推送。

如果不是自己直接对接厂商通道，那么内部的服务端可能无需做过多较为复杂繁琐的开发工作，通过接入第三方消息推送平台来实现消息的推送，比如信鸽、个推等。多数的通道会将消息是否成功推送到客户端SDK的回执数据反馈给发送方，需要提供回调地址。

### 1.4 底层通道说明

#### （1）推送通道

> 通道类型一般分为三类：厂商通道、第三方推送服务平台、长连接。

- 厂商通道是手机终端厂商推出的推送服务，通过接入厂商SDK，内部服务端可以将消息推送到手机系统的服务端,再下发至客户端内部的厂商SDK，由操作系统进行相应展示，点击后唤起相应APP，这样可以避免APP进程被杀死后消息无法触达用户，因此触达率较高。
- 第三方推送平台是推送服务公司自己搭建相关的消息服务。并且各个APP使用了同一个平台的推送服务时，客户端都是集成同一个第三方推送平台的SDK，因此形成了一个推送联盟，当联盟中的其中一个APP的消息进程没有被杀死的时候，其他的APP也可以利用进行通知用户，形成了相互唤起，提高触达率。

经过一些场景的测试，相互唤起的成功率并不是很高，需谨慎结合自身场景评估。为了提高触达率，第三方推送平台也会集成各大厂商的SDK进行推送。

- 长连接就是建立手机与服务端的一条链路进行消息数据推送，通过长连接也可以进行APP状态监控，但完全由长连接推送且保证触达的稳定，需要投入的研发资源较多,且需尽量避免自己的长连接进程不要被操作系统杀死。

#### （2）优劣势对比

![](http://image.woshipm.com/wp-files/2019/03/Q8hMeNVOQYVcXwyUR9LU.jpg) 

APP push功能的搭建需要依据产品自身的情况和公司可投入的资源成本为主，在不同的阶段应该追逐不同的目标。

### 1.5 下发推送

#### （1）推送账号

推送时客户端的PUSH SDK均会根据用户的设备号生成一个对应关系的TOKEN。

在SDK内部，如果使用的是第三方推送服务，则去第三方的SDK注册；如果是厂商，则去商城SDK注册；如果使用自己长连接，则去自己的SDK进行注册，作为后续推送的标识用户的唯一ID。

#### （2）消息路由

消息路主要见上述推送流程的讲解，此处主要讲解根据不同的业务场景，可能会定向推送给不同版本APP的用户。因此服务端在通道能力路由的时候，不仅需要能够区分通道，还要进一步能够针对用户的手机终端进行更加精细化的差异推送。

此外，消息通道并一定是100%稳定，如果下游通道出现问题，服务端需能够将由于通道问题导致的消息路由到备用通道去发送，以保证业务稳定触达。

#### （3）全量推送

一般来说，对于公司内部运营或公司的相关数据均是以产品的customer id为准，用户数据系统对接消息系统时也多为customer id，因此需建立customer id与推送TOKEN的关系，便于运营针对用户进行推送。但对于一些场景会需要针对未登录的用户也进行推送，即全量推送；比如突发重大新闻资讯、大促等活动，所以运营系统需要提供全量推送功能，针对所有TOKEN进行推送

### 1.6 数据上报

上报数据包括触达 点击 关闭 退出 注册等数据。

对于所有方式的触达消息，都离不开触达与点击，触达的数据通过厂商的需要厂商回调上报，点击数据可以由SDK上报服务端。

对于push的关闭，也是需要进行考量的，来评估push是否过度发送，打扰到了用户。关闭数据有两部分，一部分为app内部的关闭，sdk直接上报给服务端即可；另一部分为用户在手机操作系统上关闭了对应app的push，需要APP在前台时，sdk调用手机终端相关方法获取该用户是否关闭了系统通知，然后上报至服务端。

注册数据即用户首次启动APP时，去相关sdk注册token。

用户退出账号时，sdk需要上报服务端，解除token与customer id的绑定关系。

### 1.7 PUSH 特点

#### （1）强提醒，不留痕

push由于是app自己的通知渠道，是运营的一个重要工具。

如果用户未关闭PUSH通知的话，push可以从通知栏弹出进行消息显示，具有一定的强提醒性，但PUSH点击跳转后便消失，没有痕迹，因此针对于重点的通知消息，需要在APP内设置消息中心，在PUSH的同时留下通知记录。

#### （2）消息样式

对于各家PUSH来说，一些营销消息会加入EMOJI表情来吸引用户点击，这也是一个吸引用户点击的一个小方法，只要服务支持传输约定好的EMOJI码就可以了。

目前安卓系统也支持富媒体推送，推送包含图片、语音等形式，对于资讯类的APP可以增加缩略图，吸引用户点击。目前来看，语音场景还有点挖掘。

#### （3）IOS和安卓

由于APP是基于手机操作系统，因此对于IOS和安卓的推送的流程及功能基本相同，只不过细节和方法上略有不同，且国内安卓产商都在安卓系统上进行了一定改造，导致国内安卓厂商标准各不相同，需要开发同学仔细对接各个厂商。

### 1.8 触达率提升

触达率的提升需要从消息创建到实际通知到用户的建立完整流程，细化每一个交互环节，发现影响触达率的主要瓶颈，并针对性地进行解决或优化方案。

除此之外，未采用厂商通道的消息也可以采用自己的长连接和其他推送平台服务同时多条推送，在客户端的SDK内增加针对同一罅隙流水号的去重，这样可以也可以提高一部分消息的触达率。



> 相关阅读

1. [一文带你彻底了解APP PUSH推送机制](http://www.woshipm.com/pd/2076068.html) 
2. [从0到1搭建消息推送管理平台](http://www.woshipm.com/pd/2052815.html) 
3. [设计一个百万级消息推送系统](http://www.52im.net/thread-2096-1-1.html) 

























































