---
title: OpenGL着色器和程序
categories: 技术
---

​	在使用着色器绘制图形时都要执行的着色器和程序的创建、链接相关流程。本文主要来源于《OpenGL ES 3.0编程指南》第四章。
<!--more-->

​	需要创建两个基本对象才能使用着色器进行渲染：着色器对象、程序对象。理解着色器对象和程序对象的最佳方式是将它们比作C语言的编译器和链接程序。C编译器为一段源代码生成目标代码（例如.obj或者.o文件）。创建目标文件之后，C链接程序将对象文件链接为最后的程序。

​	OpenGL ES在着色器方面使用类似范式。着色器对象是包含单个着色器的对象。源代码提供给着色器对象，然后着色器对象被编译为一个目标形式（类似于.obj文件）。编译之后，着色器对象可以连接到一个程序对象。程序对象可以连接多个着色器对象。在OpenGL ES中，每个程序对象必须连接一个顶点着色器和一个片段着色器（不多也不少）。最后，程序对象被链接为用于渲染的最后“可执行程序”。（这里注意连接和链接的区分）

### 着色器对象链接步骤

​	获取链接后的着色器对象的过程，一般包括下面6个步骤：

1. 创建一个顶点着色器对象和片段着色器对象
2. 将源代码连接到每个着色器对象
3. 编译着色器对象
4. 创建一个程序对象
5. 将编译后的着色器对象连接到程序对象
6. 链接程序对象

上述步骤的示例代码如下：

```
// 顶点着色器源代码
private final String vertexShaderCode =
            "attribute vec4 vPosition;" +
                    "void main() {" +
                    "  gl_Position = vPosition;" +
                    "}";

// 片段着色器源代码
private final String fragmentShaderCode =
            "precision mediump float;" +
                    "uniform vec4 vColor;" +
                    "void main() {" +
                    "  gl_FragColor = vColor;" +
                    "}";

// step1, 创建一个顶点着色器对象。glCreateShader()返回着色器对象句柄
int vertexShader = GLES20.glCreateShader(GLES20.GL_VERTEX_SHADER);

// step2, 将源代码连接到着色器对象
GLES20.glShaderSource(vertexShader, vertexShaderCode);

// step3, 编译着色器对象
GLES20.glCompileShader(vertexShader);

// 片段着色器创建、编译过程和顶点着色器一样，step1~step3
int fragmentShader = GLES20.glCreateShader(GLES20.GL_FRAGMENT_SHADER);
GLES20.glShaderSource(fragmentShader, fragmentShaderCode);
GLES20.glCompileShader(fragmentShader);

// step4, 创建一个程序对象。glCreateProgram()返回程序对象的句柄
int mProgram = GLES20.glCreateProgram();

//step5, 将顶点着色器加入到程序
GLES20.glAttachShader(mProgram, vertexShader);

//step5, 将片元着色器加入到程序中
GLES20.glAttachShader(mProgram, fragmentShader);

//step6, 链接程序对象
GLES20.glLinkProgram(mProgram);

// 异常检测：检查链接是否成功
int[] status = new int[1];
GLES20.glGetProgramiv(mProgram, GLES20.GL_LINK_STATUS, status, 0);
if (status[0] == GLES20.GL_FALSE) {
	Log.e(TAG, "link shader error:" + GLES20.glGetProgramInfoLog(mProgram));
}

// 一般上述步骤完成后，还需要下面步骤将创建的程序对象设置为活动程序
GLES20.glUseProgram(mProgram);
```

### 示例代码

