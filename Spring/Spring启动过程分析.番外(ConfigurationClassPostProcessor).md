---
title: Spring启动过程分析.番外(ConfigurationClassPostProcessor)
subtitle: ConfigurationClassPostProcessor的处理过程
date: 2018/01/05 17:19:00
---

`ConfigurationClassPostProcessor`是一个用于处理配置类的处理器，它在Spring启动过程中在`invokeBeanFactoryPostProcessors`方法中被调用：

1. 获取beanFactory中的BeanDefinitionRegistryPostProcessor
2. 筛选实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessor
3. 调用BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法
<!--more-->
下文我们分析一下`ConfigurationClassPostProcessor`的`postProcessBeanDefinitionRegistry`方法做了什么：

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
	List<BeanDefinitionHolder> configCandidates = new ArrayList<BeanDefinitionHolder>();
	// 获取当前beanFactory中所有的beanDefinition的名称
	String[] candidateNames = registry.getBeanDefinitionNames();

	for (String beanName : candidateNames) {
	   //根据bean的名称获取beanDefinition
		BeanDefinition beanDef = registry.getBeanDefinition(beanName);
		if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
				ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
			}
		}
		// 检查这个beanDefinition是否是候选的配置类
		else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
			configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
		}
	}

	// 如果没有被@Configuration注释的类，则直接返回
	if (configCandidates.isEmpty()) {
		return;
	}

	// 根据@Order注释的顺序来排序
	Collections.sort(configCandidates, new Comparator<BeanDefinitionHolder>() {
		@Override
		public int compare(BeanDefinitionHolder bd1, BeanDefinitionHolder bd2) {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;
		}
	});

	// Detect any custom bean name generation strategy supplied through the enclosing application context
	SingletonBeanRegistry sbr = null;
	if (registry instanceof SingletonBeanRegistry) {
		sbr = (SingletonBeanRegistry) registry;
		if (!this.localBeanNameGeneratorSet && sbr.containsSingleton(CONFIGURATION_BEAN_NAME_GENERATOR)) {
			BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
			this.componentScanBeanNameGenerator = generator;
			this.importBeanNameGenerator = generator;
		}
	}

	// 解析每个被@Configuration注释的类
	ConfigurationClassParser parser = new ConfigurationClassParser(
			this.metadataReaderFactory, this.problemReporter, this.environment,
			this.resourceLoader, this.componentScanBeanNameGenerator, registry);

	Set<BeanDefinitionHolder> candidates = new LinkedHashSet<BeanDefinitionHolder>(configCandidates);
	Set<ConfigurationClass> alreadyParsed = new HashSet<ConfigurationClass>(configCandidates.size());
	do {
		// 解析候选的配置类
		parser.parse(candidates);
		parser.validate();

		Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
		configClasses.removeAll(alreadyParsed);

		// Read the model and create bean definitions based on its content
		if (this.reader == null) {
			this.reader = new ConfigurationClassBeanDefinitionReader(
					registry, this.sourceExtractor, this.resourceLoader, this.environment,
					this.importBeanNameGenerator, parser.getImportRegistry());
		}
		// 加载配置文件的bean definition
		this.reader.loadBeanDefinitions(configClasses);
		alreadyParsed.addAll(configClasses);

		candidates.clear();
		if (registry.getBeanDefinitionCount() > candidateNames.length) {
			String[] newCandidateNames = registry.getBeanDefinitionNames();
			Set<String> oldCandidateNames = new HashSet<String>(Arrays.asList(candidateNames));
			Set<String> alreadyParsedClasses = new HashSet<String>();
			for (ConfigurationClass configurationClass : alreadyParsed) {
				alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
			}
			for (String candidateName : newCandidateNames) {
				if (!oldCandidateNames.contains(candidateName)) {
					BeanDefinition bd = registry.getBeanDefinition(candidateName);
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
							!alreadyParsedClasses.contains(bd.getBeanClassName())) {
						candidates.add(new BeanDefinitionHolder(bd, candidateName));
					}
				}
			}
			candidateNames = newCandidateNames;
		}
	}
	while (!candidates.isEmpty());

	// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
	if (sbr != null) {
		if (!sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
			sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
		}
	}

	if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
		((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
	}
}
```

### 检查beanDefinition是否是候选的配置类

检查beanDefinition是否是候选的配置类由`ConfigurationClassUtils.checkConfigurationClassCandidate`完成。

```java
public static boolean checkConfigurationClassCandidate(BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {
	String className = beanDef.getBeanClassName();
	if (className == null) {
		return false;
	}

   // 首先获取beanDefinition中的metadata
	AnnotationMetadata metadata;
	if (beanDef instanceof AnnotatedBeanDefinition &&
			className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
		// Can reuse the pre-parsed metadata from the given BeanDefinition...
		metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
	}
	else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
		// Check already loaded Class if present...
		// since we possibly can't even load the class file for this Class.
		Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
		metadata = new StandardAnnotationMetadata(beanClass, true);
	}
	else {
		try {
			MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
			metadata = metadataReader.getAnnotationMetadata();
		}
		catch (IOException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Could not find class file for introspecting configuration annotations: " + className, ex);
			}
			return false;
		}
	}

   // 检查是否有Configuration注释
	if (isFullConfigurationCandidate(metadata)) {
		// 设置attribute
		beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
	}
	/**
	 * 检查是否有Component、ComponentScan、Import、ImportResource注释
	 * 如果都没有，则检查是否有被Bean注释的方法
	 */
	else if (isLiteConfigurationCandidate(metadata)) {
		beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
	}
	else {
		return false;
	}

	// 设置顺序属性
	Map<String, Object> orderAttributes = metadata.getAnnotationAttributes(Order.class.getName());
	if (orderAttributes != null) {
		beanDef.setAttribute(ORDER_ATTRIBUTE, orderAttributes.get(AnnotationUtils.VALUE));
	}

	return true;
}
```

### 解析配置文件

从beanFactory中筛选出所有的配置文件的定义，放在`Set<BeanDefinitionHolder> candidates`中。

然后新建一个`ConfigurationClassParser`:

```java
ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);
```

用`ConfigurationClassParser`来解析配置文件。

#### parse

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
	this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

   // parse调用processConfigurationClass来进行真正的解析
	for (BeanDefinitionHolder holder : configCandidates) {
		BeanDefinition bd = holder.getBeanDefinition();
		try {
			if (bd instanceof AnnotatedBeanDefinition) {
				parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
			}
			else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
				parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
			}
			else {
				parse(bd.getBeanClassName(), holder.getBeanName());
			}
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(
					"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
		}
	}

	processDeferredImportSelectors();
}
```

