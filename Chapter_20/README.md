第20章　使用JMX管理Spring Bean
-----------------------
Spring对DI的支持是通过在应用中配置bean属性，这是一种非常不错的方法。不过，一旦应用已经部署并且正在运行，单独使用DI并不能帮助我们改变应用的配置。假设我们希望深入了解正在运行的应用并要在运行时改变应用的配置，此时，就可以使用Java管理扩展（Java Management Extensions，JMX）了。

JMX这项技术能够让我们管理、监视和配置应用。这项技术最初作为Java的独立扩展，从Java 5开始，JMX已经成为标准的组件。

使用JMX管理应用的核心组件是托管bean（managed bean，MBean）。所谓的MBean就是暴露特定方法的JavaBean，这些方法定义了管理接口。JMX规范定义了如下4种类型的MBean：
- 标准MBean：标准MBean的管理接口是通过在固定的接口上执行反射确定的，bean类会实现这个接口；
- 动态MBean：动态MBean的管理接口是在运行时通过调用DynamicMBean接口的方法来确定的。因为管理接口不是通过静态接口定义的，因此可以在运行时改变；
- 开放MBean：开放MBean是一种特殊的动态MBean，其属性和方法只限定于原始类型、原始类型的包装类以及可以分解为原始类型或原始类型包装类的任意类型；
- 模型MBean：模型MBean也是一种特殊的动态MBean，用于充当管理接口与受管资源的中介。模型Bean并不像它们所声明的那样来编写。它们通常通过工厂生成，工厂会使用元信息来组装管理接口。

Spring的JMX模块可以让我们将Spring bean导出为模型MBean，这样我们就可以查看应用程序的内部情况并且能够更改配置——甚至在应用的运行期。接下来，将会介绍如何使用Spring对JMX的支持来管理Spring应用上下文中的bean。

# 1　将Spring bean导出为MBean
我们对程序清单5.10中SpittleController只做适度的改变，增加一个新的spittlesPerPage属性：
<br/>![](img/custom20-1.jpg)<br/>
之前，当我们调用SpitterService的getRecentSpittles()方法时，SpittleController传入20作为第二个参数，这会查询最近的20条Spittle。现在，不再是在构建应用时通过硬编码进行决策，而是通过使用JMX在运行时进行决策。新增的spittlesPerPage属性只是第一步而已。

我们下一步需要做的是把SpittleControllerbean暴露为MBean，而spittlePerPage属性将成为MBean的托管属性（managed attribute）。这时，我们就可以在运行时改变该属性的值。

Spring的MBeanExporter是将Spring Bean转变为MBean的关键。MBeanExporter可以把一个或多个Spring bean导出为MBean服务器（MBean server）内的模型 MBean。MBean服务器（有时候也被称为MBean代理）是MBean生存的容器。对MBean的访问，也是通过MBean 服务器来实现的。

如图20.1所示，将Spring bean导出为JMX MBean之后，可以使用基于JMX的管理工具（例如JConsole或者VisualVM）查看正在运行的应用程序，显示bean的属性并调用bean的方法。
<br/>![](img/img20-1.jpg)<br/>
图20.1　Spring的MBeanExporter可以将Spring bean的属性和方法导出为MBean服务器中的JMX属性和操作。通过JMX服务器，JMX管理工具（例如JConsole）可以查看到正在运行的应用程序的内部情况

下面的@Bean方法在Spring中声明了一个MBeanExporter，它会将spittleControllerbean导出为一个模型MBean：
<br/>![](img/MBeanExporter.jpg)<br/>

配置MBeanExporter的最简单方式是为它的beans属性配置一个Map集合，该集合中的元素是我们希望暴露为JMX MBean的一个或多个bean。每个Map条目的key就是MBean的名称（由管理域的名字和一个key-value对组成，在SpittleController MBean示例中是spitter:name=HomeController），而Map条目的值则是需要暴露的Spring bean引用。在这里，我们将输出spittleControllerbean，以便它的属性可以通过JMX在运行时进行管理。

通过MBeanExporter，spittleControllerbean将作为模型MBean以SpittleController的名称导出到MBean服务器中，以实现管理功能。图20.2展示了通过JConsole查看SpittleControllerMBean时的情况。
<br/>![](img/img20-2.jpg)<br/>
图20.2　SpittleController导出为MBean，并且可以通过JConsole查看

如图20.2的左侧所示，SpittleController所有的public成员都被导出为MBean的操作或属性。这可能并不是我们所希望看到的结果，我们真正需要的只是可以配置spittlesPerPage属性。我们不需要调用spittles()方法或SpittleController中的其他方法或属性。因此，我们需要一个方式来筛选所需要的属性或方法。

