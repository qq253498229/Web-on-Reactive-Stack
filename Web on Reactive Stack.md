# Web on Reactive Stack

本文档的这一部分介绍了对响应堆栈的支持，构建在Reactive Streams API上的Web应用程序可在非阻塞服务器（如Netty，Undertow和Servlet 3.1+容器）上运行。 各章包含Spring Web Flux框架，被动Web客户端，测试支持和反应性库。 对于Servlet堆栈，Web应用程序，请参阅关于Servlet堆栈的Web。

## 1. Spring WebFlux

### 1.1. 介绍

包含在Spring框架中的原始Web框架Spring Web MVC专门用于Servlet API和Servlet容器。 反应堆栈，Web框架，Spring WebFlux，后来在5.0版本中添加。 它完全无阻塞，支持Reactive Streams反压，并且可以在诸如Netty，Undertow和Servlet 3.1+容器的服务器上运行。

这两个Web框架都在Spring框架中镜像源模块的名称spring-webmvc和spring-webflux并且共存。 每个模块都是可选的。 应用程序可以使用一个或另一个模块，或者在一些情况下 - 例如 具有反应性WebClient的Spring MVC控制器。

#### 1.1.1. 目的

为什么要创建Spring WebFlux？

部分答案是需要一个无阻塞的Web栈来处理少量线程的并发性，并用较少的硬件资源进行扩展。 Servlet 3.1确实为非阻塞I / O提供了一个API。但是，使用它会导致Servlet API变成同步的（Filter，Servlet）或阻塞（getParameter，getPart）。这是一个新的公共API作为跨越任何非阻塞运行时的基础的动机。这一点很重要，因为诸如Netty这样的服务器已经在异步，非阻塞空间中很好地建立起来了。

答案的另一部分是函数式编程。就像在Java 5中添加注释一样，创造了机会 - 例如带注释的REST控制器或单元测试，在Java 8中添加lambda表达式为Java中的功能API创造了机会。对于非阻塞应用程序和延续式API来说，这是一个福音，正如CompletableFuture和ReactiveX所推广的那样，它允许声明式组合异步逻辑。在编程模型层面，Java 8使Spring WebFlux能够在带注释的控制器的同时提供功能性Web端点。

#### 1.1.2. 定义 "reactive"

我们谈到了非阻塞和功能，但reactive意味着什么？

术语“reactive”是指围绕对变化做出反应而建立的编程模型 - 网络组件对I / O事件作出反应，UI控制器对鼠标事件作出反应等。从这个意义上讲，非阻塞是反应性的，因为不是被阻塞，而是现在处于对操作完成或数据可用的通知作出反应的模式。

我们Spring团队的另一个重要机制是“reactive”，即非阻塞反压。在同步的，命令式的代码中，阻塞呼叫是一种自然形式的背压，迫使呼叫方等待。在非阻塞代码中，控制事件发生率非常重要，以便快速生产者不会压倒目标。

Reactive Streams是一个小规格，也在Java 9中采用，它定义了背压异步组件之间的交互。例如，数据存储库（作为发布服务器）可以生成数据，以便作为订阅服务器的HTTP服务器可以写入响应。 Reactive Streams的主要目的是允许用户控制发布者产生数据的速度有多快或多慢。

>常见问题：如果发布商不能放慢速度？
反应流的目的只是建立机制和边界。 如果发布商不能放慢速度，那么它必须决定是否缓冲，丢弃或失败。

#### 1.1.3. Reactive API

Reactive Streams在互通性方面发挥着重要作用。它对库和基础组件很感兴趣，但作为一个应用程序API可能不太有用，因为它的级别太低。应用程序需要的是更高级别和更丰富的功能API来组成异步逻辑 - 类似于Java 8 Stream API，但不仅适用于集合。这是reactive库的作用。

Reactor是Spring WebFlux的reactive库。它提供了Mono和Flux API类型，通过与ReactiveX操作符词汇对齐的一组丰富的操作符来处理0..1和0..N的数据序列。 Reactor是一个Reactive Streams库，因此它的所有操作都支持非阻塞反馈。 Reactor强烈关注服务器端Java。它是与Spring密切合作开发的。

WebFlux需要Reactor作为核心依赖项，但它可以通过Reactive Streams与其他reactive库进行互操作。作为一般规则，WebFlux API接受一个普通的发布者作为输入，在内部将它调整为Reactor类型，使用它们，然后返回Flux或Mono作为输出。因此，您可以通过任何发布服务器作为输入，并且可以对输出应用操作，但您需要调整输出以供其他反应式库使用。只要可行 - 例如带注释的控制器，WebFlux透明地适应RxJava或其他反应性库的使用。有关更多详细信息，请参阅Reactive Libraries。

#### 1.1.4. 编程模型

