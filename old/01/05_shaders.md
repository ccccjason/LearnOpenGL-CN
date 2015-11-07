本文作者JoeyDeVries，由[codeman001](https://github.com/codeman001)翻譯自[http://learnopengl.com](http://learnopengl.com/#!Getting-started/Shaders)

##著色器

在[Hello Triangle](https://github.com/Geequlim/LearnOpenGL-CN/blob/master/01%20Getting%20started/04%20Hello%20Triangle.md) 中已經提到，著色器是在GPU上運行的小程序，這些程序負責處理圖形渲染管線的特定階段，簡單來說，著色器也是一個處理輸入輸出的程序。著色器是非常孤立執行的程序，彼此之間沒有太多的交互，

在前面的教程中，我們大致介紹了表面著色器和如何正確使用它們。接下來我們講接觸更加流行的OpenGL著色語言

###GLSL
GLSL是寫法類似C語言，GLSL是專門針對圖形以及向量和矩陣變化設計的。著色器通常開始生命一個版本信息，然後是輸入、輸出列表以及常量和入口函數（main函數），每個著色器都是在入口函數處來處理輸入參數。然後輸出結果。


著色器通常具有以下結構：
```c++
#version version_number
  
in type in_variable_name;
in type in_variable_name;

out type out_variable_name;
  
uniform type uniform_name;
  
int main()
{
  // Process input(s) and do some weird graphics stuff
  ...
  // Output processed stuff to output variable
  out_variable_name = weird_stuff_we_processed;
}
```

所謂的頂點著色器就講頂點屬性作為輸入的著色器，頂點屬性的個數主要受限於硬件實現，OpenGL的保證總有至少16個四分量的頂點可以使用，但是硬件可能個多些，可以通過查詢：**GL_MAX_VERTEX_ATTRIBS**
```c++
GLint nrAttributes;
glGetIntegerv(GL_MAX_VERTEX_ATTRIBS, &nrAttributes);
std::cout << "Maximum nr of vertex attributes supported: " << nrAttributes << std::endl;
```

這個結果最小返回16，一般情況下使用。

###類型


GLSL跟一般的編程語言一樣，定義了一些常用的數據類型，基礎類型比如：int,float, double, uint and bool等，還用兩種我們在教程中經常用到的容器類型：vectors(向量) 和 matrices（矩陣），稍後會提到。


####向量
一個向量可以包含1-4個分量，分量的類型可以是上面我們提到的基礎類型。向量個類型名字規則如下：

- vecn: 默認向量，有個n浮點型分量.
- bvecn: 有n個bool分量.
- ivecn: 有n個整型分量.
- uvecn: 有n個無符號整型分量.
- dvecn: 有n個雙浮點型分量.

大多數情況下我們使用vecn，因為浮點型足夠滿足我們大多數需求。
向量的分量可以通過vec.x來訪問第一個分量，同時可以使用.x,.y,.z,.w來訪問一個向量的四個成員，同樣可以使用rgba來訪問顏色值對應的向量，紋理座標則可以使用stpq來訪問分量的值。
向量的訪問方式支持趣味性和擴展性，被稱為交叉混合性，實例如下：
```c++
vec2 someVec;
vec4 differentVec = someVec.xyxx;
vec3 anotherVec = differentVec.zyw;
vec4 otherVec = someVec.xxxx + anotherVec.yxzy;
```
可以使用任意組合來組成新向量，同時也可以把維度小的向量放在高緯度向量的構造函數裡來構成新的向量。如下代碼：
```c++
vec2 vect = vec2(0.5f, 0.7f);
vec4 result = vec4(vect, 0.0f, 0.0f);
vec4 otherResult = vec4(result.xyz, 1.0f);
```
###輸入和輸出
著色器是運行在GPU上的小程序，但是麻雀雖小五臟俱全，也會有輸入和輸出來構成完整的程序，GLSL使用in和out關鍵字來定義輸入和輸出。每個著色器可以是這些關鍵字來指定輸入和輸出，輸入變量經過處理以後會得到適合下個處理階段可以使用的輸出變量，頂點著色器和片段著色器有點小卻別。

頂點著色器接受特定格式的輸入，否則不能正確使用。頂點著色器直接接受輸入的頂點數據，但是需要在CPU一邊指定數據的對應的位置，前面的教程可以到對位置0的輸入（location=0），所以頂點著色器需要一個特定的聲明來確定CPU和GPU數據對應關聯關係，

<div style="border:solid #AFDFAF;border-radius:5px;background-color:#D8F5D8;margin:20px 20px 20px 0px;padding:15px">
也可以使用*glGetAttribLocation*來查詢對應的位置，這樣可以省略layout的聲明，但是我覺得可以是用layout聲明比較好，這也可以減少GPU的一些工作
</div>


對於片段著色器有個vec4的顏色作為特定的輸出，因為片段處理後最終是要生產一個顏色來顯示的。否則將輸出黑色或白色顏色作為輸出。

如果我們想從一個著色器向另外一個發送數據，我們需要在發送方定義一個輸出，然後再接收方頂一個輸入，同時保證這兩個變量類型和名字是相同的。
下面是一個實例來展示如何從頂點著色器傳遞一個顏色值跟片段著色器使用：

***頂點著色器***
```c++
#version 330 core
layout (location = 0) in vec3 position; // The position variable has attribute position 0
  
out vec4 vertexColor; // Specify a color output to the fragment shader

void main()
{
    gl_Position = vec4(position, 1.0); // See how we directly give a vec3 to vec4's constructor
    vertexColor = vec4(0.5f, 0.0f, 0.0f, 1.0f); // Set the output variable to a dark-red color
}
```
***片段著色器***
```c++
#version 330 core
in vec4 vertexColor; // The input variable from the vertex shader (same name and same type)
  
out vec4 color;

void main()
{
    color = vertexColor;
}
```
可以看到在頂點著色器生命了一個向量：*vertexColor* 有out修飾，同時在片段著色器聲明瞭一個*vertexColor* 使用in來修飾，這樣片段著色器就可以獲取頂點著色器處理的*vertexColor*的結果了。
根據上面shader，可以得出下圖的效果：

![shader效果](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/shaders1.jpg)


**常量**
常量是另外一箇中從CPU端向GPU傳輸數據的方式，常量方式跟頂點數據有非常明顯的不同。常量有兩個特性：

1.具有全局性，可以在著色器不同階段來獲取同一個常量

2.不變性，一旦設置了值，在渲染過程中就不能被改變，只有從新設置才能改變。

聲明常量非常簡單使用uniform 放在類型和變量名前面即可。下面看一個例子：
```c++
#version 330 core
  
out vec4 color;
  
uniform vec4 ourColor; // We set this variable in the OpenGL code.

void main()
{
    color = ourColor;
}
```
聲明瞭一個ourColor為常量類型，然後把它的值付給了輸出變量color。
<div style="border:solid #E1B3B3;border-radius:10px;background-color:#FFD2D2;margin:10px 10px 10px 0px;padding:10px">
注意：如果聲明瞭一個從來沒用到常量，GLSL的編譯器會默認刪除這個常量，由此可能導致一些莫名的問題。
</div>
現在這個常量還是個空值，接下來給ourColor在CPU端傳遞數據給它。思路：獲取ourColor在索引位置，然後傳遞數據給這個位置。另外做一些小動作，不傳遞固定的這個，傳遞一個隨時間變化的值，如下：
```c++
GLfloat timeValue = glfwGetTime();
GLfloat greenValue = (sin(timeValue) / 2) + 0.5;
GLint vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
glUseProgram(shaderProgram);
glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);
```
首先通過glfwGetTime函數獲取程序運行時間（秒數）。然後使用sin函數將greenValue的值控制在0-1。

然後使用glGetUniformLocation函數查詢ourColor的索引位置。是一個參數是要查詢的著色器程序，第二個參數是常量在著色器中聲明的變量名。如果glGetUniformLocation函數返回-1，表明沒找到對應的常量的索引位置。

最合使用glUniform4f來完成賦值。

注意：使用glGetUniformLocation 不需要在glUseProgram之後，但是glUniform4f一定要在lUseProgram之後，因為我們也只能對當前激活的著色器程序傳遞數據。
到目前為止已經學會了這麼給常量傳遞數據和渲染使用這些數據，如果我們想每幀改變常量的值，我們需要在主循環的不停的計算和更新常量的值。
```c++
while(!glfwWindowShouldClose(window))
{
    // Check and call events
    glfwPollEvents();

    // Render
    // Clear the colorbuffer
    glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    // Be sure to activate the shader
    glUseProgram(shaderProgram);
  
    // Update the uniform color
    GLfloat timeValue = glfwGetTime();
    GLfloat greenValue = (sin(timeValue) / 2) + 0.5;
    GLint vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
    glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);

    // Now draw the triangle
    glBindVertexArray(VAO);
    glDrawArrays(GL_TRIANGLES, 0, 3);
    glBindVertexArray(0);
}
```
如果運行正常的話我們能看到一個綠色到黑色，黑色到綠色變化的三角形，
可以查看完整的代碼[實例](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-interpolated)


**著色器程序管理程序**

編寫、編譯和管理著色器程序是非常繁重的工作，為了減輕這個工作量我們自己定義一個著色器程序管理器，負責讀取著色器程序文件，然後編譯他們，鏈接並檢查著色器程序有無錯誤發生。這也可以讓我們把已經學到的知識封裝到抽象的對象裡。

我們首先頂一個著色器程序的頭文件，如下：
```c++
#ifndef SHADER_H
#define SHADER_H

#include <string>
#include <fstream>
#include <sstream>
#include <iostream>
using namespace std;
  
#include <GL/glew.h>; // Include glew to get all the required OpenGL headers

class Shader
{
public:
  	// The program ID
	GLuint Program;
	// Constructor reads and builds the shader
	Shader(const GLchar* vertexSourcePath, const GLchar* fragmentSourcePath);
  	// Use the program
  	void Use();
};
  
#endif
```
著色器程序類包含一個著色器程序ID，有一個接受頂點和片段程序的接口，這個兩個路徑就是普通的文本文件就可以了。
Use函數是一個工具屬性的函數，主要是控制當前著色器程序是否激活。

讀取著色器程序文件
```c++
Shader(const GLchar* vertexPath, const GLchar* fragmentPath)
{
    // 1. Retrieve the vertex/fragment source code from filePath
    std::string vertexCode;
    std::string fragmentCode;
    std::ifstream vShaderFile;
    std::ifstream fShaderFile;
    // ensures ifstream objects can throw exceptions:
    vShaderFile.exceptions (std::ifstream::failbit | std::ifstream::badbit);
    fShaderFile.exceptions (std::ifstream::failbit | std::ifstream::badbit);
    try 
    {
        // Open files
        vShaderFile.open(vertexPath);
        fShaderFile.open(fragmentPath);
        std::stringstream vShaderStream, fShaderStream;
        // Read file's buffer contents into streams
        vShaderStream << vShaderFile.rdbuf();
        fShaderStream << fShaderFile.rdbuf();		
        // close file handlers
        vShaderFile.close();
        fShaderFile.close();
        // Convert stream into GLchar array
        vertexCode = vShaderStream.str();
        fragmentCode = fShaderStream.str();		
    }
    catch(std::ifstream::failure e)
    {
        std::cout << "ERROR::SHADER::FILE_NOT_SUCCESFULLY_READ" << std::endl;
    }
    const GLchar* vShaderCode = vertexCode.c_str();
    const GLchar* fShaderCode = fragmentCode.c_str();
    [...]
```

接下來編譯和鏈接這些程序，同時收集一些編譯和鏈接的錯誤，來幫助我們調試。
```c++
// 2. Compile shaders
GLuint vertex, fragment;
GLint success;
GLchar infoLog[512];
   
// Vertex Shader
vertex = glCreateShader(GL_VERTEX_SHADER);
glShaderSource(vertex, 1, &vShaderCode, NULL);
glCompileShader(vertex);
// Print compile errors if any
glGetShaderiv(vertex, GL_COMPILE_STATUS, &success);
if(!success)
{
    glGetShaderInfoLog(vertex, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
};
  
// Similiar for Fragment Shader
[...]
  
// Shader Program
this->Program = glCreateProgram();
glAttachShader(this->Program, vertex);
glAttachShader(this->Program, fragment);
glLinkProgram(this->Program);
// Print linking errors if any
glGetProgramiv(this->Program, GL_LINK_STATUS, &success);
if(!success)
{
    glGetProgramInfoLog(this->Program, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n" << infoLog << std::endl;
}
  
// Delete the shaders as they're linked into our program now and no longer necessery
glDeleteShader(vertex);
glDeleteShader(fragment);
```
最後來實現一個use函數
	
	void Use() { glUseProgram(this->Program); }
接下來是一個使用這個簡單實例：
```c++
Shader ourShader("path/to/shaders/shader.vs", "path/to/shaders/shader.frag");
...
while(...)
{
    ourShader.Use();
    glUniform1f(glGetUniformLocation(ourShader.Program, "someUniform"), 1.0f);
    DrawStuff();
}
```
上面我們已經把頂點著色器和片段著色器代碼分別放在shader.vs和shader.frag裡面了，這些文件的名字和後綴名都可以隨意命名的，只要符合文件名規範就好。
完整的代碼實例：

[使用的實例](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-using-object)

[著色器類](http://learnopengl.com/code_viewer.php?type=header&code=shader)

[頂點著色器代碼](http://learnopengl.com/code_viewer.php?type=vertex&code=getting-started/basic)

[片段著色器代碼](http://learnopengl.com/code_viewer.php?type=fragment&code=getting-started/basic)


**練習題**

1.通過調整的頂點著色器，以使三角形是倒置 [答案](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-exercise1)

2.通過一個常量，使得三角在x方向偏移 [答案](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-exercise2)

3.使用in 和out 關鍵字，把頂點著色器的位置數據作為片段著色器的顏色，然後看看得出的三角形顏色，進一步理解差值的問題，同時可以嘗試回答下面的問題：為什麼我們三角形左下側有黑邊?:[答案](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-exercise3)


