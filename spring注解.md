# spring 注解

## 容器

### 组件添加

- 注解

	- @Configuration
	- @ComponentScan

		- includeFilters

			- @Filter

				- TypeFilter接口

		- excludeFilters

	- @Scope

		- 单例模式是懒汉式还是饿汉式

	- @Lazy
	- @Conditional
	- @Import

- 注入bean的方式

	- 1、包扫描+组件标注注解【@Controller、@Service，@Repository、@Component】(适用于自己编写的类)
	- 2、@Bean（适用于导入的第三方包里面的组件）
	- 3、import（快速给容器中导入组件）

		- ImportSelector接口：返回需要导入的组件的全类名数组
		- ImportBeanDefinitionRegistrar接口：手动注册bean到容器中

	- 4、使用spring提供的FactoryBean(工厂bean)

		- 在容器上下文中获取factoryBean默认获得的是getObjectType()中定义的类型
		- 若想获取factoryBean本身类，需要在获取的时候加&前缀，线索：BeanFactory接口中定义FACTORY_BEAN_PREFIX

### bean生命周期
bean的创建----初始化---销毁

- 容器管理bean的生命周期
我们可以自定义初始化和销毁的方法：容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法

	- 1、指定初始化和销毁方法
通过@Bean的init-method和destroy-method
	- 2、通过让bean实现InitializingBean接口,重写afterPropertiesSet()方法
通过实现DisposableBean接口，重写destroy()方法
	- 3、可以使用JSR250
@PostConstruct：在bean创建完成并且属性赋值完成之后来执行初始化的方法
@PreDestroy：在容器销毁bean之前通知我们进行清理工作
	- 4、BeanPostProcessor:bean的后置处理器【这个类会在容器管理的bean初始化前后进行一些处理工作】
 postProcessBeforeInitialization：在初始化之前
 postProcessAfterInitialization：在初始化之后

- 流程

	- 构造（对像创建）
单实例：在容器启动的时候创建对象
多实例：在每次获取的时候创建对象
	- BeanPostProcessor.postProcessBeforeInitialization
	- 初始化：对象创建完成，并赋值好，调用初始化方法
	- BeanPostProcessor.postProcessAfterInitialization
	- 销毁：
单实例：容器关闭的时候
多实例：容器不会管理这个bean，容器不会调用销毁方法

- 源码

	- 遍历得到容器中所有BeanPostProcessor，挨个执行postProcessBeforeInitialization，一旦返回null，跳出循环，不再执行后面的BeanPostProcessor.postProcessBeforeInitialization方法
	- populateBean(beanName, mbd, instanceWrapper);先赋值，再执行之后方法

		- applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		- invokeInitMethods(beanName, wrappedBean, mbd);
		- applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

- BeanPostProcessor在spring中的使用
bean赋值，注入其他组件，@Autowired，生命周期注解功能，@Async

	- ApplicationContextAwareProcessor
	- BeanValidationPostProcessor
	- InitDestroyAnnotationBeanPostProcessor
	- AutowiredAnnotationBeanPostProcessor

### 组件赋值、注入

- @Value：只要是spring管理的bean都可以用@Value【无论是@Component还是@Bean注入到spring的】

	- 1、基本数值
	- 2、可以写SpEL：#{}
	- 3、可以写${}：取出配置文件中的值

		- @PropertySource若出现乱码，则设置encoding = "utf-8"
读取外部配置文件中的K/V保存到运行的环境变量中；加载完外部的配置文件以后使用${}取出配置文件的值

- 自动装配：spring利用依赖注入（DI）,完成IOC容器中各个组件的依赖关系赋值

	- 1、@Autowired：自动注入
	1）默认优先按照 类型 去容器中找对应的组件
	2）如果找到多个相同类型的组件，再将属性的名称作为组件的id去容器中查找
	3）@Qualifier指定需要装配的组件的id，而不是使用属性名
	4）自动装配默认一定要将属性赋值好，没有会报错【使用@Autowired(required=false)设置非必填】
	5）@Primary:让spring进行自动装配的时候。默认使用首选的bean，也可以使用@Qualifier指定需要装配的bean的名字

		- AutowiredAnnotationBeanPostProcessor：解析完成自动装配

	- 2、spring还支持使用@Resource【JSR250】和@Inject【JSR330】
	@Resource可以实现自动装配，默认是按照组件 名称 进行装配，可以使用name指定使用的注入bean的id，但不能使用required和@Primary
	@Inject和@Autowired功能意义，但没有required
	- 3、@Autowired：方法，构造器，属性
