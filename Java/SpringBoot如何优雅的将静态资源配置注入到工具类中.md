---
title: SpringBoot如何优雅的将静态资源配置注入到工具类中
date: 2017/12/23 14:06:25
---

资源注入类:

```java
@Configuration
@ConfigurationProperties(locations = "classpath:/config/qcloud.properties", ignoreUnknownFields = true, prefix = "qcloud")
public class QCloudProperties {
	public static class properties {
	}
	
	private String appid;
	private String secretId;
	private String secretKey;
	private String bucketName;
	private Strign bucketLocation;
	
	public QCloudProperties() {
	}
}
```

工具类：

```java
@Component
public class QCloudFileUtils {
	@Resource
	private QCloudProperties qCloudPropertiesAutowired;
	
	private static QCloudProperties qCloudProperties;
	
	@PostConstruct
	public void init() {
		qCloudProperties = this.qCloudPropertiesAutowired;
	}
	
	public static boolean upload() {
		String appid = qCloudProperties.getAppid();
		return false;
	}
}
```

> https://my.oschina.net/vright/blog/826184

