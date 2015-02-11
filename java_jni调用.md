#java jni调用

jni（java native interface）用于java与其他语言交互。

原因：各语言之间数据类型不一致，jni做了一层转换。

坑点：java的char是两个字节的实现。

##步骤

1. 编写带有native声明的方法的java类
2. 使用javac命令编译编写的java类
3. 用javah ...来生成后缀名为.h的头文件
4. 使用其他语言（c， c++）实现本地方法
5. 将本地方法编写的文件生成动态链接库

## load与loadLibrary方法

System.load导入绝对路径。

System.loadLibrary从java.library.path导入，但是有匹配规则，得去掉libxxx.so，只留下xxx。

###问题：如何在ant中设置java.library.path



    <target name="run" depends="deploy">
       <java dir="${jlan}" classname="org.alfresco.jlan.app.JLANServer" fork="true">
         <arg value="${jlan}/jlanConfig.xml"/>
         <sysproperty key="java.library.path" path="${jlan}/jni"/>
         <classpath>
            <filelist dir="${jlan}">
                <file name="jars/alfresco-jlan.jar" />
                <file name="libs/cryptix-jce-provider.jar" />
                <file name="service/wrapper.jar" />
                <file name="libs/bullhorn-virtualfs-0.1.jar" />
                <file name="libs/log4j-1.2.14.jar" />
            </filelist>
         </classpath>
      </java>
    </target>
    
原因是ant不允许reset变量，java.library.path是预定义的变量，因此除非重开一个新进程，也就是注意把fork设为true，才能设置java.libray.path变量。


## 把so文件打包进jar

从jar得到so，再把so复制到临时文件导入进来。

    public class Foo {
    private static final String LIB_BIN = "/lib-bin/";
    private final static Log logger = LogFactory.getLog(ACWrapper.class);
    private final static String ACWRAPPER = "acwrapper";
    private final static String AAMAPI = "aamapi51";
    private final static String LIBEAU = "libeay32";

    static {
        logger.info("Loading DLL");
        try {
            System.loadLibrary(ACWRAPPER);
            logger.info("DLL is loaded from memory");
        } catch (UnsatisfiedLinkError e) {
            loadFromJar();
        }
    }

    /**
     * When packaged into JAR extracts DLLs, places these into
     */
    private static void loadFromJar() {
        // we need to put both DLLs to temp dir
        String path = "AC_" + new Date().getTime();
        loadLib(path, ACWRAPPER);
        loadLib(path, AAMAPI);
        loadLib(path, LIBEAU);
    }

    /**
     * Puts library to temp dir and loads to memory
     */
    private static void loadLib(String path, String name) {
        name = name + ".dll";
        try {
            // have to use a stream
            InputStream in = ACWrapper.class.getResourceAsStream(LIB_BIN + name);
            // always write to different location
            File fileOut = new File(System.getProperty("java.io.tmpdir") + "/" + path + LIB_BIN + name);
            logger.info("Writing dll to: " + fileOut.getAbsolutePath());
            OutputStream out = FileUtils.openOutputStream(fileOut);
            IOUtils.copy(in, out);
            in.close();
            out.close();
            System.load(fileOut.toString());
        } catch (Exception e) {
            throw new ACCoreException("Failed to load required DLL", e);
        }
    }
        // blah-blah - more stuff
    }

##参考文献

http://chase-seibert.github.io/blog/2009/04/01/ant-javalibrarypath

http://stackoverflow.com/questions/1611357/how-to-make-a-jar-file-that-includes-dll-files
