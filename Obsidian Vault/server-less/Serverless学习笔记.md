1. ***什么是server-less***
	Server-less（无服务器）这个概念存在已经很久了，最早指不需要服务器端的软件，比如纯客户端软件以及 peer-to-peer (P2P) 软件，在云时代，这个概念才表示不需要关心服务器端的相关技术，比如按量计费的 PaaS 服务（比如 FaunaDB serverless，Aurora Serverless， 对象存储等），BaaS （Backend as a Service），以及 Google App Engine 这样的托管 Application PaaS 也可包含在内。

![[Screen Shot 2024-09-22 at 16.39.57.png]]
[Status of Serverless Computing 第一阶段的云主要解决硬件资源（网络，计算，存储）的运维和供给问题，也就是 IaaS 云，可以理解成基于硬件资源的共享经济。IaaS 云的交付的主要是资源，接口以及控制台也是面向资源的，尽量以模拟物理机房环境来降低应用的迁移成本。而云发展到当前阶段来看，出现了两种需求：and Function-as-a-Service(FaaS) in Industry and Research]
2. ***从技术的演变来看serverless***
- Common Gateway Interface（CGI）时代：把命令行软件执行的结果通过网络输出。当时的应用程序的架构比较简单，启个 web 服务器监听网络端口，然后根据请求调用 CGI 程序，同时把请求数据传递给 CGI 程序，等待程序执行后把输出结果通过网络端口返回给客户端（一般是浏览器）。这种架构下，只有 web 服务器是常驻运行的，后面的 CGI 程序都是按需运行的，运行后就释放资源。于是诞生了一批支持 CGI 协议的脚本语言，比如 perl，php，asp，一方面脚本语言可以粘结 web 服务器和应用程序，另外一方面它本身也可以包含逻辑，于是逐渐出现专门针对 web 的用脚本编写的应用。
- Fast CGI时代：这种技术的思路是执行逻辑的进程不再是按需运行，而是常驻方式，web 服务器和 FastCGI 服务之间通过 unix socket 或者 tcp 进行交互。以 php-fpm（FastCGI Process Manager） 为例，php-fpm 启动一个常驻的进程池，每个请求分配给一个进程，进程调用 php 脚本执行后释放资源。当然这种方式带来的另外一个问题是常驻进程很容易产生内存泄露等问题，于是 php-fpm 提供一个选项可以设置执行多少次后重启进程。如果单纯为了性能考虑，直接把应用程序打包到 web 服务器里，通过 web 服务器调用应该性能更好。但这样更新应用就需要同时更新 web 服务器，开发和维护成本比较高，而 web 服务器和应用程序一般是不同的提供方提供，多个应用程序共享同一台机器也会导致端口冲突。
- 互联网微服务时代：互联网服务的特点是一个服务给全球人使用，对可移植性和标准化并没那么热衷，但对并发以及性能的要求极高。所以需要一种应用的架构方式，即可以满足初期的快速研发需求，也可以承载后期的用户规模，还可以将应用的标准化功能沉淀到平台上独立提供。于是有了微服务。
3. ***从云的发展来看 FaaS***
	第一阶段的云主要解决硬件资源（网络，计算，存储）的运维和供给问题，也就是 IaaS 云，可以理解成基于硬件资源的共享经济。IaaS 云的交付的主要是资源，接口以及控制台也是面向资源的，尽量以模拟物理机房环境来降低应用的迁移成本。而云发展到当前阶段来看，出现了两种需求：
	- **按需计算** 原来云的按需计算只是虚拟机维度的，按时间计费以及弹性伸缩，并不能正真做到按需计算，计算和内存资源都是预申请规划的，和服务的请求并发数并没有明确的关系，哪怕一段时间一个请求没有，资源还是依然占用。而 FaaS 可以做到按请求计费，不需要为等待付费，可以做到更高效的资源利用率。
	- **面向应用** 本质上用户对云的期望是应用的运行环境，并且最好是只让用户关心业务逻辑，而不需要关心，或者尽量少关心技术逻辑（比如监控，性能，弹性，高可用，日志追踪等）。这也是云原生应用（Cloud Native Application）这个概念提出的背景。
4. ***云厂商比较***
	在公有 IaaS 云上支持 FaaS，可以从以下角度分析：
	- 支持的语言以及 Function 标准。
	- 触发器支持，也就是和 IaaS 上其他系统的整合。
	- 伸缩和调度策略。这个虽然对用户是个黑盒子，但通过文档和自己的一些实验可以做一定程度的分析。
	- FaaS 和私有 VPC 资源的交互。FaaS 本身是一个全局的公共池子，如果用户的 Function 要请求自己 VPC 的内网资源，就会遇到问题。
	- FaaS 之上更高级的应用支持。
 - ***语言对比***
