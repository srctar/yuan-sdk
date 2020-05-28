# About Me(前言)
This is the **SDK(`yuan-sdk`)、Quick Start** for [Alibaba Sentinel](https://github.com/alibaba/Sentinel), Make the accessing easier.

Before reading or using `yuan-sdk`, you should skilled use Sentinel;

 - You can easy access Sentinel by reading part [How To Access(如何接入)](#how-to-access-如何接入), 
 - You may know how we work at [What Did We Do(技术原理)](#what-did-we-do-技术原理),
 - Do something on me? This part: [How to rewrite me(如果修改你期望的功能)](#how-to-rewrite-me-如果修改你期望的功能)
 - Some Q&A: [Something you didnt understand(一些Q&A)](#something-you-didnt-understand-一些问题和解决方案)

In [traditional accessing](https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D#quick-start), you 
may coupling Sentinel's functionality into your logic, such as:
```
public static void main(String[] args) {
    initFlowRules();
    while (true) {
        Entry entry = null;
        try {
	    entry = SphU.entry("HelloWorld");
            /*您的业务逻辑 - 开始*/
            System.out.println("hello world");
            /*您的业务逻辑 - 结束*/
	} catch (BlockException e1) {
            /*流控逻辑处理 - 开始*/
	    System.out.println("block!");
            /*流控逻辑处理 - 结束*/
	} finally {
	   if (entry != null) {
	       entry.exit();
	   }
	}
    }
}
```
You won`t like it。

So if you can only use annotation(`@YuanCallback`) for accessing, will you happy for that?
its name is `yuan-sdk`(**THIS PROJECT**)

中文描述：
本开源项目是为了简化 Alibaba Sentinel 的接入逻辑。 
 - 降低Sentinel与目标系统的耦合性;
 - 简化Sentinel的接入流程;
 - 支持所有类型的Java项目的便捷接入;
 
 > 你在使用本组件`yuan-sdk`的时候, 需对Sentinel有一定的了解, 知道它的使用方式。

# How To Access 如何接入
## Notice (说明)
`yuan-sdk` is only for accessing easier, how to use, belong to [wiki](https://github.com/alibaba/Sentinel/wiki)

`yuan-sdk` 本意为简化Sentinel的接入， 实际使用与Sentinel官方用法别无二致. 请参阅[官方wiki](https://github.com/alibaba/Sentinel/wiki)

## Step (使用步骤)
> There are steps: **<font color='red'>A、B、C、D、E</font>**

**<font color='red'>A</font>. Clone this project, then compile it**
```
git clone git@github.com:LianjiaTech/yuan-sdk.git
```
this is a maven project:
```
mvn clean source:jar install
OR
mvn clean source:jar deploy
```

---

**<font color='red'>B</font>. Import `yuan-sdk` into your project, or copy it into your classpath.**
```
<dependency>
  <groupId>com.ke.yuan</groupId>
  <artifactId>yuan-sdk</artifactId>
  <version>0.10.1</version>
</dependency>
```
**In Gradle**, Use this:
```
compile 'com.ke.yuan:yuan-sdk:0.10.1'
annotationProcessor "com.ke.yuan:yuan-sdk:0.10.1"
```
**Notice**， `yuan-sdk` works on compile, you mush stay `yuan-sdk` on you classpath when you compile your project.

---

**<font color='red'>C</font>. Point annotation on your entrance class.**

code access, *All the following can be done only once*:
on `main` or `spring-boot` or `servlet` / `spring`

> 你的项目只会属于下面四个场景之一, 选择你合适的场景就好.

 - if you having only `main` in your application, use `@EnableYuan` On your main class。 just like:
 ```
@EnableYuan
public class Main {
    public static void main(String[] args) {
    }
}
 ```
 - if you are in a servlet environment, servlet SPI help you do everything, just enjoy it。
 - if you are in a Spring environment(include servlet), `com.ke.yuan.adaptor.SpringYuAnInitializer` should be 
 included into you Spring scan package.
 - if you are in a Spring Boot environment, use `@EnableSpringYuAn` On you main class
 ```
@EnableYuan
@SpringBootApplication
public class Main {
    public static void main(String[] args) {
        SpringApplication.run(Main.class);
    }
}
 ```
--- 

**<font color='red'>D</font>. On your protected method, have `@YuanCallback` On it.**
 
Just like this :
```
public class XX {

    @YuanCallback
    public void somethingProtected();

}
```

> **Notice**, Sentinel is not recommend for abuse, because *protected method* should be exactly marked in you system,
> so find your protected method first, mark annotation on it.
>
> **注意**, 一般情况下项目中受Sentinel保护的代码应该只占整体代码的一部分, 不应该一刀切的将所有的代码打上注解。

---

**<font color='red'>E</font>. A dashboard is in need.**
 
 > **Notice**, you can changed this by your self.
 >
 > 注意, `yuan-sdk`默认要求使用`sentinel-dashboard`, 但是你可以自行修改源码不使用dashboard, 修改方式参考后续小节
 
 `yuan-sdk` need the following 2 properties(total: `com.ke.yuan.io.SentinelProperties`):
 ```
 csp.sentinel.dashboard.server=sentinel.xxx.com    // 标记dashboard服务端的地址
 project.name=xxx                                  // 标记当前项目的client名称
 ```
 
 The two can either in  a classpath file named `sentinel_yuan_sdk.properties`,
 or in spring properties(such as `yaml`、`properties`、`remote config center`).
 
  `yuan-sdk` 需要的这两个字段(还有更多的字段以及含义可以参考`com.ke.yuan.io.SentinelProperties`)
  目前是必须的, 可以写在一个名为`sentinel_yuan_sdk.properties`的文件里, 如果是Spring环境, 可以写在远端
  配置中心、yaml、任意properties文件里.
  
# What Did We Do 技术原理
  
> we planed java agent、AOP、AspectJ, at last, we select JSR269: Java `Annotation Processor`

see the following java compiling:
![Java编译](http://img.ljcdn.com/alliance-crm-image/8de632bb-e822-4ff4-97d3-0888b8eb4043.jpg)  

Sentinel coder format a lot, JSR269 like it a lot.

Sentinel 原生推荐的写法非常格式化, 而 `JSR269`就非常适合做格式化的编码植入, 直接植入到受保护代码即可.

## mark each method like Sentinel did
Origin:
```
/**
 * 针对Hello World的限流行为, 只需要在需要限流的方法
 * （Spring Controller、普通方法、私有方法、重构的方法等等）
 * 上添加{@link com.ke.yuan.bridge.annotation.YuAnCallback} 注解即可。
 *
 * 注意需要引用SDK:
 * <dependency>
 *   <artifactId>yuan-sdk</artifactId>
 *   <groupId>com.ke.yuan</groupId>
 *   <version>0.0.1</version>
 * </dependency>
 *
 * @author yupeng.qin
 * @since 2019-07-18
 */
public class HelloWorldLimitDemo {
  
    @YuAnCallback
    public void helloWorldWillBeLimited(Integer xxx) {
    }
}
```
Target:
```
public class HelloWorldLimitDemo {
 
    public HelloWorldLimitDemo() {
    }
 
    public void helloWorldWillBeLimited_sentinel(Integer xxx) {
    }
 
    public void helloWorldWillBeLimited(Integer xxx) {
        if (SentinelOpener.shouldOpen()) {
            // sentinel 保护逻辑
            try {
                entry = SphU.entry(resource_key);
                this.helloWorldWillBeLimited_sentinel(xxx);
            } catch (Throwable var11) {
                // sentinel 保护逻辑
            } finally {
                // sentinel 保护逻辑
            }
        } else {
            this.helloWorldWillBeLimited_sentinel(xxx);
        }
 
    }
}
```

## mark a entrance for yuan_sdk 给Sentinel留一个入口
> for properties setting, this can be anywhere that can be executed, we make it in the main class.
> 入口的主要用处是配置一些Sentinel需要用到的参数, 并非必须的, 可以在代码中硬编码植入. 但是直接在入口处植入, 使用 sentinel-dashboard会非常方便.

Java app Origin:
```
@EnableYuan
public class X {

    public static void main(String[] args) {
        SpringApplication.run(X.class);
    }

}
```
Java app Target:
```
public class X {
    public X() {
    }

    public static void main(String[] args) {
        YuAnInitializer.toInit();
        SpringApplication.run(X.class, new String[0]);
    }
}
```

---

Servlet app, see `com.ke.yuan.bridge.platform.YuAnServletInitializer`.

---

Spring Boot app, see `com.ke.yuan.bridge.annotation.EnableSpringYuAn`.

## other usage(一些其他的用法)
**A. hot flow**
> 热点限流也是支持的, 使用 `@HotFlow`注解.

效果：
Origin：
```
    @RequestMapping("/")
    @ResponseBody
    @YuAnCallback
    public String s(@HotFlow String firstIndex) {
        return "1";
    }
```
Target：
```
    public String s_sentinel(String firstIndex) {
        return "1";
    }

    @RequestMapping({"/"})
    @ResponseBody
    public String s(String firstIndex) {
        if (SentinelOpener.shouldOpen()) {
            // sentinel 保护逻辑
            try {
                try {
                    entry = SphU.entry(resource_key, EntryType.IN, 1, new Object[]{firstIndex});
                    String var13 = this.s_sentinel(firstIndex);
                    return var13;
                } catch (Throwable var11) {
                    // sentinel 保护逻辑
                }

                // sentinel 保护逻辑
            } finally {
                if (null != entry) {
                    entry.exit(1, new Object[]{firstIndex});
                }

            }

            return (String)var5;
        } else {
            return this.s_sentinel(firstIndex);
        }
    }
```

**B. look for `com.ke.yuan.YuAnVersion`**
> 功能点都在这个类里面有描述, 例如清除日志、异常回调、指定日志文件夹等.

# How to rewrite me 如果修改你期望的功能
## Before do something (内置的一些功能)

`yuan-sdk` offered some feature by myself, you can active it by `properties`,
`yuan-sdk` 本身自带了一些控制参数， 也支持[Sentinel 支持的参数](https://github.com/alibaba/Sentinel/wiki/%E5%90%AF%E5%8A%A8%E9%85%8D%E7%BD%AE%E9%A1%B9), 你只需要保证进程可以读取到他们即可。

read this one: `com.ke.yuan.io.SentinelProperties`, and mark your properties readable in you environment:

 - in Spring, be found in context;
 - in other, be found in `sentinel_yuan_sdk.properties`(Java OPTS is ok)
 
 `yuan-sdk` features are showed on `com.ke.yuan.YuAnVersion`, read it!
 
## You dont like dashboard(如果你不想要控制台)
 > **Notice**, rule of flow or fuse(burn、degrade), is necessary.
 > They can be write on your java code, or form dashboard, or from config center(such as apollo)

 > **注意**, 对于Sentinel来说, 熔断、限流都需要配置规则才能使用, 官方有 [代码硬编码](https://github.com/alibaba/Sentinel/wiki/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8#%E9%80%9A%E8%BF%87%E4%BB%A3%E7%A0%81%E5%AE%9A%E4%B9%89%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6%E8%A7%84%E5%88%99)、
 [使用dashboard](https://github.com/alibaba/Sentinel/wiki/%E6%8E%A7%E5%88%B6%E5%8F%B0)(同时也是`yuan-sdk`使用的方案), 
 使用apollo之内的[配置中心](https://github.com/alibaba/Sentinel/wiki/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%99%E6%89%A9%E5%B1%95)
 
 So its easy:
 
  1. Remove the annotation `@EnableSpringYuAn` OR `@EnableYuAn` on your [entrance](#mark-a-entrance-for-yuan_sdk-给sentinel留一个入口).
  2. Ensure the rule can be read by `yuan-sdk`/Sentinel
  
## Introduce point(关键类介绍)
> `yuan-sdk` didnt copy principle of opening and closing, rewrite code by yourself.
> `yuan-sdk` 对开闭原则遵循较少, 可能需要你去修改源码(如果你要升级代码的话).

|Feature|Class|Notice|
|--|--|--|
|method compile|com.ke.yuan.jsr269.YuAnAnnotationProcessor|this is for `@YuanCallback`|
|main class compile|com.ke.yuan.jsr269.YuAnMainProcessor|this is for `@EnableYuan`|
|servlet|com.ke.yuan.bridge.platform.YuAnServletInitializer|for servlet, but spring first|
|spring|com.ke.yuan.adaptor.SpringYuAnInitializer|insert into spring environment|
|file execute|com.ke.yuan.bridge.platform.YuAnInitializer|all the file execute is here, include properties|
|open or close|com.ke.yuan.io.SentinelOpener|if you have no <br/> open or close control(http)<br/> you may return `true` forever|

> 在`yuan-sdk`中配置了一个地址(`/sdk/yuan-domain`)用来设定是否启用预案服务, 可以直接在代码中删除。

for other, look at this flow(may help rewrite code):
![流程图](http://img.ljcdn.com/alliance-crm-image/72660e09-157d-4389-a22a-ec788434e6d4.jpg)

# Something you didnt understand 一些问题和解决方案
## You are using config center(Spring)
> such as apollo or another config center.
> `yuan-sdk` use `PropertySourcesPlaceholderConfigurer` for properties getting(Spring),
> if properties can not be found, use `BeanPropertyAdaptor`

Just like this:
```$xslt
/**
 * 这个类的用处, 是在引用了第三方配置中心, 而刚好第三方配置中心与预案SDK的兼容性还不佳的情况。 将配置桥接给预案SDK
 *
 * 标准的预案SDK参数(参数如果写在 sentinel_yuan_sdk.properties中, 预案SDK自身就会去读, 不需要再写这个类):
 *
 * yuan.print=true                                  // 这个参数控制是否在标准输出中打印出Sentinel的配置信息
 * csp.sentinel.dashboard.server=sentinel.xxx.com   // 标记服务端的地址
 * project.name=ke-1234                             // 标记当前项目的名称，以业务系统module名字命名即可
 *
 * 将这三个参数塞到BeanPropertyAdaptor的Properties里面。
 * 你可以写成本类Demo这样, 也可以写在一个独立的类里面. 只要实现了 {@link com.ke.yuan.adaptor.BeanPropertyAdaptor}并委托给了Spring, 都是可以的。
 *
 * @see com.ke.yuan.io.SentinelProperties#CONFIG_FILE_NAME
 */
@Configuration
public class OfferEnvironment {
 
    @Value("${project.name}")
    private String projectName;
    @Value("${yuan.print}")
    private String projectPrint;
    @Value("${csp.sentinel.dashboard.server}")
    private String dashboardServer;
    @Value("${csp.sentinel.sdk.open.url}")
    private String url;
    @Bean
    @Primary
    public BeanPropertyAdaptor beanPropertyAdaptor() {
        return new BeanPropertyAdaptor() {
            @Override
            public Properties sentinelProperties() {
                Properties p = new Properties();
                p.setProperty("project.name", projectName);
                p.setProperty("yuan.print", projectPrint);
                p.setProperty("csp.sentinel.dashboard.server", dashboardServer);
                p.setProperty("csp.sentinel.sdk.open.url", url);
                return p;
            }
        };
    }
}
```

## Too many log file
> Ensure these properties in you environment:

```$xslt
// 删除 当前进程启动前三日的， xxx-metrics.log.pid22314.2019-12-03 这个格式的日志文件
// 运行期不会删， 例如你的程序运行了10天还没重启， 那还是会有12个日志文件
csp.clear.pre.log=true
 
// 清除 "command-center.log.pid22314.2019-12-03.0" 文件;
// 清除 "sentinel-record.log.pid22314.2019-12-03.0" 文件;
// 运行期不会删， 例如你的程序运行了10天还没重启， 那还是会有12个日志文件
csp.clear.command.log=true
 
// 指定日志的目录， 如果未指定， 它要么在 ./sentinel_log_dir 下， 要么在 ${MATRIX_PRIVDATA_DIR} 下（生产环境默认）
// 需要指定存在的目录
csp.sentinel.log.dir=/Users/dumpling/workspace/qyp/sentinel_log_dir/xxx
```

## You are using eclipse

eclipse compile java by itself, so use `mv clean package` first.

eclipse自己做了编译器， 运行的时候先不要用eclipse运行， 先 `mvn clean package`编译一遍再运行。
