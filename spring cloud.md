spring cloud

1、feign

1. 原理

   1. ```java
      @Import(FeignClientsRegistrar.class)
      public @interface EnableFeignClients {//启动类上加入此注解，开始feign
      ```

   2. ```java
      FeignClientsRegistrar导入组件
          registerDefaultConfiguration(metadata, registry);//导入default的configuration【全局配置】
      	registerFeignClients(metadata, registry);
      //1、获取所有需要使用feign的类，1、@FeignClient标注的接口，2、@EnableFeignClients的clients配置的接口，调用registerFeignClient()方法，注入FeignClientFactoryBean进容器
      //2、将每个@FeignClient对应的配置注入进容器
      
      // FeignClientFactoryBean
      @Override
      public Object getObject() throws Exception {
          FeignContext context = applicationContext.getBean(FeignContext.class);
          Feign.Builder builder = feign(context);
      ```

   3.  

      ```java
      @Configuration
      @ConditionalOnClass(Feign.class)
      @EnableConfigurationProperties({FeignClientProperties.class, FeignHttpClientProperties.class})
      public class FeignAutoConfiguration {//feign自动配置类
      
      	@Autowired(required = false)
      	private List<FeignClientSpecification> configurations = new ArrayList<>();//自动装配配置【FeignClientsRegistrar通过registerDefaultConfiguration注入的配置】
          
          //注入FeignContext，在FeignClientFactoryBean的getObject()中使用
          @Bean
      	public FeignContext feignContext() {
      		FeignContext context = new FeignContext();
      		context.setConfigurations(this.configurations);
      		return context;
      	}
      ```

         

   