##### processConfigurationClass

```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
	if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
		return;
	}

	ConfigurationClass existingClass = this.configurationClasses.get(configClass);
	if (existingClass != null) {
		if (configClass.isImported()) {
			if (existingClass.isImported()) {
				existingClass.mergeImportedBy(configClass);
			}
			// Otherwise ignore new imported config class; existing non-imported class overrides it.
			return;
		}
		else {
			// Explicit bean definition found, probably replacing an import.
			// Let's remove the old one and go with the new one.
			this.configurationClasses.remove(configClass);
			for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext();) {
				if (configClass.equals(it.next())) {
					it.remove();
				}
			}
		}
	}

	// 递归处理配置类及其子类
	SourceClass sourceClass = asSourceClass(configClass);
	do {
		sourceClass = doProcessConfigurationClass(configClass, sourceClass);
	}
	while (sourceClass != null);
  
   // 将处理过的配置类放到configurationClasses中
	this.configurationClasses.put(configClass, configClass);
}
```

###### doProcessConfigurationClass

doProcessConfigurationClass用于处理并建立一个完整的ConfigurationClass

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {

	// 递归处理成员类(内嵌类)
	processMemberClasses(configClass, sourceClass);

	// 处理@PropertySource注释
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), PropertySources.class,
			org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) {
			processPropertySource(propertySource);
		}
		else {
			logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}

	// 处理@ComponentScan注释
	Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
	if (!componentScans.isEmpty() &&
			!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
		for (AnnotationAttributes componentScan : componentScans) {
			// 根据@ComponentScan注释，执行扫描
			Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
			// Check the set of scanned definitions for any further config classes and parse recursively if needed
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				// 检查扫描出来的bean中是否有配置类，如果有则解析配置类
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(holder.getBeanDefinition(), this.metadataReaderFactory)) {
					parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
				}
			}
		}
	}

	// 处理@Import注释
	processImports(configClass, sourceClass, getImports(sourceClass), true);

	// 处理@ImportResource注释
	if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		String[] resources = importResource.getStringArray("locations");
		Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
		for (String resource : resources) {
			String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
			configClass.addImportedResource(resolvedResource, readerClass);
		}
	}

	// 处理单独由@Bean注释的方法
	Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
	for (MethodMetadata methodMetadata : beanMethods) {
		configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
	}

	// 处理接口中的默认方法
	processInterfaces(configClass, sourceClass);

	// 处理子类
	if (sourceClass.getMetadata().hasSuperClass()) {
		String superclass = sourceClass.getMetadata().getSuperClassName();
		if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
			this.knownSuperclasses.put(superclass, configClass);
			// Superclass found, return its annotation metadata and recurse
			return sourceClass.getSuperClass();
		}
	}

	// No superclass -> processing is complete
	return null;
}
```

### 处理配置类中的成员类

- processMemberClasses

    **递归处理处理配置类中的成员类(内嵌类)**
    
    ```java
    private void processMemberClasses(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {
    	// sourceClass.getMemberClasses()获取成员类
		for (SourceClass memberClass : sourceClass.getMemberClasses()) {
			if (ConfigurationClassUtils.isConfigurationCandidate(memberClass.getMetadata()) &&
					!memberClass.getMetadata().getClassName().equals(configClass.getMetadata().getClassName())) {
				if (this.importStack.contains(configClass)) {
					this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
				}
				else {
					this.importStack.push(configClass);
					try {
					    // 递归处理配置类中的成员类，
						processConfigurationClass(memberClass.asConfigClass(configClass));
					}
					finally {
						this.importStack.pop();
					}
				}
			}
		}
	}
    ```

### 处理@PropertySource注释

- AnnotationConfigUtils.attributesForRepeatable

**获取注释的属性**

    ```java
    static Set<AnnotationAttributes> attributesForRepeatable(AnnotationMetadata metadata,
			String containerClassName, String annotationClassName) {

		Set<AnnotationAttributes> result = new LinkedHashSet<AnnotationAttributes>();
		addAttributesIfNotNull(result, metadata.getAnnotationAttributes(annotationClassName, false));

		Map<String, Object> container = metadata.getAnnotationAttributes(containerClassName, false);
		if (container != null && container.containsKey("value")) {
			for (Map<String, Object> containedAttributes : (Map<String, Object>[]) container.get("value")) {
				addAttributesIfNotNull(result, containedAttributes);
			}
		}
		return Collections.unmodifiableSet(result);
	}
    ```
    
    - StandardAnnotationMetadata.getAnnotationAttibutes

        ```java
        public Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString) {
        return (this.annotations.length > 0 ? AnnotatedElementUtils.getMergedAnnotationAttributes(
        		getIntrospectedClass(), annotationName, classValuesAsString, this.nestedAnnotationsAsMap) : null);
        }
        ```
    
    - AnnotatedElementUtils.getMergedAnnotationAttributes
		
		**获取注释属性**	
		
    	```java
    	public static AnnotationAttributes getMergedAnnotationAttributes(AnnotatedElement element,
			String annotationName, boolean classValuesAsString, boolean nestedAnnotationsAsMap) {

			Assert.hasLength(annotationName, "'annotationName' must not be null or empty");
			AnnotationAttributes attributes = searchWithGetSemantics(element, null, annotationName,
					new MergedAnnotationAttributesProcessor(classValuesAsString, nestedAnnotationsAsMap));
			AnnotationUtils.postProcessAnnotationAttributes(element, attributes, classValuesAsString, nestedAnnotationsAsMap);
			return attributes;
	}
    	```

- processPropertySource

	**处理PropertySource注释，将resource包装成PropertySource，添加到AbstractEnvironment的MutablePropertySources propertySources中**

	```java
	private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
		String name = propertySource.getString("name");
		if (!StringUtils.hasLength(name)) {
			name = null;
		}
		String encoding = propertySource.getString("encoding");
		if (!StringUtils.hasLength(encoding)) {
			encoding = null;
		}
		String[] locations = propertySource.getStringArray("value");
		Assert.isTrue(locations.length > 0, "At least one @PropertySource(value) location is required");
		boolean ignoreResourceNotFound = propertySource.getBoolean("ignoreResourceNotFound");

		Class<? extends PropertySourceFactory> factoryClass = propertySource.getClass("factory");
		PropertySourceFactory factory = (factoryClass == PropertySourceFactory.class ?
				DEFAULT_PROPERTY_SOURCE_FACTORY : BeanUtils.instantiateClass(factoryClass));

		for (String location : locations) {
			try {
				String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);
				/**
				 * 根据@PropertySource注释中指定的位置加载资源
				 */
				Resource resource = this.resourceLoader.getResource(resolvedLocation);
				/**
				 * 将resource包装成PropertySource，添加到AbstractEnvironment的MutablePropertySources propertySources中
				 */
				addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
			}
			catch (IllegalArgumentException ex) {
				// Placeholders not resolvable
				if (ignoreResourceNotFound) {
					if (logger.isInfoEnabled()) {
						logger.info("Properties location [" + location + "] not resolvable: " + ex.getMessage());
					}
				}
				else {
					throw ex;
				}
			}
			catch (IOException ex) {
				// Resource not found when trying to open it
				if (ignoreResourceNotFound &&
						(ex instanceof FileNotFoundException || ex instanceof UnknownHostException)) {
					if (logger.isInfoEnabled()) {
						logger.info("Properties location [" + location + "] not resolvable: " + ex.getMessage());
					}
				}
				else {
					throw ex;
				}
			}
		}
	}
	```

- addPropertySource

	**添加PropertySource**
	
	```java
	private void addPropertySource(PropertySource<?> propertySource) {
		String name = propertySource.getName();
		MutablePropertySources propertySources = ((ConfigurableEnvironment) this.environment).getPropertySources();
		if (propertySources.contains(name) && this.propertySourceNames.contains(name)) {
			// We've already added a version, we need to extend it
			PropertySource<?> existing = propertySources.get(name);
			PropertySource<?> newSource = (propertySource instanceof ResourcePropertySource ?
					((ResourcePropertySource) propertySource).withResourceName() : propertySource);
			if (existing instanceof CompositePropertySource) {
				((CompositePropertySource) existing).addFirstPropertySource(newSource);
			}
			else {
				if (existing instanceof ResourcePropertySource) {
					existing = ((ResourcePropertySource) existing).withResourceName();
				}
				CompositePropertySource composite = new CompositePropertySource(name);
				composite.addPropertySource(newSource);
				composite.addPropertySource(existing);
				propertySources.replace(name, composite);
			}
		}
		else {
			if (this.propertySourceNames.isEmpty()) {
				propertySources.addLast(propertySource);
			}
			else {
				String firstProcessed = this.propertySourceNames.get(this.propertySourceNames.size() - 1);
				propertySources.addBefore(firstProcessed, propertySource);
			}
		}
		this.propertySourceNames.add(name);
	}
	```

### 处理@ComponentScan注释

- ComponentScanAnnotationParser.parse

	扫描@ComponentScan注释中指定的位置
	
	```java
	public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
		Assert.state(this.environment != null, "Environment must not be null");
		Assert.state(this.resourceLoader != null, "ResourceLoader must not be null");
	
		// 新建ClassPathBeanDefinitionScanner，用于扫描指定的位置，从而生成各种beanDefinition
		ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
				componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
		
		// 设置名字生成器，默认为AnnotationBeanNameGenerator
		Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
		boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
		scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
				BeanUtils.instantiateClass(generatorClass));

		// 设置代理模式，默认为AnnotationScopeMetadataResolver
		ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
		if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
			scanner.setScopedProxyMode(scopedProxyMode);
		}
		else {
			Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
			scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
		}

		// 设置匹配的资源模式，默认为"**/*.class"
		scanner.setResourcePattern(componentScan.getString("resourcePattern"));

		// 设置包含的过滤器
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addIncludeFilter(typeFilter);
			}
		}
		// 设置排除的过滤器
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addExcludeFilter(typeFilter);
			}
		}

		// 设置是否懒加载
		boolean lazyInit = componentScan.getBoolean("lazyInit");
		if (lazyInit) {
			scanner.getBeanDefinitionDefaults().setLazyInit(true);
		}
		
		// 设置扫描的路径
		Set<String> basePackages = new LinkedHashSet<String>();
		// 将basePackages属性添加到扫描的路径
		String[] basePackagesArray = componentScan.getStringArray("basePackages");
		for (String pkg : basePackagesArray) {
			String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
					ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
			basePackages.addAll(Arrays.asList(tokenized));
		}
		// 将basePackageClasses属性中类所在的package添加到扫描的路径
		for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
			basePackages.add(ClassUtils.getPackageName(clazz));
		}
		// 如果basePackages属性和basePackageClasses属性都不存在，在添加配置类所在的包作为扫描路径
		if (basePackages.isEmpty()) {
			basePackages.add(ClassUtils.getPackageName(declaringClass));
		}

		scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
			@Override
			protected boolean matchClassName(String className) {
				return declaringClass.equals(className);
			}
		});
		// 扫描basePackages
		return scanner.doScan(StringUtils.toStringArray(basePackages));
	}
	```
	
- ClassPathBeanDefinitionScanner.doScan
	
	扫描从@ComponentScan中获取要扫描的路径，生成BeanDefinition

	```java
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
		for (String basePackage : basePackages) {
			// 扫描类的路径，寻找Component，生成beanDefinition
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				// 如果candidate是AbstractBeanDefinition，经过postProcessBeanDefinition处理
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				// 如果candidate是AnnotatedBeanDefinition，经过processCommonDefinitionAnnotations处理
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				// 检查candidate的名称，检查名称是否应该被注册，或者是否和已存在的定义相冲突
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);、
					// 注册beanDefinition，添加在beanFactory的beanDefinitionMap中
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
	```

- ClassPathScanningCandidateComponentProvider.findCandidateComponents

	扫描路径，生成每个类的beanDefinition，以便之后的步骤实例化类

	```java
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
		try {
			/**
			 * 生成一个完整的扫描路径
			 * 其中，ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX为"classpath*:"
			 * resolveBasePackage将"."替换成"/"
			 * this.resourcePattern为"**/*.class"
			 */
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
			// 扫描路径，将目录下的类文件包装成Resource数组
			Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
						MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
						if (isCandidateComponent(metadataReader)) {
							/** 
							 * 将Resource包装成ScannedGenericBeanDefinition，添加到candidates中
							 */
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setResource(resource);
							sbd.setSource(resource);
							// 判断是否是一个Component
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								candidates.add(sbd);
							}
							else {
								if (debugEnabled) {
									logger.debug("Ignored because not a concrete top-level class: " + resource);
								}
							}
						}
						else {
							if (traceEnabled) {
								logger.trace("Ignored because not matching any filter: " + resource);
							}
						}
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to read candidate component class: " + resource, ex);
					}
				}
				else {
					if (traceEnabled) {
						logger.trace("Ignored because not readable: " + resource);
					}
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}
	```

- ClassPathScanningCandidateComponentProvider.isCandidateComponent

	判断是否是一个Component
	
	```java
	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
		for (TypeFilter tf : this.excludeFilters) {
			if (tf.match(metadataReader, this.metadataReaderFactory)) {
				return false;
			}
		}
		for (TypeFilter tf : this.includeFilters) {
			if (tf.match(metadataReader, this.metadataReaderFactory)) {
				return isConditionMatch(metadataReader);
			}
		}
		return false;
	}
	```

- PathMatchingResourcePatternResolver.getResource

	扫描路径，将目录下的类文件包装成Resource
	
	```java
	public Resource[] getResources(String locationPattern) throws IOException {
		Assert.notNull(locationPattern, "Location pattern must not be null");
		if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
			// a class path resource (multiple resources for same name possible)
			if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
				// a class path resource pattern
				return findPathMatchingResources(locationPattern);
			}
			else {
				// all class path resources with the given name
				return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
			}
		}
		else {
			// Generally only look for a pattern after a prefix here,
			// and on Tomcat only after the "*/" separator for its "war:" protocol.
			int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
					locationPattern.indexOf(":") + 1);
			if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
				// a file pattern
				return findPathMatchingResources(locationPattern);
			}
			else {
				// a single resource with the given name
				return new Resource[] {getResourceLoader().getResource(locationPattern)};
			}
		}
	}
	```

- PathMatchingResourcePatternResolver.findPathMatchingResource

	```java
	protected Resource[] findPathMatchingResources(String locationPattern) throws IOException {
		String rootDirPath = determineRootDir(locationPattern);
		String subPattern = locationPattern.substring(rootDirPath.length());
		// 获取根目录下的资源
		Resource[] rootDirResources = getResources(rootDirPath);
		Set<Resource> result = new LinkedHashSet<Resource>(16);
		for (Resource rootDirResource : rootDirResources) {
			rootDirResource = resolveRootDirResource(rootDirResource);
			URL rootDirURL = rootDirResource.getURL();
			/**
			 * 根据url的类型，获取url下的资源
			 */
			if (equinoxResolveMethod != null) {
				if (rootDirURL.getProtocol().startsWith("bundle")) {
					rootDirURL = (URL) ReflectionUtils.invokeMethod(equinoxResolveMethod, null, rootDirURL);
					rootDirResource = new UrlResource(rootDirURL);
				}
			}
			if (rootDirURL.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
				result.addAll(VfsResourceMatchingDelegate.findMatchingResources(rootDirURL, subPattern, getPathMatcher()));
			}
			else if (ResourceUtils.isJarURL(rootDirURL) || isJarResource(rootDirResource)) {
				result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirURL, subPattern));
			}
			else {
				result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
			}
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Resolved location pattern [" + locationPattern + "] to resources " + result);
		}
		return result.toArray(new Resource[result.size()]);
	}
	```

- PathMatchingResourcePatternResolver.findAllClassPathResources

	```java
	protected Resource[] findAllClassPathResources(String location) throws IOException {
		String path = location;
		if (path.startsWith("/")) {
			path = path.substring(1);
		}
		// 寻找path下的所有Resource
		Set<Resource> result = doFindAllClassPathResources(path);
		if (logger.isDebugEnabled()) {
			logger.debug("Resolved classpath location [" + location + "] to resources " + result);
		}
		return result.toArray(new Resource[result.size()]);
	}
	```

- PathMatchingResourceResolver.doFindAllClassPathResource

	```java
	protected Set<Resource> doFindAllClassPathResources(String path) throws IOException {
		Set<Resource> result = new LinkedHashSet<Resource>(16);
		ClassLoader cl = getClassLoader();
		// 获取URL
		Enumeration<URL> resourceUrls = (cl != null ? cl.getResources(path) : ClassLoader.getSystemResources(path));
		while (resourceUrls.hasMoreElements()) {
			URL url = resourceUrls.nextElement();
			// 将URL保证成UrlResource。
			result.add(convertClassLoaderURL(url));
		}
		if ("".equals(path)) {
			// The above result is likely to be incomplete, i.e. only containing file system references.
			// We need to have pointers to each of the jar files on the classpath as well...
			addAllClassLoaderJarRoots(cl, result);
		}
		return result;
	}
	```

- PathMatchingResourcePatternResolver.doFindPathMatchingFileResource

	```java
	protected Set<Resource> doFindPathMatchingFileResources(Resource rootDirResource, String subPattern) throws IOException {

		File rootDir;
		try {
			rootDir = rootDirResource.getFile().getAbsoluteFile();
		}
		catch (IOException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Cannot search for matching files underneath " + rootDirResource +
						" because it does not correspond to a directory in the file system", ex);
			}
			return Collections.emptySet();
		}
		// 寻找符合subPattern的资源
		return doFindMatchingFileSystemResources(rootDir, subPattern);
	}
	```
	
- PathMatchingResourcePatternResolver.doFindMatchingFileSystemResources

	```java
	protected Set<Resource> doFindMatchingFileSystemResources(File rootDir, String subPattern) throws IOException {
		if (logger.isDebugEnabled()) {
			logger.debug("Looking for matching resources in directory tree [" + rootDir.getPath() + "]");
		}
		Set<File> matchingFiles = retrieveMatchingFiles(rootDir, subPattern);
		Set<Resource> result = new LinkedHashSet<Resource>(matchingFiles.size());
		for (File file : matchingFiles) {
			// 将File包装成FileSystemResource
			result.add(new FileSystemResource(file));
		}
		return result;
	}
	```

- PathMatchingResourcePatternResolver.retrieveMatchingFiles

	```java
	protected Set<File> retrieveMatchingFiles(File rootDir, String pattern) throws IOException {
		if (!rootDir.exists()) {
			// Silently skip non-existing directories.
			if (logger.isDebugEnabled()) {
				logger.debug("Skipping [" + rootDir.getAbsolutePath() + "] because it does not exist");
			}
			return Collections.emptySet();
		}
		if (!rootDir.isDirectory()) {
			// Complain louder if it exists but is no directory.
			if (logger.isWarnEnabled()) {
				logger.warn("Skipping [" + rootDir.getAbsolutePath() + "] because it does not denote a directory");
			}
			return Collections.emptySet();
		}
		if (!rootDir.canRead()) {
			if (logger.isWarnEnabled()) {
				logger.warn("Cannot search for matching files underneath directory [" + rootDir.getAbsolutePath() +
						"] because the application is not allowed to read the directory");
			}
			return Collections.emptySet();
		}
		String fullPattern = StringUtils.replace(rootDir.getAbsolutePath(), File.separator, "/");
		if (!pattern.startsWith("/")) {
			fullPattern += "/";
		}
		fullPattern = fullPattern + StringUtils.replace(pattern, File.separator, "/");
		Set<File> result = new LinkedHashSet<File>(8);
		doRetrieveMatchingFiles(fullPattern, rootDir, result);
		return result;
	}
	```

- PathMatchingResourcePatternResolver.doRetrieveMatchingFiles

	```java
	protected void doRetrieveMatchingFiles(String fullPattern, File dir, Set<File> result) throws IOException {
		if (logger.isDebugEnabled()) {
			logger.debug("Searching directory [" + dir.getAbsolutePath() +
					"] for files matching pattern [" + fullPattern + "]");
		}
		// 获取目录下所有的文件
		File[] dirContents = dir.listFiles();
		if (dirContents == null) {
			if (logger.isWarnEnabled()) {
				logger.warn("Could not retrieve contents of directory [" + dir.getAbsolutePath() + "]");
			}
			return;
		}
		Arrays.sort(dirContents);
		for (File content : dirContents) {
			String currPath = StringUtils.replace(content.getAbsolutePath(), File.separator, "/");
			if (content.isDirectory() && getPathMatcher().matchStart(fullPattern, currPath + "/")) {
				if (!content.canRead()) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipping subdirectory [" + dir.getAbsolutePath() +
								"] because the application is not allowed to read the directory");
					}
				}
				else {
					// 递归获取目录下的文件
					doRetrieveMatchingFiles(fullPattern, content, result);
				}
			}
			if (getPathMatcher().match(fullPattern, currPath)) {
				result.add(content);
			}
		}
	}
	```
### 处理@Import注释

- ConfigurationClassParser.getImports

	```java
	private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
		Set<SourceClass> imports = new LinkedHashSet<SourceClass>();
		Set<SourceClass> visited = new LinkedHashSet<SourceClass>();
		collectImports(sourceClass, imports, visited);
		return imports;
	}
	```

- ConfigurationClassParser.processImports

	```java
	private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, boolean checkForCircularImports) throws IOException {

		if (importCandidates.isEmpty()) {
			return;
		}

		if (checkForCircularImports && isChainedImportOnStack(configClass)) {
			this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
		}
		else {
			this.importStack.push(configClass);
			try {
				for (SourceClass candidate : importCandidates) {
					// 如果类实现了ImportSelector
					if (candidate.isAssignable(ImportSelector.class)) {
						// Candidate class is an ImportSelector -> delegate to it to determine imports
						Class<?> candidateClass = candidate.loadClass();
						ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
						ParserStrategyUtils.invokeAwareMethods(
								selector, this.environment, this.resourceLoader, this.registry);
						if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
							this.deferredImportSelectors.add(
									new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
						}
						else {
							// 获取需要导入的类并处理
							String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
							Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
							processImports(configClass, currentSourceClass, importSourceClasses, false);
						}
					}
					// 如果类实现了ImportBeanDefinitionRegistrar，配置类中增加导入的beanDefinition注册器
					else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
						// Candidate class is an ImportBeanDefinitionRegistrar ->
						// delegate to it to register additional bean definitions
						Class<?> candidateClass = candidate.loadClass();
						ImportBeanDefinitionRegistrar registrar =
								BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
						ParserStrategyUtils.invokeAwareMethods(
								registrar, this.environment, this.resourceLoader, this.registry);
						configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
					}
					else {
						// Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
						// process it as an @Configuration class
						this.importStack.registerImport(
								currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
						// 处理导入的配置文件
						processConfigurationClass(candidate.asConfigClass(configClass));
					}
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to process import candidates for configuration class [" +
						configClass.getMetadata().getClassName() + "]", ex);
			}
			finally {
				this.importStack.pop();
			}
		}
	}
	```

## 处理@ImportResource注释

```java
if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
	AnnotationAttributes importResource =
			AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
	String[] resources = importResource.getStringArray("locations");
	Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
	for (String resource : resources) {
		String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
		// 将导入的资源添加到configClass的Map<String, Class<? extends BeanDefinitionReader>> importedResources中
		configClass.addImportedResource(resolvedResource, readerClass);
	}
}
```

## 处理被@Bean注释的方法

```java
// 获取所有被@Bean注释的方法
Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
for (MethodMetadata methodMetadata : beanMethods) {
	// 添加到configClass的Set<BeanMethod> beanMethods中
	configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
}
```

- ConfigurationClassParser.retrieveBeanMethodMetadata

	获取所有被@Bean注释的方法
	
	```java
	private Set<MethodMetadata> retrieveBeanMethodMetadata(SourceClass sourceClass) {
		AnnotationMetadata original = sourceClass.getMetadata();
		Set<MethodMetadata> beanMethods = original.getAnnotatedMethods(Bean.class.getName());
		if (beanMethods.size() > 1 && original instanceof StandardAnnotationMetadata) {
			// Try reading the class file via ASM for deterministic declaration order...
			// Unfortunately, the JVM's standard reflection returns methods in arbitrary
			// order, even between different runs of the same application on the same JVM.
			try {
				AnnotationMetadata asm =
						this.metadataReaderFactory.getMetadataReader(original.getClassName()).getAnnotationMetadata();
				Set<MethodMetadata> asmMethods = asm.getAnnotatedMethods(Bean.class.getName());
				if (asmMethods.size() >= beanMethods.size()) {
					Set<MethodMetadata> selectedMethods = new LinkedHashSet<MethodMetadata>(asmMethods.size());
					for (MethodMetadata asmMethod : asmMethods) {
						for (MethodMetadata beanMethod : beanMethods) {
							if (beanMethod.getMethodName().equals(asmMethod.getMethodName())) {
								selectedMethods.add(beanMethod);
								break;
							}
						}
					}
					if (selectedMethods.size() == beanMethods.size()) {
						// All reflection-detected methods found in ASM method set -> proceed
						beanMethods = selectedMethods;
					}
				}
			}
			catch (IOException ex) {
				logger.debug("Failed to read class file via ASM for determining @Bean method order", ex);
				// No worries, let's continue with the reflection metadata we started with...
			}
		}
		return beanMethods;
	}
	```

## 加载配置文件的bean definition

- ConfigurationClassBeanDefinitionReader.loadBeanDefinition

	```java
	public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
		TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
		for (ConfigurationClass configClass : configurationModel) {
			loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
		}
	}
	```

- ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForConfigurationClass

	```java
	private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

		if (trackedConditionEvaluator.shouldSkip(configClass)) {
			String beanName = configClass.getBeanName();
			if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
				this.registry.removeBeanDefinition(beanName);
			}
			this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
			return;
		}

		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			// 读取配置类中解析获得的beanMethod
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}
		// 加载@ImportResource注释导入的类
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
		// 加载@Import注释导入的ImportBeanDefinitionRegistrar实现类
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
	```


## postProcessBeanFactory

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	int factoryId = System.identityHashCode(beanFactory);
	if (this.factoriesPostProcessed.contains(factoryId)) {
		throw new IllegalStateException(
				"postProcessBeanFactory already called on this post-processor against " + beanFactory);
	}
	this.factoriesPostProcessed.add(factoryId);
	if (!this.registriesPostProcessed.contains(factoryId)) {
		// BeanDefinitionRegistryPostProcessor hook apparently not supported...
		// Simply call processConfigurationClasses lazily at this point then.
		processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
	}

	enhanceConfigurationClasses(beanFactory);
	beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}
```