都是从容器中获取组件的值

		- 1、标注在方法上

			- set方法上
			- @Bean的入参dog会自动从容器中获取

		- 2、标注在构造器上：如果组件只有一个有参构造器，这个有参构造器的@Autowired可以省略，参数位置的组件还是可以自动从容器中获取

			- person的构造器的入参 加注解
			- person的构造器方法上 加注解

		- 3、放在参数位置

	- 4、自定义组件想要使用spring容器底层的一些组件（ApplicationContex、BeanFactory）
自定义组件实现xxxAware:在创建对象的时候，会调用接口规定的方法注入相关组件
把spring底层一些组件注入到自定义的bean中
xxxAware的功能是使用xxxProcessor实现的【eg:ApplicationContextAware--->ApplicationContextAwareProcessor】

- profile：spring提供的可根据当前环境，动态的激活和切换一系列组件的功能

	- 1、加了环境标识的bean，只有和这个环境被激活的时候才能注册到容器，默认是 default 环境
	- 2、@Profile加在类上，只有在指定的环境时，整个配置类里面的所有配置才能开始生效
	- 3、没有标注环境标识的bean，在任何环境都会加载

### aop

- AOP：【动态代理】
	指在程序运行期间动态的将某段代码切入到指定方法指定位置进行运行的编程方式
	1、导入aop模块：spring aop【spring-aspects】
	2、定义一个业务逻辑类，在业务逻辑执行前，执行后，异常时有打印
	3、定义日志切面类：切面类里面的方法需要动态感知业务类运行到哪里，然后执行通知方法
		通知方法（5个）：
		前置通知(@Before)、
		后置通知(@After)【无论方法正常还是异常结束】
		返回通知(@AfterReturning)、
		异常通知(@AfterThrowing)、
		环绕通知(@Arround)：动态代理，手动推进目标方法进行（joinpoint.proceed()）
	4、给切面类的目标方法标注何时何地运行（通知注解）
	5、将切面类和业务类都加入到容器中
	6、必须告诉spring哪个类是切面类（给切面类加上一个注解：@Aspect）
	7、给配置类加上@EnableAspectJAutoProxy：开启基于注解的aop模式

	- 1、将业务逻辑和切面类都加入到容器中，并告诉spring哪个是切面类（@Aspect）
	- 2、在切面类上的每个通知方法上标注通知注解，告诉spring何时何地运行（切入点表达式）
	- 3、开启基于注解的aop模式，@EnableAspectJAutoProxy
	- @EnableAspectJAutoProxy
1、@EnableAspectJAutoProxy是什么？
	@Import(AspectJAutoProxyRegistrar.class):给容器导入AspectJAutoProxyRegistrar.class
	利用AspectJAutoProxyRegistrar自定义给容器中注册bean:
		internalAutoProxyCreator=AnnotationAwareAspectJAutoProxyCreator
	给容器注册一个AnnotationAwareAspectJAutoProxyCreator

- 原理：看容器中注册了什么组件，这个组件什么时候工作，这个组件工作实现的功能是什么

	- 2、AnnotationAwareAspectJAutoProxyCreator：
重点关注后置处理器（在bean初始化完成前后做事情）：BeanPostProcessor和自动装配：BeanFactoryAware

		- AbstractAutoProxyCreator.setBeanFactory()
AbstractAutoProxyCreator.有后置处理器的逻辑
		- AbstractAdvisorAutoProxyCreator.setBeanFactory()->initBeanFactory()
		- AnnotationAwareAspectJAutoProxyCreator.initBeanFactory()

