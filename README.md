# 《分布式逻辑管理平台XXL-GLUE》
## 一、简介

#### 1.1 概述
XXL-GLUE 是一个分布式环境下的逻辑管理平台, 扩展JVM的动态语言支持。

GlueHandler可以方便的嵌入到线上各个业务线中, 从而扩展相应业务模块的动态语言支持。可以节省部分因为项目编译、打包、部署和重启线上机器所带来的时间和人工消耗, 提高开发效率。

**Tips:** 可以参考 “配置管理服务，如disconf diamond等” 的概念来帮助我们来认识和理解XXL-GLUE。前者用于维护分布式环境下的 "配置信息", 并推送配置更新; 后者功能更强大, 支持维护 "Java逻辑代码块(基本单元是GlueHandler)" , 支持 "在线修改、推送更新和实时编译生效" ;

#### 1.2 特性
- 1、动态(groovy)：托管在平台中的GlueHandler以 "groovy" 的方式进行加载实例化, 扩展JVM的动态语言支持;
- 2、在线(Wed IDE)：提供WedIDE，支持在线管理和开发GlueHandler；
- 3、推送更新：当GlueHandler变动时, 将会通过广播组件, 实时推送接入方对应的GlueHandler进行reload更新, 保证GlueHandler中业务逻辑的实时性;
- 4、兼容Spring：无缝兼容Spring, 支持@Resource和@Autowired两种方式注入Spring容器中服务;
- 5、版本：支持50个历史版本的版本回溯；
- 6、调试: 在开发阶段可开启本地模式, 该模式下将会加载本地GlueHandler文件, 支持Debug, 可以方便的进行本地调试;