Spring-Web模块包含Spring WebFlux的基础，这些基础包括HTTP抽象，受支持服务器的Reactive Streams适配器，编解码器以及可与Servlet API相媲美但具有非阻塞协议的核心WebHandler API。

在此基础上，Spring WebFlux提供了两种编程模型的选择：

带注释的控制器 - 与Spring MVC一致，并基于spring-web模块中的相同注释。 Spring MVC和WebFlux控制器都支持响应式（Reactor，RxJava）返回类型，因此很难区分它们。一个显着的区别是WebFlux也支持被动的@RequestBody参数。

功能端点 - 基于lambda的轻量级功能编程模型。把它想象成一个小型库或应用程序可以用来路由和处理请求的一组工具。与注释控制器的最大区别在于应用程序负责从头到尾的请求处理，并通过注释声明意图并被回调。

#### 1.1.5. 适用性

Spring MVC还是WebFlux？

一个自然而然的问题，但建立一个不健全的二分法。 实际上它们一起工作来扩大可用选项的范围。 这两个设计是为了连续性和彼此的一致性，它们可以并排使用，每一方的反馈都有利于双方。 下图显示了这两者之间的关系，它们具有什么共同之处，以及各自独特的支持：

![image](https://docs.spring.io/spring/docs/current/spring-framework-reference/images/spring-mvc-and-webflux-venn.png)

以下是需要考虑的一些具体要点：

- 如果你有一个可以正常工作的Spring MVC应用程序，则不需要改变。命令式编程是编写，理解和调试代码最简单的方法。由于历史代码大多数都是阻塞的，你有最大的库选择。

- 如果您已经准备好了使用非阻塞Web栈，那么Spring WebFlux提供了与此空间中其他人相同的执行模型优势，并且还提供了多种服务器选项 - Netty，Tomcat，Jetty，Undertow，Servlet 3.1+容器，编程模型 - 带注释的控制器和功能性网络端点，以及反应库的选择 - 反应器，RxJava或其他。

- 如果您对用于Java 8 lambda或Kotlin的轻量级功能性Web框架感兴趣，请使用Spring WebFlux功能性Web端点。对于较小的应用程序或微服务来说，这也可能是一个不错的选择，这些应用程序或复杂的需求可以从更高的透明度和控制中受益。

- 在微服务体系结构中，您可以将应用程序与Spring MVC或Spring WebFlux控制器或Spring WebFlux功能端点混合使用。在两个框架中支持相同的基于注解的编程模型使得重用知识变得更加容易，同时也为正确的工作选择正确的工具。

- 评估应用程序的简单方法是检查它的依赖关系。如果您阻塞了持久性API（JPA，JDBC）或联网API，那么Spring MVC至少是常见体系结构的最佳选择。使用Reactor和RxJava在单独的线程上执行阻塞调用在技术上是可行的，但是您不会充分利用非阻塞的Web堆栈。

- 如果您有一个调用远程服务的Spring MVC应用程序，请尝试使用reactive WebClient。您可以直接从Spring MVC控制器方法返回反应类型（Reactor，RxJava或其他）。每次通话的等待时间越长，或者通话间的相互依赖性越强。 Spring MVC控制器也可以调用其他反应组件。

- 如果你有一个庞大的团队，记住转向非阻塞，功能和声明式编程的陡峭的学习曲线。没有完全切换的实用方法是使用reactive WebClient。除此之外，开始尝试一些小型项目并衡量好处。我们预计，对于广泛的应用来说，这种转变是不必要的。如果您不确定要寻找哪些好处，请先了解非阻塞I / O如何工作（例如，单线程Node.js上的并发）及其影响。

#### 1.1.6. 服务器

Tomcat，Jetty，Servlet 3.1+容器以及非Servlet运行时（如Netty和Undertow）支持Spring WebFlux。所有服务器都适用于低级的通用API，以便跨服务器支持更高级别的编程模型。

Spring WebFlux没有内置的支持来启动或停止服务器。但是，从Spring配置和WebFlux基础架构组装一个应用程序很简单，只需几行代码即可运行它。

Spring Boot有一个WebFlux初学者可以自动执行这些步骤。默认情况下，初学者使用Netty，但只需更改Maven或Gradle依赖关系即可轻松切换到Tomcat，Jetty或Undertow。 Spring Boot默认使用Netty，因为它在异步，非阻塞空间中使用得更广泛，并提供客户端和服务器共享资源。

Tomcat和Jetty可以与Spring MVC和WebFlux一起使用。请记住，他们使用的方式是非常不同的。 Spring MVC依赖于Servlet阻塞I / O，并允许应用程序在需要时直接使用Servlet API。 Spring WebFlux依赖于Servlet 3.1的非阻塞I / O，并在低级适配器后面使用Servlet API，而不是直接使用。

对于Undertow，Spring WebFlux直接使用Undertow API而不使用Servlet API。