为了对MBean的属性和操作获得更细粒度的控制，Spring提供了几种选择，包括：
- 通过名称来声明需要暴露或忽略的bean方法；
- 通过为bean增加接口来选择要暴露的方法；
- 通过注解标注bean来标识托管的属性和操作。

> MBean服务器从何处而来
根据以上配置，MBeanExporter会假设它正在一个应用服务器中（例如Tomcat）或提供MBean服务器的其他上下文中运行。但是，如果Spring应用程序是独立的应用或运行的容器没有提供MBean服务器，我们就需要在Spring上下文中配置一个MBean服务器。
在XML配置中，`<context:mbean-server>`元素可以为我们实现该功能。如果使用Java配置的话，我们需要更直接的方式，也就是配置类型为MBeanServerFactoryBean的bean（这也是在XML中`<context:mbean-server>`元素所作的事情）。

## 1.1　通过名称暴露方法
MBean信息装配器（MBean info assembler）是限制哪些方法和属性将在MBean上暴露的关键。其中有一个MBean信息装配器是MethodNameBasedMBean-InfoAssembler。这个装配器指定了需要暴露为MBean操作的方法名称列表。对于SpittleController bean来说，我们希望把spittlePerPage暴露为托管属性。基于方法名的装配器如何帮我们导出一个托管属性呢？

我们回顾下JavaBean的规则（这不是Spring Bean所必需的），spittlesPerPage属性需要定义对应的存取器（accessor）方法，方法名必须为setSpittlesPerPage()和getSpittlesPerPage()。为了限制MBean所暴露的内容，我们需要告诉MethodNameBaseMBeanInfoAssembler仅在MBean的接口中包含这两个方法。如下MethodNameBaseMBeanInfoAssembler的 bean声明就配置了这些方法：
<br/>![](img/MethodNameAssembler.jpg)<br/>

因为本示例所配置的是spittlesPerPage属性的存取器方法，所以spittlesPerPage属性也自然成为了MBean的托管属性。

为了让这个装配器能够生效，我们需要将它装配进MBeanExporter中：
<br/>![](img/MBeanExporterWithAssembler.jpg)<br/>

现在如果我们启动应用，SpittleController的spittlesPerPage将作为有效的MBean托管属性，而spittles()方法并不会暴露为MBean的托管操作。图20.3展示了通过JConsole查看SpittleController的情况。
<br/>![](img/img20-3.jpg)<br/>
图20.3　当指定了哪些方法在SpittleController MBean上暴露后，spittles()方法不再作为MBean的托管操作

> Note
另一个基于方法名称的装配器是MethodExclusionMBeanInfoAssembler。这个MBean信息装配器是MethodNameBaseMBeanInfoAssembler的反操作

## 1.2　使用接口定义MBean的操作和属性
InterfaceBasedMBeanInfoAssembler与基于方法名称的装配器很相似，只不过不再通过罗列方法名称来确定暴露哪些方法，而是通过列出接口来声明哪些方法需要暴露。
例如，假设我们定义了一个名为SpittleControllerManagedOperations的接口，如下所示：
<br/>![](img/SpittleCMO.jpg)<br/>

为了应用此装配器，我们只需要使用如下的assemblerbean替换之前基于方法名称的装配器即可：
<br/>![](img/InterfaceAssembler.jpg)<br/>

最终，这些托管操作必须在某处声明，无论是在Spring配置中还是在某个接口中。此外，从代码角度看，托管操作的声明是一种重复——在接口中或Spring上下文中声明的方法名称与实现中所声明的方法名称存在重复。之所以存在这种重复，没有其他原因，仅仅是为了满足MBeanExporter的需要而产生的。

Java注解的一项工作就是帮助消除这种重复。让我们看看如何通过使用注解标注Spring管理的bean，从而将其导出MBean。

## 1.3　使用注解驱动的MBean
用Spring context配置命名空间中的`<context:mbeanexport>`元素。这个便捷的元素装配了MBean导出器以及为了在Spring启用注解驱动的MBean所需要的装配器。我们所需要做的就是使用它来替换我们之前所使用的 MBeanExporterbean:
```xml
<context:mbean-export server="mbeanServer"/>
```

现在，要把任意一个Spring bean转变为MBean，我们所需要做的仅仅是使用@ManagedResource注解标注bean并使用@ManagedOperation或@ManagedAttribute注解标注bean的方法。例如，如下的程序清单展示了如何使用注解把SpittleController导出为MBean。

程序清单20.1　通过注解把HomeController转变为MBean
<br/>![](img/code20-1.jpg)<br/>

在类级别使用了@ManagedResource注解来标识这个bean应该被导出为MBean。objectName属性标识了域（Spitter）和MBean的名称（SpittleController）。

