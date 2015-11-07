# 著色器(Shader)

原文     | [Shaders](http://learnopengl.com/#!Getting-started/Shaders)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | Geequlim

在[Hello Triangle](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/04%20Hello%20Triangle/)教程中提到，著色器是運行在GPU上的小程序。這些小程序為圖形渲染管線的一個特定部分而運行。從基本意義上來說，著色器不是別的，只是一種把輸入轉化為輸出的程序。著色器也是一種相當獨立的程序，它們不能相互通信；只能通過輸入和輸出的方式來進行溝通。

前面的教程裡我們簡要地觸及了一點著色器的皮毛。瞭解瞭如何恰當地使用它們。現在我們會用一種更加通用的方式詳細解釋著色器，特別是OpenGL著色器語言。

## GLSL

著色器是使用一種叫GLSL的類C語言寫成的。GLSL是為圖形計算量身定製的，它包含針對向量和矩陣操作的有用特性。

著色器的開頭總是要聲明版本，接著是輸入和輸出變量、uniform和`main`函數。每個著色器的入口都是`main`函數，在這裡我們處理所有輸入變量，用輸出變量輸出結果。如果你不知道什麼是uniform也不用擔心，我們後面會進行講解。

一個典型的著色器有下面的結構：

```c++
#version version_number

in type in_variable_name;
in type in_variable_name;

out type out_variable_name;

uniform type uniform_name;

int main()
{
  // 處理輸入
  ...
  // 輸出
  out_variable_name = weird_stuff_we_processed;
}
```

當我們談論特別是談到頂點著色器的時候，每個輸入變量也叫頂點屬性(Vertex Attribute)。能聲明多少個頂點屬性是由硬件決定的。OpenGL確保至少有16個包含4個元素的頂點屬性可用，但是有些硬件或許可用更多，你可以查詢`GL_MAX_VERTEX_ATTRIB`S來獲取這個數目。

```c++
GLint nrAttributes;
glGetIntegerv(GL_MAX_VERTEX_ATTRIBS, &nrAttributes);
std::cout << "Maximum nr of vertex attributes supported: " << nrAttributes << std::endl;
```

通常情況下它會返回至少16個，大部分情況下是夠用了。

## 數據類型

GLSL有像其他編程語言相似的數據類型。GLSL有C風格的默認基礎數據類型：`int`、`float`、`double`、`uint`和`bool`。GLSL也有兩種容器類型，教程中我們會使用很多，它們是向量(Vector)和矩陣(Matrix)，其中矩陣我們會在之後的教程裡再討論。

## 向量(Vector)

GLSL中的向量可以包含有1、2、3或者4個分量，分量類型可以是前面默認基礎類型的任意一個。它們可以是下面的形式(n代表元素數量)：

  類型|含義
   ---|---
 vecn | 包含n個默認為float元素的向量
 bvecn| 包含n個布爾元素向量
 ivecn| 包含n個int元素的向量
 uvecn| 包含n個unsigned int元素的向量
 dvecn| 包含n個double元素的向量

大多數時候我們使用vecn，因為float足夠滿足大多數要求。

一個向量的元素可以通過`vec.x`這種方式獲取，這裡`x`是指這個向量的第一個元素。你可以分別使用`.x`、`.y`、`.z`和`.w`來獲取它們的第1、2、3、4號元素。GLSL也允許你使用**rgba**來獲取顏色的元素，或是**stpq**獲取紋理座標元素。

向量的數據類型也允許一些有趣而靈活的元素選擇方式，叫做重組(Swizzling)。重組允許這樣的語法：

```c++
vec2 someVec;
vec4 differentVec = someVec.xyxx;
vec3 anotherVec = differentVec.zyw;
vec4 otherVec = someVec.xxxx + anotherVec.yxzy;
```

你可以使用上面任何4個字母組合來創建一個新的和原來向量一樣長的向量(但4個元素需要是同一種類型)；不允許在一個vec2向量中去獲取.z元素。我們可以把一個向量作為一個參數傳給不同的向量構造函數，以減少參數需求的數量：

```c++
vec2 vect = vec2(0.5f, 0.7f);
vec4 result = vec4(vect, 0.0f, 0.0f);
vec4 otherResult = vec4(result.xyz, 1.0f);
```

向量是一種靈活的數據類型，我們可以把用在所有輸入和輸出上。學完教程你會看到很多如何創造性地管理向量的例子。

## 輸入與輸出(in vs out)

著色器是各自獨立的小程序，但是它們都是一個整體的局部，出於這樣的原因，我們希望每個著色器都有輸入和輸出，這樣才能進行數據交流和傳遞。GLSL定義了`in`和`out`關鍵字來實現這個目的。每個著色器使用這些關鍵字定義輸入和輸出，無論在哪兒，一個輸出變量就能與一個下一個階段的輸入變量相匹配。他們在頂點和片段著色器之間有點不同。

頂點著色器應該接收的輸入是一種特有形式，否則就會效率低下。頂點著色器的輸入是特殊的，它所接受的是從頂點數據直接輸入的。為了定義頂點數據被如何組織，我們使用`location`元數據指定輸入變量，這樣我們才可以在CPU上配置頂點屬性。我們已經在前面的教程看過`layout (location = 0)`。頂點著色器需要為它的輸入提供一個額外的`layout`定義，這樣我們才能把它鏈接到頂點數據。

!!! Important

	也可以移除`layout (location = 0)`，通過在OpenGL代碼中使用`glGetAttribLocation`請求屬性地址(Location)，但是我更喜歡在著色器中設置它們，理解容易而且節省時間。

另一個例外是片段著色器需要一個`vec4`顏色輸出變量，因為片段著色器需要生成一個最終輸出的顏色。如果你在片段著色器沒有定義輸出顏色，OpenGL會把你的物體渲染為黑色(或白色)。

所以，如果我們打算從一個著色器向另一個著色器發送數據，我們必須**在發送方著色器中聲明一個輸出，在接收方著色器中聲明一個同名輸入**。當名字和類型都一樣的時候，OpenGL就會把兩個變量鏈接到一起，它們之間就能發送數據了(這是在鏈接程序(Program)對象時完成的)。為了展示這是這麼工作的，我們會改變前面教程裡的那個著色器，讓頂點著色器為片段著色器決定顏色。

#### 頂點著色器

```c++
#version 330 core
layout (location = 0) in vec3 position; // 位置變量的屬性為0

out vec4 vertexColor; // 為片段著色器指定一個顏色輸出

void main()
{
    gl_Position = vec4(position, 1.0); // 把一個vec3作為vec4的構造器的參數
    vertexColor = vec4(0.5f, 0.0f, 0.0f, 1.0f); // 把輸出顏色設置為暗紅色
}
```
#### 片段著色器

```c++
#version 330 core
in vec4 vertexColor; // 和頂點著色器的vertexColor變量類型相同、名稱相同

out vec4 color; // 片段著色器輸出的變量名可以任意命名，類型必須是vec4

void main()
{
    color = vertexColor;
}
```

你可以看到我們在頂點著色器中聲明瞭一個`vertexColor`變量作為`vec4`輸出，在片段著色器聲明瞭一個一樣的`vertexColor`。由於它們**類型相同並且名字也相同**，片段著色器中的`vertexColor`就和頂點著色器中的`vertexColor`鏈接了。因為我們在頂點著色器中設置的顏色是深紅色的，片段著色器輸出的結果也是深紅色的。下面的圖片展示了輸出結果：

![](http://learnopengl.com/img/getting-started/shaders.png)

我們完成了從頂點著色器向片段著色器發送數據。讓我們更上一層樓，看看能否從應用程序中直接給片段著色器發送一個顏色！

## Uniform

uniform是另一種從CPU應用向GPU著色器發送數據的方式，但uniform和頂點屬性有點不同。首先，uniform是**全局的(Global)**。這裡全局的意思是uniform變量必須在所有著色器程序對象中都是獨一無二的，它可以在著色器程序的任何著色器任何階段使用。第二，無論你把uniform值設置成什麼，uniform會一直保存它們的數據，直到它們被重置或更新。

我們可以簡單地通過在片段著色器中設置uniform關鍵字接類型和變量名來聲明一個GLSL的uniform。之後，我們可以在著色器中使用新聲明的uniform了。我們來看看這次是否能通過uniform設置三角形的顏色：

```c++
#version 330 core
out vec4 color;

uniform vec4 ourColor; //在程序代碼中設置

void main()
{
    color = ourColor;
}  
```

我們在片段著色器中聲明瞭一個uniform vec4的`ourColor`，並把片段著色器的輸出顏色設置為uniform值。因為uniform是全局變量，我們我們可以在任何著色器中定義它們，而無需通過頂點著色器作為中介。頂點著色器中不需要這個uniform所以不用在那裡定義它。

!!! Attention

	如果你聲明瞭一個uniform卻在GLSL代碼中沒用過，編譯器會靜默移除這個變量，從而最後編譯出的版本中並不會包含它，如果有一個從沒用過的uniform出現在已編譯版本中會出現幾個錯誤，記住這點！

uniform現在還是空的；我們沒有給它添加任何數據，所以下面就做這件事。我們首先需要找到著色器中uniform的索引/地址。當我們得到uniform的索引/地址後，我們就可以更新它的值了。這裡我們不去給像素傳遞一個顏色，而是隨著時間讓它改變顏色：

```c++
GLfloat timeValue = glfwGetTime();
GLfloat greenValue = (sin(timeValue) / 2) + 0.5;
GLint vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
glUseProgram(shaderProgram);
glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);
```

首先我們通過`glfwGetTime()`獲取運行的秒數。然後我們使用餘弦函數在0.0到-1.0之間改變顏色，最後儲存到`greenValue`裡。

接著，我們用`glGetUniformLocation`請求`uniform ourColor`的地址。我們為請求函數提供著色器程序和uniform的名字(這是我們希望獲得的地址的來源)。如果`glGetUniformLocation`返回`-1`就代表沒有找到這個地址。最後，我們可以通過`glUniform4f`函數設置uniform值。注意，查詢uniform地址不需要在之前使用著色器程序，但是更新一個unform之前**必須**使用程序(調用`glUseProgram`)，因為它是在當前激活的著色器程序中設置unform的。

!!! Important

	因為OpenGL是C庫內核，所以它不支持函數重載，在函數參數不同的時候就要定義新的函數；glUniform是一個典型例子。這個函數有一個特定的作為類型的後綴。有幾種可用的後綴：

    後綴|含義
     ---|--
	  f | 函數需要以一個float作為它的值
	  i | 函數需要一個int作為它的值
	  ui| 函數需要一個unsigned int作為它的值
	  3f| 函數需要3個float作為它的值
	  fv| 函數需要一個float向量/數組作為它的值

    每當你打算配置一個OpenGL的選項時就可以簡單地根據這些規則選擇適合你的數據類型的重載的函數。在我們的例子裡，我們使用uniform的4float版，所以我們通過`glUniform4f`傳遞我們的數據(注意，我們也可以使用fv版本)。

現在你知道如何設置uniform變量的值了，我們可以使用它們來渲染了。如果我們打算讓顏色慢慢變化，我們就要在遊戲循環的每一幀更新這個uniform，否則三角形就不會改變顏色。下面我們就計算greenValue然後每個渲染迭代都更新這個uniform：

```c++
while(!glfwWindowShouldClose(window))
{
    // 檢測事件
    glfwPollEvents();

    // 渲染
    // 清空顏色緩衝
    glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    // 激活著色器
    glUseProgram(shaderProgram);

    // 更新uniform顏色
    GLfloat timeValue = glfwGetTime();
    GLfloat greenValue = (sin(timeValue) / 2) + 0.5;
    GLint vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
    glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);

    // 繪製三角形
    glBindVertexArray(VAO);
    glDrawArrays(GL_TRIANGLES, 0, 3);
    glBindVertexArray(0);
}
```

新代碼和上一節的很相似。這次，我們在每個循環繪製三角形前先更新uniform值。如果你成功更新uniform了，你會看到你的三角形逐漸由綠變黑再變綠。

<video src="http://learnopengl.com/video/getting-started/shaders.mp4" controls="controls"/></video>

如果你在哪兒卡住了，[這裡有源碼](http://www.learnopengl.com/code_viewer.php?code=getting-started/shaders-uniform)。

就像你所看到的那樣，uniform是個設置屬性的很有用的工具，它可以在渲染循環中改變，也可以在你的應用和著色器之間進行數據交互，但假如我們打算為每個頂點設置一個顏色的時候該怎麼辦？這種情況下，我們就不得不聲明和頂點數目一樣多的uniform了。在頂點屬性問題上一個更好的解決方案一定要能包含足夠多的數據，這是我們接下來要講的內容。

## 更多屬性

前面的教程中，我們瞭解瞭如何填充VBO、配置頂點屬性指針以及如何把它們都儲存到VAO裡。這次，我們同樣打算把顏色數據加進頂點數據中。我們將把顏色數據表示為3個float的**頂點數組(Vertex Array)**。我們為三角形的每個角分別指定為紅色、綠色和藍色：

```c++
GLfloat vertices[] = {
    // 位置                 // 顏色
     0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,   // 右下
    -0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,   // 左下
     0.0f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f    // 頂部
};
```

由於我們現在發送到頂點著色器的數據更多了，有必要調整頂點著色器，使它能夠把顏色值作為一個頂點屬性輸入。需要注意的是我們用`layout`標識符來吧`color`屬性的`location`設置為1：

```c++
#version 330 core
layout (location = 0) in vec3 position; // 位置變量的屬性position為 0 
layout (location = 1) in vec3 color;	// 顏色變量的屬性position為 1

out vec3 ourColor; // 向片段著色器輸出一個顏色

void main()
{
    gl_Position = vec4(position, 1.0);
    ourColor = color; // 把ourColor設置為我們從頂點數據那裡得到的輸入顏色
}
```

由於我們不再使用uniform來傳遞片段的顏色了，現在使用的`ourColor`輸出變量要求必須也去改變片段著色器：

```c++
#version 330 core
in vec3 ourColor
out vec4 color;
void main()
{
    color = vec4(ourColor, 1.0f);
}
```

因為我們添加了另一個頂點屬性，並且更新了VBO的內存，我們就必須重新配置頂點屬性指針。更新後的VBO內存中的數據現在看起來像這樣：

![](http://learnopengl.com/img/getting-started/vertex_attribute_pointer_interleaved.png)

知道了當前使用的layout，我們就可以使用`glVertexAttribPointer`函數更新頂點格式，

```c++
// 頂點屬性
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (GLvoid*)0);
glEnableVertexAttribArray(0);
// 顏色屬性
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (GLvoid*)(3* sizeof(GLfloat)));
glEnableVertexAttribArray(1);
```

`glVertexAttribPointer`函數的前幾個參數比較明瞭。這次我們配置屬性location為1的頂點屬性。顏色值有3個float那麼大，我們不去標準化這些值。

由於我們現在有了兩個頂點屬性，我們不得不重新計算步長值(Stride)。為獲得數據隊列中下一個屬性值(比如位置向量的下個x元素)我們必須向右移動6個float，其中3個是位置值，另外三個是顏色值。這給了我們6個步長的大小，每個步長都是float的字節數(=24字節)。

同樣，這次我們必須指定一個偏移量(Offset)。對於每個頂點來說，位置(Position)頂點屬性是先聲明的，所以它的偏移量是0。顏色屬性緊隨位置數據之後，所以偏移量就是`3*sizeof(GLfloat)`，用字節來計算就是12字節。

運行應用你會看到如下結果：
![](http://learnopengl.com/img/getting-started/shaders3.png)

如果你有困惑，可以[在這裡獲得源碼](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-interpolated)。

這個圖片可能不是你所期望的那種，因為我們只提供3個顏色，而不是我們現在看到的大調色板。這是所謂片段著色器進行**片段插值(Fragment Interpolation)**的結果。當渲染一個三角形在像素化(Rasterization 也譯為光柵化)階段通常生成比原來的頂點更多的像素。像素器就會基於每個像素在三角形的所處相對位置決定像素的位置。

基於這些位置，它**插入(Interpolate)**所有片段著色器的輸入變量。比如說，我們有一個線段，上面的那個點是綠色的，下面的點是藍色的。如果一個片段著色器正在處理的那個片段(實際上就是像素)是在線段的70%的位置，它的顏色輸入屬性就會是一個綠色和藍色的線性結合；更精確地說就是30%藍+70%綠。

這正是這個三角形裡發生的事。我們有3個頂點，和相應的3個顏色，從這個三角形的像素來看它可能包含50,000左右的像素，片段著色器為這些像素進行插值。如果你仔細看這些顏色，你會發現其中的奧祕：紅到紫再到藍。像素插值會應用到所有片段著色器的輸入屬性上。

## 我們自己的著色器類

編寫、編譯、管理著色器是件麻煩事。在著色器的最後主題裡，我們會寫一個類來讓我們的生活輕鬆一點，這個類從硬盤讀著色器，然後編譯和鏈接它們，對它們進行錯誤檢測，這就變得很好用了。這也會給你一些關於如何把我們目前所學的知識封裝到一個抽象的對象裡的靈感。

我們會在頭文件裡創建整個類，主要為了學習，也可以方便移植。我們先來添加必要的include，定義類結構：

```c++
#ifndef SHADER_H
#define SHADER_H

#include <string>
#include <fstream>
#include <sstream>
#include <iostream>

using namespace std;

#include <GL/glew.h>; // 包含glew獲取所有的OpenGL必要headers

class Shader
{
public:
        // 程序ID
        GLuint Program;
        // 構造器讀取並創建Shader
        Shader(const GLchar * vertexSourcePath, const GLchar * fragmentSourcePath);
        // 使用Program
        void Use();
};

#endif
```

!!! Important

	在上面，我們用了幾個預處理指令(Preprocessor Directives)。這些預處理指令告知你的編譯器，只在沒被包含過的情況下才包含和編譯這個頭文件，即使多個文件都包含了這個shader頭文件,它是用來防止鏈接衝突的。

shader類保留了著色器程序的ID。它的構造器需要頂點和片段著色器源代碼的文件路徑，我們可以把各自的文本文件儲存在硬盤上。`Use`函數看似平常，但是能夠顯示這個自造類如何讓我們的生活變輕鬆(雖然只有一點)。

### 從文件讀取

我們使用C++文件流讀取著色器內容，儲存到幾個string對象裡([譯註1])

```c++
Shader(const GLchar * vertexPath, const GLchar * fragmentPath)
{
    // 1. 從文件路徑獲得vertex/fragment源碼
    std::string vertexCode;
    std::string fragmentCode;

    try {
        // 打開文件
        std::ifstream vShaderFile(vertexPath);
        std::ifstream fShaderFile(fragmentPath);

        std::stringstream vShaderStream, fShaderStream;
        // 讀取文件緩衝到流
        vShaderStream << vShaderFile.rdbuf();
        fShaderStream << fShaderFile.rdbuf();

        // 關閉文件句柄
        vShaderFile.close();
        fShaderFile.close();

        // 將流轉為GLchar數組
        vertexCode = vShaderStream.str();
        fragmentCode = fShaderStream.str();
    }
    catch(std::exception e)
    {
        std::cout << "ERROR::SHADER::FILE_NOT_SUCCESFULLY_READ" << std::endl;  
    }
```

下一步，我們需要編譯和鏈接著色器。注意，我們也要檢查編譯/鏈接是否失敗，如果失敗，打印編譯錯誤，調試的時候這及其重要(這些錯誤日誌你總會需要的)：

```c++
// 2. 編譯著色器
GLuint vertex, fragment;
GLint success;
GLchar infoLog[512];

// 頂點著色器
vertex = glCreateShader(GL_VERTEX_SHADER);
glShaderSource(vertex, 1, &vShaderCode, NULL);
glCompileShader(vertex);

// 打印編譯時錯誤
glGetShaderiv(vertex, GL_COMPILE_STATUS, &success);
if(!success)
{
    glGetShaderInfoLog(vertex, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
};

// 對片段著色器進行類似處理
[...]

// 著色器程序
this->Program = glCreateProgram();
glAttachShader(this->Program, vertex);
glAttachShader(this->Program, fragment);
glLinkProgram(this->Program);

// 打印連接錯誤
glGetProgramiv(this->Program, GL_LINK_STATUS, &success);
if(!success)
{
    glGetProgramInfoLog(this->Program, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n" << infoLog << std::endl;
}

// 刪除著色器
glDeleteShader(vertex);
glDeleteShader(fragment);
```

最後我們也要實現Use函數：

```c++
void Use()
{
    glUseProgram(this->Program);
}
```

現在我們寫完了一個完整的著色器類。使用著色器類很簡單；我們創建一個著色器對象以後，就可以簡單的使用了：

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

我們把頂點和片段著色器儲存為兩個叫做`shader.vs`和`shader.frag`的文件。你可以使用自己喜歡的名字命名著色器文件；我自己覺得用`.vs`和`.frag`作為擴展名很直觀。

使用新著色器類的[程序](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-using-object)，[著色器類](http://learnopengl.com/code_viewer.php?type=header&code=shader)，[頂點著色器](http://learnopengl.com/code_viewer.php?type=vertex&code=getting-started/basic)，[片段著色器](http://learnopengl.com/code_viewer.php?type=fragment&code=getting-started/basic)。

## 練習

1. 修改頂點著色器讓三角形上下顛倒：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-exercise1)
2. 通過使用uniform定義一個水平偏移，在頂點著色器中使用這個偏移量把三角形移動到屏幕右側：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-exercise2)
3. 使用`out`關鍵字把頂點位置輸出到片段著色器，把像素的顏色設置為與頂點位置相等(看看頂點位置值是如何在三角形中進行插值的)。做完這些後，嘗試回答下面的問題：為什麼在三角形的左下角是黑的?：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/shaders-exercise3)

[譯註1]: http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/05%20Shaders/#_5 "譯者注：實際上把著色器代碼保存在文件中適合學習OpenGL的時候，實際開發中最好把一個著色器直接儲存為多個字符串，這樣具有更高的靈活度。"
