---
title: "解决 Spring Boot Process finished with exit code 0的问题"
date: 2018-05-07
categories: ["Spring Boot"]
tags: ["Spring Boot"]
---

## 问题描述
在启动Spring Boot项目的时候，直接返回`Process finished with exit code 0`退出了，无法正常启动。
```
 .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.0.2.RELEASE)


Process finished with exit code 0
```
## 原因
通过debug spring boot的启动过程，在spring boot启动过程中任何的错误，都会通过`org.springframework.boot.SpringApplication#handleRunFailure`这个方法处理启动过程中的异常，并且通过`org.springframework.boot.SpringApplication#reportFailure`来报告启动异常的信息。

```
private void reportFailure(Collection<SpringBootExceptionReporter> exceptionReporters,
			Throwable failure) {
		try {
			for (SpringBootExceptionReporter reporter : exceptionReporters) {
				if (reporter.reportException(failure)) {
					registerLoggedException(failure);
					return;
				}
			}
		}
		catch (Throwable ex) {
			// Continue with normal handling of the original failure
		}
		if (logger.isErrorEnabled()) {
			logger.error("Application run failed", failure);
			registerLoggedException(failure);
		}
	}
```
该方法的源代码中捕获了`catch (Throwable ex)`是一个空实现，而后面的`logger.isErrorEnabled()`由于logger的配置并没有输出错误日志。

所以在这个方法内部打个断点，通过查看`failure`变量，就可以找到启动过程中真正的异常。
我的情况则是由于拷贝了一个`RestController`，而没有配置Spring bean的名称，造成有两个相同名称的bean，造成启动失败。

## 解决方法
知道了无法启动的真正原因，为拷贝出来的`RestController`设置一个其他的名称，就可以正常启动了。