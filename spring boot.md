# spring boot

## 1、spring boot helloword 原理

### 1、pom文件

#### 1、父项目

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.9.RELEASE</version>
</parent>
  
spring-boot-starter-parent的父项目
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>1.5.9.RELEASE</version>
    <relativePath>../../spring-boot-dependencies</relativePath>
</parent>
spring-boot-dependencies的意义：用来管理spring boot应用里面的所有依赖版本
```

spring boot的版本依赖仲裁

以后导入的依赖默认是不需要写版本的（没有在dependencies里面管理的依赖仍然需要声明版本）

#### 2、启动类

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**spring-boot-starter**-web：

​	spring-boot-starter：spring-boot场景启动器；帮助我们导入web模块正常运行所依赖的组件

springboot将所有的功能都抽取出来，做成一个个的starter（启动器），只需要在项目里面引入这些starter，相关场景的所有依赖就会自动导入进来，要用什么功能就导入什么场景的启动器

### 2、主程序类，主入口类

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class);
    }
}
```

#### 1、@SpringBootApplication

该注解标注的类就是springboot的主配置类，springboot会运行这个类的main方法来启动springboot应用

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

##### 1、@SpringBootConfiguration

该注解标注的类，表示这是一个springboot的配置类

​	@Configuration：配置类上标注这个注解【配置类-----配置文件】

```java
@Configuration
public @interface SpringBootConfiguration {


@Component
public @interface Configuration {
```

##### 2、@EnableAutoConfiguration

开启自动配置功能

​	以前我们需要配置的东西，springboot会帮我们自动配置【**@EnableAutoConfiguration**就是告诉springboot要开启自动配置功能，这样自动配置才能生效】

```java
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    
    
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
```

###### 1、@AutoConfigurationPackage

​		@Import(AutoConfigurationPackages.Registrar.class)

​				spring的底层注解@import：给容器中导入组件，导入的组件由AutoConfigurationPackages.Registrar.class导入。

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
	static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

		@Override
		public void registerBeanDefinitions(AnnotationMetadata metadata,
				BeanDefinitionRegistry registry) {
			register(registry, new PackageImport(metadata).getPackageName());
		}
