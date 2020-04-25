# Spring对循环依赖的处理 : 从一个例子开始


通过分析一个最简单的循环依赖的例子，解释Spring对循环依赖的处理思路。

## bean 定义和加载代码

### BeanA

``` java
public class BeanA {
	private BeanB beanB;

	public void sayHi () {
		System.out.println("Hi, I am beanA, I have beanB = " + beanB.toString());
	}

	public BeanB getBeanB() {
		return beanB;
	}

	public void setBeanB(BeanB beanB) {
		this.beanB = beanB;
	}
}
```

### BeanB

``` java
public class BeanB {
	private BeanA beanA;

	public void sayHi () {
		System.out.println("Hi I am beanB, I have beanA = " + beanA.toString());
	}

	public BeanA getBeanA() {
		return beanA;
	}

	public void setBeanA(BeanA beanA) {
		this.beanA = beanA;
	}
}
```

### xml 配置

``` xml
<bean id="beanA" class="com.gmail.stridehuan.bean.BeanA">
	<property name="beanB">
		<ref bean="beanB"></ref>
	</property>
</bean>
<bean id="beanB" class="com.gmail.stridehuan.bean.BeanB">
	<property name="beanA">
		<ref bean="beanA"></ref>
	</property>
</bean>
```

### bean 加载代码

``` java
ClassPathResource resource = new ClassPathResource("spring/spring-config.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(resource);
BeanA beanA = (BeanA) factory.getBean("beanA");
```

## 执行路径

### 第一层递归，加载 BeanA

将调用路径分成 9 步，也就是实际 debug 中的堆栈状态，到第 9 步开始递归 加载依赖的 BeanB。

#### 1. AbstractBeanFactory#doGetBean

核心方法 AbstractBeanFactory#doGetBean

通过这个方法 递归调用加载相互依赖的 bean

#### 2. DefaultSingletonBeanRegistry#getSingleton(java.lang.String)

DefaultSingletonBeanRegistry#getSingleton 有多个重载的方法，当前方法的执行逻辑是：

尝试从缓存中 获得 bean 对象。

此处的执行结果是 null，也就是没有从缓存中得到 名为 beanA 的 bean 对象。

#### 3. AbstractBeanFactory#markBeanAsCreated

将当前 bean名 添加到 AbstractBeanFactory#alreadyCreated 集合中。

``` java
/** Names of beans that have already been created at least once. */
private final Set<String> alreadyCreated = Collections.newSetFromMap(new ConcurrentHashMap<>(256));
```

#### 4. AbstractBeanFactory#getMergedLocalBeanDefinition

获得当前 bean 的 RootBeanDefinition 对象。

``` java
final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
```
``
#### 5. DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory)

第 2 步的方法的重载，调用处是:

``` java
sharedInstance = getSingleton(beanName, () -> {
	try {
		return createBean(beanName, mbd, args);
	}
	catch (BeansException ex) {
		destroySingleton(beanName);
		throw ex;
	}
});
```

主体的处理逻辑是：

如果没有从缓存中获得 bean，

调用 DefaultSingletonBeanRegistry#beforeSingletonCreation 方法，尝试 将当前 bean 的名字添加到 DefaultSingletonBeanRegistry#singletonsCurrentlyInCreation 集合中。

通过工厂对象获得 bean 对象。

这里的工厂对象是通过lmdba表达式实现的匿名对象，最终调用 AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[]) 方法生成对象。

#### 6. AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[]) 

实际调用的方法是：AbstractAutowireCapableBeanFactory#doCreateBean。

- 实例化BeanA对象的代码段：

``` java
if (mbd.isSingleton()) {
	instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
}
if (instanceWrapper == null) {
	instanceWrapper = createBeanInstance(beanName, mbd, args);
}
```

这段代码主要是解析到 bean 的构造方法，实例化 bean，并将其包装成 BeanWrapper 对象。不过此时的 bean 尚未进行初始化和依赖处理。