- ConfigurationClassPostProcessor.enhanceConfigurationClasses

```java
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
	Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<String, AbstractBeanDefinition>();
	for (String beanName : beanFactory.getBeanDefinitionNames()) {
		BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
		if (ConfigurationClassUtils.isFullConfigurationClass(beanDef)) {
			if (!(beanDef instanceof AbstractBeanDefinition)) {
				throw new BeanDefinitionStoreException("Cannot enhance @Configuration bean definition '" +
						beanName + "' since it is not stored in an AbstractBeanDefinition subclass");
			}
			else if (logger.isWarnEnabled() && beanFactory.containsSingleton(beanName)) {
				logger.warn("Cannot enhance @Configuration bean definition '" + beanName +
						"' since its singleton instance has been created too early. The typical cause " +
						"is a non-static @Bean method with a BeanDefinitionRegistryPostProcessor " +
						"return type: Consider declaring such methods as 'static'.");
			}
			configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
		}
	}
	if (configBeanDefs.isEmpty()) {
		// nothing to enhance -> return immediately
		return;
	}
	// 使用ConfigurationClassEnhancer处理配置类
	ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
	for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
		AbstractBeanDefinition beanDef = entry.getValue();
		// If a @Configuration class gets proxied, always proxy the target class
		beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
		try {
			// Set enhanced subclass of the user-specified bean class
			Class<?> configClass = beanDef.resolveBeanClass(this.beanClassLoader);
			Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
			if (configClass != enhancedClass) {
				if (logger.isDebugEnabled()) {
					logger.debug(String.format("Replacing bean definition '%s' existing class '%s' with " +
							"enhanced class '%s'", entry.getKey(), configClass.getName(), enhancedClass.getName()));
				}
				beanDef.setBeanClass(enhancedClass);
			}
		}
		catch (Throwable ex) {
			throw new IllegalStateException("Cannot load configuration class: " + beanDef.getBeanClassName(), ex);
		}
	}
}
```

- ConfigurationClassEnhancer.enhance

	```java
	public Class<?> enhance(Class<?> configClass, ClassLoader classLoader) {
		if (EnhancedConfiguration.class.isAssignableFrom(configClass)) {
			if (logger.isDebugEnabled()) {
				logger.debug(String.format("Ignoring request to enhance %s as it has " +
						"already been enhanced. This usually indicates that more than one " +
						"ConfigurationClassPostProcessor has been registered (e.g. via " +
						"<context:annotation-config>). This is harmless, but you may " +
						"want check your configuration and remove one CCPP if possible",
						configClass.getName()));
			}
			return configClass;
		}
		Class<?> enhancedClass = createClass(newEnhancer(configClass, classLoader));
		if (logger.isDebugEnabled()) {
			logger.debug(String.format("Successfully enhanced %s; enhanced class name is: %s",
					configClass.getName(), enhancedClass.getName()));
		}
		return enhancedClass;
	}
	```

