# Boat的结构和实现

Boat 作为Mcinabox的后端，早已经在安卓端的启动器名声大噪。

早在Boardwalk无法在安卓6及以上版本出现运行时，横空出世。但是很多人只是用，并不知道Boat是怎么运行的，包括很多二改作者。。。

早在我写McgcLauncher（一个已经鸽掉的启动器，并没有使用Boat作为后端）

那时我拥有着部分逆向的知识（Boat SE，我逆向早期Boat H2O拿到实现的，当然还是因为Boat H2O写的太容易，毕竟那时候我技术水平不高）

不说题外话了，在我们了解Boat是怎么样运行的时候，我们先来了解Minecraft，也就是我们口中的MC是怎么样运行，绘制画面的。

Minecraft采用lwjgl进行游戏输入输出等处理。

## 什么是LWJGL？

LWJGL取自全称首字母Lightweight Java Game Library（轻量级Java游戏库）。 它是用来处理Minecraft的图形，声效与输入。目前LWJGL的最新版本为3.3.2。

目前Minecraft发布的版本1.5.2以及Alpha和Beta都是采用LWJGL 2.4.2版进行游戏处理，Classic用的是LWJGL 2.1.0，Indev 20100131之前的版本是LWJGL 2.2.0[需要验证]，Indev 20100131之后的版本是LWJGL 2.2.2（包括Indev 20100131），Infdev的LWJGL版本目前不明。Mojang现已采用较新版本的LWJGL来处理Minecraft1.6版。

不止如此lwjgl还是基成了多个底层图形库和系统api，但是这和我们要说的主题有什么关系？

当然有关系，因为lwjgl对OpenGL是有依赖的，Minecraft也是通过这些进行图像渲染等等，就如我前文所提到的。

### 疑惑？？
可是我们都知道安卓开发绘制方面都是依赖api接口的，而且LWJGL并没有对安卓端口提供支持，我们无法通过LWJGL进行直接的绘制，那怎么办呢？

这时我们就要来到我们这期主题的小高潮了，修改LWJGL！

是的，你没有听错，就是修改LWJGL，Boat修改了LWJGL使它可以在安卓上进行绘制操作！

现在让我们看两行代码，我只挑重点讲，如果想要了解其他的，可以去本篇文章末尾的GitHub链接去翻看。（只是帮助理解，讲的十分片面，很多实现都没细说，但是不影响我们的思路！）
``` c
ANativeWindow* boatGetNativeWindow(){	
        return mBoat.window;
}

JNIEXPORT void JNICALL Java_cosine_boat_BoatActivity_setBoatNativeWindow(JNIEnv* env, jclass clazz, jobject surface) {
	//mBoat是一个结构体
	mBoat.window = ANativeWindow_fromSurface(env, surface);
	__android_log_print(ANDROID_LOG_ERROR, "Boat", "setBoatNativeWindow : %p", mBoat.window);
	mBoat.display = 0;
}
```

``` java
public static native void setBoatNativeWindow(Surface surface);

public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
        
                // TODO: Implement this method
		System.out.println("SurfaceTexture is available!");
		BoatActivity.setBoatNativeWindow(new Surface(surface));
		// 后面会提，现在先不说
		new Thread(){
			@Override
			public void run(){
				
				LauncherConfig config = LauncherConfig.fromFile(getIntent().getExtras().getString("config"));
				LoadMe.exec(config);		
				Message msg = new Message();
				msg.what = -1;
				mHandler.sendMessage(msg);
			}
	   	}.start();
	  }
```

上面是通过jni技术，将SurfaceTexture通过native传给C （ANativeWindow就是SurfaceTexture的C语言版本）并且保存下来。

可以通过boatGetNativeWindow这个函数获取到指向这个ANativeWindow的指针。

**注意，接下来的内容十分重要**

在我们得到这个SurfaceTexture对象后，肯定要对它进行绘制操作。可是，应该在哪里进行绘制呢？

相信聪明的小伙伴已经想到了，对！就是LWJGL，上文提到的LWJGL就是如此！