- 将实例封装到单例工厂中，并进行注册

``` java
// 将 bean（刚刚new出来的未初始化的bean）封装到 factory 对象中
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

``` java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(singletonFactory, "Singleton factory must not be null");
	synchronized (this.singletonObjects) {
		if (!this.singletonObjects.containsKey(beanName)) {
			this.singletonFactories.put(beanName, singletonFactory); // 三级缓存
			this.earlySingletonObjects.remove(beanName); // 二级缓存
			this.registeredSingletons.add(beanName); // 一级缓存
		}
	}
}
```

通过 lamdba 表达式将 bean 封装成一个 factory 对象，并将其添加到 singletonFactories（三级缓存） 中。

#### 7. AbstractAutowireCapableBeanFactory#populateBean

解析 bean 类的属性，最终调用 AbstractAutowireCapableBeanFactory#applyPropertyValues 方法处理属性信息。

#### 8. AbstractAutowireCapableBeanFactory#applyPropertyValues

将 bean 信息包装成 BeanDefinitionValueResolver 类型，调用 BeanDefinitionValueResolver#resolveValueIfNecessary 处理具体的属性。

最终调用 BeanDefinitionValueResolver#resolveReference 处理依赖的 bean

#### 9. BeanDefinitionValueResolver#resolveReference

最终调用 .AbstractBeanFactory#getBean(java.lang.String) 方法尝试加载依赖的 beanB，到此开始进行第一次递归调用。

此时 BeanA 的状态是，对象已经 new 出来了，被封装成一个单例 factory 并放到 三级缓存 singletonFactories 中。

### 第二层递归，加载 BeanB

调用步骤和 BeanA 一模一样，到第 9 步的时候，尝试加载依赖的 BeanA，进入第三层递归。

### 第三层递归，再次加载 BeanA

此时 BeanA 已经在 三级缓存 singletonFactories 中了，和第一次加载 BeanA 一样，会在第 2 步调用 DefaultSingletonBeanRegistry#getSingleton(java.lang.String) 方法。

此时，将 BeanA 从三级缓存挪到二级缓存，并返回 BeanA 的引用（此时 BeanA 仍然是一个未初始化的对象，因为它还未解决完依赖）。

### 回到第二层递归

回到第二层递归，BeanB 获得了 依赖的 BeanA 的引用，BeanB 的初始化完成，BeanB 从三级缓存直接转移到一级缓存 singletonObjects 中，至此，完成了 BeanB 的加载。

当然，还需要将 BeanB 的引用返回给上一层的递归。

### 回到第一层递归

回到第一层递归，此时 BeanA 还未初始化完成，它还位于二级缓存中。BeanA 获得了 依赖的 BeanB 的引用，BeanA 完成了初始化，将 BeanA 从二级缓存转移到一级缓存中。

当然，需要将完成加载的 BeanA 对象返回给 getBean 方法的调用处。

至此，完成了 BeanA 和与它循环依赖的 BeanB 的加载，保障了 BeanA 和 BeanB 的单例状态，也没有出现无限递归的情况。

## 总结

循环依赖问题的本质是：

使用递归尝试加载一个 bean 的时候，发现依赖的 bean 也依赖自己，出现了当前 bean 的重复加载，并造成程序的无限递归。

Spring 使用了三级缓存来解决循环依赖问题，所谓三级缓存依次对应以下三个map：

- DefaultSingletonBeanRegistry#singletonObjects （一级缓存，保存加载完成的 bean）
- DefaultSingletonBeanRegistry#earlySingletonObjects （二级缓存，保存被依赖引用的 bean）
- DefaultSingletonBeanRegistry#singletonFactories （三级缓存，保存未被依赖引用的 bean 工厂）

例子中的 BeanA 在加载过程中有三个阶段：实例化、处理依赖、完成

实例化阶段位于 三级缓存；

处理依赖时，如果被依赖bean所依赖，则会放到二级缓存；

处理完所有的依赖，初始化完成，放到一级缓存。