spittlesPerPage属性的存取器方法都使用了@ManagedAttribute注解来进行标注，这表示该属性应该暴露为MBean的托管属性。注意，其实并不需要使用注解同时标注这两个存取器方法。如果我们选择仅标注setSpittlesPerPage()方法，那我们仍可以通过JMX设置该属性，但这样的话我们将不能查看该属性的值。相反，如果仅仅标注getSpittlesPerPage()方法，那我们可以通过JMX查看该属性的值，但无法修改该属性的值。

同样需要提醒一下，我们还可以使用@ManagedOperation注解替换@ManagedAttribute注解来标注存取器方法。如下所示:
<br/>![](img/ManagedOperation.jpg)<br/>

- 更严格的方法、属性暴露方式
这会将方法暴露为MBean的托管操作，但是并不会把spittlesPerPage属性暴露为MBean的托管属性。这是因为在暴露MBean功能时，使用@ManagedOperation注解标注方法是严格限制方法的，并不会把它作为JavaBean的存取器方法。因此，使用@ManagedOperation可以用来把bean的方法暴露为MBean托管操作，而使用@ManagedAttribute可以把bean的属性暴露为MBean托管属性。

## 1.4　处理MBean冲突
到目前为止，我们已经看到可以使用多种方式在MBean服务器中注册MBean。在所有的示例中，我们为MBean指定的对象名称是由管理域名和key-value对组成的。如果MBean服务器中不存在与我们MBean名字相同的已注册的MBean，那我们的MBean注册时就不会有任何问题。但是如果名字冲突时，将会发生什么呢？

默认情况下，MBeanExporter将抛出InstanceAlreadyExistsException异常，该异常表明MBean服务器中已经存在相同名字的MBean。不过，我们可以通过MBeanExporter的registrationBehaviorName属性或者`<context:mbeanexport>`的registration属性指定冲突处理机制来改变默认行为。

Spring提供了3种借助registrationBehaviorName属性来处理MBean名字冲突的机制：
- FAIL_ON_EXISTING：如果已存在相同名字的MBean，则失败（默认行为）；
- IGNORE_EXISTING：忽略冲突，同时也不注册新的MBean；
- REPLACING_EXISTING：用新的MBean覆盖已存在的MBean；
例如，如果我们使用MBeanExporter，我们可以通过设置registration-BehaviorName属性为RegistrationPolicy.IGNORE_EXISTING来忽略冲突，如下所示：
<br/>![](img/registPolicy.jpg)<br/>

# 2　远程MBean
在本小节中，我们将看到Spring如何远程访问MBean。我们首先从配置Spring把SpittleController导出为远程MBean开始，然后我们再了解如何使用Spring远程操纵MBean。

## 2.1　暴露远程MBean
使MBean成为远程对象的最简单方式是配置Spring的ConnectorServerFactoryBean：
<br/>![](img/ConnectorSFB.jpg)<br/>
ConnectorServerFactoryBean会创建和启动JSR-160 JMXConnectorServer。默认情况下，服务器使用JMXMP协议并监听端口9875——因此，它将绑定“service:jmx:jmxmp://localhost:9875”。但是我们导出MBean的可选方案并不局限于JMXMP。

根据不同JMX的实现，我们有多种远程访问协议可供选择，包括远程方法调用（Remote Method Invocation，RMI）、SOAP、Hessian/Burlap和IIOP（Internet InterORB Protocol）。为MBean绑定不同的远程访问协议，我们仅需要设置ConnectorServerFactoryBean的serviceUrl属性。例如，如果我们想使用RMI远程访问MBean，我们可以像下面示例这样配置：
<br/>![](img/ConnectorSFBWithServiceUrl.jpg)<br/>

在这里，我们将ConnectorServerFactoryBean绑定到了一个RMI注册表，该注册表监听本机的1099端口。这意味着我们需要一个RMI注册表运行时，并监听该端口。我们可以回顾下第15章，RmiServiceExporter可以为我们自动启动一个RMI注册表。但是，我们在本示例中不使用RmiServiceExporter，而是通过在Spring中声明RmiRegistryFactoryBean来启动一个RMI注册表，如下面的@Bean方法所示：
<br/>![](img/RmiRFB.jpg)<br/>

## 2.2　访问远程MBean
要想访问远程MBean服务器，我们需要在Spring上下文中配置MBeanServerConnectionFactoryBean。下面的bean声明装配了一个MBeanServerConnectionFactoryBean，该bean用于访问我们在上一节中所创建的基于RMI的远程服务器。
<br/>![](img/MBeanSConnFB.jpg)<br/>
由MBeanServerConnectionFactoryBean所生成的MBeanServerConnection实际上是作为远程MBean服务器的本地代理。它能够以MBeanServerConnection的形式注入到其他bean的属性中：
<br/>![](img/JmxClient.jpg)<br/>

远程访问 MBean服务器的真正价值在于访问远程服务器上已注册MBean的属性以及调用它们的方法。

