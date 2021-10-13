---
title: "SpringResource"
date: 2021-10-11T19:04:33+08:00
draft: false
---

在Spring中, 所有的class, 配置文件等都被抽象为资源, 即Spring的Resource接口. 今天主要来分析Spring加载class文件的过程.

Spring是通过ComponentScan这个注解来配置需要扫描的包路径，而ComponentScanAnnotationParser类就是用来解析ComponentScan注解的. 具体的步骤为：

1. 入口为ComponentScanAnnotationParser.parse, 解析ComponentScan, 获取配置的basePackages和basePackageClasses(会被转为包路径)属性. 当未配置时, 获取启动类的包路径.
2. 接着调用ClassPathBeanDefinitionScanner的doScan方法
3. 调用父类ClassPathScanningCandidateComponentProvider.findCandidateComponents -> scanCandidateComponents
4. 调用PathMatchingResourcePatternResolver.getResources方法, 在这个方法中, 因为我们传入的locationPattern参数是classpath*:为前缀的值, 所以最终调用findPathMatchingResources方法
5. 在findPathMatchingResources方法中, 首先加载目录的Resource(底层还是调用ClassLoader.getResources方法), 然后遍历目录, 根据得到的URL的协议(file、jar、vfs等)分别进行资源加载
6. 本次分析Jar和FileSystem文件资源的加载过程.

首先分析FileSystem资源的加载, 在本地IDE进行调试时, 本项目内的class文件都会最终加载为FileSystemResource, 路径为target/classes/目录下的class文件。

由ClassLoader.getResources(path)方法可以得到资源目录的URL, 并将其封装为UrlResource, 代码如下:

```java
/**
* 查找指定路径, 并将其转为URLResource, 路径可能在多个位置存在, 所以返回Set集合
* @param path  包的相对路径 例如: org/example/simple
* @return
* @throws IOException
*/
private static Set<Resource> doFindAllClassPathResource(String path) throws IOException {
    Set<Resource> result = new LinkedHashSet<>(16);

    ClassLoader cl = FileSystemResourceTest.class.getClassLoader();
    Enumeration<URL> resourceUrls = cl.getResources(path);

    while (resourceUrls.hasMoreElements()) {
        URL url = resourceUrls.nextElement();
        result.add(convertClassLoaderURL(url));
    }
    return result;
}

/**
* 将url封装为UrlResource
* @param url
* @return
*/
private static Resource convertClassLoaderURL(URL url) {
    return new UrlResource(url);
}
```

将上一步得到的UrlResource作为 rootDirResource, 将下面的所有文件都递归查询出来, 封装为FileSystemResource

```java
private static Resource[] getResource(String basePackagePath) throws IOException {

    // 根据basePackage先获取目录级资源
    Set<Resource> rootDirResources = doFindAllClassPathResource(basePackagePath);

    Set<Resource> result = new LinkedHashSet<>(16);
    for (Resource rootDirResource : rootDirResources) {
        // 依次加载每个根目录下的资源
        Set<Resource> fileResources = doFindPathMatchingFileResources(rootDirResource);
        result.addAll(fileResources);
    }

    return result.toArray(new Resource[0]);
}

private static final Set<Resource> doFindPathMatchingFileResources(Resource rootDirResource) throws IOException {
    File rootDir;
    try {
        // 查找文件的绝对路径
        rootDir = rootDirResource.getFile().getAbsoluteFile();
    } catch (Exception ex) {
        ex.printStackTrace();
        return Collections.emptySet();
    }
    return doFindMatchingFileSystemResources(rootDir);
}

/**
    * 查找根目录下的所有文件, 并将其转为FileSystemResource
    * @param rootDir
    * @return
    * @throws IOException
    */
private static final Set<Resource> doFindMatchingFileSystemResources(File rootDir) throws IOException {
    Set<File> matchingFiles = retrieveMatchingFiles(rootDir);

    Set<Resource> result = new LinkedHashSet<>(matchingFiles.size());
    for (File file : matchingFiles) {
        result.add(new FileSystemResource(file));
    }
    return result;
}
```

下面分析如何根据文件的绝对路径来查询所有文件, 即retrieveMatchingFile(rootDir)方法

```java
/**
* 查询目录下的所有文件
* @param rootDir
* @return
* @throws IOException
*/
private static final Set<File> retrieveMatchingFiles(File rootDir) throws IOException {
    if (!rootDir.exists()) {
        return Collections.emptySet();
    }

    if (!rootDir.isDirectory()) {
        return Collections.emptySet();
    }

    if (!rootDir.canRead()) {
        return Collections.emptySet();
    }
    Set<File> result = new LinkedHashSet<>(8);
    doRetrieveMatchingFiles(rootDir, result);
    return result;
}

/**
* 递归查询目录下的所有文件
* @param dir
* @param result
* @throws IOException
*/
private static final void doRetrieveMatchingFiles(File dir, Set<File> result) throws IOException {
    for (File content : listDirectory(dir)) {
        if (content.isDirectory()) {
            // 对可读目录执行递归操作, 获取所有文件
            if (content.canRead()) {
                doRetrieveMatchingFiles(content, result);
            } else {
                // ignore
            }
        }
        // 暂时忽略match的问题, 只要是文件, 就返回
        if (content.isFile()) {
            result.add(content);
        }
    }
}

/**
* 列出指定目录下的所有文件
* @param dir
* @return
*/
private static final File[] listDirectory(File dir) {
    File[] files = dir.listFiles();
    if (files == null) {
        return new File[0];
    }
    Arrays.sort(files, Comparator.comparing(File::getName));
    return files;
}
```

