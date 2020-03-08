Android开发opengl程序，通常是用GLSurfaceView显示OpenGL对象，其实GLSurfaceView在setRender()时，内部已经封装了创建EGL窗口的过程，使得开发者无需关心OpenGL的初始化流程就能写好程序。

​	之前写过一个功能，利用生成的Bitmap作为OpenGL的纹理，然后使用MediaCodec编码成视频，整个过程没有用户可以看到的“View”。这种脱离GLSurfaceView的情况，创建EGL窗口的过程就需要自己来写了。此外，了解EGL窗口的创建过程，也能更好的理解OpenGL的运作流程。



下面从《OpenGL ES 3.0编程指南》中摘的代码，说明了从EGL初始化开始到EGLContext绑定到EGLSurface的全过程。稍作修改，拿EGL14举例。

```
/**
*	@param nativeWindow Android开发中，此window可以使用Surface
*/
EGLBoolean initializeWindow(EGLNativeWindow nativeWindow) {

		// step1，创建显示连接
		EGLDisplay display = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);
		if (display == EGL14.EGL_NO_DISPLAY) {
			return EGL14.EGL_FALSE;
		}
		
		// step2，初始化egl
		int[] version = new int[2];
		if (!EGL14.eglInitialize(display, version, 0, version, 1)) {
			return EGL14.EGL_FALSE;
		}
		
		// step3，选择渲染表明的配置
		int[] configAttribs = {
		  	EGL14.EGL_RED_SIZE, 8,
		  	EGL14.EGL_GREEN_SIZE, 8,
		  	EGL14.EGL_BLUE_SIZE, 8,
		  	EGL14.EGL_ALPHA_SIZE, 8,
		  	EGL14.EGL_RENDERABLE_TYPE, EGL14.EGL_OPENGL_ES2_BIT,
		  	EGL_RECORDABLE_ANDROID, 1,
		  	EGL14.EGL_NONE
		}
		EGLConfig[] configs = new EGLConfig[1];
		int[] numConfigs = new int[1];
		if (!EGL14.eglChooseConfig(display, configAttribs, 0, configs, 0, configs.length,
                numConfigs, 0)) {
				return EGL14.EGL_FALSE;
		}
		
		// step4，创建渲染区域，egl窗口
		int[] surfaceAttribs = { EGL14.EGL_NONE };
		EGLSuface window = EGL14.eglCreateWindowSurface(display, configs[0], nativeWindow, surfaceAttribs);
		if (window == EGL14.NO_SURFACE) {
			return EGL14.EGL_FALSE;
		}

		// step5，创建渲染上下文
    int[] contextAttribs = {
      EGL14.EGL_CONTEXT_CLIENT_VERSION, 2,
      EGL14.EGL_NONE
    }
		EGLContext context = EGL14.eglCreateContext(display, configs[0], EGL14.EGL_NO_CONTEXT, contextAttribs);
		if (context == EGL14.EGL_NO_CONTEXT) {
			return EGL14.EGL_FALSE;
		}
		
		// step6，指定某个EGLContext作为当前上下文
		if (EGL14.eglMakeCurrent(display, window, context)) {
			return EGL14.EGL_FALSE;
		}
		return EGL14.EGL_TRUE;
}
```

解释下每个步骤的含义：

step1，创建显示连接。每个系统（Windows/Mac OS/Android）都有不同的语义，EGL提供了基本的不透明类型——EGLDisplay，用于封装系统相关性，建立和系统原生窗口的连接。任何使用EGL的应用程序必须执行的第一个操作就是创建和初始化与本地EGL显示的连接。

step2，第一步创建完显示连接后，就初始化EGL。

step3，确定渲染表明的配置，比如颜色分量位数等。eglChooseConfig()是让EGL自行选择最匹配的配置。

step4，创建渲染区域，egl窗口。窗口即为显示或者opengl数据要渲染到的地方。其入参nativeWindow可以是Android平台的Surface。像文章刚开始提到之前做的一个功能，将OpenGL的纹理数据交给MediaCodec进行编码，这里的Surface，就是MediaCodec.createInputSurface()。

step5，创建上下文。Context，渲染上下文，是OpenGL的内部数据结构，包含操作所需的所有状态信息。比如画三角形时顶点、片段着色器及顶点数据数组的引用。网上其他解释，Context可以理解为一个容器，主要存放两个东西，（1）内部状态信息，包括view port, depth range, clear color, texture, FBO, VBO；（2）调用缓存，保存了在当前context下发起的GL指令（OpenGL调用是异步的）。简单来说，Context用来存储渲染相关的输入数据。

step6，指定某个EGLContext作为当前上下文。一个应用程序可能创建多个EGLContext用于不同的用途，所以我们需要关联特定的EGLContext和渲染表明——这一过程常常被称作“指定当前上下文”。