- 流程：

	- 1、传入配置类，创建ioc容器
	- 2、注册配置类，调用refresh()刷新容器
	- 3、registerBeanPostProcessors(beanFactory);注册bean的后置处理器来方便拦截bean的创建

		- 1、先获取ioc容器已经定义了的需要创建对象的所有beanProcess
		- 2、给容器中加别的BeanPostProcessor
		- 3、优先注册实现了PriorityOrdered接口的BeanPostProcessor
		- 4、再在容器中注册实现了Ordered接口的BeanPostProcessor
		- 5、注册没有实现优先级接口的BeanPostProcessor
		- 6、注册BeanPostProcessor，实际上就是创建BeanPostProcessor对象，保存在容器中：

			- 创建internalAutoProxyCreator的BeanPostProcessor【AnnotationAwareAspectJAutoProxyCreator】
			- 1、创建bean的实例
			- 2、populateBean()给bean的各种属性赋值
			- 3、initializeBean：初始化bean

				- 1、invokeAwareMethods：处理Aware接口的方法回调
				- 2、applyBeanPostProcessorsBeforeInitialization：应用后置处理器的postProcessBeforeInitialization()
				- 3、invokeInitMethods()：执行自定义的初始化方法
				- 4、applyBeanPostProcessorsAfterInitialization()：执行后置处理器的postProcessAfterInitialization()

			- 4、BeanPostProcessor【AnnotationAwareAspectJAutoProxyCreator】创建成功：---》aspectJAdvisorsBuilder【】

		- 7、把BeanPostProcessor注册到BeanFactory中：beanFactory.addBeanPostProcessor(postProcessor);

	- 4、finishBeanFactoryInitialization(beanFactory);完成beanFactory初始化工作：创建剩下的单实例bean
AnnotationAwareAspectJAutoProxyCreator=>InstantiationAwareBeanPostProcessor

		- 1、遍历获取容器中所有的bean，依次创建对象getBean(beanName);

			- 1、getBean()->doGetBean()->getSingleton()

		- 2、创建bean
【AnnotationAwareAspectJAutoProxyCreator在所有bean创建之前会有一个拦截，因为实现了InstantiationAwareBeanPostProcessor，会调用postProcessBeforeInstantiation()】

			- 1、先从缓存中获取当前bean，如果能获取到，说明bean是之前创建过的，直接使用，否则再创建：只要创建好的bean都会被缓存起来
			- 2、createBean():创建bean
【AnnotationAwareAspectJAutoProxyCreator会在任何bean创建之前先尝试返回bean的实例】
BeanPostProcessor是在bean对象创建完成初始化前后调用的
InstantiationAwareBeanPostProcessor是在创建bean实例之前尝试用后置处理器返回对象

				- 1、resolveBeforeInstantiation(beanName, mbdToUse);解析BeforeInstantiation【希望后置处理器在此能返回一个代理对象，如果能返回代理对象则使用，如果不能就继续创建bean】

					- 1、后置处理器先尝试返回对象
bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
	拿到所有后置处理器，如果是InstantiationAwareBeanPostProcessor就执行后置处理器的postProcessBeforeInstantiation()
if (bean != null) {
	bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
}

						- 1、在每个bean创建之前，调用postProcessBeforeInstantiation()
关心MyCaculator【业务类】和MyAdviser【切面类】的创建

							- 1、判断当前bean是否在advisedBeans中（保存了所有需要增强bean）
							- 2、判断当前bean是否是基础类型的Advice、Pointcut、Advisor、AopInfrastructureBean
或者是否是切面（@Aspect）
							- 3、是否需要跳过

								- 1、获取候选的增强器（切面里面的通知方法）【List<Advisor> candidateAdvisors】