![[Screen Shot 2024-09-22 at 17.06.18 1.png]]
- ***触发器支持***
![[Screen Shot 2024-09-22 at 17.14.35.png]]
FaaS 是一种事件驱动架构，所以事件源很重要。这一方面 AWS 做的最全面，几乎将 FaaS 所有可能支持的事件类型都囊括了，并且探索 FaaS 在边缘计算和 IoT 上的场景。这个表格中并没有把 AWS 支持的事件类型全列出来。

Google 的 Firebase/Firestore 的支持其实是 Firebase 本身提供的，并不是标准的 Google Cloud Function 的 trigger，开发的界面以及语言标准都不一样，Crashlytics/Analytics 也是通过 Firebase 支持的，应该是最早的 Cloud Function 只是 Firebase 中的一个功能子集，Google 当前正在整合中。

微服务场景下，AWS，阿里云，华为云的做法都是通过 API Gateway 配置规则，将请求转发给 Function，Google 和 Azure 是直接提供 http trigger 地址。前者的好处是可以复用 API Gateway 的其他功能，但使用起来略复杂，后者简单易用。
- ***运维***
![[Screen Shot 2024-09-22 at 17.14.25.png]]

5. ***开源项目对比***![[Screen Shot 2024-09-22 at 17.20.38.png]]
- **Openwisk** 是 IBM 捐献给 Apache 的项目，当前还在孵化中。IBM 同时在自己的 Bluemix 上提供了 Openwisk 服务。它的架构是基于 Akka 的 actor 模型（不熟悉 actor 模型的可以参看本人的文章《[并发之痛 Thread，Goroutine，Actor](https://jolestar.com/parallel-programming-model-thread-goroutine-actor/)》）构建，中间以 Kafka 作为消息中间件。OpenWhisk 的抽象概念考虑的场景比较多。它的 Trigger 是可以独立定义的，而不是系统中确定的几种。然后通过 Rule 把 Trigger 和 Action（相当于 Function） 关联起来。有 Package 的概念，相当于可以把多个 Function 打包成同一个 Package，然后进行分发。另外它又抽象了一种 Feed 的概念，相当于一个事件流，可以做一些流式处理上的逻辑。同时它支持 Chain 的机制，把多个 Action 串起来进行 Pipeline 调用。

- **Fnproject** 是 Oracle 收购了 iron.io 后，从 iron-io/function 这个项目直接改造过来的。iron function 创建于 2012 年，2013 项目中断，2015年重新启动，2017 年被 Oracle 收购。它的架构比较简单，Function 的交付结果是容器镜像，每次请求都运行一个 Docker 容器，然后从标准输出获取结果输出，再销毁容器。所以它基本上只需要一个 API Gateway 做路由，然后后面挂一个容器运行环境就行，单机或者容器编排引擎都行。Fnproject 在 FaaS 之上发布了一个叫做 flow 的项目，一种工作流引擎，特点是基于编程语言来调度，而不是编排文件或者自定义描述语言，对开发者友好，并且容易编写复杂的工作流逻辑。

- **Funktion** 最近已经停止更新，提示 Redhat 停止赞助这个项目。看相关公告，Redhat 当前试图整合 openwhisk 以及 fission，还有公有云的 aws lambda 到 openshift，重心应该还是在 openshift，主要是做整合，不再试图创造新的 FaaS 平台。

- **Fission** 是基于 Kubernetes 构建，它的 Function 是 Kubernetes 中的对象，每种语言环境维护一个 Pod 池子，用 Deployment 控制数量，然后自己的调度器把 Function 分配给合适的 Pod 运行。前面有一层 API Gateway，创建 Function 后需要设置路由，通过 HTTP 触发。Fission 在 FaaS 之上也构建了一个工作流引擎，叫做 fission-workflows。通过 yaml 描述语言描述，编排多个 Function 作为工作流。它的工作流本质上也是一种 Function，不过运行在叫做 workflow 的环境中。

- **Kubeless** 也是基于 Kubernetes，但更 Native ，它充分利用了 Kubernetes 的 CRD 特性，将 Function 作为 Kubernetes 中的一种在 Pod 之上的对象，每个 Function 实际上会创建出一个 Deployment，同时暴露一个 Service。自动伸缩机制直接沿用 Kubernetes 的 autoscale 机制，只需要输出自定义的监控数据给自动伸缩组件即可。没有独立的 API 服务，直接复用 Kubernetes 的 API。

- **Openfaas** 的机制和 Fnproject 有点像，只是 Openfaas 并不是直接抓取容器的标准输出，而是写一个 Function Watchdog 作为容器的启动进程，暴露 http 服务，用于和调度系统交互，然后直接调用进程运行 Function 获取输出。

- **Qinling** 的定位是 Openstack 上的 FaaS，特性是和 Openstack 上的对象存储等系统整合。

- **Funcatron** 的特点是和 Swagger 整合，编程规范试图弥合异步和同步事件的差异，以及复用已有的框架，比如 Java SpringBoot。

reference:
1. https://jolestar.com/serverless-faas-current-status-and-future/