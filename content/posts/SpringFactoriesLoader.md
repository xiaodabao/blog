---
title: "SpringFactoriesLoader"
date: 2021-09-29T15:16:34+08:00
draft: false
---

SPI, 全称 Service Provider Interface, 是一种服务发现机制。今天来分析一下Spring中的SPI机制是如何实现的。

Spring中的SPI机制是通过类SpringFactoriesLoader来实现的，主要步骤如下:
1. 查找指定路径classpath:META-INF/spring.factories的所有文件。
2. 通过PropertiesLoaderUtils加载为Properties对象
3. 解析Properties中的数据, key为抽象类型interface/abstractType, value按照逗号分割为实现类型的名称集合implementationNames，在这个过程中合并不同spring.factories文件中相同类型的数据
4. 将implementationNames去重，并转换为不可变List
5. 将全部数据以ClassLoader为key存入cache中
6. 根据本次要加载的类型，实例化此类型下所有的实现类
7. 所有的实现类按照PriorityOrdered、Ordered优先级依次排序

首先分析方法 loadSpringFactories(ClassLoader classLoader), 加载所有的classpath:META-INF/spring.factories文件，并将其解析为Map<Interface, Implements> 的形式。
```java
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
static final Map<ClassLoader, Map<String, List<String>>> cache = new ConcurrentReferenceHashMap<>();

/**
* 根据ClassLoader加载所有的数据  key: interfaceName --> value: implementationNames
* @param classLoader
* @return
*/
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
    // 首先从缓存中获取
    Map<String, List<String>> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    result = new HashMap<>();
    try {
        // 在所有的jar包中搜索classpath:META-INF/spring.factories
        Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            // 将url封装为Resource, 并加载到Properties中
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                // key 为 抽象类型 interface、abstractType
                String factoryTypeName = ((String) entry.getKey()).trim();
                // 将实现类使用, 分割为实现类数组
                String[] factoryImplementationNames =
                        StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
                for (String factoryImplementationName : factoryImplementationNames) {
                    // 处理实现类, 如果多个jar包中存在多个相同的接口key, 在这里也会进行合并
                    result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
                            .add(factoryImplementationName.trim());
                }
            }
        }

        // 实现类去重, 并将其设置为不可变List
        result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
                .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
        // 加入缓存
        cache.put(classLoader, result);
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
    return result;
}
```

SpringFactoriesLoader共有两个public方法
1. 第一个根据给定的Class获取其实现类名称List, 方法比较简单，不再详细解释
```java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
}
```

2. 第二个根据给定的Class获取其实现类实例List
```java
public static <T> List<T> loadFactories(Class<T> factoryType, @Nullable ClassLoader classLoader) {
    Assert.notNull(factoryType, "'factoryType' must not be null");
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    // 获取实现类名称集合
    List<String> factoryImplementationNames = loadFactoryNames(factoryType, classLoaderToUse);
    if (logger.isTraceEnabled()) {
        logger.trace("Loaded [" + factoryType.getName() + "] names: " + factoryImplementationNames);
    }
    List<T> result = new ArrayList<>(factoryImplementationNames.size());
    // 实例化所有的实现类
    for (String factoryImplementationName : factoryImplementationNames) {
        result.add(instantiateFactory(factoryImplementationName, factoryType, classLoaderToUse));
    }
    // 按照PriorityOrder、Order排序
    AnnotationAwareOrderComparator.sort(result);
    return result;
}
```

使用反射得到实现类的实例
```java
@SuppressWarnings("unchecked")
private static <T> T instantiateFactory(String factoryImplementationName, Class<T> factoryType, ClassLoader classLoader) {
    try {
        // 获取实现类的Class对象, 因为只配置了名称
        Class<?> factoryImplementationClass = ClassUtils.forName(factoryImplementationName, classLoader);
        // 必须是factoryType的实现类
        if (!factoryType.isAssignableFrom(factoryImplementationClass)) {
            throw new IllegalArgumentException(
                    "Class [" + factoryImplementationName + "] is not 
                    assignable to factory type [" + factoryType.getName() + "]");
        }
        // 使用newInstance 反射的方式创建对象, 必须存在无参构造函数
        return (T) ReflectionUtils.accessibleConstructor(factoryImplementationClass).newInstance();
    }
    catch (Throwable ex) {
        throw new IllegalArgumentException(
                "Unable to instantiate factory class [" + 
                factoryImplementationName + "] for factory type [" + factoryType.getName() + "]",
                ex);
    }
}
```

下面简单说下SpringFactoriesLoader的用法

创建Interface: Color, 实现类: Red、Green

在classpath下创建 META-INF/spring.factories, 内容是：

```
Color=Red,Green
```

测试方法如下:

```java
/**
* 通过自定义 META-INF/spring.factories, 可以使用Spring SPI机制
* 当实现Spring Interface的时候, 就可以进行自定义扩展了
*/
public static void testColor() {
    List<Color> colors = SpringFactoriesLoader.loadFactories(Color.class, null);

    colors.forEach(System.out::println);
}
```