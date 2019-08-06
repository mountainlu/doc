
### JDK 和 JRE
	
	1. 定义:
	JRE(Java Runtime Enviroment)是Java的运行环境。面向Java程序的使用者，而不是开发者。
	如果你仅下载并安装了JRE，那么你的系统只能运行Java程序。
	JRE是运行Java程序所必须环境的集合，包含JVM标准实现及 Java核心类库。它包括Java虚拟机、
	Java平台核心类和支持文件。它不包含开发工具(编译器、调试器等)。
	
	JDK(Java Development Kit)又称J2SDK(Java2 Software Development Kit)，是Java开发工具包，
	它提供了Java的开发环境(提供了编译器javac等工具，用于将java文件编译为class文件)
	和运行环境(提 供了JVM和Runtime辅助包，用于解析class文件使其得到运行)。
	如果你下载并安装了JDK，那么你不仅可以开发Java程序，也同时拥有了运 行Java程序的平台。
	JDK是整个Java的核心，包括了Java运行环境(JRE)，一堆Java工具tools.jar和Java标准类库 (rt.jar)。
	
	2. 区别:
	JRE主要包含：java类库的class文件(都在lib目录下打包成了jar)和虚拟机(jvm.dll)；
	JDK主要包含：java类库的 class文件(都在lib目录下打包成了jar)并自带一个JRE。
	那么为什么JDK要自带一个JRE呢？而且jdk/jre/bin下的client 和server
	两个文件夹下都包含jvm.dll(说明JDK自带的JRE有两个虚拟机)。
	
	记得在环境变量path中设置jdk/bin路径麽？老师会告诉大家不设置的话javac和java是用不了的。
	确实jdk/bin目录下包含了所有的命令。可是有没有人想过我们用的java命令并不是jdk/bin目录下的而是jre/bin目录下的呢？
	不信可以做一个实验，大家可以把jdk /bin目录下的java.exe剪切到别的地方再运行java程序，发现了什么？一切OK！
	(JRE中没有javac命令，原因很简单，它不是开发环境)那么有人会问了？我明明没有设置jre/bin目录到环境变量中啊？
	试想一下如果java为了提供给大多数人使用，他们是不需要jdk做开发的，只需 要jre能让java程序跑起来就可以了，
	那么每个客户还需要手动去设置环境变量多麻烦啊？所以安装jre的时候安装程序自动帮你把jre的
	 java.exe添加到了系统变量中，验证的方法很简单，去Windows/system32下面去看看吧，发现了什么？有一个java.exe。
	
	3. 难点:
	如果安装了JDK，你的电脑就有两套JRE(JRE本身和JDK中的JRE)，前面这套比后面那套少了Server端的Java虚拟机。
	
	(1)为什么Sun要让JDK安装两套相同的JRE？这是因为JDK里面有很多用Java所编写的开发工具(如javac.exe、jar.exe 等)，而且都放置在/lib/tools.jar里。
	    如果我们将tools.jar改名为tools1.jar，然后运行javac.exe，显示如下结果：
	    Exception in thread "main" java.lang.NoClassDefFoundError: com/sun/tools/javac/Main。
	    这个意思是说，你输入javac.exe与输入java -cp c:/jdk/lib/tools.jar com.sun.tools.javac.Main 是一样的，会得到相同的结果。
	    从这里我们可以证明javac.exe只是一个包装器（Wrapper），而制作的目的是为了让开发者免于输入太长的指命。
	    而且可以发现/lib目录下的程序都很小，不大于29K，从这里我们可以得出一个结论。
	    就是JDK里的工具几乎是用Java所编写，所以也是Java应用 程序，因此要使用JDK所附的工具来开发Java程序，也必须要自行附一套JRE才行，
	    所以位于JDK目录下的那套JRE就是用来运行一般Java程序 的。
	
	(2)如果一台电脑安装两套以上的JRE，谁来决定呢？这个重大任务就落在java.exe身上。java.exe的工作就是找到合适的JRE来运 行Java程序。
	    java.exe依照以下的顺序来查找JRE：  1)自己的目录下有没有JRE；
	                                    2)父目录有没有JRE；
	                                    3)查询注册表： [HKEY_LOCAL_MACHINE/SOFTWARE/JavaSoft/Java Runtime Environment]。
	                                    所以java.exe的运行结果与你的电脑里面哪个JRE被执行有很大的关系。
	
	(3)JDK-->JRE-->Bin目录下有两个文件夹：server与client，这是真正的jvm.dll所在。 jvm.dll无法单独工作，当jvm.dll启动后，
	     会使用explicit的方法(就是使用Win32 API之中的LoadLibrary()与GetProcAddress()来载入辅助用的动态链接库)，
	     而这些辅助用的动态链接库(.dll)都必须位 于jvm.dll所在目录的父目录之中。
	     因此想使用哪个JVM，只需要设置PATH，指向JRE所在目录下的jvm.dll。
	