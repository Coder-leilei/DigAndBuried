NodeManager解析系列二：Container的启动
==============

在分析Container启动源码之前，我们先自己思考一下怎么实现。  

+   NodeManager是调度模块，它复杂接受来自AM的`StartContainer`的请求来启动Container
+   Container是一个进程级别的执行器，NodeManager需要从请求信息中生成进程执行命令和启动脚本
+   NodeManager针对每个Container起一个监控线程，通过该线程阻塞调度起Container执行器进程，并等待直到Container进程退出。
+   NodeManager可以通过向Container进程发送sig来kill掉Container进程
+   Container执行结束以后，NodeManager读取进程结束代码来判断进程是否是正常退出，被kill还是异常退出。

上面简单的陈述了container进程的启停的简单过程过程，其中最重要的内容是：NodeManager中维持一个线程池，针对每个Container请求，创建一个线程并监听线程的结束。我们就这个线程开始进行分析。

###containermanager.launcher.ContainerLaunch
ContainLaunch是位于NodeManager中负责启停Container执行器进程的线程，它继承`Callable<Integer>`接口，返回Integer的值，这个值即为所监控的Container执行器进程退出的`ExitCode`。

首先从最简单的进程退出错误码来看Container状态。

	public enum ExitCode {
		FORCE_KILLED(137),
		TERMINATED(143),
		LOST(154);
	}

Container执行器进程结束可能有下面几种可能状态：

+   CONTAINER_EXITED_WITH_FAILURE：ContainLaunch会尝试的去将Container进程调度起来，但是调度可能会失败，比如创建一些基础目录失败之类的。
+   CONTAINER_KILLED_ON_REQUEST：Container进程在调度过程中被kill。此时Container进程会返回FORCE_KILLED或TERMINATED
+   CONTAINER_EXITED_WITH_FAILURE：Container执行器计算过程中返回了非0的错误码。
+   CONTAINER_EXITED_WITH_SUCCESS：Container进程执行成功，并正常退出，返回0错误码。

至于ExitCode.LOST超出本文讨论的访问，这里就不描述了。。。


