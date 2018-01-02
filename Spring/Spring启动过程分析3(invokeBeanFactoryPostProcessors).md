---
title: Spring启动过程分析3(invokeBeanFactoryPostProcessors)
---

在Spring容器中找出实现了BeanFactoryPostProcessor接口的processor并执行。Spring容器会委托给PostProcessorRegistrationDelegate的invokeBeanFactoryProcessors方法执行。

```java
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
```

其中`getBeanFactoryPostProcessors()`函数返回了`AbstractApplicationContext`中保存的`beanFactoryPostProcessors`：

```java
public List<BeanFactoryPostProcessor> getBeanFactoryPostProcessors() {
	return this.beanFactoryPostProcessors;
}
```

invokeBeanFactoryPostProcessors代码如下：

```java
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
	
	// 首先调用BeanDefinitionRegistryPostProcessors
	Set<String> processedBeans = new HashSet<String>();
	
	if (beanFactory instanceof BeanDefinitionRegistry) {
		BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
		List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
      	List<BeanDefinitionRegistryPostProcessor> registryPostProcessors = new LinkedList<BeanDefinitionRegistryPostProcessor>();
      
    	for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
        	if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
            	BeanDefinitionRegistryPostProcessor registryPostProcessor = (BeanDefinitionRegistryPostProcessor) postProcessor;
            	registryPostProcessor.postProcessBeanDefinitionRegistry(registry);
            	registryPostProcessors.add(registryPostProcessor);
            } else {
            	regularPostProcessors.add(postProcessor);
            }
    	}
    
    	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    
      	/**
      	 * 实现PriorityOrdered接口的BeanDefinitionRegistryPostProcessor先全部找出来，然后排序后依次执行
      	 */
    	List<BeanDefinitionRegistryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();
      	for (String ppName : postProcessorNames) {
        	if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            	priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
              	processedBeans.add(ppName);
        	}
      	}
      	sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
		registryPostProcessors.addAll(priorityOrderedPostProcessors);
		invokeBeanDefinitionRegistryPostProcessors(priorityOrderedPostProcessors, registry);
      
      	/**
      	 * 实现Ordered接口的BeanDefinitionRegistryPostProcessor找出来，然后排序后依次执行
      	 */
		postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      	List<BeanDefinitionRegistryPostProcessor> orderedPostProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();
    	for (String ppName : postProcessorNames) {
        	if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
            	orderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
              	processedBeans.add(ppName);
        	}
    	}
      	sortPostProcessors(beanFactory, orderedPostProcessors);
      	registryPostProcessors.addAll(orderedPostProcessors);
      	invokeBeanDefinitionRegistryPostProcessors(orderedPostProcessors, registry);
      	
      	/**
      	 * 没有实现PriorityOrdered和Ordered接口的BeanDefinitionRegistryPostProcessor找出来依次执行
      	 */
      	boolean reiterate = true;
      	while (reiterate) {
        	reiterate = false;
          	postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, trye, false);
          	for (String ppName : postProcessorNames) {
            	if (!processedBeans.contains(ppName)) {
                  	BeanDefinitionRegistryPostProcessor pp = beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class);
                  	registryPostProcessors.add(pp);
                  	processedBeans.add(ppName);
                  	pp.postProcessBeanDefinitionRegistry(registry);
                  	reiterate = true;
                }
          	}
      	}
      
      	invokeBeanFactoryPostProcessors(registryPostProcessors, beanFactory);
      	invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
	}
  	else {
    	invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
  	}
  
  	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
  	
  	// 将BeanFactoryPostProcessors分成实现了PriorityOrdered, Ordered的类以及其他的类
  	List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
  	List<String> orderedPostProcessorNames = new ArrayList<String>();
  	List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
  	for (String ppName : postProcessorNames) {
      	if (processedBeans.contains(ppName)) {
        	// 跳过第一阶段已经处理过的
      	}
      	else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        	priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
      	}
      	else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
        	orderedPostProcessorNames.add(ppName);
      	}
      	else {
        	nonOrderedPostProcessorNames.add(ppName);
      	}
  	}
  	
  	// 排序并执行实现了PriorityOrdered的BeanFactoryPostProcessors
  	sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
  	invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);
  
  	// 排序并执行实现了Ordered的BeanFactoryPostProcessors
  	List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
  	for (String postProcessorName : orderedPostProcessorNames) {
    	orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  	}
  	sortPostProcessors(beanFactory, orderedPostProcessors);
  	invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);
  
  	// 执行其他的BeanFactoryPostProcessors
  	List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
  	for (String postProcessorName : nonOrderedPostProcessors) {
    	nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  	}
  	invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);
  
  	beanFactory.clearMetadataCache();
}
```