为了访问MBean属性，我们可以使用getAttribute()和setAttribute()方法。例如，为了获取MBean属性的值，我们可以按照下面的方法调用getAttribute()方法：
<br/>![](img/getAttr.jpg)<br/>

同样，我们可以使用setAttribute()方法改变MBean属性的值：
<br/>![](img/setAttr.jpg)<br/>

如果希望调用MBean的操作，那我们需要使用invoke()方法。下面的内容描述了如何调用SpittleController MBean的setSpittlesPerPage()方法：
<br/>![](img/invokeMBeanOpr.jpg)<br/>

不过，通过MBeanServerConnection对远程MBean进行方法调用和属性设置是一种很笨拙的方法。要想调用setSpittlesPerPage()这样一个简单的方法，我们需要创建一个ObjectName实例，并向invoke()方法传递几个参数。它并不是直观的方法调用。为了更直接地调用方法，我们需要代理远程MBean。

## 2.3　代理MBean
MBcanProxyFactoryBean可以让我们可以直接访问远程的MBean（就如同配置在本地的其他bean一样）。图20.4展示了它的工作原理。
<br/>![](img/img20-4.jpg)<br/>
图20.4　MBeanFactoryBean创建远程MBean的代理。客户端通过此代理与远程MBean进行交互，就像它是本地Bean一样

例如，考虑如下的MBeanProxyFactoryBean声明：
<br/>![](img/MBeanProxyFB.jpg)<br/>

- objectName属性指定了远程MBean的对象名称。在这里是引用我们之前导出的SpittleControllerMBean。
- server属性引用了MBeanServerConnection，通过它实现MBean所有通信的路由。在这里，我们注入了之前配置的MBeanServerConnectionFactoryBean。
- 最后，proxyInterface属性指定了代理需要实现的接口。在本示例中，我们使用20.1.2小节所定义的SpittleControllerManagedOperations接口。

对于上面声明的remoteSpittleControllerMBean，我们现在可以把它注入到类型为SpittleControllerManagedOperations的bean属性中，并使用它来访问远程的MBean。这样，我们就可以调用setSpittlesPerPage()和getSpittlesPerPage()方法了。

# 3　处理通知
通过查询MBean获得信息只是查看应用状态的一种方法。但当应用发生重要事件时，如果希望能够及时告知我们，这通常不是最有效的方法。例如，假设Spittr应用保存了已发布的Spittle数量，而我们希望知道每发布一百万Spittle时的精确时间（例如一百万、两百万、三百万等）。一
种解决方法是编写代码定期查询数据库，计算Spittle的数量。但是执行这种查询会让应用和数据库都很繁忙，因为它需要不断的检查Spittle的数量。

与重复查询数据库获得Spittle的数量相比，更好的方式是当这类事件发生时让MBean通知我们。JMX通知（JMX notification，如图20.5 所示）是MBean与外部世界主动通信的一种方法，而不是等待外部应用对MBean进行查询以获得信息。

<br/>![](img/img20-5.jpg)<br/>
图20.5　JMX通知使MBean与外部世界进行主动通信

Spring通过NotificationPublisherAware接口提供了发送通知的支持。任何希望发送通知的MBean都必须实现这个接口。例如，请查看如下程序清单中的SpittleNotifierImpl。

程序清单20.2　使用NotificationPublisher来发送JMX通知
<br/>![](img/code20-2.jpg)<br/>

正如我们所看到的，SpittleNotifierImpl实现了NotificationPublisherAware接口。这并不是一个要求苛刻的接口，它仅要求实现一个方法：setNotificationPublisher。

SpittleNotificationImpl也实现了SpittleNotifier接口的方法：millionthSpittlePosted()。这个方法使用了setNotificationPublisher()方法所注入的NotificationPublisher来发送通知：我们的Spittle数量又到了一个新的百万级别。

一旦sendNotification()方法被调用，就会发出通知。嗯……好像我们还没决定谁来接收这个通知。那就让我们建立一个通知监听器来监听和处理通知。

## 3.1　监听通知
接收MBean通知的标准方法是实现javax.management.NotificationListener接口。例如，考虑一下PagingNotificationListener：
<br/>![](img/NotificationListener.jpg)<br/>
PagingNotificationListener是一个典型的JMX通知监听器。当接收到通知时，将会调用handleNotification()方法处理通知。

剩下的工作只需要使用MBeanExporter注册PagingNotificationListener：
<br/>![](img/MBeanExporterWithPageNL.jpg)<br/>

MBeanExporter的notificationListenerMappings属性用于在监听器和监听器所希望监听的MBean之间建立映射。在本示例中，我
们建立了PagingNotificationListener来监听由SpittleNotifier MBean所发布的通知。

# 源码
无
