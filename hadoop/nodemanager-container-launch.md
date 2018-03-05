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


```
public enum ExitCode {
		FORCE_KILLED(137),
		TERMINATED(143),
		LOST(154);
	}


```

Container执行器进程结束可能有下面几种可能状态：

+   CONTAINER_EXITED_WITH_FAILURE：ContainLaunch会尝试的去将Container进程调度起来，但是调度可能会失败，比如创建一些基础目录失败之类的。
+   CONTAINER_KILLED_ON_REQUEST：Container进程在调度过程中被kill。此时Container进程会返回FORCE_KILLED或TERMINATED
+   CONTAINER_EXITED_WITH_FAILURE：Container执行器计算过程中返回了非0的错误码。
+   CONTAINER_EXITED_WITH_SUCCESS：Container进程执行成功，并正常退出，返回0错误码。

至于ExitCode.LOST超出本文讨论的访问，这里就不描述了。。。
下面熟悉一下NodeMananger中一些目录规范，NodeManager中很多任务都和这几个目录在打交道。

+ `hadoop.tmp.dir`：Hadoop的全局tmp目录，包括Yarn在内的各个模块在运行过程中默认都是在该目录中创建相应的临时目录
+ `yarn.nodemanager.local-dirs`：NodeManager负责Container的运行和调度，运行过程中涉及到大量的文件包括map的输出，这个目录就是NodeManager运行过程中所有目录的根目录。
+ `yarn.nodemanager.log-dirs`：NodeManager在运行过程中所调度的所有进程和container都有相应的日志目录，该配置即为所有日志目录的根目录
+ `distribuction cache`：Yarn在NodeManager中实现了传统MapReduce中的`distribuction cache`的逻辑，其中cache分为public,private,app三个级别。

> `public`：全局权限，由NodeManager直接下载到`{*.local-dirs}/filecache`目录下面
> `private`：用户私有权限。NodeManager为每个用户维护一个Cache目录，其中`{*.local-dirs}/usercache/{username}/filecache`为用户层面的文件
>`application`：应用层面。NodeManager为指定用户的每个app创建一个本地目录,其中`{*.local-dirs}/usercache/{username}/appcache/{app_id}/filecache`内部维护app层面的文件，app结束以后会被清理。 具体文件本地化后面会专门进行分析，差不多是Nodemanager中一个饿比较拗口的模块。

+ `{*.local-dirs}/nmPrivate`:是NodeManager运行过程中私有目录，这个运行不包括Container进程运行过程中生成的数据。NodeManager在调度过程中，会在该`{*.local-dirs}/nmPrivate`目录下为每个app的每个container创建一个临时目录，从而可以为container运行之前做好一些准备。这些准备包括:

> container运行之前，需要针对进程生成执行脚本，脚本就放在`{*.local-dirs}/nmPrivate/appid/containerid/launch_container`文件中
> container运行之前，需要针对进程生成token文件，内容就放在`{*.local-dirs}/nmPrivate/appid/containerid/containerid.tokens`文件中
> container运行过程中，会在`{*.local-dirs}/nmPrivate/appid/containerid/containerid.pid`中创建container的pid文件，从而实现运行过程中监控该文件来确定container进程是否退出
>container运行结束后，会在`{*.local-dirs}/nmPrivate/appid/containerid/containerid.pid.exitcode`中写入container进程执行退出码，从而实现NodeManager获取container的执行结果。
>
> nmPrivate目录是NodeManager运行的私有目录，而不是container运行的pwd。上述生成container执行脚本和token在进行container启动时，会将它复制到container的pwd中。默认`pwd={*.local-dirs}/appcache/{username}/appcache/{app_id}/{containerid}`

在上面谈到`{*.local-dirs}/nmPrivate`目录时候，我们说到在真正对container的进程进行调度之前，需要创建container执行脚本和token文件等步骤，下面针对ContainLaunch在将container调度起来之前所做的事情做一个详细的描述：

+ `container.getLaunchContext().getCommands()`返回一个字符串list。AM在请求NodeManager启动container会写这个List。
+ AM提交的commands列表中对于container执行过程中日志目录和操作系统类型不清楚，因此NodeManager针对这块提供三个常量