#### 1.3 下载
源码地址 (将会在两个git仓库同步发布最新代码)
- [github地址](https://github.com/xuxueli/xxl-glue)
- [git.oschina地址](https://git.oschina.net/xuxueli0323/xxl-glue)

博客地址
- [oschina地址](http://my.oschina.net/xuxueli/blog/732499)
- [cnblog地址](http://www.cnblogs.com/xuxueli/p/5115036.html)

##### 技术交流群 (仅作技术交流)

- 群2：438249535    [![image](http://pub.idqqimg.com/wpa/images/group.png)](http://shang.qq.com/wpa/qunwpa?idkey=e288e6a50a82a1eeed89117f45b4839b4ba69db9a87da63ea915fae5294cc50d )
- 群1：367260654    [![image](http://pub.idqqimg.com/wpa/images/group.png)](http://shang.qq.com/wpa/qunwpa?idkey=4686e3fe01118445c75673a66b4cc6b2c7ce0641528205b6f403c179062b0a52 )   （群1已满，请加群2）

#### 1.4 环境
- Maven3+
- Jdk1.7+
- Tomcat7+
- Mysql5.5+

### 二、系统设计

#### 架构图

![输入图片说明](https://static.oschina.net/uploads/img/201609/28120739_AEeQ.png "在这里输入图片标题")

#### 1、角色组成
- GlueHandler: 业务中抽象且离散的逻辑单元, 本质上是实现统一父接口的子类, 约定了公共方法以及公共的输入输出。
- Broadcase: 广播组件, 当GlueHandler更新时, 将会触发广播消息;
- GLUE管理中心: 管理中心, 提供对GlueHandler的管理功能, 同时提供 WebIDE 支持在线开发GlueHandler, 并借助 "Broadcase" 组件提供逻辑单元"GlueHandler"实时推送功能;
- GLUE服务: Glue加载RPC服务, 提供GlueHandler对应源码的加载服务;
- Client: GLUE的接入方, 可以动态的使用托管的GlueHandler, 并享受实时推送功能。避免了逻辑变更带来的一系列编译、打包、部署和重启线上机器等流程;

#### GlueHandler 剖析

GlueHandler本质上是实现统一父接口的子类, 约定了公共方法以及公共的输入输出。

其源码维护在数据库表中, 接入方通过GroovyClassLoader加载相应源码并实例化为 "GlueHandler对象", 调用时将会执行父类公共方法。

统一父接口:
```
package com.xxl.glue.core.handler;
import java.util.Map;

/**
 * default glue iface, it could be use in your biz service
 * @author xuxueli 2016-1-2 21:31:56
 */
public interface GlueHandler {
	
	/**
	 * defaule method
	 * @param params
	 * @return
	 */
	public Object handle(Map<String, Object> params);
	
}

```
 
Groovy简介 : 用于 Java 虚拟机的一种敏捷的动态语言;
- 1、以强大的Java为基础；
- 2、包含Python、Ruby、Smalltalk等语言强带附加功能，例如动态类型转换、闭包和元编程支持；
- 3、一种成熟的面向对象语言，同时可用作纯粹的校验语言；
- 4、适合与Spring动态语言支持一起使用，因为它专门为JVM设计，充分考虑Java继承；
- 5、与Java代码互操作很容易；

#### 2、GlueHandler 执行步骤

![输入图片说明](https://static.oschina.net/uploads/img/201609/28170939_xiE7.png "在这里输入图片标题")

接入方,执行托管在GLUE平台上的一个GlueHandler中的代码逻辑时, 执行步骤如下:

- 1、缓存命中: 首先, 在本地缓存中匹配 "GlueHandler实例" , 匹配成功则调用统一公共方法即可; 否则执行步骤2;
- 2、实例化: RPC方式加载相应GlueHandler对应源码, 并通过 "GroovyClassLoader" 最终实例化为 "GlueHandler实例" ;
- 3、依赖注入: 执行 "injectService", 反射方式为 "GlueHandler实例" 注入依赖的Spring服务;
- 4、加入缓存: 将 "GlueHandler实例" 加入本地缓存;
- 5、注册监听: 注册对该 "GlueHandler" 的广播消息监听;
- 6、invoke: 执行 "GlueHandler实例" 的公共方法, 返回执行结果;
- 7、Finish;

#### 3、Spring服务注入
支持 “Resource.class” 和 “Autowired.class” 两种方式为GlueHandler输入Spring服务，实现逻辑如下：
- 1、 反射获取GlueHandler的Field数组；
- 2、 遍历Field数组，根据其注解 “@Resource” 和 “@Autowired” 在Spring容器匹配服务（注入规则同Spring默认规则：@Autowired按类型注入；@Resource按照首先名称注入，失败则按照类型注入；）；
- 3、将匹配到的Spring服务注入到该Field中。

#### 4、广播组件

![输入图片说明](https://static.oschina.net/uploads/img/201609/28195353_3QpP.png "在这里输入图片标题")

Glue中通过ZK实现了一套广播机制, 采用广播的方式进行触发主动更新。

如图所示, 每个GlueHandler在ZK中对应一个node节点, 当需要广播更新时将会将广播消息(消息对象如右图)序列化后赋值给节点。该GlueHandler的接入方项目将会监听到事件通知并及时刷新缓存;


#### 5、缓存更新策略（异步 + 覆盖）

![输入图片说明](https://static.oschina.net/uploads/img/201609/28201225_vF6z.png "在这里输入图片标题")

- 1、Glue中缓存的对象是“groovyClassLoader”解析生成的GlueHandler实例。

- 2、GlueHandler缓存支持设置Timeout时间，单位毫秒，缓存失效时将会实例化加载新的GLueHander实例，Timeout设置为-1时将永不失效。

- 3、常规缓存更新，通常是通过remove(key)的方式进行缓存清理，然后在下次请求时将会以懒加载的方式进行缓存初始化，但是这样在并发环境中有雪崩的隐患。

    为避免缓存雪崩情况，GlueHandler采用 “异步（queue + thread）”+“覆盖”的方式进行GlueHandler更新，步骤如下：
    - 1、在接收到缓存更新的广播消息时，首先会将待更新的GlueHandler的名称push到待更新队列中；
    - 2、异步线程监控待更新队列，获取待更新GlueHandler名称，加载并实例化新GlueHandler实例；
    - 3、将新的GlueHandler实例，覆盖缓存中旧的GlueHandler实例，后续调用将会执行新的业务逻辑。

#### 6、灰度更新
GlueHandler通过广播的方式进行推送更新，在推送广播消息时支持输入待刷新该GlueHandler的项目名列表，只有匹配到的项目才会对本项目中GlueHandler进行覆盖更新，否则则忽视该条广播消息。

#### 7、版本回溯
GlueHandler的每次更新都会进行对上个版本的代码进行备份，备份数量支持配置，原则上无上限。
同时，在Web IDE界面上，可以查看到所有的备份记录，并且可以方便的进行版本回退。

#### 8、预热
GlueHandler创建时和项目关联起来，这样在项目启动时会主动加载关联到的GlueHandler，避免懒加载引起的并发问题。

#### 9、避免因Permanet Generation空间被占满引起的Full GC
PermanetGeneration中存放的为一些class的信息等，当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满。

GLUE底层基于Groovy实现，Groovy之前使用时曾经出现过频繁Full GC的问题，原因如下：

系统在执行 “groovy.lang.GroovyClassLoader.parseClass(groovyScript)” 进行groovy代码解析时，Groovy为了保证解析后执行的都是最新的脚本内容，每进行一次解析都会生成一次新命名的Class文件，如下图：

![输入图片说明](https://static.oschina.net/uploads/img/201608/14200626_QHdj.jpg "在这里输入图片标题")

因此，如果Groovy类加载器设置为静态，当对同一段脚本均多次执行该方法时，会导致 “GroovyClassLoader” 装载的Class越来越多，从而导致PermGen被用满。

为了避免出现类似问题，GLUE做了以下几点优化。
- 1、针对每个GlueHandler解析后生成的Java对象实例做缓存，而不是代码本身做缓存；
- 2、仅仅在接收到清除缓存的广播时解析生成新的GlueHandler实例对象，避免groovy的频繁解析，减少Class装载频率；
- 3、周期性的异步刷新类加载器，避免因全局类加载器装载Class过多且不及时卸载导致的PermGen被用满。

#### 10、调试      
由于GlueHandler源码存储在云端, 调试不便。因此系统提供了 "com.xxl.glue.core.loader.impl.FileGlueLoader" ,用于开启 "本地模式", 该模式系统将会从项目资源目录的子目录 "config/glue" 中加载源码, 同时支持debug, 可以方便的进行业务代码调试。


### 三、快速入门
#### 源码目录介绍

![输入图片说明](https://static.oschina.net/uploads/img/201609/28154219_vkTh.png "在这里输入图片标题")

- /db                : 数据库交表脚本位置
- /xxl-glue-admin    : GLUE管理中心
- /xxl-glue-core     : 公共依赖
- /xxl-glue-core-example : 接入XXL-GLUE的Demo项目, 可以参考它来学习如何在项目中接入GLUE

#### 初始化数据库

执行数据库建表脚本: /xxl-glue/db/mysql_xxl_glue.sql

#### 配置“Zookeeper地址列表”

    - 配置文件位置: /data/webapps/xxl-conf.properties     (使用硬盘绝对路径, 好处是: 该机器上所有项目可共享该Zookeeper地址配置)
    - 配置文件内容:
    ```
    // 支持zookeeper地址集群配置, 如有多个地址用逗号分隔
    zkserver=127.0.0.1:2181
    ```
    - 配置文件作用: 配置Zookeeper的地址信息


#### 部署"GLUE管理中心"(xxl-glue-admin)

- 配置JDBC地址: 配置文件地址 "/xxl-glue/xxl-glue-admin/src/main/resources/jdbc.properties"

#### 部署"接入XXL-GLUE的Demo项目"(xxl-glue-core-example)

- 配置JDBC地址: 配置文件地址 "/xxl-glue/xxl-glue-admin/src/main/resources/jdbc.properties"

#### 开发第一个GlueHandler (Hello World) 

启动"GLUE管理中心"(xxl-glue-admin),登录系统;

登陆GLUE系统，点击新增，填写 “GLUE名称”（该名称是该GLUE项的唯一标示）和简介，确定后即新增一条GLUE。
点击 “编译”按钮，即可进入GlueHandler开发界面，可在该界面开发GlueHandler代码，也可以在IDE中开发完成后粘贴进来，默认已经初始化一个Demo。
每个GlueHandler必须是实现统一父接口GlueHandler的子类；

	
#### 测试

启动"接入XXL-GLUE的Demo项目"(xxl-glue-core-example),访问以下链接测试

    http://localhost:8080/xxl-glue-core-example/code/GLUE名称
    
#### 如何Debug测试GlueHandler

源码加载器配置,见项目"xxl-glue-core-example"的"applicationcontext-glue.xml"配置中"GlueFactory"实例的"glueLoader"属性;

- dbGlueLoader(数据库加载器,不支持Debug,源码线上维护): 默认XXL-GLUE在接入方通过加载数据库中GLUE源码进而实例化并执行, 需要配置一个"dbGlueLoader"。
- FileGlueLoader(本地加载器,支持Debug): XXL-GLUE支持配置本地加载器, 只是掉上述配置中的"glueLoader"属性即可,则使用默认的文件加载器;拥有以下特点:
    - 默认加载项目"resources/config/glue"目录下的groovy文件,文件名称必须和配置的GlueHandler名称一致;
    - 会循环遍历该目录下子目录文件进行匹配,因此文件可以在该目录下自由存放;
    - FileGlueLoader方式使用XXL-GLUE,支持GlueHandler的debug断点;
    
#### 业务中如何执行自己的GlueHandler

执行以下一行代码即可    
```
	Object result = GlueFactory.glue("glue名称", "glue入参，Map类型");
```

#### 如何广播刷新GlueHandler

进入GLUE管理中心, 在GlueHandler的右侧, 点击 "清除缓存" 按钮将会弹框确认是否刷新GLUE。

该弹框中有一个输入框 "Witch APP", 输入接入方项目的AppName, 即可精确的灰度刷新该项目中的相应GlueHandler。如果不输入, 则广播刷新所有项目中响应的GlueHandler。

### 四、GlueHandler的三种经典使用场景, 
#### 场景A：托管 “配置信息” ，尤其适用于数据结构比较复杂的配置项
```
package com.xxl.groovy.example.service.impl;

import java.util.HashSet;
import java.util.Set;

import com.xxl.glue.core.handler.GlueHandler;

/**
 * 场景A：托管 “配置信息” ，尤其适用于数据结构比较复杂的配置项
 * 优点：在线编辑；推送更新；+ 直观；
 * @author xuxueli 2016-4-14 15:36:37
 */
public class DemoHandlerAImpl implements GlueHandler {

	@Override
	public Object handle(Map<String, Object> params) {
		
		// 【列表配置】
		Set<Integer> blackShops = new HashSet<Integer>();								// 黑名单列表
		blackShops.add(15826714);
		blackShops.add(15826715);
		blackShops.add(15826716);
		blackShops.add(15826717);
		blackShops.add(15826718);
		blackShops.add(15826719);
		
		return blackShops;
	}
	
}
```

#### 场景B：托管 “静态方法”，可以将配置解析逻辑一并托管，只关注返回结果即可
```
package com.xxl.groovy.example.service.impl;

import java.util.HashSet;
import java.util.Map;
import java.util.Set;

import com.xxl.glue.core.handler.GlueHandler;

/**
 * 场景B：托管 “静态方法”
 * 优点：...； + 组件共享；
 * @author xuxueli 2016-4-14 16:07:03
 */
public class DemoHandlerBImpl implements GlueHandler {
	private static final String SHOPID = "shopid";
	
	private static Set<Integer> blackShops = new HashSet<Integer>();
	static {
		blackShops.add(15826714);
		blackShops.add(15826715);
		blackShops.add(15826716);
		blackShops.add(15826717);
		blackShops.add(15826718);
		blackShops.add(15826719);
	}
	
	/**
	 * 商户黑名单判断
	 */
	@Override
	public Object handle(Map<String, Object> params) {
		int shopid = 0;
		if (params!=null && params.get(SHOPID)!=null) {
			shopid = (Integer) params.get(SHOPID);
		}
		if (shopid > 0 && blackShops.contains(shopid)) {
			return true;
		}
		return false;
	}
	
}
```

#### 场景C：托管 “动态服务”，可以灵活组装接口和服务, 扩展服务的动态特性，作为公共模块。
```
package com.xxl.groovy.example.service.impl;

import java.util.Map;

import com.xxl.glue.core.handler.GlueHandler;

/**
 * 场景B：托管 “抽象且离散的逻辑单元”
 * 优点：...；逻辑封装（伪服务）；
 * @author xuxueli 2016-4-14 16:07:03
 */
public class DemoHandlerCImpl implements GlueHandler {
	private static final String SHOPID = "shopid";
	
	/*
	@Resource
	private AccountService accountService;
	@Autowired
	private AccountService accountServiceB;
	*/
	
	/**
	 * 商户黑名单判断
	 */
	@Override
	public Object handle(Map<String, Object> params) {
		
		/*
		int shopid = 0;
		if (params!=null && params.get(SHOPID)!=null) {
			shopid = (Integer) params.get(SHOPID);
		}
		if (shopid > 0 && accountService.isBlackShop()) {
			return true;
		}
		*/
		
		return false;
	}
	
}
```



### 五、操作指南
#### 第一步：登陆GLUE
![输入图片说明](https://static.oschina.net/uploads/img/201608/14200804_Yuox.png "在这里输入图片标题")

#### 第二步：新建GLUE项
![输入图片说明](https://static.oschina.net/uploads/img/201608/14200827_kkQX.png "在这里输入图片标题")

#### 第三步：GLUE列表
![输入图片说明](https://static.oschina.net/uploads/img/201608/14200852_Q3Mj.png "在这里输入图片标题")

#### 第四步：开发GLUE代码
点击右侧 “编辑” 按钮，进入GLUE开发的Wed IDE界面。默认已经初始化Hello World示例代码，如需开发业务代码，只需要在handle方法中开发即可。

![输入图片说明](https://static.oschina.net/uploads/img/201608/14200932_HAsd.png "在这里输入图片标题")

#### 第五步：一句话执行GlueHandler
首先确定项目中已经接入GLUE（参考上文“三步接入GLUE”，接入非常方便）；
然后执行一行代码即可；

![输入图片说明](https://static.oschina.net/uploads/img/201608/14200956_vxNj.png "在这里输入图片标题")

#### 第六步：推送更新
GlueHandler在第一次加载之后将会缓存在内存中，点击右侧 “清楚缓存” 按钮可以推送更新，填写AppName将会精确定位单个项目进行缓存更新，如果为空则全站广播。

![输入图片说明](https://static.oschina.net/uploads/img/201608/14201023_Y3Be.png "在这里输入图片标题")


## 六、历史版本
#### 版本1.0.0
- 1、动态(groovy)：托管在平台中的GlueHandler以 "groovy" 的方式进行加载实例化, 扩展JVM的动态语言支持;
- 2、在线(Wed IDE)：提供WedIDE，支持在线管理和开发GlueHandler；
- 3、推送更新：当GlueHandler变动时, 将会通过广播组件, 实时推送接入方对应的GlueHandler进行reload更新, 保证GlueHandler中业务逻辑的实时性;
- 4、兼容Spring：无缝兼容Spring, 支持@Resource和@Autowired两种方式注入Spring容器中服务;
- 5、版本：支持50个历史版本的版本回溯；

#### 版本1.1.0
- 1、系统重构;
- 2、调试: 在开发阶段可开启本地模式, 该模式下将会加载本地GlueHandler文件, 支持Debug, 可以方便的进行本地调试;

#### 版本1.2.0
- 1、广播组件由Activemq改为自主实现的基于ZK的广播组件, 减少系统第三方依赖;
- 2、新增Local模式，提供GLUE本地加载器, 支持加载本地GlueHandler, 方便进行Debug调试;
- 3、异步刷新缓存逻辑更新,新实例正常则覆盖,否则remove掉旧实例;
- 4、修复一处因ReentrantLock导致可能死锁的问题;
- 5、导航菜单更新;
- 6、底层代码重构, 结构优化;

#### 规划中
- 分组管理

## 七、其他

#### 7.1 报告问题
XXL-GLUE托管在Github上，如有问题可在 [ISSUES](https://github.com/xuxueli/xxl-glue/issues) 上提问，也可以加入上文技术交流群；

#### 7.2 接入登记
更多接入公司，欢迎在github [登记](https://github.com/xuxueli/xxl-glue/issues/1 )