每一个封装的通知方法的增强器是InstantiationModelAwarePointcutAdvisor
判断每一个增强器是否是AspectJpointcutAdvisor类型的，是则返回true
								- 2、

						- 2、创建对象

							- postProcessAfterInitialization

								- 如果需要，则包装对象 ：return wrapIfNecessary(bean, beanName, cacheKey);

									- 1、获取当前bean的所有增强器（通知方法）

										- 1、找到所有的增强器
										- 2、获取到能在当前bean使用的增强器（找到那些通知方法是需要切入当前bean方法的）
										- 3、给增强器排序

									- 2、在缓存中设置此bean已被增强：advisedBeans.put(cacheKey, Boolean.TRUE)
									- 3、如果当前bean需要增强，创建bean的代理对象

										- 1、获取所有增强器（通知方法）
										- 2、保存到proxyFactory
										- 3、创建代理对象【spring自动决定代理对象的创建方式】

											- jdk动态代理：JdkDynamicAopProxy(config)
											- cglib动态代理：ObjenesisCglibAopProxy(config)

									- 4、给容器中返回当前组件使用cglib增强了的代理对象
									- 5、以后容器中获取到的就是这个组件的代理对象，执行目标方法的时候，代理对象就是执行通知方法的流程

				- 2、doCreateBean(beanName, mbdToUse, args);真正的去创建一个bean实例【和3.6流程一样】

			- 3、目标方法的执行（容器中保存了组件的代理对象【cglib增强后的对象】，这个对象里面保存了详细的消息【eg:增强器、目标对象】）

				- 1、CglibAopProxy.intercept()拦截目标方法的执行
				- 2、根据ProxyFactory对象获取将要执行的目标方法拦截器链【List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);】

					- 1、List<Object> interceptorList保存所有的拦截器【一个默认的ExposeInvocationInterceptor和4个增强器】
					- 2、遍历所有的增强器，将其转为Interceptor：registry.getInterceptors(advisor)
					- 3、将增强器转为List<MethodInterceptor>：如果增强器是MethodInterceptor，则直接加入集合中，如果不是，使用AdvisorAdapter将增强器转为MethodInterceptor

				- 3、如果没有拦截器链，直接执行目标方法

					- 拦截器链【每一个通知方法又被包装为方法拦截器，利用MethodInterceptor机制处理目标方法】

				- 4、如果有拦截器链，把需要执行的目标对象，目标方法，拦截器链等信息传入创建一个CglibMethodInvocation对象，并调用proceed()
				- 5、拦截器链的触发过程

					- 1、如果没有拦截器，或者执行到最后一个拦截器，则执行目标方法
					- 2、链式获取每一个拦截器，拦截器执行invoke方法，每一个拦截器等待下一个拦截器执行完成返回后再来执行【拦截器链机制，保证通知方法和目标方法的执行顺序】

- 总结

	- 1、@EnableAspectJAutoProxy 开启AOP功能
	- 2、@EnableAspectJAutoProxy 会给容器中注册一个组件AnnotationAwareAspectJAutoProxyCreator
	- 3、AnnotationAwareAspectJAutoProxyCreator是一个后置处理器
	- 4、容器的创建流程

		- 1、registerBeanPostProcessors(beanFactory)注册后置处理器：创建AnnotationAwareAspectJAutoProxyCreator
		- 2、finishBeanFactoryInitialization(beanFactory)初始化剩下的单实例bean

			- 1、创建业务逻辑组件和切面组件
			- 2、AnnotationAwareAspectJAutoProxyCreator 拦截组件的创建过程
			- 3、组件创建完成后，判断组件是否需要增强

				- 是：切面的通知方法，包装成增强器（Advisor），给业务逻辑组件创建一个代理对象（cglib）
				- 否：

	- 5、执行目标方法

		- 1、代理对象执行目标方法
		- 2、CglibAopProxy.intercept()

			- 1、得到目标方法的拦截器链（增强器包装成拦截器MethodInterceptor）
			- 2、利用拦截器的链式机制，依次进入每一个拦截器进行执行
			- 3、效果

				- 正常执行：前置通知-》目标方法-》后置通知-》返回通知
				- 出现异常：前置通知-》目标方法-》后置通知-》异常通知

### 声明式事务

- 示例

	- 1、依赖【spring-jdbc】
	- 2、配置数据源，jdbcTemplate

		- spring 对@Configuration类会特殊处理，给容器加组件的方法，多次调用都只是从容器中找组件

	- 3、给方法上标注@Transactional表示当前方法是一个事务方法
	- 4、@EnableTransactionManagement 开启基于注解的事务管理功能
	- 5、配置事务管理器PlatformTransactionManager