file：[org_lwjgl_opengl_Display.c](https://github.com/CosineMath/lwjgl-boat/blob/boat-2.9.3/src%2Fnative%2Fboat%2Fopengl%2Forg_lwjgl_opengl_Display.c#L117-L151)
``` C
static ANativeWindow* createWindow(JNIEnv* env, EGLDisplay disp, jint window_mode, BoatPeerInfo *peer_info, int x, int y, int width, int height, jboolean resizable) {
        ANativeWindow* win;

        win = boatGetNativeWindow();
//        printfDebugJava(env, "Created window");

        return win;
}

JNIEXPORT jlong JNICALL Java_org_lwjgl_opengl_BoatDisplay_nCreateWindow(JNIEnv *env, jclass clazz, jlong display, jobject peer_info_handle, jobject mode, jint window_mode, jint x, jint y, jboolean resizable) {
        EGLDisplay disp = (EGLDisplay)(intptr_t)display;
        BoatPeerInfo *peer_info = (*env)->GetDirectBufferAddress(env, peer_info_handle);
        EGLConfig *fb_config = NULL;
        fb_config = getFBConfigFromPeerInfo(env, peer_info);
        if (fb_config == NULL)
                return 0;
        jclass cls_displayMode = (*env)->GetObjectClass(env, mode);
        jfieldID fid_width = (*env)->GetFieldID(env, cls_displayMode, "width", "I");
        jfieldID fid_height = (*env)->GetFieldID(env, cls_displayMode, "height", "I");
        int width = (*env)->GetIntField(env, mode, fid_width);
        int height = (*env)->GetIntField(env, mode, fid_height);
        ANativeWindow* win = createWindow(env, disp, window_mode, peer_info, x, y, width, height, resizable);
        if ((*env)->ExceptionOccurred(env)) {
                return 0;
        }
        egl_window = lwjgl_eglCreateWindowSurface(disp, *fb_config, win, NULL);
        free(fb_config);
        return (jlong)(intptr_t)win;
}

JNIEXPORT void JNICALL Java_org_lwjgl_opengl_BoatDisplay_nDestroyWindow(JNIEnv *env, jclass clazz, jlong display, jlong window_ptr) {
        EGLDisplay disp = (EGLDisplay)(intptr_t)display;
        ANativeWindow* window = (ANativeWindow*)(intptr_t)window_ptr;
        destroyWindow(env, disp, window);
}
```
这样修改后，我们绘制的对象是不是就变成了原来我们通过jni的native传入的那个SurfaceTexture！然后我们就可以接收到绘制的效果了！这样LWJGL创建的窗口我们就可以看见了是不是？

可是单单要看见可不行，我们还要对事件进行处理啊，比如我按下了W，要反应W啊。当然，这方面也有修改，但是考虑篇幅原因，这里不细说。

## 运行JVM
前面说了那么多，都是在说LWJGL和怎么绘制窗口，并没有提到是怎么样运行JVM的，很多人都有一个疑惑，你只说了怎么样绘制窗口，但是运行JVM才是最重要的的吧，MC和LWJGL不都是依赖JVM吗？没有JVM那怎么修改LWJGL都没用吧？为什么要把这部分放后面讲？对此我只能说（我忘了 划掉

怎么样去运行JVM，这是一个很好的话题，目前我们有两个方案
1. ProcessBuilder 
2. dlopen("libjli.so")

第一种方案有着非常强的弊端，我们无法拿到JVM在我们的应用程序中，也就是说我们前文描述的操作在这里直接失效了。。。

因此我们只能选择第二种方法，相信熟悉Open JDK源码的小伙伴对于dlopen("libjli.so")这个操作并不陌生

``` C
//copy java.c
/*
 * Entry point.
 */
JNIEXPORT int JNICALL
JLI_Launch(int argc, char ** argv,              /* main argc, argv */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* UNUSED dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* unused */
)

```

上述是jvm的启动入口，对于这些参数，我们一一解读
- int argc, char ** argv: 这两个参数通常表示 main 函数的参数。argc 是参数的数量，argv 是一个指向字符串数组的指针，该数组包含命令行参数。
int jargc, const char** jargv: 这些参数是专门用于 Java 应用程序的参数。jargc 是 Java 参数的数量，jargv 是一个指向字符串数组的指针，该数组包含 Java 应用程序的参数（如 -Xmx512m，-DsomeProperty=value 等）。
- int appclassc, const char** appclassv: 这些参数定义了 Java 应用程序的类路径。appclassc 是类路径条目的数量，appclassv 是一个指向字符串数组的指针，该数组包含类路径条目。
- const char* fullversion: 这个参数通常表示 Java 的完整版本号，例如 "1.8.0_251"。
- const char* dotversion: 这个参数通常表示 Java 的点分隔版本号（例如 "1.8"），但在你给出的函数签名中，它已被标记为未使用（UNUSED）。
- const char* pname: 这可能是程序的名称，但具体含义可能取决于 JVM 的实现。
- const char* lname: 这可能是启动器（launcher）的名称，也取决于 JVM 的实现。
- jboolean javaargs: 这是一个布尔值，可能用于指示是否使用 Java 风格的参数解析（即，参数前带有 - 或 --）。
- jboolean cpwildcard: 这也是一个布尔值，可能用于指示是否应该在类路径解析中使用通配符（如 * 和 ?）。
- jboolean javaw: 这是一个特定于 Windows 的布尔值，用于指示是否应使用 
- javaw.exe（而不是 java.exe）来启动 JVM。javaw.exe 与 java.exe 的主要区别在于它不会打开控制台窗口。
jint ergo: 在你给出的函数签名中，这个参数已被标记为未使用。

上面这些可能有些晦涩难懂，但是不妨碍我们讲下面的内容

``` C
int
(*JLI_Launch)(int argc, char ** argv,              /* main argc, argc */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard */
        jboolean javaw,                         /* windows-only javaw */
        jint     ergo_class                     /* ergnomics policy */
);

JNIEXPORT void JNICALL Java_cosine_boat_LoadMe_setupJLI(JNIEnv* env, jclass clazz){
	
	void* handle;
	//通过dlopen拿到JLI_Launch，一方面方便启动，另一方面为前面提到的LWJGL做辅助
	handle = dlopen("libjli.so", RTLD_LAZY);
	JLI_Launch = (int (*)(int, char **, int, const char**, int, const char**, const char*, const char*, const char*, const char*, jboolean, jboolean, jboolean, jint))dlsym(handle, "JLI_Launch");
	
}

// 使用jni给java层一个native方法，让java层传一个数组上来，也就是启动命令....
JNIEXPORT jint JNICALL Java_cosine_boat_LoadMe_jliLaunch(JNIEnv *env, jclass clazz, jobjectArray argsArray){
	int argc = (*env)->GetArrayLength(env, argsArray);
	char* argv[argc];
	for (int i = 0; i < argc; i++) {
		jstring str = (*env)->GetObjectArrayElement(env, argsArray, i);
		int len = (*env)->GetStringUTFLength(env, str);
		char* buf = malloc(len + 1);
		int characterLen = (*env)->GetStringLength(env, str);
		(*env)->GetStringUTFRegion(env, str, 0, characterLen, buf);
		buf[len] = 0;
		argv[i] = buf;
	}
	// 返回JLI_Launch的启动状态，一般是结束或异常时才返回
	return JLI_Launch(argc, argv,
                   sizeof(const_jargs) / sizeof(char *), const_jargs,
                   sizeof(const_appclasspath) / sizeof(char *), const_appclasspath,
                   FULL_VERSION,
                   DOT_VERSION,
                   (const_progname != NULL) ? const_progname : *argv,
                   (const_launcher != NULL) ? const_launcher : *argv,
                   (const_jargs != NULL) ? JNI_TRUE : JNI_FALSE,
                   const_cpwildcard, const_javaw, const_ergo_class);
	
}
```
``` java
// 以上一些配置例如dlopen其他动态库，实现环境变量等等
public static native int jliLaunch(String[] strArr);
```

还是通过native，不过这次是我们通过dlopen拿到JLI_Launch，对Java进行实现，对于Java层面我们只需要传入参数，就可以启动JVM了。

也就是我们之前见到的
``` java
		new Thread(){
			@Override
			public void run(){
				
				LauncherConfig config = LauncherConfig.fromFile(getIntent().getExtras().getString("config"));
				LoadMe.exec(config);		
				Message msg = new Message();
				msg.what = -1;
				mHandler.sendMessage(msg);
			}
	   	}.start();
	   	
```
这个实现中LoadMe.exec(config);的重要部分，而对于LauncherConfig你只需要知道是配置信息就行了

****注意我这里跳过了一些环境变量等等的设置情况，实际情况比这要复杂，还是那句话，本文只是帮你了解这一部分，并不是完全的读源码！****

而且！现在JVM还在内存当中！是不是很棒！这种方法简直巧妙极了，我们现在有了JVM又有了LWJGL，Boat也就可以正常的绘制处理Minecraft的图像了

然后就是一些杂七杂八的东西了，如果想要更深入的了解这方面的知识，可以打开下面的链接：


 仓库名 | 链接
---|---
Boat | https://github.com/AOF-Dev/Boat
lwjgl2 | https://github.com/AOF-Dev/lwjgl-boat
jdk8u-android | https://github.com/Eurya2233369/jdk8u-android
