---
title: SpringBootJpa自定义ID主键策略注册到Hibernate默认策略列表中（自定义全局主键策略）
date: 2022-03-04 10:55:46
updated: 2022-03-04 10:55:46
tags:
- SpringBoot
- Jpa
- Hibernate
---

[如何在Hibernate JPA EntityManager中注册自定义IdentifierGenerator？](https://stackoverflow.com/questions/34731783/how-to-register-custom-identifiergenerator-in-hibernate-jpa-entitymanager)



在使用SpringBootJpa自定义主键策略的时候，通常需要使用三个注解：

```java
@javax.persistence.Id
@javax.persistence.GeneratedValue(generator = "id")
@org.hibernate.annotations.GenericGenerator(name = "id", strategy = "com.xxx.XXXIdentifierGenerator")
private String id;
```

此时的第三个注解 `@org.hibernate.annotations.GenericGenerator(name = "id", strategy = "com.xxx.xxx.XXX")` 的 `strategy` 参数需要写完整的对象包名信息。



实际上 Hibernate 中已经预设了一组策略可以提供使用，例如使用UUID作为主键，此时也是需要使用三个注解：

```java
@javax.persistence.Id
@javax.persistence.GeneratedValue(generator = "system-uuid")
@org.hibernate.annotations.GenericGenerator(name = "system-uuid", strategy = "uuid")
private String id;
```

> 注：GeneratedValue 的 generator 参数需要与 GenericGenerator 的 name 参数保持一致

这时候 `GenericGenerator` 的 `strategy` 参数只需要填写一个关键词就行了。

这是因为 Hibernate 在 DefaultIdentifierGeneratorFactory 中已经为我们预设了一组可以正常使用的策略：

```java
public DefaultIdentifierGeneratorFactory() {
    register( "uuid2", UUIDGenerator.class );
    register( "guid", GUIDGenerator.class );
    register( "uuid", UUIDHexGenerator.class );
    register( "uuid.hex", UUIDHexGenerator.class );
    register( "assigned", Assigned.class );
    register( "identity", IdentityGenerator.class );
    register( "select", SelectGenerator.class );
    register( "sequence", SequenceStyleGenerator.class );
    register( "seqhilo", SequenceHiLoGenerator.class );
    register( "increment", IncrementGenerator.class );
    register( "foreign", ForeignGenerator.class );
    register( "sequence-identity", SequenceIdentityGenerator.class );
    register( "enhanced-sequence", SequenceStyleGenerator.class );
    register( "enhanced-table", TableGenerator.class );
}
```

正是由于上面的配置使得我们 `strategy` 参数可以不写完整的包名，只需要一个关键字就行了。



那么在 SpringBootJpa 中我们如何把自己编写的主键策略注册到 `DefaultIdentifierGeneratorFactory` 中，让我们在使用自己的主键策略的时候也可以让 `strategy` 参数不写完整的策略包名，而是像系统策略一样直接使用一个关键字就行了呢？

**先看结论，结论如下：**

那就是向SpringBoot中注入一个Bean ： `HibernatePropertiesCustomizer` 来向 Hibernate 追加参数信息，代码如下：

```java
@Component
public class SnowflakeHibernatePropertiesCustomizer implements HibernatePropertiesCustomizer {
    @Override
    public void customize(Map<String, Object> hibernateProperties) {
        hibernateProperties.put(
                AvailableSettings.IDENTIFIER_GENERATOR_STRATEGY_PROVIDER,
                new IdentifierGeneratorStrategyProvider(){
                    @Override
                    public Map<String, Class<?>> getStrategies() {
                    	Map<String, Class<?>> map = new HashMap<>();
                    	map.put("snowflake", XXXXXIdentifierGenerator.class);
                        return map;
                    }
                });
    }
}

```



**看完结论，再说过程**

接下来说一下上面的代码如何生效的。

**以下过程采用逆向断点跟踪操作，从最后面的代码往前推断。**



我们通过 `DefaultIdentifierGeneratorFactory.register(String, Class)` 方法可以发现它在 `EntityManagerFactoryBuilderImpl.configureIdentifierGenerators(StandardServiceRegistry)` 的方法中被调用过，而且整个项目只有这一个在非 `DefaultIdentifierGeneratorFactory` 对象中的调用，因此在这个 `configureIdentifierGenerators` 方法中一定可以追加策略到预设列表中。

```java
private void configureIdentifierGenerators(StandardServiceRegistry ssr) {
	final StrategySelector strategySelector = ssr.getService( StrategySelector.class );

	// apply id generators
	final Object idGeneratorStrategyProviderSetting = configurationValues.remove( AvailableSettings.IDENTIFIER_GENERATOR_STRATEGY_PROVIDER );
	if ( idGeneratorStrategyProviderSetting != null ) {
		final IdentifierGeneratorStrategyProvider idGeneratorStrategyProvider =
				strategySelector.resolveStrategy( IdentifierGeneratorStrategyProvider.class, idGeneratorStrategyProviderSetting );
		final MutableIdentifierGeneratorFactory identifierGeneratorFactory = ssr.getService( MutableIdentifierGeneratorFactory.class );
		if ( identifierGeneratorFactory == null ) {
			throw persistenceException(
					"Application requested custom identifier generator strategies, " +
							"but the MutableIdentifierGeneratorFactory could not be found"
			);
		}
		for ( Map.Entry<String,Class<?>> entry : idGeneratorStrategyProvider.getStrategies().entrySet() ) {
            // =========================================================
			// 在这里被调用
			identifierGeneratorFactory.register( entry.getKey(), entry.getValue() );
    		// =========================================================
		}
	}
}
```

从上面的代码中可以发现它是从 `configurationValues.remove( AvailableSettings.IDENTIFIER_GENERATOR_STRATEGY_PROVIDER );` 这段代码中获取到我们的其他主键策略信息。



那么接下来我们再追踪 `configurationValues` 这个对象，看看如何向 `configurationValues` 对象中添加一个 KEY 为 `AvailableSettings.IDENTIFIER_GENERATOR_STRATEGY_PROVIDER` 值为 `IdentifierGeneratorStrategyProvider.class` 类型的实例对象。

我们首先会找到 `EntityManagerFactoryBuilderImpl.java` 构造方法中有以下几行初始化配置：

```java
final MergedSettings mergedSettings = mergeSettings( persistenceUnit, integrationSettings, ssrBuilder );

// flush before completion validation
if ( "true".equals( mergedSettings.configurationValues.get( Environment.FLUSH_BEFORE_COMPLETION ) ) ) {
	LOG.definingFlushBeforeCompletionIgnoredInHem( Environment.FLUSH_BEFORE_COMPLETION );
	mergedSettings.configurationValues.put( Environment.FLUSH_BEFORE_COMPLETION, "false" );
}

// keep the merged config values for phase-2
this.configurationValues = mergedSettings.getConfigurationValues();
```

我们会发现 `configurationValues` 的值是来自 `persistenceUnit, integrationSettings, ssrBuilder` 三个参数，而此时的代码还在 Hibernate 内部，还没有到 SpringBoot 的部分，因此需要继续往外面跟踪调试代码。接下来就主要跟踪 `integrationSettings` 这个 Map 对象是从哪里来的。



然后我们会找到 `integrationSettings` 这个参数来自 `LocalContainerEntityManagerFactoryBean.java` 的 `getJpaPropertyMap()` 方法，代码如下：

```java
protected EntityManagerFactory createNativeEntityManagerFactory() throws PersistenceException {
	Assert.state(this.persistenceUnitInfo != null, "PersistenceUnitInfo not initialized");

	PersistenceProvider provider = getPersistenceProvider();
	if (provider == null) {
		String providerClassName = this.persistenceUnitInfo.getPersistenceProviderClassName();
		if (providerClassName == null) {
			throw new IllegalArgumentException(
					"No PersistenceProvider specified in EntityManagerFactory configuration, " +
					"and chosen PersistenceUnitInfo does not specify a provider class name either");
		}
		Class<?> providerClass = ClassUtils.resolveClassName(providerClassName, getBeanClassLoader());
		provider = (PersistenceProvider) BeanUtils.instantiateClass(providerClass);
	}

	if (logger.isDebugEnabled()) {
		logger.debug("Building JPA container EntityManagerFactory for persistence unit '" +
				this.persistenceUnitInfo.getPersistenceUnitName() + "'");
	}
    // ==============================================================
    // 关键代码部分
	EntityManagerFactory emf =
			provider.createContainerEntityManagerFactory(this.persistenceUnitInfo, getJpaPropertyMap());
    // ==============================================================
	postProcessEntityManagerFactory(emf, this.persistenceUnitInfo);

	return emf;
}
```

接下来跟踪 `getJpaPropertyMap()` 方法中 `jpaPropertyMap` 这个对象的数据是如何改变的，给 `AbstractEntityManagerFactoryBean.java` 的 216 和 227 设置断点。



然后我们会发现  `getJpaPropertyMap()` 方法中 `jpaPropertyMap` 这个对象在 `EntityManagerFactoryBuilder.Builder.build()` 方法中有两个调用：

```java
entityManagerFactoryBean.getJpaPropertyMap().putAll(EntityManagerFactoryBuilder.this.jpaProperties);
entityManagerFactoryBean.getJpaPropertyMap().putAll(this.properties);
```

这里第一个 `EntityManagerFactoryBuilder.this.jpaProperties` 参数是读取配置文件里面 `spring.jpa.properties` 参数得来的，但是我们在配置文件中无法提供一个 `IdentifierGeneratorStrategyProvider.class` 实例对象，因此需要关注第二个 `this.properties` 里面的配置是如何设置的。



再使用老方法，断点 `this.properties` 的数据是如何更改的，在 `EntityManagerFactoryBuilder.java` 的 171 设置断点。

然后会发现在 `JpaBaseConfiguration.entityManagerFactory(EntityManagerFactoryBuilder)` 方法中有一行代码

```java
public LocalContainerEntityManagerFactoryBean entityManagerFactory(EntityManagerFactoryBuilder factoryBuilder) {
	Map<String, Object> vendorProperties = getVendorProperties();
	customizeVendorProperties(vendorProperties);
	return factoryBuilder.dataSource(this.dataSource).packages(getPackagesToScan()).properties(vendorProperties)
			.mappingResources(getMappingResources()).jta(isJta()).build();
}
```

里面的 `properties(vendorProperties)` 更改了前面（上一个步骤）的 `properties` 参数，接下来通过代码得知 `vendorProperties` 参数是通过调用 `JpaBaseConfiguration.getVendorProperties()` 方法得来的，但是 `getVendorProperties()` 是一个抽象方法，我们找到它的实现类 `HibernateJpaConfiguration.java`

```java
protected Map<String, Object> getVendorProperties() {
	Supplier<String> defaultDdlMode = () -> this.defaultDdlAutoProvider.getDefaultDdlAuto(getDataSource());
	return new LinkedHashMap<>(this.hibernateProperties
			.determineHibernateProperties(getProperties().getProperties(), new HibernateSettings()
					.ddlAuto(defaultDdlMode).hibernatePropertiesCustomizers(this.hibernatePropertiesCustomizers)));
}
```

在这段代码的最后有一个 `hibernatePropertiesCustomizers(this.hibernatePropertiesCustomizers)` 属性定制器，`this.hibernatePropertiesCustomizers` 是一个 `List<HibernatePropertiesCustomizer>` 对象，它是通过构造方法注入一个 `ObjectProvider<HibernatePropertiesCustomizer>` 参数来得到的，而实际上 `ObjectProvider<HibernatePropertiesCustomizer>` 会是一个拥有相同类型的Bean对象列表。



那么到这里基本上就解决了，我们只要在SpringBoot中提供一个实现了 `HibernatePropertiesCustomizer` 的 Bean 对象即可对 `properties` 进行设置、插入数据。



现在再回到 `EntityManagerFactoryBuilderImpl` 中，`configurationValues` 对象中需要一个 KEY 为 `AvailableSettings.IDENTIFIER_GENERATOR_STRATEGY_PROVIDER` 值为 `IdentifierGeneratorStrategyProvider.class` 类型的实例对象，那我们就在 `HibernatePropertiesCustomizer` 中给 `properties` 添加一个这样的 Key-Value 对象信息。