> LOG_DIR：表示container运行时的日志目录，AM采用该常量进行标示
> CPS：window和linux针对目录分隔符的不同，AM提供的commands中如果有目录路径分隔符，用该常量进行替换
> `{{和}}`来对系统常量进行替换。window下面用%VAR%来表示系统常量，而linux用$VAR。为了保证代码平台无关，采用`{{ }}`来对系统常量进行标示

+ ENV的设置。AM提交的container请求中包含一部分用户自定义的container，但是NodeManager需要对这部分进行处理，主要是添加一些内部环境变量，用内部的环境变量覆盖用户
设置可能存在风险和错误的环境变量。涉及到环境变量还是很多，参阅`ContainerLaunch.sanitizeEnv()`函数。
+ 最后就写token和执行脚本到上面谈到两个文件中，执行脚本的生成内容很丰富，上面设置的环境变量也会写到该文件中。

到目前为止ContainLaunch已经完成对container所有的初始化工作，此时需要做的工作就是将container的进程起起来，这个过程是通过调用ContainerExecutor来实现的
```
exec.activateContainer(containerID, pidFilePath);
ret = exec.launchContainer(container, nmPrivateContainerScriptPath,nmPrivateTokensPath, user, appIdStr, containerWorkDir,localDirs, logDirs);

```
注意该函数的调用是堵塞的，在调度的进程退出之前，该函数是不会退出。

最后，针对ContainerLaunch附上一个所生成的ContainLaunch脚本的，通过该脚本，可以看出ContainerLaunch对环境变量等做了什么工作。
###ContainerExecutor
ContainerExecutor类在NodeManager的根包下面，第一次阅读NodeManager源码，就误以为它充当上文提到的ContainerLaunch角色。   经过对NodeManager里面的角色进行梳理，ContainerExecutor和ContainerLaunch最大的区别是，ContainerExecutor在NodeManager是一个全局的对象，整个NodeManager中只有一个ContainerExecutor，在NodeManager的serviceInit中进行初始化。而ContainerLaunch是一个线程，针对每个Container都会新建一个对象。

既然ContainerExecutor是NodeManager中全局对象，那么它肯定掌握了一些NodeManager中的全局信息。没错，参考如下：

      private ConcurrentMap<ContainerId, Path> pidFiles =new ConcurrentHashMap<ContainerId, Path>();
      protected boolean isContainerActive(ContainerId containerId) {
        try {
          readLock.lock();
          return (this.pidFiles.containsKey(containerId));
        } finally {
          readLock.unlock();
        }
      }
      public void activateContainer(ContainerId containerId, Path pidFilePath) {
        try {
          writeLock.lock();
          this.pidFiles.put(containerId, pidFilePath);
        } finally {
          writeLock.unlock();
        }
      }
      public void deactivateContainer(ContainerId containerId) {
        try {
          writeLock.lock();
          this.pidFiles.remove(containerId);
        } finally {
          writeLock.unlock();
        }
      }

ContainerExecutor全局维护了当前NodeManager所有处于Active状态的Container，并关联每个运行中的Container的pidFiles。通过pidFile，我们可以通过ContainerExecutor的reacquireContainer来监控指定的container是否运行结束。

除了维护当前NodeManager中所有Container的pidFile以外，ContainerExecutor最重要的两个功能是“资源的加载清理”和"Container的进程启停"。针对资源的加载和清理，后面再详细讨论，这里我们核心针对Container的进程启停进行讨论。

在上面讨论的ContainerLaunch中，ContainerLaunch负责生成Container的运行脚本等基础信息在NodeManager的nmprivate目录下面，但是将这些信息复制到Container的pwd目录中，并通过什么样的方式来将Container进程调度起来则由ContainerLaunch请求ContainerExecutor来实现。

特别是进程调起方式，不同ContainerExecutor实现是不同的。比如默认的DefaultContainerExecutor和LinuxContainerExecutor之间，LinuxContainerExecutor可以在指定的CGroup中启动Container进程，而DefaultContainerExecutor仅仅通过传统的方式来启动进程。

