---
title: Spring启动过程分析6(finishRefresh)
date: 2018/01/02 19:58:00
---

finishRefresh方法调用LifecycleProcessor的onRefresh()方法，发送ContextRefreshedEvent
<!--more-->
`AbstractApplicationContext.finishRefresh`:

```java
protected void finishRefresh() {
	// 实例化lifecycle processor
	initLifecycleProcessor();
	// 调用lifecycle
	getLifecycleProcessor().onRefresh()
	
	publishEvent(new ContextRefreshedEvent(this));
	LiveBeansView.registerApplicationContext(this);
}
```

## onRefresh

`onRefresh`调用的是`DefaultLifecycleProcessor.onRefresh`:

```java
public void onRefresh() {
	startBeans(true);
	this.running = true;
}
```

- DefaultLifecycle.startBeans(boolean autoStartupOnly)

	```java
	private void startBeans(boolean autoStartupOnly) {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<Integer, LifecycleGroup>();
		for (Map.Entry<String, ? extends Lifecycle> entry : lifecycleBeans.entrySet()) {
			Lifecycle bean = entry.getValue();
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(entry.getKey(), bean);
			}
		}
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<Integer>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
	```

- DefaultLifecycleProcessor.start()

	```java
	public void start() {
		if (this.members.isEmpty()) {
			return;
		}
		if (logger.isInfoEnabled()) {
			logger.info("Starting beans in phase " + this.phase);
		}
		Collections.sort(this.member);
		for (LifecycleGroupMember member : this.members) {
			if (this.lifecycleBeans.containsKey(member.name)) {
				doStart(this.lifecycleBeans, member.name, this.autoStartupOnly);
			}
		}
	}
	```

- DefaultLifecycleProcessor.doStart

	```java
	private void doStart(Map<String, ? extends Lifecycle> lifecycleBeans, String beanName, boolean autoStartupOnly) {
		Lifecycle bean = lifecycleBeans.remove(beanName);
		if (bean != null && !this.equals(bean)) {
			String[] dependenciesForBean = this.beanFactory.getDependenciesForBean(beanName);
			for (String dependency : dependenciesForBean) {
				doStart(lifecycleBeans, dependency, autoStartupOnly);
			}
			if (!bean.isRunning() && (!autoStartupOnly || !(bean instanceof SmartLifecycle) || ((SmartLifecycle) bean).isAutoStartup())) {
				if (logger.isDebugEnabled()) {
					logger.debug("Starting bean '" + beanName + "' of type [" + bean.getClass() + "]");
				}
				try {
					bean.start();
				}
				catch (Throwable ex) {
					throw new ApplicationContextException("Failed to start bean '" + beanName + "'", ex);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Successfully started bean '" + beanName + "'");
				}
			}
		}
	}
	```
	
	可以看到最终调用的Lifecycle接口的start方法