```

> ***AutoConfigurationPackages.Registrar.class：将主配置类（@SpringBootApplication标注的类）所在包及下面所有子包里面的所有组件扫描到spring容器中***

###### 2、@Import(EnableAutoConfigurationImportSelector.class)

ImportSelector：将所有需要导入到组件以全类名的方式返回，这些组件就会被添加到容器中

EnableAutoConfigurationImportSelector.class：会给容器中导入非常多的自动配置类（xxxAutoConfiguration）:就是给容器中导入这个场景需要的所有组件，并配置好这些组件

![1570954586735](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1570954586735.png)

有了自动配置类，免去了我们手动编写配置注入功能组件等的工作

​	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class, classLoader);

> `spring boot 在启动的时候从类路径下的META-INF/spring.factories中获取EnableAutoConfiguration指定的值，将这些值作为自动配置类导入到容器中，自动配置类就生效，帮我们进行自动配置工作。`

J2EE：spring-boot-autoconfigure-1.5.9.RELEASE.jar



@ResponseBody标注在方法上，则**这个方法**返回的对象或者字符串都可以直接输出到页面
@ResponseBody标注在类上，则**这个类中的方法**返回的对象或者字符串都可以直接输出到页面

**@ResponseBody+@Controller=@RestController**

## 2、配置

### 1、yaml配置

#### 1、语法

todo...

#### 2、@ConfigurationProperties和@Value

- 区别

|                      | @ConfigurationProperties                         | @Value       |
| :------------------- | :----------------------------------------------- | ------------ |
| 功能                 | 批量注入配置文件中的属性                         | 一个个的注入 |
| 松散绑定（松散语法） | 支持【**userName\user-name\user_name的等价的**】 | 不支持       |
| SpEL                 | 不支持                                           | 支持         |
| JSR303数据校验       | 支持【**@Validated**】                           | 不支持       |
| 复杂类型封装         | 支持                                             | 不支持       |

@ConfigurationProperties：用在复杂JavaBean的赋值【**默认从全局配置文件中加载**】

@Value：简单取值

- 说明

  - @ConfigurationProperties生效的原因：ConfigurationPropertiesBindingPostProcessor
  - @EnableConfigurationProperties的使用： 
    -    @EnableConfigurationProperties(HttpEncodingProperties.class)：可以将HttpEncodingProperties注入到容器中，并设置好属性【HttpEncodingProperties类上必须要有@ConfigurationProperties注解】
    - EnableConfigurationProperties做了两件事
      - 1、将此注解上value设置的类加载到容器中【value设置的类，必须要标注@ConfigurationProperties注解】
      - 2、将ConfigurationPropertiesBindingPostProcessor类加载到容器中

  ```java
  1、
      @EnableConfigurationProperties(HttpEncodingProperties.class)
  
  2、
      @Import(EnableConfigurationPropertiesImportSelector.class)
  	public @interface EnableConfigurationProperties {
  
  
  3、
      class EnableConfigurationPropertiesImportSelector implements ImportSelector {
  
  	@Override
  	public String[] selectImports(AnnotationMetadata metadata) {
  		MultiValueMap<String, Object> attributes = metadata.getAllAnnotationAttributes(
  				EnableConfigurationProperties.class.getName(), false);
  		Object[] type = attributes == null ? null
  				: (Object[]) attributes.getFirst("value");
  		if (type == null || type.length == 0) {
  			return new String[] {
  					ConfigurationPropertiesBindingPostProcessorRegistrar.class
  							.getName() };
  		}
  		return new String[] { ConfigurationPropertiesBeanRegistrar.class.getName(),
  				ConfigurationPropertiesBindingPostProcessorRegistrar.class.getName() };
  	}
  
  3.1、
  	public static class ConfigurationPropertiesBeanRegistrar
  			implements ImportBeanDefinitionRegistrar {
  
  		@Override
  		public void registerBeanDefinitions(AnnotationMetadata metadata,
  				BeanDefinitionRegistry registry) {
  			MultiValueMap<String, Object> attributes = metadata
  					.getAllAnnotationAttributes(
  							EnableConfigurationProperties.class.getName(), false);
  			List<Class<?>> types = collectClasses(attributes.get("value"));
  			for (Class<?> type : types) {
  				String prefix = extractPrefix(type);
  				String name = (StringUtils.hasText(prefix) ? prefix + "-" + type.getName()
  						: type.getName());
  				if (!registry.containsBeanDefinition(name)) {
  					registerBeanDefinition(registry, type, name);
  				}
  			}
  		}
  3.2
      public class ConfigurationPropertiesBindingPostProcessorRegistrar
  		implements ImportBeanDefinitionRegistrar {
  	@Override
  	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
  			BeanDefinitionRegistry registry) {
  		if (!registry.containsBeanDefinition(BINDER_BEAN_NAME)) {
  			BeanDefinitionBuilder meta = BeanDefinitionBuilder
  					.genericBeanDefinition(ConfigurationBeanFactoryMetaData.class);
  			BeanDefinitionBuilder bean = BeanDefinitionBuilder.genericBeanDefinition(
  					ConfigurationPropertiesBindingPostProcessor.class);
  			bean.addPropertyReference("beanMetaDataStore", METADATA_BEAN_NAME);
  			registry.registerBeanDefinition(BINDER_BEAN_NAME, bean.getBeanDefinition());
  			registry.registerBeanDefinition(METADATA_BEAN_NAME, meta.getBeanDefinition());
  		}
  	}
  ```

  



#### 3、@PropertySource、@ImportResource和@Configuration的区别

|      | @PropertySource    | @ImportResource                                       | @Configuration                                        |
| ---- | ------------------ | ----------------------------------------------------- | ----------------------------------------------------- |
| 作用 | 加载指定的配置文件 | 导入spring的配置文件，让配置文件里面的内容生效【xml】 | 添加组件【**spring boot推荐给容器中添加组件的方式**】 |



### 4、profile

#### 1、激活profile方式

1. 配置文件

   1. properties配置文件：配置application-**dev**.properties，然后在application.properties指定spring.profiles.active=dev
   2. yml配置文件：分段配置

   ```yml
   server:
     port: 8081
   spring:
     profiles:
       active: dev
   ---
   spring:
     profiles: dev #指定环境
   server:
     port: 8082
   ---
   spring:
     profiles: test #指定环境
   server:
     port: 8083
   ```

2. 命令行参数：--spring.profiles.active=dev

3. 虚拟机参数：-Dspring.profiles.active=dev

### 5、配置文件加载位置【非jar】

springboot启动会扫描一下位置的application.properties/yml文件作为springboot的默认配置文件

- file:/config/

- file:./

- classpath:/config/

- classpath:/

  以上位置是按照==优先级从高到低==的顺序，所有位置的文件都会被加载，高优先级配置内容会**覆盖**低优先级配置内容，形成**互补配置**

  可通过**spring.config.location**来修改默认配置路径
  
  项目打包好之后，我们可以使用命令行参数的形式，启动项目的时候来指定配置文件的新位置；指定配置文件和默认加载的这些配置文件会共同起作用，形成互补配置。

### 6、外部配置加载顺序【jar】

总结：

- ==sprong boot 也可以从以下位置加载配置，优先级从高到低，高优先的配置**覆盖**低优先级的配置，所有诶之会形成互补配置==

- ==优先加载带profile==
- ==由jar包外向jar包内进行查找==：外部优先级高的原因：因为内部的配置是在打包的是打进去的，是默认的，而外部的是再不用重复打包的情况下，覆盖默认配置的一种方式

明细：

1. [Devtools global settings properties](https://docs.spring.io/spring-boot/docs/2.1.9.RELEASE/reference/html/using-boot-devtools.html#using-boot-devtools-globalsettings) on your home directory (`~/.spring-boot-devtools.properties` when devtools is active).
2. [`@TestPropertySource`](https://docs.spring.io/spring/docs/5.1.10.RELEASE/javadoc-api/org/springframework/test/context/TestPropertySource.html) annotations on your tests.
3. `properties` attribute on your tests. Available on [`@SpringBootTest`](https://docs.spring.io/spring-boot/docs/2.1.9.RELEASE/api/org/springframework/boot/test/context/SpringBootTest.html) and the [test annotations for testing a particular slice of your application](https://docs.spring.io/spring-boot/docs/2.1.9.RELEASE/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-tests).
4. ==Command line arguments.==
5. Properties from `SPRING_APPLICATION_JSON` (inline JSON embedded in an environment variable or system property).
6. `ServletConfig` init parameters.
7. `ServletContext` init parameters.
8. JNDI attributes from `java:comp/env`.
9. Java System properties (`System.getProperties()`).
10. OS environment variables.
11. A `RandomValuePropertySource` that has properties only in `random.*`.
12. ==[Profile-specific application properties](https://docs.spring.io/spring-boot/docs/2.1.9.RELEASE/reference/html/boot-features-external-config.html#boot-features-external-config-profile-specific-properties) outside of your packaged jar (`application-{profile}.properties` and YAML variants).==
13. ==[Profile-specific application properties](https://docs.spring.io/spring-boot/docs/2.1.9.RELEASE/reference/html/boot-features-external-config.html#boot-features-external-config-profile-specific-properties) packaged inside your jar (`application-{profile}.properties` and YAML variants).==
14. ==Application properties outside of your packaged jar (`application.properties` and YAML variants).==
15. ==Application properties packaged inside your jar (`application.properties` and YAML variants).==
16. [`@PropertySource`](https://docs.spring.io/spring/docs/5.1.10.RELEASE/javadoc-api/org/springframework/context/annotation/PropertySource.html) annotations on your `@Configuration` classes.
17. Default properties (specified by setting `SpringApplication.setDefaultProperties`).

### 7、自动配置原理

示例：HttpEncodingAutoConfiguration

```java
@Configuration //表示当前类是一个配置类，可以给容器中添加组件
@EnableConfigurationProperties({HttpEncodingProperties.class}) //在容器中添加HttpEncodingProperties，并设置好其属性
@ConditionalOnWebApplication
@ConditionalOnClass({CharacterEncodingFilter.class})//CharacterEncodingFilter：springmvc 中进行乱码解决的过滤器
@ConditionalOnProperty(
    prefix = "spring.http.encoding",
    value = {"enabled"},
    matchIfMissing = true
)
public class HttpEncodingAutoConfiguration {
```

在application.properties中可以配置的属性都是在xxx自动配置类的xxxProperties类中封装的

​	配置文件中能配置什么就需要参考某个功能对应的这个属性配置类

@Condition注解：根据不同的条件判断，决定配置类是否生效

一旦这个配置类生效，这个配置类就会给容器加各种组件，这些组件的属性是从对应的properties类中获取的，这些类里面的每个属性又是和配置文件绑定的



**spring boot的精髓**

1. springboot启动会加载大量的自动配置类
2. 我们看我们需要的功能有没有在springboot默认写好的自动配置类
3. 如果存在自动配置类，则再看这个自动配置类中到底配置了哪些组件，只要我们要用的组件有，就不需要再来配置
4. 给容器中自动配置类添加组件的时候，会从properties类中获取某些属性，我们就可以在配置文件中指定这些属性的值

```java
xxxAutoConfiguration：自动配置类
xxxProperties：封装配置文件中的相关属性
```

**加入spring容器的bean，如果只有一个有参的构造器，则spring会将有参构造器的参数从容器中获取并传入**

==在application.properties中配置debug=true，可以使控制台打印自动配置报告，这样我们就可以很方便的知道哪些自动配置类生效，也可以使用actuator==



## 3、日志

**logback.xml和logback-spring.xml区别**：

​	logback.xml是日志框架可识别的配置

​	logback-spring.xml是spring可识别的配置，可使用**profile特性**

### 1、日志适配器：sfl4j

- slf4j的具体使用

  ![1573270280147](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1573270280147.png)

- 浅蓝色：抽象层；
  浅绿色：适配层；
  深蓝色：具体实现层

- 若需要使用log4j，则需要使用适配层。
  适配层的作用：因为log4j编写时还没有slf4j，其本身是不支持slf4j的，所以需要使用适配层做中转

- 每个日志的实现框架都有自己的配置文件，使用slf4j后，配置文件还是用日志实现框架自身的配置文件

### 2、日志框架

- log4j
- logback
- log4j2
- jul【java.util.logging】

### 3、日志框架的统一【应用依赖的框架由于历史原因没有使用slf4j，直接使用了日志具体实现框架】

 ![img](http://www.slf4j.org/images/legacy.png) 

- eg：spring（commons-logging）,hibernate（jboss-logging）
- jcl-over-slf4j.jar、log4j-over-slf4j.jar、jul-to-slf4j.jar是日志具体实现框架的替换包
  替换包的作用：不同的依赖包可能使用到不到的日志框架，需要依赖包使用的框架替换下，使其支持slf4j【与适配包的区别：适配包并不会覆盖日志实现框架，替换包会直接覆盖日志实现框架的代码】
- ==**统一日志的步骤**==：
  1、将系统中其他日志框架先排除出去
  2、使用替换包替换原有的日志框架
  3、导入slf4j和日志的实现包

### 4、spring boot使用日志

#### 1、application.properties的配置

- logging.level.package
- logging.file：只指定文件名，则在当前项目下生产日志文件；若指定了路径，则在相应路径下生成日志
- logging.path：与logging.file是冲突的，logging.file优先级高【springboot默认日志名称为spring.log】
- logging.pattern.console：控制台日志输出的格式
- logging.pattern.file：文件日志输出的格式

#### 2、配置

- 使用日志配置文件：在resource添加日志配置文件

  - 日志框架的配置文件

    | Logging System          | Customization                                                |
    | ----------------------- | ------------------------------------------------------------ |
    | Logback                 | `logback-spring.xml`, `logback-spring.groovy`, `logback.xml`, or `logback.groovy` |
    | Log4j2                  | `log4j2-spring.xml` or `log4j2.xml`                          |
    | JDK (Java Util Logging) | `logging.properties`                                         |

- logback.xml和logback-spring.xml区别：logback.xml是日志框架可识别的配置，logback-spring.xml是spring可识别的配置，可使用profile特性

  ![1573270896121](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1573270896121.png)

#### 3、切换日志框架

1. slf4j+logback 转成 slf4j+**log4j**
   1. 排除logback依赖
   2. 排除log4j的替换包
   3. 加入log4j的适配包【含log4j的jar】
   4. 加入log4j的配置文件
2. slf4j+logback 转成 slf4j+**log4j2**
   1. 排除spring-boot-starter-loggin
   2. 加入spring-boot-starter-log4j2

## 4、web开发

### 1、简介

使用springboot

1. 创建springboot应用，选中我们需要的模块
2. springboot已经默认将这些场景配置hapless，只需要在配置文件中指定少量配置就可以运行起来
3. 自己编写业务代码

自动配置原理：

1. ​	这个场景springboot帮我们配置了什么？能不能修改？能修改哪些配置？能不能扩展？

   xxxAutoConfiguration

   xxxProperties

### 2、springboot对静态资源的映射规则

静态资源配置

```java
@ConfigurationProperties(prefix = "spring.resources", ignoreUnknownFields = false)
public class ResourceProperties implements ResourceLoaderAware {
```

org.springframework.boot.autoconfigure.web.WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter#addResourceHandlers

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
        return;
    }
    Integer cachePeriod = this.resourceProperties.getCachePeriod();
    if (!registry.hasMappingForPattern("/webjars/**")) {
        customizeResourceHandlerRegistration(
            registry.addResourceHandler("/webjars/**")
            .addResourceLocations(
                "classpath:/META-INF/resources/webjars/")
            .setCachePeriod(cachePeriod));
    }
    String staticPathPattern = this.mvcProperties.getStaticPathPattern();
    if (!registry.hasMappingForPattern(staticPathPattern)) {
        customizeResourceHandlerRegistration(
            registry.addResourceHandler(staticPathPattern)
            .addResourceLocations(
                this.resourceProperties.getStaticLocations())
            .setCachePeriod(cachePeriod));
    }
}
```

1. 所有"/webjars/**"，都去"classpath:/META-INF/resources/webjars/"查找

   [webjars文档](https://www.webjars.com/)

   ![1573885968026](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1573885968026.png)

   - 引入webjars的jar
   - 直接使用：http://localhost:8088/webjars/jquery/3.4.1/jquery.js

2. "/**"访问当前项目的任何资源，

   1. **静态资源文件夹**

   ```
   "classpath:/META-INF/resources/",
   "classpath:/resources/",
   "classpath:/static/",
   "classpath:/public/",
   "/"：当前项目的根目录
   ```

   

3. **欢迎页**：静态资源文件夹里面定义的index.html页面；被“/**”映射

   ```java
   @Bean
   public WelcomePageHandlerMapping welcomePageHandlerMapping(
       ResourceProperties resourceProperties) {
       return new WelcomePageHandlerMapping(resourceProperties.getWelcomePage(),
                                            this.mvcProperties.getStaticPathPattern());
   }
   ```

4. **图标**：所有"**/favicon.ico"都在静态资源文件下查找

   ```java
   @Configuration
   @ConditionalOnProperty(value = "spring.mvc.favicon.enabled", matchIfMissing = true)
   public static class FaviconConfiguration {
   
       private final ResourceProperties resourceProperties;
   
       public FaviconConfiguration(ResourceProperties resourceProperties) {
           this.resourceProperties = resourceProperties;
       }
   
       @Bean
       public SimpleUrlHandlerMapping faviconHandlerMapping() {
           SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
           mapping.setOrder(Ordered.HIGHEST_PRECEDENCE + 1);
           mapping.setUrlMap(Collections.singletonMap("**/favicon.ico",
                                                      faviconRequestHandler()));
           return mapping;
       }
   
       @Bean
       public ResourceHttpRequestHandler faviconRequestHandler() {
           ResourceHttpRequestHandler requestHandler = new ResourceHttpRequestHandler();
           requestHandler
               .setLocations(this.resourceProperties.getFaviconLocations());
           return requestHandler;
       }
   
   }
   ```

   

5. **自定义静态资源文件**：spring.resources.static-locations

### 3、模板引擎

jsp,velocity,thymeleaf,freemarker

spring boot推荐thymeleaf

1. **引入thymeleaf依赖**

   ```xml
   <!--webjars 的jquery-->
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-thymeleaf</artifactId>
   </dependency>
   <!--修改默认版本-->
   <properties>
       <thymeleaf.version>3.0.2.RELEASE</thymeleaf.version>
       <thymeleaf-layout-dialect.version>2.1.1</thymeleaf-layout-dialect.version>
   </properties>
   ```

   

2. thymeleaf使用和语法

   ```java
   @ConfigurationProperties(prefix = "spring.thymeleaf")
   public class ThymeleafProperties {
   
   	private static final Charset DEFAULT_ENCODING = Charset.forName("UTF-8");
   
   	private static final MimeType DEFAULT_CONTENT_TYPE = MimeType.valueOf("text/html");
   
   	public static final String DEFAULT_PREFIX = "classpath:/templates/";
   
   	public static final String DEFAULT_SUFFIX = ".html";
   ```

   只需要将文件放在"classpath:/templates/",thymeleaf就会自动渲染

   使用：

   1. 导入thymeleaf名称空间：<html xmlns:th="http://www.thymeleaf.org">

   2. 语法

      1. th:text：转义【<h1></h1>原样输出】：行内写法：[[]]

      2. th:utext：不转义【<h1></h1>转成一号标题输出】：行内写法：[()]

      3.  th:任意HTML属性，来替换原生属性的值 

         ![1573899932700](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1573899932700.png)

      4. 表达式

      ```properties
      Simple expressions:
          Variable Expressions: ${...}：获取变量值：OGNL
          	1、获取对象的属性，调用方法
          	2、内置对象的基本对象
          		#ctx : the context object.
                  #vars: the context variables.
                  #locale : the context locale.
                  #request : (only in Web Contexts) the HttpServletRequest object.
                  #response : (only in Web Contexts) the HttpServletResponse object.
                  #session : (only in Web Contexts) the HttpSession object.
                  #servletContext : (only in Web Contexts) the ServletContext object.
      		3、内置的工具对象
      			#execInfo : information about the template being processed.
                  #messages : methods for obtaining externalized messages inside variables expressions, in the same way as they would be obtained using #{…} syntax.
                  #uris : methods for escaping parts of URLs/URIs Page 20 of 106
                  #conversions : methods for executing the configured conversion service (if any).
                  #dates : methods for java.util.Date objects: formatting, component extraction, etc.
                  #calendars : analogous to #dates , but for java.util.Calendar objects.
                  #numbers : methods for formatting numeric objects.
                  #strings : methods for String objects: contains, startsWith, prepending/appending, etc.
                  #objects : methods for objects in general.
                  #bools : methods for boolean evaluation.
                  #arrays : methods for arrays.
                  #lists : methods for lists.
                  #sets : methods for sets.
                  #maps : methods for maps.
                  #aggregates : methods for creating aggregates on arrays or collections.
                  #ids : methods for dealing with id attributes that might be repeated (for example, as a result of an iteration).
          Selection Variable Expressions: *{...}：选择表达式，与${}功能上是类似的，配合th:object使用
          	#示例
                  <div th:object="${session.user}">
                      <p>Name: <span th:text="*{firstName}">Sebastian</span>.</p>
                      <p>Surname: <span th:text="*{lastName}">Pepper</span>.</p>
                      <p>Nationality: <span th:text="*{nationality}">Saturn</span>.</p>
                   </div>
          
          Message Expressions: #{...}：获取国际化值
          Link URL Expressions: @{...}：定义URL
          	#示例：
          		@{/order/process(execId=${execId},execType='FAST')}
          Fragment Expressions: ~{...}：片段引入
          	#示例
          		<div th:insert="~{commons :: main}">...</div>
      
      Literals:字面量
          Text literals: 'one text' , 'Another one!' ,…
          Number literals: 0 , 34 , 3.0 , 12.3 ,…
          Boolean literals: true , false
          Null literal: null
          Literal tokens: one , sometext , main ,…
      Text operations:文本操作
          String concatenation: +
          Literal substitutions: |The name is ${name}|
      Arithmetic operations:数学运行
          Binary operators: + , - , * , / , %
          Minus sign (unary operator): -
      Boolean operations:逻辑运算
          Binary operators: and , or
          Boolean negation (unary operator): ! , not
      Comparisons and equality:比较运算
          Comparators: > , < , >= , <= ( gt , lt , ge , le )
          Equality operators: == , != ( eq , ne )
      Conditional operators:条件运算
          If-then: (if) ? (then)
          If-then-else: (if) ? (then) : (else)
          Default: (value) ?: (defaultvalue)
      Special tokens:特殊操作
          No-Operation: _
      ```

### 4、springboot MVC自动配置原理

Spring Boot 自动配置好了Spring MVC.

默认的自动配置:

- Inclusion of `ContentNegotiatingViewResolver` and `BeanNameViewResolver` beans.
- Support for serving static resources, including support for WebJars (see below).
- Automatic registration of `Converter`, `GenericConverter`, `Formatter` beans.
- Support for `HttpMessageConverters` (see below).
- Automatic registration of `MessageCodesResolver` (see below).
- Static `index.html` support.
- Custom `Favicon` support (see below).
- Automatic use of a `ConfigurableWebBindingInitializer` bean (see below).