- 原理

	- 1、@EnableTransactionManagement：利用TransactionManagementConfigurationSelector给容器中导入组件（2个）

		- 1、AutoProxyRegistrar

			- 给容器中注册一个组件：InfrastructureAdvisorAutoProxyCreator

				- 继承了InstantiationAwareBeanPostProcessor
				- 利用后置处理器机制在对象创建以后，包装对象，返回一个代理对象（增强器），代理对象执行方法利用拦截器链进行调用

		- 2、ProxyTransactionManagementConfiguration

			- 1、给容器中注册事务增强器：BeanFactoryTransactionAttributeSourceAdvisor

				- 1、事务增强器要用事务注解的信息：AnnotationTransactionAttributeSource（解析事务注解）
				- 2、事务拦截器：TransactionInterceptor（保存了事务属性信息和事务管理器）

					- 实现了MethodInterceptor
					- 在目标方法执行的时候

						- 执行拦截器链

							- 事务拦截器

								- 1、先获取事务相关属性
								- 2、再获取PlatformTransactionManager，如果事先没有添加指定任何TransactionManager，最终会从容器中按照类型获取一个PlatformTransactionManager
								- 3、执行目标方法

									- 如果异常，获取到事务管理器，利用事务管理器回滚操作
									- 如果正常，利用事务管理器，提交事务

## 扩展原理

### BeanFactoryPostProcessor

- BeanPostProcessor：bean后置处理器，bean创建对象初始化前后进行拦截工作的
- BeanFactoryPostProcessor：
beanFactory后置处理器：在BeanFactory标准初始化之后调用，来定制和修改BeanFactory的内容，
所有的bean定义已经保存加载到BeanFactory，但是bean的实例还未创建
- 流程

	- 1、IOC容器创建对象
	- 2、invokeBeanFactoryPostProcessors执行beanFactoryPostProcessor

		- 如何找到所有的beanFactoryPostProcessor并执行他们的方法
		- 1、直接在BeanFactory中找到所有类型是BeanFactoryPostProcessor的组件，并执行他们的方法
		- 2、在初始化创建其他组件前面执行

### BeanDefinitionRegistryPostProcessor

- postProcessBeanDefinitionRegistry()：在所有bean定义信息将要被加载，bean实例还未创建时
- 优先于BeanFactoryPostProcessor执行，利用BeanDefinitionRegistryPostProcessor给容器中再额外添加一些组件
- BeanDefinitionRegistry是 bean定义信息的保存中心，以后beanFactory就是按照BeanDefinitionRegistry里面保存的每一个bean的定义信息创建bean实例的
- 流程

	- 1、IOC容器创建对象
	- 2、invokeBeanFactoryPostProcessors
	- 3、从容器中获取到所有的BeanDefinitionRegistryPostProcessor组件

		- 1、依次触发所有的postProcessBeanDefinitionRegistry()方法
		- 2、再来触发postProcessBeanFactory()方法【BeanFactoryPostProcessor接口声明的】

	- 4、再来从容器中找到BeanFactoryPostProcessor组件，然后依次触发postProcessBeanFactory()方法

### ApplicationListener：监听容器中发布的事件【事件驱动模型开发】
监听ApplicationEvent 及其子类的子事件

-  步骤

	- 1、写一个监听器来监听某个事件（ApplicationEvent及其子类）
	- 2、把监听器加入到容器
	- 3、只要容器中有相关事件的发布，我们就能监听到这个事件

		- ContextRefreshedEvent：容器刷新完成（所有bean都完成创建）会发布这个事件
		- ContextClosedEvent：关闭容器会发布这个事件

	- 4、自定义发布一个事件：context.publishEvent()

- 原理

	- ContextRefreshedEvent事件

		- 1、IOC容器创建对象
		- 2、refresh()->finishRefresh()：容器刷新完成
		- 3、publishEvent()【发布事件流程】

			- 1、获取事件的多播器（派发器）：getApplicationEventMulticaster()

				- 创建

					- refresh()->initApplicationEventMulticaster();
