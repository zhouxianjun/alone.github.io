---
title: Spring boot JDBC动态数据源 starter
date: 2019-09-17 10:25:48
tags:
- Java
- Spring boot
- starter
- jdbc
- 动态数据源
- 多数据源

categories:
- 后端
- Java
- Spring
---

在平时开发过程中，很多内部的项目都是直接访问多个数据库，这样平时一个项目一个数据库就不够用了，spring支持多数据源。
现在很流行Spring boot自动配置，我在这里分享一个基于Spring boot starter方式的多数据源（动态自动切换）整合方案。

由于我们需要做一个starter，所以我们参照mybatis-spring-boot-start的方式新建两个工程：
   
### 1、创建 dynamic-datasource-spring-boot-autoconfigure
#### pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.8.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.alone</groupId>
    <artifactId>dynamic-datasource-spring-boot-autoconfigure</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>dynamic-datasource-spring-boot-autoconfigure</name>
    <description>dynamic datasource auto configure</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```
因为我们是一个spring boot项目，所以我们这里需要继承spring boot的parent，然后依赖spring-boot-starter，
和jdbc starter（数据源）、aop starter（切面拦截自动切换数据源）、configuration-processor（为了把实体类的属性在properties中配置的时候可以提示）

#### DynamicDataSourceProperties 配置类
```java
@ConfigurationProperties(prefix = "dynamic.ds")
@Data
public class DynamicDataSourceProperties {
	/**
	 * 数据源map
	 */
	private Map<String, DataSourceProperties> datasource;
	/**
	 * 默认数据源
	 */
	private String defaultDataSource;
	/**
	 * 未找到指定数据源是否使用默认数据源
	 */
	private Boolean notFoundUseDefault = false;

	public boolean haveDefault() {
		return defaultDataSource != null && datasource.containsKey(defaultDataSource);
	}

	public boolean has(String name) {
		return datasource.containsKey(name);
	}
}
```
定义配置属性自动配置类，前缀为`dynamic.ds`, 为了简化操作，我们这里数据源的配置直接使用的是Spring中的`DataSourceProperties`，我们配置类中主要的分为：

1、datasource 配置多个数据源以Key-Value形式，key为数据源名称，Value为`DataSourceProperties`，例如:dynamic.ds.datasource.xxx.url 其中的xxx为key，url为DataSourceProperties中的属性

2、defaultDataSource 默认数据源，如果配置了`spring.datasource.xxx`则使用该数据源为默认数据源。

3、notFoundUseDefault 未找到指定数据源是否使用默认数据源，如果使用了一个不存在（没有在第1点中配置的）的数据源，是否使用默认数据源。

@Data 这个注解为lombok中的，主要是自动生成属性的setter和getter。@ConfigurationProperties 为spring中的属性配置注解，会自动把properties中相匹配的属性注入。

#### DynamicDataSource 继承 AbstractRoutingDataSource
```java
public class DynamicDataSource extends AbstractRoutingDataSource {
	@Override
	protected Object determineCurrentLookupKey() {
		// 返回当前线程中保存的值
		return DynamicDataSourceContextHolder.get();
	}
}
```
Spring JDBC 包中有一个`AbstractRoutingDataSource`类，该类为数据源的路由抽象类，
我们这里主要使用`determineCurrentLookupKey`这个方法，返回一个数据源名称即可。

#### DynamicDataSourceAutoConfiguration 自动初始化配置
```java
@Configuration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DynamicDataSourceProperties.class)
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
@Import(DynamicDataSourceAop.class)
public class DynamicDataSourceAutoConfiguration implements EnvironmentAware {
	private final DynamicDataSourceProperties properties;
	/**
	 * 其它数据源
	 */
	private Map<Object, Object> dataSources;
	/**
	 * 默认数据源
	 */
	private DataSource defaultDateSource;

