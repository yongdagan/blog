#taobao-pamirs-proxycache框架的实现原理
[taobao-pamirs-proxycache](http://code.taobao.org/p/taobao-pamirs-proxycache/wiki/index/)是一个使缓存配置和业务代码分离的缓存管理框架。缓存代理通过XML配置，框架使用Spring AOP的方式与业务代码无缝结合。

本文从下面几个方面介绍taobao-pamirs-proxycache框架的实现原理：
>1. taobao-pamirs-proxycache的配置使用；
2. 以CacheModule为代表的几个基本构件；
3. CacheManager初始化缓存框架所涉及的工作；
4. TairStore和MapStore的基本运行机制，其中会简要介绍Tair和ConcurrentLRUCacheMap的实现机制；
5. CacheManagerHandle 缓存管理处理类如何使用AOP的方式进行缓存代理；
6. ThreadCacheHandle 本地线程缓存机制。

另外，本文以taobao-pamirs-proxycache 2.1.0后的版本作为例子，在配置上与之前的版本略有差别。

###taobao-pamirs-proxycache 的配置使用
**springCache.xml**：基本bean配置，其中tairManager是TairStore调用Tair的入口类，负责与Tair打交道，cacheManager是缓存框架的入口类，负责初始化缓存配置，cacheManagerHandle是缓存框架的处理类，负责依据cacheManager初始化的配置，采用AOP的方式管理缓存代理。
```xml
<?xml version="1.0" encoding="GBK"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="byName">
	<!-- 升级到mc-tair -->
	<bean id="tairManager" class="com.taobao.tair.impl.mc.MultiClusterTairManager"
		init-method="init">
		<property name="configID">
			<value>ldbcommon-daily</value>
		</property>
		<property name="dynamicConfig">
			<value type="java.lang.Boolean">true</value>
		</property>
	</bean>

	<bean id="cacheManager" class="com.taobao.pamirs.cache.load.impl.LocalConfigCacheManager" init-method="init"
		depends-on="tairManager">
		<property name="storeType" value="tair/map" />
		<property name="tairNameSpace" value="318" />					<!-- just for tair store -->
		<property name="storeMapCleanTime" value="0 * * * * ? *" />		<!-- just for map store，可选 -->
		<property name="localMapSize" value="1024" />					<!-- just for map store，可选，默认1024 -->
		<property name="localMapSegmentSize" value="8" />				<!-- just for map store，可选，默认16 -->
		<property name="storeRegion" value="${store.tair.region}" />	<!-- 环境分区，可选 -->
		<property name="configFilePaths">
			<list>
				<value>bean/cache/cache-config-article.xml</value>
				<value>bean/cache/cache-config-item.xml</value>
				<value>bean/cache/cache-config-pack.xml</value>
				<value>bean/cache/cache-config-activity.xml</value>
				<value>bean/cache/cache-config-promotion.xml</value>
			</list>
		</property>
		<property name="tairManager" ref="tairManager" />				<!-- just for tair store -->
	</bean>

	<bean class="com.taobao.pamirs.cache.framework.aop.handle.CacheManagerHandle">
		<property name="cacheManager" ref="cacheManager" />
	</bean>

</beans>
```

**beanCache.xml**：缓存配置，这里配置了promotionReadService这个bean的getPromotionByCode使用缓存，以及cleanCacheById作为getPromotionById的清理缓存方法。目前，cleanCacheBean必须是cacheBean的子集。在这里，我们可以看到缓存框架的几个关键构件：cacheModule, cacheBean, cacheCleanBean, methodConfig, cacheCleanMethod。
```xml
<?xml version="1.0" encoding="GBK"?>
<cacheModule>
	<!-- 提醒：spring bean 内部调用（inner method）会绕过AOP代理，从而绕过缓存 -->
	<!-- 解决方案示例： -->
	<!-- 1. ASerivce selfAopProxy = (ASerivce) AopContext.currentProxy(); -->
	<!-- 2. selfAopProxy.innerMethod();// 替换this. -->

	<!-- 缓存Bean配置 -->
	<cacheBeans>
		<cacheBean>
			<beanName>promotionReadService</beanName>
			<cacheMethods>
				<methodConfig>
					<methodName>getPromotionByCode</methodName>
					<parameterTypes>								<!-- 方法参数，如果有重载方法时，必须要指定，可选 -->
						<java-class>java.lang.String</java-class>
					</parameterTypes>
					<expiredTime></expiredTime>						<!-- 失效时间，单位：秒。 可以是相对时间，也可以是绝对时间(大于当前时间戳是绝对时间过期)。不传或0都是不过期，可选 -->
				</methodConfig>
				<methodConfig>
					...
				</methodConfig>
			</cacheMethods>
		</cacheBean>
		<cacheBean>...</cacheBean>
	</cacheBeans>
	<!-- 缓存清理Bean配置 -->
	<cacheCleanBeans>
		<cacheCleanBean>
			<beanName>promotionReadService</beanName>
			<methods>
				<cacheCleanMethod>
					<methodName>cleanCacheById</methodName>
					<parameterTypes>								<!-- 方法参数，如果有重载方法时，必须要指定，可选 -->
						<java-class>java.lang.Long</java-class>
					</parameterTypes>
					<cleanMethods>
						<methodConfig>
							<methodName>getPromotionById</methodName>
							<beanName />							<!-- 不用配置，会继承上面的beanName -->
							<parameterTypes />						<!-- 不用配置，会继承，目前只支持参数必须和上面的一致的方法 -->
						</methodConfig>
					</cleanMethods>
				</cacheCleanMethod>
				<cacheCleanMethod>
					...
				</cacheCleanMethod>
			</methods>
		</cacheCleanBean>
		<cacheCleanBean>...</cacheCleanBean>
	</cacheCleanBeans>
</cacheModule>
```
###以CacheModule为代表的几个基本构件
![配置相关的UML类图](http://upload-images.jianshu.io/upload_images/209608-f8c443d16b1c8212.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

缓存框架根据XML文件可以很容易地得到对应的CacheModule，而实际使用的CacheConfig只是比CacheModule多了一些配置项参数。

###CacheManager 初始化缓存框架
CacheManager是缓存框架的入口类，负责读取XML配置并初始化缓存框架。在这里，会借助ConfigUtil来做一些加载和校验的工作。具体来说，它要完成下面的工作：
1. loadConfig()：加载并校验XML配置文件。利用XStream可以很方便地将正确配置的XML缓存配置文件转换为CacheModule的Java对象。将得到的CacheModule附上在springCache.xml的一些配置项就得到缓存的总配置CacheConfig。关键代码如下：
```java
XStream xStream = new XStream(new DomDriver());
xStream.alias("cacheModule", CacheModule.class);
xStream.alias("cacheBean", CacheBean.class);
xStream.alias("methodConfig", MethodConfig.class);
xStream.alias("cacheCleanBean", CacheCleanBean.class);
xStream.alias("cacheCleanMethod", CacheCleanMethod.class);
```
2. autoFillCacheConfig(cacheConfig)：在配置XML缓存文件中，我们可以不配置方法的parameterTypes，框架在此时会自动寻找同名的方法（反射机制），但如果存在多个同名方法，则会报错。因此个人认为还是手动配置完整的parameterTypes比较好。另外，cacheCleanMethod里面的cleanMethod的参数列表必须与cacheCleanMethod的参数列表一致。例如，在上面的配置中，getPromotionById方法的参数列表必须和cleanCacheById的参数列表一致，即Long。关键代码如下：
```java
if (cacheConfig.getCacheCleanBeans() != null) {
	for (CacheCleanBean cleanBean : cacheConfig.getCacheCleanBeans()) {
		for (CacheCleanMethod method : cleanBean.getMethods()) {
			for (MethodConfig clearMethod : method.getCleanMethods()) {
				clearMethod.setParameterTypes(method.getParameterTypes());
			}
		}
	}
}
```
3. verifyCacheConfig(cacheConfig)：缓存配置合法性校验，包括三个方面的校验。(1)静态校验，校验对象的域注解；(2)动态Spring校验，校验bean及其方法是否与Spring中注入的一致；(3)配置重复校验，校验缓存方法、缓存清理方法以及缓存清理方法关键的方法是否存在重复配置。

4. initCache()：初始化缓存。目前cacheCleanBeans必须是cacheBeans的子集，在这里只会对每个cacheBean初始化对应的缓存对象。region（配置项），bean和method是缓存的唯一标识。初始化缓存涉及到的关键类如下：
![初始化缓存相关的UML类图](http://upload-images.jianshu.io/upload_images/209608-93769a72bc86a9d4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
具体来说，initCacheAdapters(...)初始化bean/method对应的缓存需要做如下操作：
>1. 根据region, bean和method生成当前缓存适配器的key，key的格式为region@beanName#methodName#{String|Long}；
>2. 根据storeType初始化缓存类型TairStore/MapStore，得到相应的ICache。在日常应用中，我们一般选用TairStore；
>3. 初始化CacheProxy，如上图所示，它是一个使用了观察者模式的ICache适配器。与ICache的区别在于，在执行方法完成时，会通知实现了CacheOprateListener接口的监听者做相关操作。将key和cacheProxy放进类型为Map的cacheProxys，供后续缓存框架使用；
>4. 若使用MapStore，这里会启动定时清理任务进程，根据配置的storeMapCleanTime表达式定时清理MapStore的缓存；
>5. 注册JMX，支持使用JMX控制台管理缓存数据；
>6. 注册xray log打印日志统计信息，这里利用了CacheProxy的观察者模式。

至此，缓存框架初始化完成。从上面可以看出，缓存管理的基本元素是CacheProxy，而其内部功能实现就是TairStore或MapStore。因此，接下来简单介绍一下TairStore与MapStore及其相关的运行机制。

###TairStore与Tair
TairStore采用淘宝Tair的统一缓存存储方案，通过Key/Value的形式将序列化后的对象存入Tair服务器。TairStore只支持put，put_expireTime，get，remove四种key方法，不能使用clear和clean这种范围清除操作。TairStore包装了Tair的Client接口TairManager的相应方法，并额外做一些格式转换和超时校验操作。缓存在Tair的唯一标识是namespace和key（region@beanName#methodName#{String|Long}abc@@123）。

这里需要特别说明一下超时校验操作：
> Tair支持在put方法中传入expiredTime，当缓存超过expiredTime时，Tair会将缓存在服务器中移除。若expiredTime为0，则表示永不失效。
> TairStore在此基础上，支持在需要的时候手动设置invalidBeforeDate。设置后，在get方法拿到缓存后，会校验缓存是否在invalidBeforeDate之前。若是，则调用TairManager的remove方法手动移除缓存。这样使得缓存失效方式更加灵活。关键代码如下：
> ```java
public void invalidBefore() {
	this.invalidBeforeDate = new Date();
}
public V get(K key) {
...
	if (invalidBeforeDate != null) {
		DateFormat f = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		Date mDate = f.parse(formatDate(tairData.getModifyDate()));
		if (invalidBeforeDate.after(mDate)) {
			this.remove(key);
			return null;
		}
	}
...
}
>```

为了更好地理解TairStore是如何工作，下面简单介绍一下Tair的运作机制，具体可参见[这篇文章](http://www.infoq.com/cn/articles/taobao-tair)。
> ![Tair的三大模块](http://upload-images.jianshu.io/upload_images/209608-49c58d1da1eab315.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> **Tair的三大模块**：
>- **client**：从configserver获取数据的分布信息（对照表），并与相应的dataserver交互完成用户的请求
>- **configserver（主从）**：通过接收dataserver的心跳信息(HeartBeat)维护集群中的可用节点，并构建数据分布信息
>- **dataserver（集群）**：负责数据存储，并按照configserver的指示完成数据的复制和迁移工作
>
> ![对照表模型示例](http://upload-images.jianshu.io/upload_images/209608-cb03e7b37ce1b3dc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> **轻量级configserver**：
>- client与configserver的交互主要是为了获得对照表，而client会缓存这张表直接与dataserver集群进行交互。因此，在大部分时候，configserver不可用对集群的服务不会造成太大影响。
>- 在configserver生成对照表的同时会产生一个版本号，并同步给dataserver。当client向dataserver请求数据时，dataserver会一并将版本号返回给client。只有当client发现版本号不一致时，才会向configserver请求新的对照表。
>- 另一种情况是，当有新的client加入时，需要向configserver请求对照表。
>
>**dataserver的数据自动复制与迁移**：
>- 当数据写入到主节点时，主节点会根据对照表自动将数据写入到其他备份节点
>- 当有新节点或节点不可用时，configserver重新生成对照表，数据节点会自动将新表中不由自己负责的数据迁移到目标节点

虽然TairStore只需与TairManager即Tair的client打交道，但理解上述知识对理解整个缓存框架有帮助。

###MapStore与ConcurrentLRUCacheMap
MapStore是使用本地ConcurrentLRUCacheMap缓存存储方案，其中采用线程安全的最近最少使用淘汰策略。MapStore支持get, put, remove, clear等key方法，但不支持invalidBefore失效缓存操作。MapStore的核心元素是ConcurrentLRUCacheMap，它提供线程安全的缓存存储结构。为了支持expiredTime，使用缓存结果包装类ObjectBoxing。MapStore的整体类结构如下：
![MapStore](http://upload-images.jianshu.io/upload_images/209608-9d71608f03452e3a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Java原生的LinkedHashMap提供一种accessOrder淘汰策略，本质上就是最近最少使用LRU淘汰策略。因此，LRUMap是一个实现了LRU算法但线程不安全的Map。LRUMapLocked更进一步，使用了ReentrantLock锁机制（类似synchronized但更灵活高效）对key方法进行包装，提供了线程安全的特性。ConcurrentLRUCacheMap其实是LRUMapLocked的一个装饰器，在LRUMapLocked线程安全的基础上，采用分区策略提高并发性能，至此实现了一个高效的线程安全本地缓存结构。

为了支持超时失效缓存的特性，ObjectBoxing对缓存结果进行包装，在取得结果前先校验结果是否超时。其关键代码如下：
```java
if (expireTime != null && expireTime != 0) {
	long now = System.currentTimeMillis() / 1000;

	if (timestamp.longValue() > expireTime.longValue()) {// 相对时间
		if (now >= (expireTime.longValue() + timestamp.longValue()))
			return null;
	} else {// 绝对时间
		if (now >= expireTime.longValue())
			return null;
	}
}
```

###CacheManagerHandle 缓存管理处理类
首先给出框架生成代理及处理缓存时所涉及的UML类图，关键思想是利用Spring AOP机制来实现缓存代理。
![缓存代理及其处理的相关类](http://upload-images.jianshu.io/upload_images/209608-2db83a54b7e3c413.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CacheManagerHandle继承Spring AOP的AbstractAutoProxyCreator，当spring context初始化完成后（包括bean注入和CacheManager的init方法初始化CacheConfig），CacheManagerHandle的getAdvicesAndAdvisorsForBean会为需要缓存的bean/method生成缓存代理。具体来说，它会检查每个bean是否在cacheConfig中存在配置，若存在配置则会生成缓存代理。关键代码如下：
```java
protected Object[] getAdvicesAndAdvisorsForBean(Class beanClass,
		String beanName, TargetSource targetSource) throws BeansException {
	log.debug("CacheManagerHandle in:" + beanName);
	if (ConfigUtil.isBeanHaveCache(cacheManager.getCacheConfig(), beanName)) {
		log.warn("CacheManager start... ProxyBean:" + beanName);
		return new CacheManagerAdvisor[] { new CacheManagerAdvisor(
				cacheManager, beanName) };
	}
	return DO_NOT_PROXY;
}
```

从上面的代码可以看出，为缓存生成的代理是CacheManagerAdvisor。这里涉及到Spring AOP的一个适配器模式Advisor和Advice。实际进行工作的是实现Advice接口的CacheManagerRoundAdvice，而为了对cacheConfig配置的bean进行缓存代理，它同时实现了MethodInterceptor接口对调用的方法进行拦截。下面简单说说其中涉及的步骤（invoke方法）：
>1. 利用MethodInvocation取得当前代理bean调用的方法，获取方法签名；
>2. 利用bean和方法签名获取满足条件的CacheMethod和CacheCleanMethod的cleanMethods；
>3. 获取HSF consumer IP，这个IP主要是CacheProxy的监听者用来统计信息；
>4. 当前bean/method的三种情况（优先级由高到低）：
>    (1)获取到相应的cacheMethod：生成相应的适配key并取得cacheProxy，生成缓存的key。执行处理缓存方法：若缓存命中，走缓存；若缓存未命中，走原生方法，若原生方法返回结果不为空，则更新到缓存。关键代码如下：
> ```java
// 生成key
...
String adapterKey = CacheCodeUtil.getCacheAdapterKey(storeRegion, beanName, cacheMethod);
CacheProxy<Serializable, Serializable> cacheAdapter = cacheManager.getCacheProxy(adapterKey);
String cacheCode = CacheCodeUtil.getCacheCode(storeRegion,beanName, cacheMethod, invocation.getArguments());
return useCache(cacheAdapter, cacheCode,cacheMethod.getExpiredTime(), invocation, fromHsfIp);
...
// 处理缓存
private Object useCache(CacheProxy<Serializable, Serializable> cacheAdapter,
		String cacheCode, Integer expireTime, MethodInvocation invocation,
		String ip) throws Throwable {
	if (cacheAdapter == null)
		return invocation.proceed();
	long start = System.currentTimeMillis();
	Object response = cacheAdapter.get(cacheCode, ip);
	if (response == null) {
		response = invocation.proceed();
		long end = System.currentTimeMillis();
		// 缓存未命中，走原生方法，通知listener
		cacheAdapter.notifyListeners(GET, new CacheOprateInfo(cacheCode,
				end - start, false, cacheAdapter.getBeanName(),
				cacheAdapter.getMethodConfig(), null, ip));
		if (response == null)// 如果原生方法结果为null，不put到缓存了
			return response;
		if (expireTime == null) {
			cacheAdapter.put(cacheCode, (Serializable) response, ip);
		} else {
			cacheAdapter.put(cacheCode, (Serializable) response,
					expireTime, ip);
		}
	}
	return response;
}
> ```
>    (2)获取到相应的cleanMethods：调用原生的cacheCleanMethod做相关清理工作，然后获取到需要被清理缓存的方法cleanMethods，使它们涉及的缓存失效。关键代码如下：
>```java
...
try {
	return invocation.proceed();
} finally {
	cleanCache(beanName, cacheCleanMethods, invocation, storeRegion, fromHsfIp);
}
...
// 清理缓存
private void cleanCache(String beanName, List<MethodConfig> cacheCleanMethods, MethodInvocation invocation,
		String storeRegion, String ip) throws Throwable {
	if (cacheCleanMethods == null || cacheCleanMethods.isEmpty())
		return;
	for (MethodConfig methodConfig : cacheCleanMethods) {
		String adapterKey = CacheCodeUtil.getCacheAdapterKey(storeRegion,
				beanName, methodConfig);
		CacheProxy<Serializable, Serializable> cacheAdapter = cacheManager
				.getCacheProxy(adapterKey);
		if (cacheAdapter != null) {
			String cacheCode = CacheCodeUtil.getCacheCode(storeRegion,
					beanName, methodConfig, invocation.getArguments());
			cacheAdapter.remove(cacheCode, ip);
		}
	}
}
>```
>    (3)都不存在：走原生方法。
>```java
return invocation.proceed();
>```

这里特别说明一下清理缓存。当不需要做一些额外的统计或清理操作的时候，只需定义一个空的清理缓存方法，让框架帮我们做缓存失效的工作就够了。

回过头来说说为什么当cacheCleanBeans是cacheBeans的子集时，在CacheManager初始化缓存配置时只需初始化cacheBeans而CacheManagerHandle在处理缓存代理时却可以处理缓存清理方法。

这是因为CacheManagerHandle使用AOP创建缓存代理的最小粒度是bean，因此可以在bean的维度拦截整个bean的所有方法。而CacheManager初始化缓存框架的主要目的是初始化cacheProxys，用于后续与真正的缓存存储结构打交道。只有需要缓存的方法才需要cacheProxy，而缓存清理方法是不需要与缓存打交道的，但其中的cleanMethods必须存在缓存方法配置才可失效缓存。

###ThreadCacheHandle 本地线程缓存
在流程中（单线程），涉及到对多个method的重复调用且调用结果相同时，即使单个接口性能很高，也可能导致整个流程性能降低。使用ThreadCacheHandle可以将method结果缓存起来，下次调用时直接在本线程的缓存中获取，不需要再次重复调用method。

要使用ThreadCacheHandle需要做额外的XML配置，表明需要使用本地缓存服务的beans。配置方式如下所示：
```xml
<bean class="com.taobao.pamirs.cache.store.threadcache.ThreadCacheHandle">
	<property name="beansMap">
	<!-- the void method not support， will ignore cache -->
		<map>
			<entry key="articleReadClient" value="getArticleById,getPromotionIds,getMarketSaleConditions" />
			<entry key="itemReadClient" value="getItemById,getMutexItemIds,getPromotionIds,getSaleConditions" />
			<entry key="prodSubscriptionService" value="countProdSubNumByProductId" />
			<entry key="bizVerifyService" value="existAppstoreSubParam" />
		</map>
	</property>
</bean>
```

在流程启动时，启动线程缓存的方式如下：
```java
public void process() {
	// 线程缓存启动
	ThreadContext.startLocalCache();
	try {
		...
	} finally {
		ThreadContext.remove();
	}
}
```
在上述的例子中，当线程缓存启动后，调用articleReadClient.getArticleById方法时会缓存其结果，后续在本流程再次调用此方法时，会直接取得缓存的结果而不去重复调用方法。

大体上，ThreadCacheHandle的实现方式与CacheManagerHandle非常相似，同样是利用Spring AOP机制来实现本地线程缓存。下面给出其中涉及的UML类图：
![本地线程缓存的相关类](http://upload-images.jianshu.io/upload_images/209608-dc9869b680b20173.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

与此前CacheManagerHandle的类图是非常相似，因此这里直接跳过AOP的相关内容来说说ThreadMethodRoundAdvice的invoke方法是如何做本地线程缓存代理的。
>1. 利用MethodInvocation取得当前代理bean调用的方法，获取方法签名；
>2. 取得beansMap中配置的当前bean的方法列表；
>3. 当前bean/method的三种情况：
>    (1)不存在相关配置，即不需做本地线程缓存：直接走原生方法。
>```java
if (!handle.isOpenThreadCache() || mothods == null
		|| !mothods.contains(methodName)) {
	// do nothing
	return invocation.proceed();
}
>```
>    (2)存在相关配置且缓存有结果：直接返回缓存结果。
>```java
if (value != null) {
	doPrintLog(key, value);
	return value;
}
>```
>    (3)存在相关配置但缓存没有结果：走原生方法，并缓存结果。
>```java
value = invocation.proceed();
ThreadContext.put(key, value);
return value;
>```

目前还没有说明线程缓存结果是如何保存的，这是ThreadContext所做的工作。它利用ThreadLocal<Map<String, Object>>将缓存的结果保存在当前线程的Map里，相对比较简单。

至此，整个框架的主要功能部分大体上已介绍完毕。
