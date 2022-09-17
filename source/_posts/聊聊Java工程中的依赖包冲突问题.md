---
title: 聊聊 Java 工程中的依赖包冲突问题
date: 2022-08-23 17:35:30
tags: 
    - Java
    - ClassLoader
    - 隔离
---

先说一下背景，我们的项目需要对接很多家银行，每个银行都会给我们一个 SDK 或者干脆一个工程文件, 然后这个这个 SDK 带了一堆依赖包，有的是我们系统中使用到了的，有的是我们系统中没有的，但是即使是使用了的，版本也不尽相同。之前我们尝试过直接让我们的工程依赖银行的 SDK 和它附带的一堆依赖，遇到 jar 包版本冲突后，改改冲突 jar 包的版本，项目也能正常运行。但是后来发现这种办法越来越费劲儿，比如 bcprov 这个包，向后兼容做的一般。为了找到那个既适合银行 A 的依赖版本，又适合银行 B 的依赖版本很难，于是不得不寻找一种更有效的解决办法。

## 方案选择
### 使用类加载器隔离的机制
这个就是说，JVM 不允许同一个 JVM 进程里面存在相同 classLoader 加载的且相同类限定名的 class 存在于一个进程里。也就是说，只要类加载器不一样，是可以有两个同名的类共存于进程里的。我们可以在使用银行 A 的 SDK时，让 classloader A 加载银行 A 的SDK。这样，在加载银行 A 的 SDK 的过程中，他会顺带使用 classloader A 把 SDK 的依赖都加载一遍。而我们要做的主要是告诉 classloader A， SDK-A 及其依赖的路径，但是该路径下是不可能找到 SDK-B 及其依赖的（这个是关键）。  
以下是具体的代码实现：
```java
import java.io.Closeable;
import java.io.File;
import java.io.IOException;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * @author wencheng
 * @since 2022/8/23
 */
public class ChannelSdkManager implements Closeable {

	private final Map<String, URLClassLoader> classLoaderMap = new HashMap<>();

	private final Map<String, String> sdkClassNameMap = new HashMap<>();

	private static volatile ChannelSdkManager channelSdkManager;

	private ChannelSdkManager() {
	}

	public static ChannelSdkManager getInstance() {
		if(channelSdkManager == null) {
			synchronized (ChannelSdkManager.class) {
				if(channelSdkManager == null) {
					channelSdkManager = new ChannelSdkManager();
				}
			}
		}

		return channelSdkManager;
	}

	public void registerSdk(String channelCode, File libPath, String sdkClassName) {
		File[] file = libPath.listFiles();
		URL[] libUrls = Arrays.stream(file).map(f -> {
			try {
				return f.toURI().toURL();
			} catch (MalformedURLException e) {
				throw new RuntimeException(e);
			}
		}).collect(Collectors.toList()).toArray(new URL[file.length]);
		URLClassLoader classLoader = new URLClassLoader(libUrls);
		classLoaderMap.put(channelCode, classLoader);
		sdkClassNameMap.put(channelCode, sdkClassName);
	}

	public Class<?> getSdk(String channelCode) throws ClassNotFoundException {
		if (!classLoaderMap.containsKey(channelCode)) {
			throw new IllegalArgumentException("Channel '" + channelCode + "' not registered");
		}

		Class<?> sdkClass = classLoaderMap.get(channelCode).loadClass(sdkClassNameMap.get(channelCode));
		return sdkClass;
	}

	@Override
	public void close() {
		classLoaderMap.values().forEach(cl -> {
			try {
				cl.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		});
	}

}
```
以下是工程依赖的文件结构：

```
lib
├── channel-a
│   ├── dep.jar
│   └── sdk-a.jar
└── channel-b
    ├── dep.jar
    └── sdk-b.jar
```


## 附录：
[SOFAArk 介绍](https://www.sofastack.tech/projects/sofa-boot/sofa-ark-readme/)