	public DynamicDataSourceAutoConfiguration(DynamicDataSourceProperties properties) {
		this.properties = properties;
		this.dataSources = properties.getDatasource().entrySet().stream().collect(
				Collectors.toMap(Map.Entry::getKey, entry -> entry.getValue().initializeDataSourceBuilder().build())
		);
		Assert.notEmpty(properties.getDatasource(), "请配置动态数据源");
	}

	@Bean
	public DataSource dataSource() {
		DynamicDataSource dataSource = new DynamicDataSource();
		dataSource.setTargetDataSources(this.dataSources);
		dataSource.setDefaultTargetDataSource(defaultDateSource);
		return dataSource;
	}

	@Override
	public void setEnvironment(Environment environment) {
		if (!properties.haveDefault()) {
			if (hasPrefix(environment, "spring.datasource")) {
				Binder binder = Binder.get(environment);
				defaultDateSource = binder
						.bind("spring.datasource", DataSourceProperties.class)
						.get()
						.initializeDataSourceBuilder()
						.build();
			} else {
				int size = dataSources.size();
				Assert.isTrue(size <= 1, "当前数据源中有多个，但没有指定默认数据源.");
				defaultDateSource = (DataSource) dataSources.values().iterator().next();
			}
		} else {
			defaultDateSource = (DataSource) dataSources.get(properties.getDefaultDataSource());
		}
	}

	private boolean hasPrefix(Environment environment, String prefix) {
		AbstractEnvironment env = (AbstractEnvironment) environment;
		return env.getPropertySources().stream().filter(p -> p instanceof MapPropertySource)
				.anyMatch((p) -> ((MapPropertySource) p).getSource().keySet().stream()
						.anyMatch(k -> k.startsWith(prefix)));
	}
}
```
@Configuration 注解，告诉spring该类是一个配置类，@ConditionalOnClass(DataSource.class) 表示在含有DataSource这个类的情况下有效。
@EnableConfigurationProperties 启用属性配置，@AutoConfigureBefore 表示在某个类初始化之前，这里为DataSourceAutoConfiguration。
@Import 导入其他需要配置的类。
该类实现了EnvironmentAware接口，用于获取用户是否有配置spring.datasource前缀的属性。如果用户没有配置默认数据源，并且配置了spring.datasource则使用该数据源为默认数据源。
构造函数很简单，主要是把配置的数据源初始化，没什么可以讲的。
然后使用@Bean注解暴露了一个DataSource，就是我们上面的DynamicDataSource，这里主要是把我们的动态数据源放入spring容器，并设置数据源集合以及默认数据源，这样在spring的jdbc流程中可以直接获取到该数据源。

#### DynamicDataSourceContextHolder
```java
public class DynamicDataSourceContextHolder {
	private static final ThreadLocal<String> HOLDER = new ThreadLocal<>();

	public static void set(String type) {
		HOLDER.set(type);
	}


	public static String get() {
		return HOLDER.get();
	}


	public static void clear() {
		HOLDER.remove();
	}
}
```
这里主要是利用ThreadLocal的特性，把数据源标示保存在当前线程中，避免多线程操作数据源时互相干扰。

#### TargetDataSource 注解
```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface TargetDataSource {
	String value();
}
```
这个注解主要是配合aop切面自动切换数据源使用的，和@Transaction同理。

#### DynamicDataSourceAop 动态数据源切面
```java
@Aspect
@Order(-1)
@Slf4j
public class DynamicDataSourceAop {
	@Resource
	private DynamicDataSourceProperties properties;

	@Around("@annotation(source)")
	@SneakyThrows
	public Object method(ProceedingJoinPoint joinPoint, TargetDataSource source){
		return around(joinPoint, source);
	}

	@Around("@within(source)")
	@SneakyThrows
	public Object targetClass(ProceedingJoinPoint joinPoint, TargetDataSource source){
		return around(joinPoint, source);
	}

