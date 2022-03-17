---
description: Analysis of  Skywalking
---

# Skywalking解析

## skywalking组件

Skywalking是一款比较火的APM(Application Performance Management)程序，负责应用程序的可观测性。其主要流程如下：首先是各种agent负责采集数据(Trace、Metrics、Log)，agent可以是自动注入和手动采集方式，通过grpc或kafka或http发送到后端OAP系统，OAP负责分析和展示。

![skywalking官方架构图](<../.gitbook/assets/image (3) (1).png>)



写这篇文章的目的主要是介绍Skywalking主要模块的功能，让自己对其源码更加熟悉。

## skywalking-agent启动流程

我们介绍java端是如何进行自动采集的，依赖于java-agent机制，在我们的程序运行之前，先运行java-agent的代码，将目标字节码替换成注入采集逻辑的代码，比如在方法之前开启Trace、在方法之后结束Trace。

`org.apache.skywalking.apm.agent.SkyWalkingAgent`类是整个agent机制的入口，需要按照java-agent机制的要求实现`premain`方法，我们只需要关注测方法即可。

### 1.初始化配置

```
SnifferConfigInitializer.initializeCoreConfig(agentArgs);
```

看一下具体怎么实现`initializeCoreConfig`，进入`org.apache.skywalking.apm.agent.core.conf.SnifferConfigInitializer`类，我们一步步看下`initializeCoreConfig`都做了什么，在关键方法上都做了注释，详细的实现就不跟进去了，比较基础。

```
 public static void initializeCoreConfig(String agentOptions) {
        //利用jdk的Properties类进行配置存储
        AGENT_SETTINGS = new Properties();
        //加载配置文件的输入流
        try (final InputStreamReader configFileStream = loadConfig()) {
            AGENT_SETTINGS.load(configFileStream);
            //遍历Properties进行占位符替换
            for (String key : AGENT_SETTINGS.stringPropertyNames()) {
                String value = (String) AGENT_SETTINGS.get(key);
                AGENT_SETTINGS.put(key, PropertyPlaceholderHelper.INSTANCE.replacePlaceholders(value, AGENT_SETTINGS));
            }

        } catch (Exception e) {
            LOGGER.error(e, "Failed to read the config file, skywalking is going to run in default config.");
        }

        try {
            //用环境变量的配置来覆盖配置
            overrideConfigBySystemProp();
        } catch (Exception e) {
            LOGGER.error(e, "Failed to read the system properties.");
        }

        agentOptions = StringUtil.trim(agentOptions, ',');
        if (!StringUtil.isEmpty(agentOptions)) {
            try {
                agentOptions = agentOptions.trim();
                LOGGER.info("Agent options is {}.", agentOptions);
                //用agent的args来覆盖配置
                overrideConfigByAgentOptions(agentOptions);
            } catch (Exception e) {
                LOGGER.error(e, "Failed to parse the agent options, val is {}.", agentOptions);
            }
        }
        //初始化config类，这里是用Properties里的值来反射塞到config的static变量中
        initializeConfig(Config.class);
        // reconfigure logger after config initialization
        //根据配置重新设置loggerManager的Resolver:分为JSON和PATTERN两种
        configureLogger();
        //重新获取logger
        LOGGER = LogManager.getLogger(SnifferConfigInitializer.class);

        //防御变成，判断一些配置的不可缺失性
        if (StringUtil.isEmpty(Config.Agent.SERVICE_NAME)) {
            throw new ExceptionInInitializerError("`agent.service_name` is missing.");
        }
        if (StringUtil.isEmpty(Config.Collector.BACKEND_SERVICE)) {
            throw new ExceptionInInitializerError("`collector.backend_service` is missing.");
        }
        if (Config.Plugin.PEER_MAX_LENGTH <= 3) {
            LOGGER.warn(
                "PEER_MAX_LENGTH configuration:{} error, the default value of 200 will be used.",
                Config.Plugin.PEER_MAX_LENGTH
            );
            Config.Plugin.PEER_MAX_LENGTH = 200;
        }
        //配置初始完成
        IS_INIT_COMPLETED = true;
    }
```

### 2.重新获取LOGGER

因为用户可能会配置logger

```
LOGGER = LogManager.getLogger(SkyWalkingAgent.class);
```

#### 3.加载插件并封装成PluginFinder

```
PluginFinder pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
```

首先要知道什么是Plugin，看一下下图就大概明白了，此处插件的意思就是针对每个组件的各种采集逻辑。

![plugins](<../.gitbook/assets/image (2).png>)

有两个重要的步骤`loadPlugins()` ,`new PluginFinder()`,我们依次来分析。

3.1 加载插件