不过这里我们不会详细的讨论不同的ContainerExecutor的实现，我们以DefaultContainerExecutor的例子来进一步分析一个Container进程怎么被调度起来。

回到ContainerLaunch的实现，在ContainerLaunch完成对Container的初始化以后，首先通过`activateContainer`在ContainerExecutor中将该容器进行激活，然后将刚刚生成的scriptPath，tokenPath，localDir，logDirs以及pwd等Container环境文件和目录传递给launchContainer，并堵塞线程的执行，直到launchContainer完成container的进程的启动和运行，并返回container的进程返回错误码。

    exec.activateContainer(containerID, pidFilePath);
    ret = exec.launchContainer(container, nmPrivateContainerScriptPath,
            nmPrivateTokensPath, user, appIdStr, containerWorkDir,
            localDirs, logDirs);

launchContainer所做的工作主要有三件事：

+   初始化container的pwd目录，将token/script等path复制到pwd中。
+   container的进程脚本的生成。
+   启动container的进程脚本并等待进程运行结束。

第一件事情很简单，我们这里就不详细阐述了。但是第二件事就是要讲一下。在ContainerLaunch线程中，我们已经生成了一个launch_container.sh文件的脚本文件
那么在ContainerExecutor还生成什么脚本呢？其实真正的功能脚本就是ContainerLaunch中生成的launch_container.sh,在ContainerExecutor是对launch_container.sh进行包装。具体描述如下：

+   在launch_container.sh脚本外包围一个default_container_executor_session.sh脚本。用于将container进程的pid写入到pidfile中。由于采用的exec的方式来运行launch_container.sh，进程的pid是不改变

			//default_container_executor_session.sh
			echo $$ > pidfile.tmp
			/bin/mv -f pidfile.tmp pidfile
			exec setsid /bin/bash "launch_container.sh"

+   在default_container_executor_session.sh外部包围一个default_container_executor.sh。但是default_container_executor.sh不是通过exec的方式来启动default_container_executor_session.sh脚本。所以整个container是由两个进程组成，一个default_container_executor.sh和launch_container.sh组成


			//default_container_executor.sh
			/bin/bash default_container_executor_session.sh
			rc=$?
			echo $rc > tmpfile
			/bin/mv -f tmpfile pidfile.exitcode
			exit $rc


default_container_executor.sh进程由于将default_container_executor_session.sh进程执行退出码写到exitcode文件中。

完成了default_container_executor.sh脚本的生成，ContainerExecutor后面的工作就比较简单，直接调度起来并等待进程退出。同时ContainerExecutor拥有所有启动container的pid文件，向指定进程的pid发送kill -9/kill -0等信号可以进程container的探活以及杀死。具体就不描述了。

#containermanager.launcher.ContainersLauncher
上面谈到了ContainerLaunch，它是一个继承了Call的线程对象，对ContainerLaunch线程进行调度是由ContainersLauncher来负责。ContainersLauncher是ContainerManager中一个service模块。

首先它负责containerLaunch线程的调度，那么它内部肯定有一个线程池。

    public ExecutorService containerLauncher =
        Executors.newCachedThreadPool(
            new ThreadFactoryBuilder().setNameFormat("ContainersLauncher #%d").build());

其次，它是一个service，被注册了ContainersLauncherEventType.class的Event

    dispatcher.register(ContainersLauncherEventType.class, containersLauncher);

该Event包括三类事件：

+   LAUNCH_CONTAINER：接受来自ContainerManager中启动一个container的事件，ContainersLauncher负责创建一个ContainerLaunch线程，并交由线程池进行调度。
+   RECOVER_CONTAINER：Container的恢复，这里就不详说了。后面再分析。
+   CLEANUP_CONTAINER：Container的清理。杀死当前运行的container，以及对临时文件的清理。

总结：到目前为止，我们已经针对每个container生成一个ContainerLaunch线程，到调用ContainerExecutor来实现Container进程进程的调度，已经走通了Container进程启动和结束错误码的收集。至于说ContainersLauncher什么时候被event来启动一个containerLaunch进程，需要对container的资源本地化进行分析以后再能描述清楚。

end。
