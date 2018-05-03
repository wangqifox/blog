---
title: Zuul探究(三)——Zuul动态路由的实现
date: 2018/05/03 17:01:25
---

在前一篇文章的`路由的初始化`章节，我们探究了Zuul的路由是如何在事件的驱动下刷新路由的。了解了路由的刷新就已经非常接近动态刷新的功能了。

<!-- more -->
通过继承`DiscoveryClientRouteLocator`来实现一个可以读取数据库的路由定位器：

```java
public class CustomRouteLocator extends DiscoveryClientRouteLocator {
    private ZuulRouteMapper zuulRouteMapper;

    public CustomRouteLocator(ServerProperties server, DiscoveryClient discovery, ZuulProperties zuulProperties, ServiceRouteMapper serviceRouteMapper, Registration registration) {
        super(server.getServlet().getServletPrefix(), discovery, zuulProperties, serviceRouteMapper, registration);
    }

    public void setZuulRouteMapper(ZuulRouteMapper zuulRouteMapper) {
        this.zuulRouteMapper = zuulRouteMapper;
    }

    @Override
    protected LinkedHashMap<String, ZuulRoute> locateRoutes() {
        LinkedHashMap<String, ZuulRoute> routesMap = new LinkedHashMap<>();
        routesMap.putAll(super.locateRoutes());
        routesMap.putAll(locateRoutesFromDB());
        return routesMap;
    }

    private LinkedHashMap<String, ZuulRoute> locateRoutesFromDB() {
        LinkedHashMap<String, ZuulRoute> routes = new LinkedHashMap<>();
        List<ZuulRouteVO> results = zuulRouteMapper.selectAll();
        for (ZuulRouteVO zuulRouteVO : results) {
            ZuulRoute zuulRoute = new ZuulRoute();
            zuulRoute.setId(zuulRouteVO.getId());
            zuulRoute.setPath(zuulRouteVO.getPath());
            zuulRoute.setServiceId(zuulRouteVO.getServiceId());
            zuulRoute.setUrl(zuulRouteVO.getUrl());
            zuulRoute.setStripPrefix(zuulRouteVO.getStripPrefix());
            zuulRoute.setRetryable(zuulRouteVO.getRetryAble());
            routes.put(zuulRoute.getPath(), zuulRoute);
        }
        return routes;
    }
}
```

可以看到，在`locateRoutes`方法中首先加载配置文件和Eureka中注册的service，然后读取数据库中的路由数据。

然后再实现一个可以发送刷新事件的service：

```java
@Service
public class RefreshRouteService {
    @Autowired
    ApplicationEventPublisher publisher;
    @Autowired
    RouteLocator routeLocator;

    public void refreshRoute() {
        RoutesRefreshedEvent routesRefreshedEvent = new RoutesRefreshedEvent(routeLocator);
        publisher.publishEvent(routesRefreshedEvent);
    }
}
```

每次加入新的路由时，调用`RefreshRouteService.refreshRoute`方法就可以实现动态刷新路由的功能了。



> https://blog.csdn.net/u013815546/article/details/68944039