```
    public List<AbstractClassEnhancePluginDefine> loadPlugins() throws AgentPackageNotFoundException {
        //初始化类加载器，这里是自己实现的AgentClassLoader
        AgentClassLoader.initDefaultLoader();

        PluginResourcesResolver resolver = new PluginResourcesResolver();
        //加载所有插件的url，即每个插件skywalking-plugin.def文件下的东西
        List<URL> resources = resolver.getResources();

        if (resources == null || resources.size() == 0) {
            LOGGER.info("no plugin files (skywalking-plugin.def) found, continue to start application.");
            return new ArrayList<AbstractClassEnhancePluginDefine>();
        }

        //根据每个plugin的位置加载输入流(def文件),然后转成PluginDefine类，同时会去除配置中EXCLUDE_PLUGINS配置的插件
        for (URL pluginUrl : resources) {
            try {
                PluginCfg.INSTANCE.load(pluginUrl.openStream());
            } catch (Throwable t) {
                LOGGER.error(t, "plugin file [{}] init failure.", pluginUrl);
            }
        }

        //获取到所有PluginDefine的列表
        List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();

        List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
        //对PluginDefine一次进行类加载，注意是使用defaultAgentClassLoader进行加载，并加入到plugins中
        for (PluginDefine pluginDefine : pluginClassList) {
            try {
                LOGGER.debug("loading plugin class {}.", pluginDefine.getDefineClass());
                AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(pluginDefine.getDefineClass(), true, AgentClassLoader
                    .getDefault()).newInstance();
                plugins.add(plugin);
            } catch (Throwable t) {
                LOGGER.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
            }
        }
        //加载基于xml配置的插件
        plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));

        return plugins;

    }
```

3.2 初始化PluginFinder，从名字可以看出此类的作用查找，是根据TypeDescription来查找

List\<AbstractClassEnhancePluginDefine>，即根据类描述来查找器对应的增强插件。

```
   //PluginFinder初始化
     public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {
        for (AbstractClassEnhancePluginDefine plugin : plugins) {
            //获取到要增强的类
            ClassMatch match = plugin.enhanceClass();

            if (match == null) {
                continue;
            }

            //是名称匹配
            if (match instanceof NameMatch) {
                NameMatch nameMatch = (NameMatch) match;
                //添加到累的增加列表中
                LinkedList<AbstractClassEnhancePluginDefine> pluginDefines = nameMatchDefine.get(nameMatch.getClassName());
                if (pluginDefines == null) {
                    pluginDefines = new LinkedList<AbstractClassEnhancePluginDefine>();
                    nameMatchDefine.put(nameMatch.getClassName(), pluginDefines);
                }
                pluginDefines.add(plugin);
            } else {
                //不是名称匹配就添加到签名匹配中
                signatureMatchDefine.add(plugin);
            }

            //添加到启动类匹配中，这里值得是jdk中的类
            if (plugin.isBootstrapInstrumentation()) {
                bootstrapClassMatchDefine.add(plugin);
            }
        }
    }
```

类匹配机制除了NameMatch(名称匹配，判断字符串相等)之外，还有RegexMatch(正则匹配)、PrefixMatch(前缀匹配)、AnnotationMatch(注解匹配，包括class和method的注解)，匹配机制还是挺丰富的。下图是ClassMatch的继承体系。

![ClassMatch继承体系](<../.gitbook/assets/image (3).png>)

3.3 PluginFinder的查找

```
    public List<AbstractClassEnhancePluginDefine> find(TypeDescription typeDescription) {
        List<AbstractClassEnhancePluginDefine> matchedPlugins = new LinkedList<AbstractClassEnhancePluginDefine>();
        String typeName = typeDescription.getTypeName();
        //名称匹配就是简单的字符串匹配
        if (nameMatchDefine.containsKey(typeName)) {
            matchedPlugins.addAll(nameMatchDefine.get(typeName));
        }

        //签名匹配要调用match的isMatch方法
        for (AbstractClassEnhancePluginDefine pluginDefine : signatureMatchDefine) {
            IndirectMatch match = (IndirectMatch) pluginDefine.enhanceClass();
            if (match.isMatch(typeDescription)) {
                matchedPlugins.add(pluginDefine);
            }
        }

        return matchedPlugins;
    }
```

4.初始化ByteBuddy，并忽略掉一些类。

```
 final ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));

        AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy).ignore(
                nameStartsWith("net.bytebuddy.")
                        .or(nameStartsWith("org.slf4j."))
                        .or(nameStartsWith("org.groovy."))
                        .or(nameContains("javassist"))
                        .or(nameContains(".asm."))
                        .or(nameContains(".reflectasm."))
                        .or(nameStartsWith("sun.reflect"))
                        .or(allSkyWalkingAgentExcludeToolkit())
                        .or(ElementMatchers.isSynthetic()));
```

5\.

```
 JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();

```

5\.

```
agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);

```

6.针对jdk9的模块化做处理

```
agentBuilder = JDK9ModuleExporter.openReadEdge(instrumentation, agentBuilder, edgeClasses);

```

7.判断是否开启缓存，注入缓存机制

```
       if (Config.Agent.IS_CACHE_ENHANCED_CLASS) {
            try {
                agentBuilder = agentBuilder.with(new CacheableTransformerDecorator(Config.Agent.CLASS_CACHE_MODE));
                LOGGER.info("SkyWalking agent class cache [{}] activated.", Config.Agent.CLASS_CACHE_MODE);
            } catch (Exception e) {
                LOGGER.error(e, "SkyWalking agent can't active class cache.");
            }
        }
```

8\. 实现字节码增强

```
        agentBuilder.type(pluginFinder.buildMatch()) //要修改的类
                    .transform(new Transformer(pluginFinder)) //字节码修改逻辑，直接集成bytebuddy的类
                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
                    .with(new RedefinitionListener())
                    .with(new Listener())
                    .installOn(instrumentation);
```

9.启动skywalking的服务

```
//ServiceManager.INSTANCE.boot();
```

### skywalking 链路采集

### skywalking 指标采集

### skywalking 日志采集

### skywalking-oap接收