以创建三角形为例，走下上述创建着色器对象并链接程序的过程，每一步都有基本的注释。对细节不是特别清楚的可以参考这个网站，入门OpenGL非常好的一个[教程网站](https://learnopengl-cn.github.io/01 Getting started/04 Hello Triangle/))。

<img src="/img/opengl_triangle.png" style="zoom: 50%;" />

```
class TriangleRenderer : BaseRenderer() {

    private val vertextShaderCode =
            "attribute vec4 vPosition;" +
                    "void main() {" +
                    "   gl_Position = vPosition;" +
                    "}"

    private val fragmentShaderCode =
            "precision mediump float;" +
                    "uniform vec4 vColor;" +
                    "void main() {" +
                    "   gl_FragColor = vColor;" +
                    "}"

    private val vertexVecSize = 3 // 每个顶点矢量的坐标分量，因为是三维，所以是3
    private val triangleCoordinate = floatArrayOf(
            0f, 0.5f, 0f,
            -0.5f, -0.5f, 0f,
            0.5f, -0.5f, 0f
    )
    private var vertexBuffer: FloatBuffer? = null
    private var triangleColor = floatArrayOf(1f, 1f, 1f, 0f)
    private val sizeOfFloat = 4	// Float是4个字节
    private var mProgram = -1

    override fun onInit() {
        // 准备顶点数据
        val bb = ByteBuffer.allocateDirect(triangleCoordinate.size * sizeOfFloat)
        bb.order(ByteOrder.nativeOrder())
        vertexBuffer = bb.asFloatBuffer()
        vertexBuffer?.put(triangleCoordinate)
        vertexBuffer?.position(0)

        // 编译vertexShader和fragmentShader
        val vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, vertextShaderCode)
        val fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, fragmentShaderCode)

        // 将着色器加入程序
        mProgram = GLES20.glCreateProgram()
        GLES20.glAttachShader(mProgram, vertexShader)
        GLES20.glAttachShader(mProgram, fragmentShader)

        // 链接程序
        GLES20.glLinkProgram(mProgram)
        checkLinkProgramStatus(mProgram)
    }

    override fun onDrawFrame(gl: GL10) {
        super.onDrawFrame(gl)
        // 使用程序
        GLES20.glUseProgram(mProgram)

        // 找到顶点坐标（attribute变量）的句柄，并对其赋值
        val positionLocation = GLES20.glGetAttribLocation(mProgram, "vPosition")
        
        // 启用属性句柄
        GLES20.glEnableVertexAttribArray(positionLocation)
        
        // @param indx 属性句柄。即上面用glGetAttribLocation找到的属性结果positionLocation
        // @param size 顶点属性大小，和矢量维度一致。顶点坐标是一个vec3，它由3个值组成，所以大小是3
        // @param type 顶点属性数据类型
        // @param normalized 是否希望数据被标准化。如果是true，所有数据会被映射到-1~1之间
        // @param stride 步长。也是每个顶点属性的字节数，OpenGL通过该参数才知道一步跨多大能找到下一个顶点属性。一般是size*数据类型byte大小
        // @param ptr 属性数据
        GLES20.glVertexAttribPointer(positionLocation, 3, GLES20.GL_FLOAT, false, 3 * sizeOfFloat, vertexBuffer)

        // 找到片段颜色（uniform变量）的句柄，并对其赋值
        val colorLocation = GLES20.glGetUniformLocation(mProgram, "vColor")
        GLES20.glUniform4fv(colorLocation, 1, triangleColor, 0)

        // 绘制三角形
        GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, triangleCoordinate.size/vertexVecSize)
        GLES20.glDisableVertexAttribArray(positionLocation)
    }
}
```

Renderer基类包含一部分检查工作，比如着色器是否创建成功、程序是否链接成功。

```
abstract class BaseRenderer: GLSurfaceView.Renderer {

    private val TAG = this.javaClass.simpleName

    fun loadShader(type: Int, shaderCode: String): Int {
        //根据type创建顶点着色器或者片元着色器
        val shader = GLES20.glCreateShader(type)
        //将着色器源代码连接到着色器对象上
        GLES20.glShaderSource(shader, shaderCode)
        //编译着色器对象
        GLES20.glCompileShader(shader)
        checkShaderCompileStatus(shader, type)
        return shader
    }

    abstract fun onInit()

    override fun onSurfaceCreated(gl: GL10, config: EGLConfig) {
        gl.glClearColor(0.3f, 0.3f, 0.3f, 0.7f)
        onInit()
    }

    override fun onSurfaceChanged(gl: GL10, width: Int, height: Int) {
        GLES20.glViewport(0, 0, width, height)
    }

    override fun onDrawFrame(gl: GL10) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT or GLES20.GL_DEPTH_BUFFER_BIT)
    }

    /**
     * 检查着色器编译是否成功
     */
    fun checkShaderCompileStatus(shader: Int, shaderType: Int) {
        val status = IntArray(1)
        GLES20.glGetShaderiv(shader, GLES20.GL_COMPILE_STATUS, status, 0)
        if (status[0] == GLES20.GL_FALSE) {
            Log.e(TAG, "shader compile error, shader type: $shaderType; log: ${GLES20.glGetShaderInfoLog(shader)}")
        } else {
            Log.d(TAG, "shader compile success")
        }
    }

    /**
     * 检查程序链接是否成功
     */
    fun checkLinkProgramStatus(program: Int) {
        val status = IntArray(1)
        GLES20.glGetProgramiv(program, GLES20.GL_LINK_STATUS, status, 0)
        if (status[0] == GLES20.GL_FALSE) {
            Log.e(TAG, "link shader error:${GLES20.glGetProgramInfoLog(program)}")
        } else {
            Log.d(TAG, "link shader success")
        }
    }
}
```



