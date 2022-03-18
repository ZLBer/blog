---
description: Skywalking agent startup process
---

# skywalking-agent启动流程

我们介绍java端是如何进行自动采集的，依赖于java-agent机制，在我们的程序运行之前，先运行java-agent的代码，将目标字节码替换成注入采集逻辑的代码，比如在方法之前开启Trace、在方法之后结束Trace。skywalking用ByteBuddy进行字节码增强的，可以先看下ByteBuddy怎么操作java-agent：

* [https://notes.diguage.com/byte-buddy-tutorial/#creating-java-agents](https://notes.diguage.com/byte-buddy-tutorial/#creating-java-agents)

`org.apache.skywalking.apm.agent.SkyWalkingAgent`类是整个agent机制的入口，需要按照java-agent机制的要求实现`premain`方法，我们一次介绍agent加载的具体步骤，也就是premain方法里的逻辑。

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

因为用户可能会在配置里重新配置logger

```
LOGGER = LogManager.getLogger(SkyWalkingAgent.class);
```

### 3.加载插件并封装成PluginFinder

```
PluginFinder pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
```

首先要知道什么是Plugin，看一下下图就大概明白了，此处插件的意思就是针对每个组件的各种采集逻辑。

![plugins](<../../.gitbook/assets/image (2).png>)

有两个重要的步骤`loadPlugins()` ,`new PluginFinder()`,我们依次来分析。

#### 3.1 加载插件`loadPlugins方法`

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

#### 3.2 PluginFinder的构造器

从名字可以看出此类的作用查找，是根据TypeDescription来查找

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

![ClassMatch继承体系](<../../.gitbook/assets/image (3) (1).png>)

#### 3.3 PluginFinder类的find方法

find方法没出现在此处，但可以简单介绍下。

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

### 4.初始化ByteBuddy

利用ByteBuddy的AgentBuilder进行初始化构造，主要是忽略到一些类。

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

### 5.获取边界类

```
 JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();

```

### 6.处理jdk注入

BootstrapInstrumentBoost主要用来处理对jdk类的增强。

```
agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);

```

#### 6.1 inject方法

inject来进行jkd增强的具体构造

```
   public static AgentBuilder inject(PluginFinder pluginFinder, Instrumentation instrumentation,
        AgentBuilder agentBuilder, JDK9ModuleExporter.EdgeClasses edgeClasses) throws PluginException {
        Map<String, byte[]> classesTypeMap = new HashMap<>();

        /*
        针对目标类是jdk核心类库的插件，这里根据的拦截点的不同（实例方法、静态方法、构造方法）
        使用不同的模板（XXTemplate）来构造新的拦截器的核心处理逻辑，并且将插件本身定义的拦截器的全类名
        赋值给模板的 XX  子段
        最终，这些新的拦截器的核心处理逻辑会放入Bootstrap ClassLoader中
         */
        if (!prepareJREInstrumentation(pluginFinder, classesTypeMap)) {
            return agentBuilder;
        }

        if (!prepareJREInstrumentationV2(pluginFinder, classesTypeMap)) {
            return agentBuilder;
        }
        //加载高优先级的类到classesTypeMap中
        for (String highPriorityClass : HIGH_PRIORITY_CLASSES) {
            loadHighPriorityClass(classesTypeMap, highPriorityClass);
        }
        //加载ByteBuddy的核心类到classesTypeMap中
        for (String highPriorityClass : ByteBuddyCoreClasses.CLASSES) {
            loadHighPriorityClass(classesTypeMap, highPriorityClass);
        }

        /**
         * Prepare to open edge of necessary classes.
         */
        for (String generatedClass : classesTypeMap.keySet()) {
            edgeClasses.add(generatedClass);
        }

        /**
         * Inject the classes into bootstrap class loader by using Unsafe Strategy.
         * ByteBuddy adapts the sun.misc.Unsafe and jdk.internal.misc.Unsafe automatically.
         */
        //把这些类注入到bootstrapClassLoader中
        ClassInjector.UsingUnsafe.Factory factory = ClassInjector.UsingUnsafe.Factory.resolve(instrumentation);
        factory.make(null, null).injectRaw(classesTypeMap);
        agentBuilder = agentBuilder.with(new AgentBuilder.InjectionStrategy.UsingUnsafe.OfFactory(factory));

        return agentBuilder;
    }
```

在classesTypeMap中添加的类都是要用BoostrapClassLoader去加载的，其中有一项是HIGH\_PRIORITY\_CLASSES 高优先类，我们看一下其具体的内容，这为后面的类加载打通通道。

![HIGH\_PRIORITY\_CLASSES](<../../.gitbook/assets/image (3).png>)

#### 6.2 prepareJREInstrumentation方法

根据不同的模板为bytebuddy生成jdk的注入代码

```
    private static boolean prepareJREInstrumentation(PluginFinder pluginFinder,
        Map<String, byte[]> classesTypeMap) throws PluginException {
        TypePool typePool = TypePool.Default.of(BootstrapInstrumentBoost.class.getClassLoader());
        List<AbstractClassEnhancePluginDefine> bootstrapClassMatchDefines = pluginFinder.getBootstrapClassMatchDefine();
        //开始遍历bootstrapClassMatchDefines
        //根据不同的拦截点选择不同的java模板来生成增强类
        for (AbstractClassEnhancePluginDefine define : bootstrapClassMatchDefines) {
            //处理实例方法拦截点
            if (Objects.nonNull(define.getInstanceMethodsInterceptPoints())) {
                for (InstanceMethodsInterceptPoint point : define.getInstanceMethodsInterceptPoints()) {
                    if (point.isOverrideArgs()) {
                        generateDelegator(
                            classesTypeMap, typePool, INSTANCE_METHOD_WITH_OVERRIDE_ARGS_DELEGATE_TEMPLATE, point
                                .getMethodsInterceptor());
                    } else {
                        generateDelegator(
                            classesTypeMap, typePool, INSTANCE_METHOD_DELEGATE_TEMPLATE, point.getMethodsInterceptor());
                    }
                }
            }
            //处理构造器方法拦截点
            if (Objects.nonNull(define.getConstructorsInterceptPoints())) {
                for (ConstructorInterceptPoint point : define.getConstructorsInterceptPoints()) {
                    generateDelegator(
                        classesTypeMap, typePool, CONSTRUCTOR_DELEGATE_TEMPLATE, point.getConstructorInterceptor());
                }
            }
            //处理静态方法拦截点
            if (Objects.nonNull(define.getStaticMethodsInterceptPoints())) {
                for (StaticMethodsInterceptPoint point : define.getStaticMethodsInterceptPoints()) {
                    if (point.isOverrideArgs()) {
                        generateDelegator(
                            classesTypeMap, typePool, STATIC_METHOD_WITH_OVERRIDE_ARGS_DELEGATE_TEMPLATE, point
                                .getMethodsInterceptor());
                    } else {
                        generateDelegator(
                            classesTypeMap, typePool, STATIC_METHOD_DELEGATE_TEMPLATE, point.getMethodsInterceptor());
                    }
                }
            }
        }
        //是不是进行了jdk类增强
        return bootstrapClassMatchDefines.size() > 0;
    }
```

#### 6.3 generateDelegator方法

生成委派器

```
    private static void generateDelegator(Map<String, byte[]> classesTypeMap, TypePool typePool,
        String templateClassName, String methodsInterceptor) {
        //internalInterceptorName = methodName_internal
        String internalInterceptorName = internalDelegate(methodsInterceptor);
        try {
            TypeDescription templateTypeDescription = typePool.describe(templateClassName).resolve();

            //根据模板构造class类， 将TARGET_INTERCEPTOR替换成真正的methodsInterceptor
            //这里的classloader是BootstrapInstrumentBoost的类加载器
            DynamicType.Unloaded interceptorType = new ByteBuddy().redefine(templateTypeDescription, ClassFileLocator.ForClassLoader
                .of(BootstrapInstrumentBoost.class.getClassLoader()))
                                                                  .name(internalInterceptorName)
                                                                  .field(named("TARGET_INTERCEPTOR"))
                                                                  .value(methodsInterceptor)
                                                                  .make();

            //将根据模板构造的类添加到传入的classesTypeMap中
            classesTypeMap.put(internalInterceptorName, interceptorType.getBytes());

            InstrumentDebuggingClass.INSTANCE.log(interceptorType);
        } catch (Exception e) {
            throw new PluginException("Generate Dynamic plugin failure", e);
        }
    }
```

#### 6.4 InstanceMethodInterTemplate模板类

我们看一下模板具体是怎么样的，这是根据byteBuddy做的模板，prepare方法理解起来比较困难，我已经做了详细的注释。除此之外，还有ConstructorInterTemplate、StaticMethodInterTemplate等等。

```
public class InstanceMethodInterTemplate {
    /**
     * This field is never set in the template, but has value in the runtime.
     */
    private static String TARGET_INTERCEPTOR;

    private static InstanceMethodsAroundInterceptor INTERCEPTOR;
    private static IBootstrapLog LOGGER;

    /**
     * Intercept the target instance method.
     *
     * @param obj          target class instance.
     * @param allArguments all method arguments
     * @param method       method description.
     * @param zuper        the origin call ref.
     * @return the return value of target instance method.
     * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
     *                   bug, if anything triggers this condition ).
     */
    @RuntimeType
    public static Object intercept(@This Object obj, @AllArguments Object[] allArguments, @SuperCall Callable<?> zuper,
        @Origin Method method) throws Throwable {
        EnhancedInstance targetObject = (EnhancedInstance) obj;

        //填充INTERCEPTOR和LOGGER
        prepare();

        MethodInterceptResult result = new MethodInterceptResult();
        try {
            if (INTERCEPTOR != null) {
                INTERCEPTOR.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
            }
        } catch (Throwable t) {
            if (LOGGER != null) {
                LOGGER.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
            }
        }

        Object ret = null;
        try {
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                ret = zuper.call();
            }
        } catch (Throwable t) {
            try {
                if (INTERCEPTOR != null) {
                    INTERCEPTOR.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
                }
            } catch (Throwable t2) {
                if (LOGGER != null) {
                    LOGGER.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
                }
            }
            throw t;
        } finally {
            try {
                if (INTERCEPTOR != null) {
                    ret = INTERCEPTOR.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
                }
            } catch (Throwable t) {
                if (LOGGER != null) {
                    LOGGER.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
                }
            }
        }

        return ret;
    }

    /**
     * Prepare the context. Link to the agent core in AppClassLoader.
     */
    //此处是bootstrapclassloader加载的 ,根据双亲委派模型，是获取不到AgentClassLoader加载的类的，
    // 而拦截器都是AgentClassLoader加载的，所以要打通BootStrapClassLoader和AgentClasssLoader
    private static void prepare() {
        if (INTERCEPTOR == null) {
            //获取Agentclassloader 通过反射获取default_agentclassloader字段
            ClassLoader loader = BootstrapInterRuntimeAssist.getAgentClassLoader();

            if (loader != null) {
                //获取logger loggerManager是appclassloader加载的, 此处是BootStrapClassLoader执行，所以直接获取不到loggerManager
                //此处用的方法是用Agentclassloader去加载一个桥接器BootstrapPluginLogBridge，桥接器集成IBootstrapLog(此类是BootstrapClassloader加载的)
                // 然后桥接器调用getLogger去调用loggerManager获取到logger, 并封装成IBootstrapLog返回
                //此处比较复杂，需要对类加载器的流程比较熟悉，主要做法就是：用BootStrapClassLoader的类IBootstrapLog 封装logger的方法，
                // 让Agentclassloader的类BootstrapPluginLogBridge去实现此接口，然后去实际操作logger
                // 父加载器持有接口，子加载器负责实现  ---重点
                //为什么不直接反射获取logger？因为BootStrapClassLoader不认识logger的类
                IBootstrapLog logger = BootstrapInterRuntimeAssist.getLogger(loader, TARGET_INTERCEPTOR);
                if (logger != null) {
                    LOGGER = logger;
                    //用defaultAgentClassLoader去加载interceptor，也是父加载器持有接口，子加载器负责实现
                    //因为是BootStrapClassloader加载的InstanceMethodsAroundInterceptor接口
                    INTERCEPTOR = BootstrapInterRuntimeAssist.createInterceptor(loader, TARGET_INTERCEPTOR, LOGGER);
                }

            } else {
                LOGGER.error("Runtime ClassLoader not found when create {}." + TARGET_INTERCEPTOR);
            }
        }
    }
}
```

至此，针对jdk增强的构造器已经生成。

### 7.针对jdk9的模块化做处理

打开模块类的读边界

```
agentBuilder = JDK9ModuleExporter.openReadEdge(instrumentation, agentBuilder, edgeClasses);

```

### 8.判断是否开启缓存，注入缓存机制

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

### 9. 实现字节码增强

除了jdk的增强，其余组件的增强都在这里处理 。transform是具体的转化逻辑，with是添加监听器，installOn进行字节码增强替换。

```
        agentBuilder.type(pluginFinder.buildMatch()) //要修改的类
                    .transform(new Transformer(pluginFinder)) //字节码修改逻辑，直接集成bytebuddy的类
                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
                    .with(new RedefinitionListener())
                    .with(new Listener())
                    .installOn(instrumentation);
```

#### 9.1 Transformer类

Transformer是byteBuddy定义的接口，用来封装类的转换逻辑

```
   private static class Transformer implements AgentBuilder.Transformer {
        private PluginFinder pluginFinder;

        Transformer(PluginFinder pluginFinder) {
            this.pluginFinder = pluginFinder;
        }

        //tranform主要是返回新的builder，进行类增强
        //TypeDescription 是类定义
        //ClassLoader是当前类的类加载器
        @Override
        public DynamicType.Builder<?> transform(final DynamicType.Builder<?> builder,
                                                final TypeDescription typeDescription,
                                                final ClassLoader classLoader,
                                                final JavaModule module) {
            //此处的classLoader是加载这个类的类加载器
            LoadedLibraryCollector.registerURLClassLoader(classLoader);
            //获取这个类的增强定义
            List<AbstractClassEnhancePluginDefine> pluginDefines = pluginFinder.find(typeDescription);
            if (pluginDefines.size() > 0) {
                DynamicType.Builder<?> newBuilder = builder;
                EnhanceContext context = new EnhanceContext();
                for (AbstractClassEnhancePluginDefine define : pluginDefines) {
                    //循环define
                    DynamicType.Builder<?> possibleNewBuilder = define.define(
                            typeDescription, newBuilder, classLoader, context);
                    if (possibleNewBuilder != null) {
                        newBuilder = possibleNewBuilder;
                    }
                }
                if (context.isEnhanced()) {
                    LOGGER.debug("Finish the prepare stage for {}.", typeDescription.getName());
                }

                return newBuilder;
            }

            LOGGER.debug("Matched class {}, but ignore by finding mechanism.", typeDescription.getTypeName());
            return builder;
        }
    }
```

#### 9.2 define方法

AbstractClassEnhancePluginDefine.define&#x20;

```
 
     public DynamicType.Builder<?> define(TypeDescription typeDescription, DynamicType.Builder<?> builder,
        ClassLoader classLoader, EnhanceContext context) throws PluginException {
        String interceptorDefineClassName = this.getClass().getName();
        String transformClassName = typeDescription.getTypeName();
        if (StringUtil.isEmpty(transformClassName)) {
            LOGGER.warn("classname of being intercepted is not defined by {}.", interceptorDefineClassName);
            return null;
        }

        LOGGER.debug("prepare to enhance class {} by {}.", transformClassName, interceptorDefineClassName);
        WitnessFinder finder = WitnessFinder.INSTANCE;
        /**
         * find witness classes for enhance class
         */
        //判断要增强的类是不是存在，不存在就打warn日志
        String[] witnessClasses = witnessClasses();
        if (witnessClasses != null) {
            for (String witnessClass : witnessClasses) {
                if (!finder.exist(witnessClass, classLoader)) {
                    LOGGER.warn("enhance class {} by plugin {} is not working. Because witness class {} is not existed.", transformClassName, interceptorDefineClassName, witnessClass);
                    return null;
                }
            }
        }
        //判断要增强的方法是不是存在，不存在就打warn日志
        List<WitnessMethod> witnessMethods = witnessMethods();
        if (!CollectionUtil.isEmpty(witnessMethods)) {
            for (WitnessMethod witnessMethod : witnessMethods) {
                if (!finder.exist(witnessMethod, classLoader)) {
                    LOGGER.warn("enhance class {} by plugin {} is not working. Because witness method {} is not existed.", transformClassName, interceptorDefineClassName, witnessMethod);
                    return null;
                }
            }
        }

        /**
         * find origin class source code for interceptor
         */
        DynamicType.Builder<?> newClassBuilder = this.enhance(typeDescription, builder, classLoader, context);

        context.initializationStageCompleted();
        LOGGER.debug("enhance class {} by {} completely.", transformClassName, interceptorDefineClassName);

        return newClassBuilder;
    } 
```

#### 9.3 enhance方法

enhance是增强逻辑的入口，包括enhanceClass和enhanceInstance。这里我们只看下enhanceInstance。

```
    protected DynamicType.Builder<?> enhance(TypeDescription typeDescription, DynamicType.Builder<?> newClassBuilder,
                                             ClassLoader classLoader, EnhanceContext context) throws PluginException {
        //增强静态方法
        newClassBuilder = this.enhanceClass(typeDescription, newClassBuilder, classLoader);

        //增强实例方法
        newClassBuilder = this.enhanceInstance(typeDescription, newClassBuilder, classLoader, context);

        return newClassBuilder;
    }
```

#### 9.4 enhanceInstance 方法

实例增强逻辑，其实现在ClassEnhancePluginDefine。

```
    protected DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription,
        DynamicType.Builder<?> newClassBuilder, ClassLoader classLoader,
        EnhanceContext context) throws PluginException {
        //获取各种增强点
        ConstructorInterceptPoint[] constructorInterceptPoints = getConstructorsInterceptPoints();
        InstanceMethodsInterceptPoint[] instanceMethodsInterceptPoints = getInstanceMethodsInterceptPoints();
        String enhanceOriginClassName = typeDescription.getTypeName();
        boolean existedConstructorInterceptPoint = false;
        if (constructorInterceptPoints != null && constructorInterceptPoints.length > 0) {
            existedConstructorInterceptPoint = true;
        }
        boolean existedMethodsInterceptPoints = false;
        if (instanceMethodsInterceptPoints != null && instanceMethodsInterceptPoints.length > 0) {
            existedMethodsInterceptPoints = true;
        }

        /**
         * nothing need to be enhanced in class instance, maybe need enhance static methods.
         */
        if (!existedConstructorInterceptPoint && !existedMethodsInterceptPoints) {
            return newClassBuilder;
        }

        /**
         * Manipulate class source code.<br/>
         *
         * new class need:<br/>
         * 1.Add field, name {@link #CONTEXT_ATTR_NAME}.
         * 2.Add a field accessor for this field.
         *
         * And make sure the source codes manipulation only occurs once.
         *
         */
        //添加新的字段并设置private
        if (!typeDescription.isAssignableTo(EnhancedInstance.class)) {
            if (!context.isObjectExtended()) {
                newClassBuilder = newClassBuilder.defineField(
                    CONTEXT_ATTR_NAME, Object.class, ACC_PRIVATE | ACC_VOLATILE)
                                                 .implement(EnhancedInstance.class)
                                                 .intercept(FieldAccessor.ofField(CONTEXT_ATTR_NAME));
                context.extendObjectCompleted();
            }
        }

        /**
         * 2. enhance constructors
         */
        //对构造器进行增强
        if (existedConstructorInterceptPoint) {
            for (ConstructorInterceptPoint constructorInterceptPoint : constructorInterceptPoints) {
                if (isBootstrapInstrumentation()) {
                    newClassBuilder = newClassBuilder.constructor(constructorInterceptPoint.getConstructorMatcher())
                                                     .intercept(SuperMethodCall.INSTANCE.andThen(MethodDelegation.withDefaultConfiguration()
                                                                                                                 .to(BootstrapInstrumentBoost
                                                                                                                     .forInternalDelegateClass(constructorInterceptPoint
                                                                                                                         .getConstructorInterceptor()))));
                } else {
                    newClassBuilder = newClassBuilder.constructor(constructorInterceptPoint.getConstructorMatcher())
                                                     .intercept(SuperMethodCall.INSTANCE.andThen(MethodDelegation.withDefaultConfiguration()
                                                                                                                 .to(new ConstructorInter(constructorInterceptPoint
                                                                                                                     .getConstructorInterceptor(), classLoader))));
                }
            }
        }

        /**
         * 3. enhance instance methods
         */
        //对实例方法进行增强
        if (existedMethodsInterceptPoints) {
            for (InstanceMethodsInterceptPoint instanceMethodsInterceptPoint : instanceMethodsInterceptPoints) {
                String interceptor = instanceMethodsInterceptPoint.getMethodsInterceptor();
                if (StringUtil.isEmpty(interceptor)) {
                    throw new EnhanceException("no InstanceMethodsAroundInterceptor define to enhance class " + enhanceOriginClassName);
                }
                //junction是匹配的实例方法
                ElementMatcher.Junction<MethodDescription> junction = not(isStatic()).and(instanceMethodsInterceptPoint.getMethodsMatcher());
                if (instanceMethodsInterceptPoint instanceof DeclaredInstanceMethodsInterceptPoint) {
                    junction = junction.and(ElementMatchers.<MethodDescription>isDeclaredBy(typeDescription));
                }
                if (instanceMethodsInterceptPoint.isOverrideArgs()) {
                    if (isBootstrapInstrumentation()) {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                    .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                    } else {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .withBinders(Morph.Binder.install(OverrideCallable.class))
                                                                                    .to(new InstMethodsInterWithOverrideArgs(interceptor, classLoader)));
                    }
                } else {
                    if (isBootstrapInstrumentation()) {
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(BootstrapInstrumentBoost.forInternalDelegateClass(interceptor)));
                    } else {
                        //method()是匹配的方法，intercept是拦截具体的实现
                        newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(new InstMethodsInter(interceptor, classLoader)));
                    }
                }
            }
        }
        //返回新的builder
        return newClassBuilder;
    }
```

#### 9.5 InstMethodsInter方法

我们只看下实例方法的增强类，和jdk增强模板是一样的，只是没有了prepare方法更简单了。在构造方法里回去加载intercepter。

```
public class InstMethodsInter {
    private static final ILog LOGGER = LogManager.getLogger(InstMethodsInter.class);

    /**
     * An {@link InstanceMethodsAroundInterceptor} This name should only stay in {@link String}, the real {@link Class}
     * type will trigger classloader failure. If you want to know more, please check on books about Classloader or
     * Classloader appointment mechanism.
     */
    private InstanceMethodsAroundInterceptor interceptor;

    /**
     * @param instanceMethodsAroundInterceptorClassName class full name.
     */
    public InstMethodsInter(String instanceMethodsAroundInterceptorClassName, ClassLoader classLoader) {
        try {
            //加载interceptor,这里的类加载器是组件的类加载器
            interceptor = InterceptorInstanceLoader.load(instanceMethodsAroundInterceptorClassName, classLoader);
        } catch (Throwable t) {
            throw new PluginException("Can't create InstanceMethodsAroundInterceptor.", t);
        }
    }

    /**
     * Intercept the target instance method.
     *
     * @param obj          target class instance.
     * @param allArguments all method arguments
     * @param method       method description.
     * @param zuper        the origin call ref.
     * @return the return value of target instance method.
     * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
     *                   bug, if anything triggers this condition ).
     */
    @RuntimeType
    public Object intercept(@This Object obj, @AllArguments Object[] allArguments, @SuperCall Callable<?> zuper,
        @Origin Method method) throws Throwable {
        EnhancedInstance targetObject = (EnhancedInstance) obj;

        MethodInterceptResult result = new MethodInterceptResult();
        try {
            interceptor.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
        } catch (Throwable t) {
            LOGGER.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
        }

        Object ret = null;
        try {
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                ret = zuper.call();
            }
        } catch (Throwable t) {
            try {
                interceptor.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
            } catch (Throwable t2) {
                LOGGER.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
            }
            throw t;
        } finally {
            try {
                ret = interceptor.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
            } catch (Throwable t) {
                LOGGER.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
            }
        }
        return ret;
    }
}
```

#### 9.6 interceptor加载

把当前类的类加载器当成parent，新生成一个AgentClassLoader 去加载拦截器。

```
    public static <T> T load(String className,
        ClassLoader targetClassLoader) throws IllegalAccessException, InstantiationException, ClassNotFoundException, AgentPackageNotFoundException {
        if (targetClassLoader == null) {
            targetClassLoader = InterceptorInstanceLoader.class.getClassLoader();
        }
        String instanceKey = className + "_OF_" + targetClassLoader.getClass()
                                                                   .getName() + "@" + Integer.toHexString(targetClassLoader
            .hashCode());
        Object inst = INSTANCE_CACHE.get(instanceKey);
        if (inst == null) {
            //加锁进行访问
            INSTANCE_LOAD_LOCK.lock();
            ClassLoader pluginLoader;
            try {
                //看看这个classloader是否存在
                pluginLoader = EXTEND_PLUGIN_CLASSLOADERS.get(targetClassLoader);
                if (pluginLoader == null) {
                    //不存在就 用targetClassLoader->的父子关系new一个AgentClassLoader
                    pluginLoader = new AgentClassLoader(targetClassLoader);
                    EXTEND_PLUGIN_CLASSLOADERS.put(targetClassLoader, pluginLoader);
                }
            } finally {
                INSTANCE_LOAD_LOCK.unlock();
            }
            //用新的AgentClassLoader加载这个interceptor
            inst = Class.forName(className, true, pluginLoader).newInstance();
            if (inst != null) {
                INSTANCE_CACHE.put(instanceKey, inst);
            }
        }

        return (T) inst;
    }
```

### 10.启动skywalking的服务

加载skywalking定义的服务 BootService

```
ServiceManager.INSTANCE.boot();
```

#### 10.1 boot方法

```
        public void boot() {
        //加载所有服务
        bootedServices = loadAllServices();

        //调用所有服务的prepare boot  onComplete方法
        prepare();
        startup();
        onComplete();
    }
```

#### 10.2 loadAllServices方法

```
    private Map<Class, BootService> loadAllServices() {
        Map<Class, BootService> bootedServices = new LinkedHashMap<>();
        List<BootService> allServices = new LinkedList<>();
        //spi机制加载服务
        load(allServices);
        for (final BootService bootService : allServices) {
            Class<? extends BootService> bootServiceClass = bootService.getClass();
            boolean isDefaultImplementor = bootServiceClass.isAnnotationPresent(DefaultImplementor.class);
            //如果是默认实现 就放在bootedServices里
            if (isDefaultImplementor) {
                if (!bootedServices.containsKey(bootServiceClass)) {
                    bootedServices.put(bootServiceClass, bootService);
                } else {
                    //ignore the default service
                }
              //如果是覆盖实现
            } else {
                OverrideImplementor overrideImplementor = bootServiceClass.getAnnotation(OverrideImplementor.class);
                if (overrideImplementor == null) {
                    if (!bootedServices.containsKey(bootServiceClass)) {
                        bootedServices.put(bootServiceClass, bootService);
                    } else {
                        throw new ServiceConflictException("Duplicate service define for :" + bootServiceClass);
                    }
                } else {
                    Class<? extends BootService> targetService = overrideImplementor.value();
                    //已经有这个服务了
                    if (bootedServices.containsKey(targetService)) {
                        boolean presentDefault = bootedServices.get(targetService)
                                                               .getClass()
                                                               .isAnnotationPresent(DefaultImplementor.class);
                       //如果这个服务存在默认实现就覆盖掉
                        if (presentDefault) {
                            bootedServices.put(targetService, bootService);
                        } else {
                            throw new ServiceConflictException(
                                "Service " + bootServiceClass + " overrides conflict, " + "exist more than one service want to override :" + targetService);
                        }
                    } else {
                        //没有这个服务直接put进去
                        bootedServices.put(targetService, bootService);
                    }
                }
            }

        }
        return bootedServices;
    }
```

可以看下BootService spi定义文件中所有的实现：JVMService和JVMMetricsSender前者负责采集jvm的数据，后者负责向aop发送数据。GRPCChannelManager负责管理grpc的连接。TraceSegmentServiceClient负责将Trace数据发送到aop。MeterService和MeterSender分别负责Metrics的注册和发送。SamplingService负责采样相关的。LogReportServiceClient负责日志的上报。、KafkaXXX是上报逻辑的kafka实现，因为skywalking的上报分为直接上报和消息队列间接上报。

![service spi](<../../.gitbook/assets/image (4).png>)

#### 10.3 prepare方法

因为都不难，我们只看下prepare方法。

```
    private void prepare() {
        //现根据服务的优先级进行排序
        bootedServices.values().stream().sorted(Comparator.comparingInt(BootService::priority)).forEach(service -> {
            try {
                service.prepare();
            } catch (Throwable e) {
                LOGGER.error(e, "ServiceManager try to pre-start [{}] fail.", service.getClass().getName());
            }
        });
    }
```



至此，skywalking-agent端的启动流程我们就分析完毕了，我觉得自己都看懂了，但是写出了可能就不好理解了，信息的传播毕竟是熵减的嘛，哈哈哈。
