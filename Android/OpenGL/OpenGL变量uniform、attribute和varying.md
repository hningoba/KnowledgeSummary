---
title: OpenGL变量uniform、attribute和varying
categories: 技术
---

本篇文章主要梳理下OpenGL几种变量的区别。

<!--more-->

下面是一段顶点、片段着色器代码：

```
 private final String vertexShaderCode =
            "attribute vec4 vPosition;" +
                    "uniform mat4 vMatrix;"+
                    "varying  vec4 vColor;"+
                    "attribute vec4 aColor;"+
                    "void main() {" +
                    "  gl_Position = vMatrix*vPosition;" +
                    "  vColor=aColor;"+
                    "}";

    private final String fragmentShaderCode =
            "precision mediump float;" +
                    "varying vec4 vColor;" +
                    "void main() {" +
                    "  gl_FragColor = vColor;" +
                    "}";
```



### 统一变量uniform

统一变量相当于应用程序传递给着色器（vertex、fragment）的只读值，即常量。

uniform一般用来表示变换矩阵、照明参数、颜色等，比如

```
uniform mat4 viewProjMatrix;	// 投影观察矩阵
uniform mat4 viewMatrix;			// 观察矩阵
uniform vec3 lightPosition;		// 光源坐标
```

统一变量的命名空间在顶点着色器和片段着色器中是共享的。也就是说，如果顶点和片段着色器一起链接到一个程序对象，它们就会共享同一组统一变量，前提是要求uniform变量在vertex和fragment两者之间声明方式完全一样。

程序中一般通过glUniform*()给统一变量赋值。比如开头示例代码中，顶点着色器的vMatrix：

```
// 找到变量索引
int mMatrixHandler= GLES20.glGetUniformLocation(mProgram,"vMatrix");
// 给变量赋值，其中mMVPMatrix是提前算好的变换矩阵
GLES20.glUniformMatrix4fv(mMatrixHandler,1,false,mMVPMatrix,0);
```



### 属性attribute

attribute变量只能用在顶点着色器中，不能再片段着色器中声明或使用。

attribute一般用来表示顶点相关的数据，比如顶点坐标、顶点法线、纹理坐标、顶点颜色等。

程序中一般用glGetAttribLocation、glBindAttribLocation等获取attribute变量的索引，再用glVertexAttribPointer给变量赋值。比如开头示例中顶点着色器中的vPosition：

```
// 获取变量索引
int mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");
// 给变量赋值
GLES20.glVertexAttribPointer(mPositionHandle, COORDS_PER_VERTEX,
                GLES20.GL_FLOAT, false, vertexStride, vertexBuffer);
```



### 变量varying

varying变量一般用于顶点着色器和片段着色器之间数据传递。一般在顶点着色器中对varying变量赋值，片段着色器中使用该变量。

varying变量的声明在顶点着色器和片段着色器中必须一致。且varying变量的赋值一般是通过一个attribute变量对齐赋值，比如示例代码中的vColor。

```
"varying  vec4 vColor;"+
"attribute vec4 aColor;"+
"void main() {" +
"  vColor=aColor;"+
"}";
```







