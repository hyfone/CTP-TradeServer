# CTP多账户多策略-交易程序

<p data-anchor-id="d480"><code>C++</code> <code>ctp接口</code> <code>程序化交易</code> <code>经验分享</code></p>

---

## CTP简介

> 综合交易平台CTP（Comprehensive Transaction Platform）是由上海期货信息技术有限公司（上海期货交易所的全资子公司）开发的期货交易平台，CTP平台以“新一代交易所系统”的核心技术为基础，稳定、高速、开放式接口，适合程序化交易软件运用和短线炒单客户使用。

[ctp接口下载地址](http://www.sfit.com.cn/5_2_DocumentDown.htm)

## 本文目的

该程序是我大二暑假参加一个金融软件比赛写的，这是比赛作品的其中一部分，**专门用来进行交易的**。作品建立在多账户、多策略的基础上，其中策略算法分析用别的语言编写，然后发送交易指令（即算法根据实时行情发出的买卖信号，其中包含：交易账户、交易合约、交易）给交易主机，交易主机的主要职责是接收、解析并执行交易指令，跟踪汇报指令的交易情况。具体情况可以看我我上传的介绍视频，那是后来提交作品时录的。

比赛后有三个人来找我要这个程序，其中有位老师想在实盘中测试下自己的交易策略怎样，就找了两位师弟给他做那个东西，然后让我去给他们讲要注意些什么东西，这让我想起自己一开始接触ctp的接口时，花了不少时间去测试接口看是怎么一回事。**鉴于网上ctp的开发介绍不多，我就分享下经验，让大家能少走些弯路就尽量少走一些。**

**有以下地方需要注意**：
> * 开发的环境是VS2013 + Qt5.3
> * 一定要看**“开发资料”**中的内容，特别是ppt。我只是分享些开发的经验，不是起步教程，但结合资料里的看就足够起步了
> * 在初步理解概念后试着自己写一个登录发请求的例子，试着调用不同的API函数，这些可以参考**noQtCTP**压缩包里的内容（结合文档中的示例）。这是我开始接触ctp接口时为了理解写的一些代码（就是登录、调用简单的API），不需要Qt库也可以编译
> * 当需要测试交易API时，可以参考**tdspiTestWithQt**压缩包里的内容，这是我学交易API操作和研究回调函数时用得最多的工具了！需要Qt进行编译，可以不断修改里面的源代码然后点击界面中的按键来执行发送指令，控制台输出回调信息，大家也可以自己写一个适合自己的研究API的小工具，既加深理解，又方便自己开发。基本上会写这个小工具ctp接口就已经学会怎么用了，剩下的就是想怎么把它应用到软件需求中了
> * 虽然是程序支持“多账户多策略”，但是对于ctp接口的使用是一样的，这篇文章主要说的是关于ctp的
> * 该程序遵循[BSD协议](http://www.linfo.org/bsdlicense.html)，即可以自由的使用，修改源代码，也可以将修改后的代码作为开源或者专有软件再发布

## 基础

> 理解CTP的基本用法

1. windows的接口文件咋一看比较多，其实是有规律的（很多东西名字很长，这是因为每个前面都添加了ThostFtdc这几个字，去掉了再读就是变量or文件的含义）

    **ThostFtdcUserApiDataType.h** : 定义业务数据类型，用typedef为现有类型创建同义字，比如期货账号typedef char TThostFtdcUserIDType[16]，密码类型typedef char TThostFtdcPasswordType[41]
    
    **ThostFtdcUserApiStruct.h** ： 定义业务数据结构，使用ThostFtdcUserApiDataType.h的数据类型，调用api时就要传这里面定义好的数据结构过去，比如执行登录操作时传一个CThostFtdcReqUserLoginField数据结构过去，这里面就放置了上面说的账号类型和密码类型
    
    **ThostFtdcMdApi.h、 thostmduserapi.lib、 thostmduserapi.dll** : 用于获取行情的API，关键字是md，表示market data
    
    **ThostFtdcTraderApi.h、 thosttraderapi.lib、thosttraderapi.dll**  ：用于交易操作的API，关键字是trader
    
2. 开发前要有期货交易账号，经纪商代码，前置机地址（前置机地址又分行情和交易的），这些又分为模拟的真实的，可以自己去开一个户，然后和期货公司要一些模拟的账号（记得问前置机地址和经纪商代码）。如果需要真实环境的前置机地址，可以下载一个快期交易软件，在登陆界面点击代理可以分别看到行情和交易的地址。

    大家可以加一些ctp开发的讨论群，里面有各期货公司的人提供模拟账户的，当然这不是免费的午餐咯，当需要提供手机号码时大家最好不要报真实号码，否则你懂的。另外这些模拟账户是大家都可以用到的，导致一些回调你不知道是不是自己的操作，因此大家可以在早上登录账号并调用API把密码改了，那样那个号那天就是你的了。我的tdspiTestWithQt里有这样的功能（大把号码根本没人用的，不用担心影响到别人）
    
3. **无论是行情还是交易API，里面都有两个类，一个是xxxSpi，另一个是xxxApi，分别代表着回调和调用**。Api的函数是**主动**调用的，这些函数的参数要你填，交易所收到你的请求后会把信息返回给你——通过回调Spi的函数，Spi是**被**调用的。默认Spi是什么也不做的，因此你可能需要继承Spi并重载需要的回调函数。回调函数里的参数就是交易所返回的信息，不是需要你填的东西，可以直接输出。每个Api都需要绑定一个Spi，这样才知道请求出去的东西怎么返回回来。

    大家可以仔细看下noQtCTP的代码，我写的TdSpi类就是继承CThostFtdcTraderSpi，而MdSpi继承CThostFtdcMdSpi，它们的实现都是重载回调函数输出需要的信息。我里面写的调用就像是链式的反应，首先主动调用api的Init()函数，Spi的OnFrontConnected()就被调用，在里面我又调用api的ReqUserLogin()函数，Spi的OnRspUserLogin()就会被调用告诉我登录结果等等等。一个主动一个被动，一环扣一环，只是方便观察和理解而已，实际生产肯定不是这样写。
    
    **调用Api的函数时，传的那些结构体中，不是每个参数都要填的**，比如请求登录只需要填名、密码和经纪商代码，那个结构体别的域空着就好了。
    
4. **交易API的函数远远多于行情API，但是实际要使用的其实只是其中的几个！**
    
    可以想象我们做程序化交易无非需要的功能就是登录、查询资金、下单、撤单这些功能，而回调函数就是围绕这几个展开的。

    首先你需要对照着看我之前提到的PPT里的介绍，里面都提到了我说的那几个需要的函数；然后你可以看我**用到了那些，传的参数里面填了什么**，别忘了参数的含义在CTP的两个业务数据定义的头文件中；接着看下我实现了哪些Spi回调函数，这些Spi函数又分别对应哪些Api函数，这样你就理解全部了。
    
    我指的代码是：
    1. 项目**tdspiTestWithQt**中的"MainWindow.h"及实现、"MySpi.h"及其实现（代码简单实用）
    2. 项目**TradeServer**中的"Trader.h"及实现（ctp交易API在交易机中的应用）
    <br/>
    我代码中有简单足够的注释，你需要做的就是对照ppt教程和别的参考资料看下我怎么用的，特别是传参，有哪些是需要填的，分别又代表了什么。先运行我的代码，然后改我的代码，接着自己写一个，就达到目的了。
    
    如果我在这里再一个一个解释函数就显得很累赘了。

5. 在网上的一些基础教程里，以及我的noQtCTP项目中，都会调用api的Join()函数，这个是为了防止主线程运行完直接结束程序用的，在Qt开发中并不需要。Join的意思是让另一个线程(比如a)插入到当前线程(比如b)直到a运行结束才返回让b运行。

6. 行情API是不需要用户名登陆的，也就是说登录字段中全部用户代码、密码、经纪商全部传空就好了
    
## 进阶
> 项目中需要注意的地方

1. Spi创建时都会新建一个线程，Spi被总是在它的那个线程中被调用。有时候你从主线程调用Api发出的请求a，而请求b需要在a成功之后才能执行，那你只能等待Spi返回的结果A(对应a)，然后从A中判断是不是成功再发出请求b，此时你又是在Spi所在的线程使用Api了。如果业务需求中存在些共享变量，那样就要注意**线程安全**了

2. Api创建时要求你告诉它放置***.con文件**的位置，这些文件是CTP跟踪记录你当前运行情况用的，可以不理。但如果做多账户的开发，那样就要注意为每个账户的交易Api创建单独的文件夹，一面互相覆盖

3. **ctp的Api还有一个要求，它限制每秒只能发送一个查询**，比如一秒就不可以连着下单了。这个问题我在Tradeser中的解决方法是给每个API开一条新的线程，使用命令模式把一条请求封装到一条命令中，有一个命令队列，在线程中每隔1秒检查命令队列有无命令，有就发送。这样在外部看来指令都直接执行了，不需要关心限制。可参考**TradeServer**中的CommandQueue.h及其实现，以及各个*Command.h及其实现。

4. 报单中有几个重要的状态标志：
    * systemID
        报单的系统编号，由交易所返回给你，对于一天的一次交易它是唯一性的
    * orderRef(erence)
        报单的引用，由我们填了交上去，这个报单的后续情况（如提交成功、交易了多少手等等）回报时都会带上这个数字，所以我们应该保持它的唯一性，最简单的办法就是每个交易账户每天从1开始随着提交报单来递增它
    * sequenceID
        报单的顺序编号，对于同一个orderRef的报单，它后面发生的情况的顺序编号会大于前面发生的事的编号，由交易所返回给你
    * orderStatus
        报单的状态编号，由交易所返回，它有好几个值，我所遇到的全部情况有：
    THOST_FTDC_OST_AllTraded **'0'** 全部成交，报单提交上去后全部交易成功就返回该状态  
    THOST_FTDC_OST_PartTradedQueueing **'1'**、THOST_FTDC_OST_NoTradeQueueing **'3'**表示报单提交成功了，但还在交易状态，没有完成  
    THOST_FTDC_OST_Canceled **'5'**撤单，当你发出撤单请求时，它会返回相应的带有该标志的报单给你；或者错单，表示该单错误  
    THOST_FTDC_OST_Unknown **'a'**。表示Thost已经接受用户 的委托指令，还没有转发到交易所，一般这个状态的时间极短，可以忽略  
    **总的来说有5个状态会遇见**，只需要关注这些情况就好了，a,1,3,5,0；在CTP的定义中还有别的状态，但我一直没遇到过  

5. **报单回报会遇到的全部情况**：
    报单回报首先要分本地和远程。
    * 本地的回报就是调用那个发报单的函数的返回值，返回值-2表示“未处理请求超过许可数”， -3表示“每秒发送请求数超过许可数”，而0表示成功。一般而言极少失败的，起码我做好1秒限制的措施后没试过失败。
    *  重点是远程的回报
        所有回报中分为有systemID和没systemID的，有表示这个单交易所已经接收到并认为是合规的，已经摆上去交易了，以后你只需要根据这个systemID和你经纪商的代码就可以准确查到报单的信息；没有表示为一个错单，交易所不接收
        从状态来详细分类的话，遇到a状态的报单可以什么事都不做；遇到1、3状态的报单可以表明在等待，可以根据sequenceID看是否要刷新本地信息；遇到5状态的报单看有无systemID，如果有则为撤单，否则是错单；0表示该报单交易完成，本地如果没记录过这个信息就可以记录下来然后计算费用。
    我们自己试验时常见的情况有  
    错单： a——>5  
    等待： a——>3  
    瞬间成功: a——>a——>0  
    等待一会成功： a——>3——>3——>0  
    总之，任何一个报单，只要你产生了它并提交了出去，那保单的状态最后都会归到0或者5的，表示成功、撤单或错单。

6. 报单引用的作用
    乍一看报单引用似乎没什么作用，但从上面可以看到，每个报单不一定有系统编号，但是只要我们赋给它们独一的引用编号，那对我们而言每个报单可以视为独一无二的，可以跟踪到。而相同报单引用的情况下，顺序编号则可以让我们跟踪到保单的信息。
    想象下对于交易所而言某账户的报单都是一样的，但对于本地而言，每个报单可能由不同的策略发出，这样区别哪个报单对应哪个策略就需要我们自己动手了，没错，报单引用的作用就在于此。
    但有时候一个策略发出的一个交易指令，需要几个报单来完成，这时候我们就要自己为交易指令编一个独一无二的号了。这里说远了，但对于多策略而言这是必须思考的问题。

7. **过滤**报单回报
    每次登录帐号或者短线重连，CTP那端都会把之前的报单状态发过来，这样就很有必要过滤那些已经处理过的报单状态了。程序的数据库中应该存有已经处理的报单编号集合，内存中应该有需要处理的报单编号集合，每次初始化程序需要初始化内存中这个集合。正常情况下，需要经过过滤在判断报单的状态再做相应的本地处理。

8. 风险控制
    如果要投入实际生产中使用，在设计时就要无时无刻考虑“在此时宕机这么办”，“断网怎么办”。宕机问题就要做好内存状态和数据库存储的关系，起码内存的信息可以从数据库中初始化出来，这样我们什么时候断掉都行；而断网这个就难说了，这不是一个程序可以解决的问题，实际中应该有一个强制全平的守护程序和别的保证可以上网的方法，一旦出现问题且长时间无法恢复，就需要在调用那个守护程序，从数据库中读取状态并把仓位全部平掉，而且要“安抚”好策略。。总之宁愿清清楚楚地亏损，也不要让状态不在自己的掌控之中，特别是做私募的这类风险问题可一点都出不得。

## 结束语

首先得懂期货  
  
如果你从零开始接触ctp的开发，那么就从我的**参考资料**看起，结合**noQtCTP**和**tdspiTestWithQt**里面的代码，自己动手写一写。此时你只要看基础部分就好了  
  
进一步开发时，可以看下进阶里的一些注意问题，顺便看下我**TradeServer**对这些问题的处理。多少可能会有一些帮助。我做的时候的难点是跟踪多账户多策略的报单情况，我需要根据这些报单实时计算出每个账户的每个策略的可用资金，而且平仓后要计算本次平仓的盈亏，平今判断等等。这些都和具体业务需求关系太大了，所以我就没多写  
  
我并不是行内的高手，可能有很多关键的地方还没有提到，也有可能有提了很多幼稚的地方，见笑:bowtie:  
  
**希望这些经验对大家有帮助，欢迎转载或介绍给新手朋友**
