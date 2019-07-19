vim打开spark-shell脚本可以看到下面这段脚本

spark-shell代码片段：
![spark-shell代码片段](https://note.youdao.com/yws/public/resource/f908d084a11f2c97ddb1e777c29e3968/xmlnote/3F6BC2499F844C61BF40C7D1188B5AE9/1286)

可以看到在spark-shell脚本中调用了spark-submit脚本，打开spark-submit脚本发现包含如下脚本：
![image](https://note.youdao.com/yws/public/resource/f908d084a11f2c97ddb1e777c29e3968/xmlnote/E588D58808DA4A8099C0F37A57987932/1303)

可以看到在spark-submit脚本中，首先检查是否设置了SPARK_HOME,然后调用了spark-class，增加了参数SparkSubmit。

打开spark-class脚本


![image](https://note.youdao.com/yws/public/resource/f908d084a11f2c97ddb1e777c29e3968/xmlnote/B91101B49CEB44069A82A4DCD28DBD25/1315)
首先调用了load-spark-env.sh脚本去加载spark-env.sh，设置scala版本，然后寻找java，并赋值给变量RUNNER。

在spark-class中最重要的是下面这段脚本，首先循环读取ARG参数，加入到CMD中。然后执行了
> "$RUNNER" -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@ 

这个是真正执行的第一个spark的类。
![image](https://note.youdao.com/yws/public/resource/f908d084a11f2c97ddb1e777c29e3968/xmlnote/7CCEDAD0047F4DB6A326EC1C85032DE7/1337)

> **org.apache.spark.launcher.Main类**：

```
 public static void main(String[] argsArray) throws Exception {
    checkArgument(argsArray.length > 0, "Not enough arguments: missing class name.");

    List<String> args = new ArrayList<>(Arrays.asList(argsArray));
    String className = args.remove(0);

    boolean printLaunchCommand = !isEmpty(System.getenv("SPARK_PRINT_LAUNCH_COMMAND"));
    AbstractCommandBuilder builder;
    //根据传进来的参数创建命令SparkSubmitCommandBuilder或者SparkClassCommandBuilder
    if (className.equals("org.apache.spark.deploy.SparkSubmit")) {
      try {
      
        builder = new SparkSubmitCommandBuilder(args);
      } catch (IllegalArgumentException e) {
        printLaunchCommand = false;
        System.err.println("Error: " + e.getMessage());
        System.err.println();

        MainClassOptionParser parser = new MainClassOptionParser();
        try {
          parser.parse(args);
        } catch (Exception ignored) {
          // Ignore parsing exceptions.
        }

        List<String> help = new ArrayList<>();
        if (parser.className != null) {
          help.add(parser.CLASS);
          help.add(parser.className);
        }
        help.add(parser.USAGE_ERROR);
        builder = new SparkSubmitCommandBuilder(help);
      }
    } else {
      builder = new SparkClassCommandBuilder(className, args);
    }

    Map<String, String> env = new HashMap<>();
    //buildCommand
    List<String> cmd = builder.buildCommand(env);
    if (printLaunchCommand) {
      System.err.println("Spark Command: " + join(" ", cmd));
      System.err.println("========================================");
    }
    //返回有效的参数，存到CMD中，最后spark-class脚本中执行exec "${CMD[@]}"
    if (isWindows()) {
      System.out.println(prepareWindowsCommand(cmd, env));
    } else {
      // In bash, use NULL as the arg separator since it cannot be used in an argument.
      List<String> bashCmd = prepareBashCommand(cmd, env);
      for (String c : bashCmd) {
        System.out.print(c);
        System.out.print('\0');
      }
    }
  }
```


spark-class脚本里面的执行逻辑较多，整体上是：检查spark_home是否配置 -> 执行load_spark-env.sh去加载spark-env.sh文件并设置scala环境 -> 检查java的执行路径变量 -> 寻找spark jars -> 执行类文件org.apache.spark.launcher.Main，返回解析后的参数给CMD -> 执行org.apache.spark.deploy.SparkSubmit这个类s

在org.apache.spark.launcher.Main中会创建SparkSubmitCommandBuilder或者SparkClassCommandBuilder，这里主要讲解SparkSubmitCommandBuilder类
> **SparkSubmitCommandBuilder类**：


```
//org.apache.spark.launcher.Main中new SparkSubmitCommandBuilder(args)
SparkSubmitCommandBuilder(List<String> args) {
    this.allowsMixedArguments = false;
    this.sparkArgs = new ArrayList<>();
    boolean isExample = false;
    List<String> submitArgs = args;

    if (args.size() > 0) {
    //选择类型解析参数
      switch (args.get(0)) {
        case PYSPARK_SHELL:
          this.allowsMixedArguments = true;
          appResource = PYSPARK_SHELL;
          submitArgs = args.subList(1, args.size());
          break;
    //执行这一步，进行参数截取，去掉第一个类型选择参数
        case SPARKR_SHELL:
          this.allowsMixedArguments = true;
          appResource = SPARKR_SHELL;
          submitArgs = args.subList(1, args.size());
          break;

        case RUN_EXAMPLE:
          isExample = true;
          submitArgs = args.subList(1, args.size());
      }

      this.isExample = isExample;
    //创建解析器，解析参数
      OptionParser parser = new OptionParser();
      parser.parse(submitArgs);
      this.isAppResourceReq = parser.isAppResourceReq;
    }  else {
      this.isExample = isExample;
      this.isAppResourceReq = false;
    }
  }
```
> **SparkSubmitOptionParser类的parse方法**：


```
//根据opts进行参数解析与验证
 final String[][] opts = {
    { ARCHIVES },
    { CLASS },
    { CONF, "-c" },
    { DEPLOY_MODE },
    { DRIVER_CLASS_PATH },
    { DRIVER_CORES },
    { DRIVER_JAVA_OPTIONS },
    { DRIVER_LIBRARY_PATH },
    { DRIVER_MEMORY },
    { EXECUTOR_CORES },
    { EXECUTOR_MEMORY },
    { FILES },
    { JARS },
    { KEYTAB },
    { KILL_SUBMISSION },
    { MASTER },
    { NAME },
    { NUM_EXECUTORS },
    { PACKAGES },
    { PACKAGES_EXCLUDE },
    { PRINCIPAL },
    { PROPERTIES_FILE },
    { PROXY_USER },
    { PY_FILES },
    { QUEUE },
    { REPOSITORIES },
    { STATUS },
    { TOTAL_EXECUTOR_CORES },
  };
protected final void parse(List<String> args) {
    Pattern eqSeparatedOpt = Pattern.compile("(--[^=]+)=(.+)");

    int idx = 0;
    for (idx = 0; idx < args.size(); idx++) {
      String arg = args.get(idx);
      String value = null;

      Matcher m = eqSeparatedOpt.matcher(arg);
      if (m.matches()) {
        arg = m.group(1);
        value = m.group(2);
      }

      // Look for options with a value.
      String name = findCliOption(arg, opts);
      if (name != null) {
        if (value == null) {
          if (idx == args.size() - 1) {
            throw new IllegalArgumentException(
                String.format("Missing argument for option '%s'.", arg));
          }
          idx++;
          value = args.get(idx);
        }
        if (!handle(name, value)) {
          break;
        }
        continue;
      }

      // Look for a switch.
      name = findCliOption(arg, switches);
      if (name != null) {
        if (!handle(name, null)) {
          break;
        }
        continue;
      }

      if (!handleUnknown(arg)) {
        break;
      }
    }

    if (idx < args.size()) {
      idx++;
    }
    handleExtraArgs(args.subList(idx, args.size()));
  }
```

org.apache.spark.launcher.Main执行成功后会接着org.apache.spark.deploy.SparkSubmit的执行，SparkSubmit部分源码如下：

```
//SparkSubmit Main方法
def main(args: Array[String]): Unit = {
    val appArgs = new SparkSubmitArguments(args)
    if (appArgs.verbose) {
      // scalastyle:off println
      printStream.println(appArgs)
      // scalastyle:on println
    }
    appArgs.action match {
      执行这一句，转到submit方法
      case SparkSubmitAction.SUBMIT => submit(appArgs)
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
    }
  }
```
进入submit方法：
```
/**
   * Submit the application using the provided parameters.
   *
   * This runs in two steps. First, we prepare the launch environment by setting up
   * the appropriate classpath, system properties, and application arguments for
   * running the child main class based on the cluster manager and the deploy mode.
   * Second, we use this launch environment to invoke the main method of the child
   * main class.
   */
  @tailrec
  private def submit(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
  //运行参数
    val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)

    def doRunMain(): Unit = {
      if (args.proxyUser != null) {
        val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,
          UserGroupInformation.getCurrentUser())
        try {
          proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
            override def run(): Unit = {
            //执行runmain方法
              runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)
            }
          })
        } catch {
          case e: Exception =>
            // Hadoop's AuthorizationException suppresses the exception's stack trace, which
            // makes the message printed to the output by the JVM not very helpful. Instead,
            // detect exceptions with empty stack traces here, and treat them differently.
            if (e.getStackTrace().length == 0) {
              // scalastyle:off println
              printStream.println(s"ERROR: ${e.getClass().getName()}: ${e.getMessage()}")
              // scalastyle:on println
              exitFn(1)
            } else {
              throw e
            }
        }
      } else {
      //执行runmain方法
        runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)
      }
    }
```
进入runmain方法：

```
/**
   * Run the main method of the child class using the provided launch environment.
   *
   * Note that this main class will not be the one provided by the user if we're
   * running cluster deploy mode or python applications.
   */
  private def runMain(
      childArgs: Seq[String],
      childClasspath: Seq[String],
      sparkConf: SparkConf,
      childMainClass: String,
      verbose: Boolean): Unit = {
    // scalastyle:off println
    if (verbose) {
      printStream.println(s"Main class:\n$childMainClass")
      printStream.println(s"Arguments:\n${childArgs.mkString("\n")}")
      // sysProps may contain sensitive information, so redact before printing
      printStream.println(s"Spark config:\n${Utils.redact(sparkConf.getAll.toMap).mkString("\n")}")
      printStream.println(s"Classpath elements:\n${childClasspath.mkString("\n")}")
      printStream.println("\n")
    }
    // scalastyle:on println

    val loader =
      if (sparkConf.get(DRIVER_USER_CLASS_PATH_FIRST)) {
        new ChildFirstURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      } else {
        new MutableURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      }
    Thread.currentThread.setContextClassLoader(loader)

    for (jar <- childClasspath) {
      addJarToClasspath(jar, loader)
    }

    var mainClass: Class[_] = null

    try {
    //通过反射加载childMainClass
      mainClass = Utils.classForName(childMainClass)
    } catch {
      case e: ClassNotFoundException =>
        e.printStackTrace(printStream)
        if (childMainClass.contains("thriftserver")) {
          // scalastyle:off println
          printStream.println(s"Failed to load main class $childMainClass.")
          printStream.println("You need to build Spark with -Phive and -Phive-thriftserver.")
          // scalastyle:on println
        }
        System.exit(CLASS_NOT_FOUND_EXIT_STATUS)
      case e: NoClassDefFoundError =>
        e.printStackTrace(printStream)
        if (e.getMessage.contains("org/apache/hadoop/hive")) {
          // scalastyle:off println
          printStream.println(s"Failed to load hive class.")
          printStream.println("You need to build Spark with -Phive and -Phive-thriftserver.")
          // scalastyle:on println
        }
        System.exit(CLASS_NOT_FOUND_EXIT_STATUS)
    }
    //构建 SparkApplication
    val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
      mainClass.newInstance().asInstanceOf[SparkApplication]
    } else {
      // SPARK-4170
      if (classOf[scala.App].isAssignableFrom(mainClass)) {
        printWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
      }
      new JavaMainApplication(mainClass)
    }

    @tailrec
    def findCause(t: Throwable): Throwable = t match {
      case e: UndeclaredThrowableException =>
        if (e.getCause() != null) findCause(e.getCause()) else e
      case e: InvocationTargetException =>
        if (e.getCause() != null) findCause(e.getCause()) else e
      case e: Throwable =>
        e
    }

    try {
    //启动app
      app.start(childArgs.toArray, sparkConf)
    } catch {
      case t: Throwable =>
        findCause(t) match {
          case SparkUserAppException(exitCode) =>
            System.exit(exitCode)

          case t: Throwable =>
            throw t
        }
    }
  }
  
  
  /**
 * Implementation of SparkApplication that wraps a standard Java class with a "main" method.
 *
 * Configuration is propagated to the application via system properties, so running multiple
 * of these in the same JVM may lead to undefined behavior due to configuration leaks.
 */
private[deploy] class JavaMainApplication(klass: Class[_]) extends SparkApplication {
// start方法：
  override def start(args: Array[String], conf: SparkConf): Unit = {
  //通过反射机制加载main方法
    val mainMethod = klass.getMethod("main", new Array[String](0).getClass)
    if (!Modifier.isStatic(mainMethod.getModifiers)) {
      throw new IllegalStateException("The main method in the given main class must be static")
    }

    val sysProps = conf.getAll.toMap
    sysProps.foreach { case (k, v) =>
      sys.props(k) = v
    }
    //执行main方法  至此spark程序的提交流程完毕
    mainMethod.invoke(null, args)
  }
```

