初始化ApplicationEventMulticaster

						- 1、先去容器中找有没有id="applicationEventMulticaster"的组件【beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)】
						- 2、如果没有，则new一个：this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
且加入容器中【beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);】
我们就可以在其他组件要派发事件时，自动注入这个applicationEventMulticaster

			- 2、multicastEvent(applicationEvent, eventType)派发事件
			- 3、获取到所有的ApplicationListener

				- 如果有Executor，可以异步进行派发
				- 否则，同步的执行listener方法invokeListener()

					- 拿到listener回调onApplicationEvent(event)方法

				- 监听器的获取

					- 1、refresh()->registerListeners();

						- 从容器中拿到所有的监听器，把他们注册到applicationEventMulticaster中【getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);】

- @EventListener

	- EventListenerMethodProcessor->SmartInitializingSingleton

		- 1、ioc容器创建对象并refresh()
		- 2、finishBeanFactoryInitialization(beanFactory)初始化剩下的单实例bean

			- 1、先创建所有的单实例bean：getBean()
			- 2、获取所有创建好的单实例bean，判断是否是SmartInitializingSingleton类型的

				- 如果是，则调用smartSingleton.afterSingletonsInstantiated()

					-  1、ApplicationListener<?> applicationListener =
									factory.createApplicationListener(beanName, targetType, methodToUse);
					- 2、context.addApplicationListener(applicationListener);

### spring容器创建过程

- 1、prepareRefresh()

	- initPropertySources();
	- getEnvironment().validateRequiredProperties();

- 2、ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()

	- refreshBeanFactory();

		- DefaultListableBeanFactory beanFactory = createBeanFactory();
		- customizeBeanFactory(beanFactory);
		- loadBeanDefinitions(beanFactory);

	- getBeanFactory();

- 3、prepareBeanFactory(beanFactory);

	- beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	- beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

- 4、postProcessBeanFactory(beanFactory);
- 5、invokeBeanFactoryPostProcessors(beanFactory);

	- BeanDefinitionRegistryPostProcessor

		- PriorityOrdered
		- Ordered
		- 子主题 3

	- BeanFactoryPostProcessor

		- PriorityOrdered
		- Ordered
		- 子主题 3

- 6、registerBeanPostProcessors(beanFactory);

	- beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
	- 接口

		- BeanPostProcessor
		- DestructionAwareBeanPostProcessor
		- InstantiationAwareBeanPostProcessor

			- SmartInstantiationAwareBeanPostProcessor

		- MergedBeanDefinitionPostProcessor【internalPostProcessors】

	- beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));

- 7、initMessageSource();

	- DelegatingMessageSource dms = new DelegatingMessageSource();
	- beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);

- 8、initApplicationEventMulticaster();

	- this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
	- beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);