至此FileSystemResource解析完毕, 可以看到, 最终是通过File.listFiles()方法, 加上递归操作获取了根目录下的所有文件, 这里我们忽略了match的过程, 毕竟我们是要加载**.*class文件, 不符合条件的文件是不必加载的, 但这里我们只研究Resource的加载过程, 所以只要是文件, 我们都作为Resource返回.

下面分析Jar包中的class文件如何加载, 参数basePackage为jar包中的路径的话, 解析出的rootDirResource的protocol是jar, Spring会判断这个protocol, 进而调用Jar中class文件的加载

我们直接省略判断过程, 指定basePackage为 org/springframework/core/ 加载SpringCore中的class文件, 代码如下:

```java
private static Resource[] getResource(String basePackage) throws IOException {

    // 根据basePackage先获取目录级资源
    Set<Resource> rootDirResources = doFindAllClassPathResource(basePackage);

    Set<Resource> result = new LinkedHashSet<>(16);
    for (Resource rootDirResource : rootDirResources) {
        // 这里根据rootDirResource开始加载其下的classResource
        Set<Resource> fileResources = doFindPathMatchingJarResources(rootDirResource, rootDirResource.getURL());
        result.addAll(fileResources);
    }

    return result.toArray(new Resource[0]);
}

/**
* 查找指定路径, 并将其转为URLResource, 路径可能在多个位置存在, 所以返回Set
* @param path
* @return
* @throws IOException
*/
private static Set<Resource> doFindAllClassPathResource(String path) throws IOException {
    Set<Resource> result = new LinkedHashSet<>(16);

    ClassLoader cl = JarResourceTest.class.getClassLoader();
    Enumeration<URL> resourceUrls = cl.getResources(path);

    while (resourceUrls.hasMoreElements()) {
        URL url = resourceUrls.nextElement();
        result.add(convertClassLoaderURL(url));
    }
    return result;
}

private static Resource convertClassLoaderURL(URL url) {
    return new UrlResource(url);
}
```

上段代码和FileSystemResource加载根目录资源的代码基本一样, 唯一不同在于doFindPathMatchingJarResources方法

```java
/**
* 加载Jar包中rootDirResource中的class文件
* @param rootDirResource
* @param rootDirURL
* @return
* @throws IOException
*/
private static Set<Resource> doFindPathMatchingJarResources(Resource rootDirResource, URL rootDirURL) throws IOException {

    URLConnection con = rootDirURL.openConnection();
    JarFile jarFile;
    String jarFileUrl;
    String rootEntryPath;
    boolean closeJarFile;

    if (con instanceof JarURLConnection) {
        JarURLConnection jarCon = (JarURLConnection) con;
        ResourceUtils.useCachesIfNecessary(jarCon);

        jarFile = jarCon.getJarFile();
        jarFileUrl = jarCon.getJarFileURL().toExternalForm();
        JarEntry jarEntry = jarCon.getJarEntry();
        rootEntryPath = (jarEntry != null ? jarEntry.getName() : "");
        closeJarFile = !jarCon.getUseCaches();
    } else {
        String urlFile = rootDirURL.getFile();
        try {
            int separatorIndex = urlFile.indexOf(ResourceUtils.WAR_URL_SEPARATOR);
            if (separatorIndex == -1) {
                separatorIndex = urlFile.indexOf(ResourceUtils.JAR_URL_SEPARATOR);
            }
            if (separatorIndex != -1) {
                jarFileUrl = urlFile.substring(0, separatorIndex);
                rootEntryPath = urlFile.substring(separatorIndex + 2);
                jarFile = getJarFile(jarFileUrl);
            } else {
                jarFile = new JarFile(urlFile);
                jarFileUrl = urlFile;
                rootEntryPath = "";
            }
            closeJarFile = true;
        } catch (ZipException ex) {
            return Collections.emptySet();
        }
    }

    try {
        if (StringUtils.hasLength(rootEntryPath) && !rootEntryPath.endsWith("/")) {
            rootEntryPath = rootEntryPath + "/";
        }

        Set<Resource> result = new LinkedHashSet<>(8);
        // 迭代jarFile.entries, 通过createRelative转化为UrlResource(因为rootDirResource是UrlResource的实例)
        for (Enumeration<JarEntry> entries = jarFile.entries(); entries.hasMoreElements();) {
            JarEntry entry = entries.nextElement();
            String entryPath = entry.getName();
            if (entryPath.startsWith(rootEntryPath)) {
                String relativePath = entryPath.substring(rootEntryPath.length());
                if (relativePath.endsWith("class")) {
                    result.add(rootDirResource.createRelative(relativePath));
                }
            }
        }
        return result;
    } finally {
        if (closeJarFile) {
            jarFile.close();
        }
    }
}

private static JarFile getJarFile(String jarFileUrl) throws IOException {
    if (jarFileUrl.startsWith(ResourceUtils.FILE_URL_PREFIX)) {
        try {
            return new JarFile(ResourceUtils.toURI(jarFileUrl).getSchemeSpecificPart());
        } catch (URISyntaxException ex) {
            return new JarFile(jarFileUrl.substring(ResourceUtils.FILE_URL_PREFIX.length()));
        }
    } else {
        return new JarFile(jarFileUrl);
    }
}
```

可以看到是通过得到JarFile, 并遍历其中的entries, 并封装成UrlResource的实例来完成Resource的加载. 

注意在FileSystemResource的listDirectory中有一段对文件排序的代码, Arrays.sort(files, Comparator.comparing(File::getName)); 但是在加载Jar文件时, 只是遍历, 并没有排序。这在本地运行和打包后在服务器运行的代码加载顺序有可能不同, 有时会造成隐形偶发的bug, 主要表现在一些静态加载的资源被后续文件依赖时.

具体代码已上传至 https://github.com/xiaodabao/artofsource 代码库中