	private Object around(ProceedingJoinPoint joinPoint, TargetDataSource source) throws Throwable {
		String name = source.value();
		if (!properties.has(name)) {
			Assert.isTrue(properties.getNotFoundUseDefault(), "指定的数据源: " + name + " 未配置，并 notFoundUseDefault is false.");
			log.warn("指定的数据源: {} 未配置，使用默认数据源.", name);
			return joinPoint.proceed();
		}
		DynamicDataSourceContextHolder.set(name);
		try {
			return joinPoint.proceed();
		} finally {
			DynamicDataSourceContextHolder.clear();
		}
	}

	@PostConstruct
	public void init() {
		log.info("dynamic datasource aop init ok.");
	}
}
```
这里的@Order必须要比spring事务的order要小，也就是说该切面必须比spring的事务先执行，否则无效。
我们这里使用了两个@Around环绕通知，一个拦截方法上含有TargetDataSource注解的，另一个拦截方法的目标类上含有TargetDataSource注解的。
在调用目标方法之前把需要的数据源标示保存到当前线程中，然后执行目标方法。最后别忘记要把当前线程清空，否则会导致内存溢出。

写到这里，我们的动态数据源的配置类就已经写完了，但是这个时候spring并不认识我们，所以我们需要告诉它：
在resource目录新建META-INF目录并且新建spring.factories文件，内容如下：
```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.alone.spring.boot.autoconfigure.dynamic.datasource.DynamicDataSourceAutoConfiguration
```
告诉spring我们的DynamicDataSourceAutoConfiguration类为自动配置类。

### 2、创建 dynamic-datasource-spring-boot-starter
这个工程可要可不要，这里主要是为了按照spring boot starter的规范，所以才分为了两个工程，如果你嫌麻烦可以不要这个工程即可。
该工程不需要resource以及class目录，只需要pom中依赖第一步的工程即可，并且该工程不需要继承spring boot parent。

### 测试
#### 新建测试工程 dynamic-datasource-demo
#### pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.8.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.alone</groupId>
    <artifactId>dynamic-datasource-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>dynamic-datasource-demo</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alone</groupId>
            <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.46</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```
依赖，我们的dynamic-datasource-spring-boot-starter，并且依赖数据驱动包，我这里用的mysql。

#### 创建 测试服务 DemoService
```java
@Service
public class DemoService {
	@Resource
	private JdbcTemplate jdbcTemplate;

	public void defaultMethod() {
		System.out.println(jdbcTemplate.queryForList("select * from t_order"));
	}

	@TargetDataSource("other")
	public void other() {
		System.out.println(jdbcTemplate.queryForList("select * from tab_flow"));
	}
}
```
第一个 defaultMethod 为默认数据源，第二个我们在方法上添加了@TargetDataSource注解并且赋值为other数据源。

#### application.properties
```properties
dynamic.ds.datasource.other.url = jdbc:mysql://url/other?characterEncoding=utf8&useSSL=false&autoReconnect=true&failOverReadOnly=false&allowMultiQueries=true
dynamic.ds.datasource.other.username = root
dynamic.ds.datasource.other.password = xxx
dynamic.ds.datasource.other.driver-class-name = com.mysql.jdbc.Driver

spring.datasource.url = jdbc:mysql://url/default?characterEncoding=utf8&useSSL=false&autoReconnect=true&failOverReadOnly=false&allowMultiQueries=true 
spring.datasource.username = root
spring.datasource.password = xxx
spring.datasource.driver-class-name = com.mysql.jdbc.Driver
```
大功告成。

全部代码在Github： [dynamic-datasource-spring-boot-starter](https://github.com/zhouxianjun/dynamic-datasource-spring-boot-starter)、[dynamic-datasource-spring-boot-autoconfigure](https://github.com/zhouxianjun/dynamic-datasource-spring-boot-autoconfigure),
也可以直接使用jitpack源：[![](https://jitpack.io/v/zhouxianjun/dynamic-datasource-spring-boot-starter.svg)](https://jitpack.io/#zhouxianjun/dynamic-datasource-spring-boot-starter)，如果你使用了私服仓库请配置你的setting.xml中的mirrorOf为：`*,!jitpack.io`