- 9、onRefresh();
- 10、registerListeners();
- 11、finishBeanFactoryInitialization(beanFactory);

	- getBean(name)

		- doGetBean(name, null, null, false)

			- getSingleton(beanName);
			- createBean(beanName, mbd, args);

				- doCreateBean(beanName, mbdToUse, args)

					- Object bean = instanceWrapper.getWrappedInstance();
					- applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);

						- MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition(mbd, beanType, beanName);

					- (beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
					- populateBean(beanName, mbd, instanceWrapper);
					- initializeBean(beanName, exposedObject, mbd);

						- invokeAwareMethods(beanName, bean);

							- BeanNameAware
							- BeanClassLoaderAware
							- BeanFactoryAware

						- applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
						- invokeInitMethods(beanName, wrappedBean, mbd);
						- applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

- 12、finishRefresh();

## servlet3.0

### 注解

- @WebServlet：直接在servlet类上加注解，不需要在web.xml中配置
- @WebFilter
- @WebListener

### 8.2.4Shared libraries / runtimes pluggability【共享库和运行时插件功能】

- 1、servlet容器（tomcat）启动会扫描：当前应用里面每一个jar包的ServletContainerInitializer的实现
- 2、提供ServletContainerInitializer的实现类：
必须绑定在：META-INF/services/javax.servlet.ServletContainerInitializer
文件的内容就是ServletContainerInitializer实现类的全类名
- 总结：容器在启动应用的时候，会扫描当前应用每一个jar包里面的META-INF/services/javax.servlet.ServletContainerInitializer指定的实现类，启动并运行这个实现类的方法，传入感兴趣的类型

	- ServletContainerInitializer
	- @HandlesTypes

- 注册三大组件

	- 1、使用servletContext注册web组件（Servlet,Filter,Listener）

	- 2、使用编码的方式，在项目启动的时候给ServletContext添加组件【必须在项目启动的时候添加】
- 1、ServletContainerInitializer得到ServletContext可添加组件
		- 2、ServletContextListener得到的ServletContext可添加组件

### spring mvc【servlet与springmvc整合】

- xml配置
- spring-web
- 原理

	- 1、web容器在启动的时候，会扫描每个jar包下的META-INF/services/javax.servlet.ServletContainerInitializer
	- 2、加载这个文件指定的类：org.springframework.web.**Spring**ServletContainerInitializer
	- 3、spring的应用一启动就会加载@HandlesTypes({WebApplicationInitializer.class})接口下的所有组件
	- 4、并且为WebApplicationInitializer组件创建对象（组件不是接口，不是抽象类）

		- 1、AbstractContextLoaderInitializer

			- 创建根容器：createRootApplicationContext()

		- 2、AbstractDispatcherServletInitializer

			- 创建一个web的IOC容器：createServletApplicationContext();
			- 创建了DispatcherServlet：createDispatcherServlet(servletAppContext);
			- 将创建的DispatcherServlet添加到ServletContext中：servletContext.addServlet(servletName, dispatcherServlet)，并设置映射：registration.addMapping(this.getServletMappings());

		- 3、AbstractAnnotationConfigDispatcherServletInitializer

			- 创建根容器：createRootApplicationContext()

				- 传入一个配置类：getRootConfigClasses();

			- 创建web的IOC容器：createServletApplicationContext();

				- 传入一个配置类：getServletConfigClasses();

		- 总结：

			- 以注解方式来启动springMvc：继承AbstractAnnotationConfigDispatcherServletInitializer，实现抽象方法，指定DispatcherServlet的配置信息

- 实现

	- spring整合servlet--》springmvc

	- xml配置

		- <mvc:default-servlet-handler>：将springmvc处理不了的请求交给tomcat：eg:静态资源
		- <mvc:annotation-driver>：springmvc的高级功能开启
		- <mvc:interceptors>
		- <mvc:view-controller>

	- 注解方式定制springmvc

		- 1、@EnableWebMvc：开启springmvc定制配置功能【等价于<mvc:annotation-driver>】

流程：

servle **----->** spring与servlet整合成为springmvc **----->** 如何定制springmvc【但是没有说明这些配置生效的原理？是不是在springmvc中有讲解呢？】

 ![img](https://img-blog.csdn.net/20180522162744760?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbndlaWhwdQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 

### 异步请求

@Async注解与springMVC的异步的**区别**

1. @Async注解标注的方法的异步，相当于自己开启一个线程执行任务，并不会影响springMVC的响应
2. **servlet的异步相当于对springMVC响应的“回调”**

#### springMVC的异步方式

1. 返回Callable<response对象>
   1. 控制器返回一个callable
   2. spring异步处理，将callable提交到TaskExecutor，使用一个隔离的线程执行
   3. DispatchServlet和所有的Filter退出web容器的线程，但是response保持打开状态
   4. callable返回结果，springMVC将请求重新派发给容器，恢复之前的处理
   5. 根据callable返回的结果，springMVC继续进行视图渲染等（从接收请求-视图渲染）
2. 返回DeferredResult<response对象>【相当于mq消息中间件】

#### 异步的拦截器：

1、原生api的AsyncListener
2、spring mvc: 实现AsyncHandlerInterceptor