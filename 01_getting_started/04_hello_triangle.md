# 你好，三角形

原文     | [Creating a window](http://www.learnopengl.com/#!Getting-started/Hello-Triangle)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | Geequlim

## 圖形渲染管線(Pipeline)

在OpenGL中任何事物都在3D空間中，但是屏幕和窗口是一個2D像素陣列，所以OpenGL的大部分工作都是關於如何把3D座標轉變為適應你屏幕的2D像素。3D座標轉為2D座標的處理過程是由OpenGL的**圖形渲染管線**(Pipeline，大多譯為管線，實際上指的是一堆原始圖形數據途經一個輸送管道，期間經過各種變化處理最終出現在屏幕的過程)管理的。圖形渲染管線可以被劃分為兩個主要部分：第一個部分把你的3D座標轉換為2D座標，第二部分是把2D座標轉變為實際的有顏色的像素。這個教程裡，我們會簡單地討論一下圖形渲染管線，以及如何使用它創建一些像素，這對我們來說是有好處的。

!!! Important

	2D座標和像素也是不同的，2D座標是在2D空間中的一個點的非常精確的表達，2D像素是這個點的近似值，它受到你的屏幕/窗口解析度的限制。

圖形渲染管線接收一組3D座標，然後把它們轉變為你屏幕上的有色2D像素。圖形渲染管線可以被劃分為幾個階段，每個階段需要把前一個階段的輸出作為輸入。所有這些階段都是高度專門化的(它們有一個特定的函數)，它們能簡單地並行執行。由於它們的並行執行特性，當今大多數顯卡都有成千上萬的小處理核心，在GPU上為每一個(渲染管線)階段運行各自的小程序，從而在圖形渲染管線中快速處理你的數據。這些小程序叫做 **著色器**(Shader)。

有些著色器允許開發者自己配置，我們可以用自己寫的著色器替換默認的。這樣我們就可以更細緻地控制圖形渲染管線中的特定部分了，因為它們運行在GPU上，所以它們會節約寶貴的CPU時間。OpenGL著色器是用**OpenGL著色器語言**(OpenGL Shading Language, GLSL)寫成的，我們在下一節會花更多時間研究它。

在下面，你會看到一個圖形渲染管線的每個階段的抽象表達。要注意藍色部分代表的是我們可以自定義的著色器。

![](http://geequlim.com/assets/img/blog/LearnOpenGL/01 Getting started/OpenGL_pipline_cn.png)

如你所見，圖形渲染管線包含很多部分，每個都是將你的頂點數據轉變為最後渲染出來的像素這個大過程中的一個特定階段。我們會概括性地解釋渲染管線的每個部分，從而使你對圖形渲染管線的工作方式有個大概瞭解。

我們以數組的形式傳遞3個3D座標作為圖形渲染管線的輸入，它用來表示一個三角形，這個數組叫做頂點數據(Vertex Data)；這裡頂點數據是一些頂點的集合。一個**頂點**是一個3D座標的集合(也就是x、y、z數據)。而頂點數據是用**頂點屬性**(Vertex Attributes)表示的，它可以包含任何我們希望用的數據，但是簡單起見，我們還是假定每個頂點只由一個3D位置([譯註1])和幾個顏色值組成的吧。

[譯註1]: http://learnopengl-cn.readthedocs.org "譯註:當我們談論一個“位置”的時候，它代表在一個“空間”中所處地點的這個特殊屬性；同時“空間”代表著任何一種座標系，比如x、y、z三維座標系，x、y二維座標系，或者一條直線上的x和y的線性關係，只不過二維座標系是一個扁扁的平面空間，而一條直線是一個很瘦的長長的空間。"

!!! Important

	為了讓OpenGL知道我們的座標和顏色值構成的到底是什麼，OpenGL需要你去提示你希望這些數據所表示的是什麼類型。我們是希望把這些數據渲染成一系列的點？一系列的三角形？還是僅僅是一個長長的線？做出的這些提示叫做**基本圖形**(Primitives)，任何一個繪製命令的調用都必須把基本圖形類型傳遞給OpenGL。這是其中的幾個：**GL_POINTS**、**GL_TRIANGLES**、**GL_LINE_STRIP**。

圖形渲染管線的第一個部分是**頂點著色器**(Vertex Shader)，它把一個單獨的頂點作為輸入。頂點著色器主要的目的是把3D座標轉為另一種3D座標(後面會解釋)，同時頂點著色器允許我們對頂點屬性進行一些基本處理。

**基本圖形裝配**(Primitive Assembly)階段把頂點著色器的表示為基本圖形的所有頂點作為輸入(如果選擇的是`GL_POINTS`，那麼就是一個單獨頂點)，把所有點組裝為特定的基本圖形的形狀；本節例子是一個三角形。

基本圖形裝配階段的輸出會傳遞給**幾何著色器**(Geometry Shader)。幾何著色器把基本圖形形式的一系列頂點的集合作為輸入，它可以通過產生新頂點構造出新的(或是其他的)基本圖形來生成其他形狀。例子中，它生成了另一個三角形。

**細分著色器**(Tessellation Shaders)擁有把給定基本圖形**細分**為更多小基本圖形的能力。這樣我們就能在物體更接近玩家的時候通過創建更多的三角形的方式創建出更加平滑的視覺效果。

細分著色器的輸出會進入**光柵化**(Rasterization也譯為像素化)階段，這裡它會把基本圖形映射為屏幕上相應的像素，生成供片段著色器(Fragment Shader)使用的片段(Fragment)。在片段著色器運行之前，會執行**裁切**(Clipping)。裁切會丟棄超出你的視圖以外的那些像素，來提升執行效率。


!!! Important

	OpenGL中的一個fragment是OpenGL渲染一個獨立像素所需的所有數據。

**片段著色器**的主要目的是計算一個像素的最終顏色，這也是OpenGL高級效果產生的地方。通常，片段著色器包含用來計算像素最終顏色的3D場景的一些數據(比如光照、陰影、光的顏色等等)。

在所有相應顏色值確定以後，最終它會傳到另一個階段，我們叫做**alpha測試**和**混合**(Blending)階段。這個階段檢測像素的相應的深度(和Stencil)值(後面會講)，使用這些，來檢查這個像素是否在另一個物體的前面或後面，如此做到相應取捨。這個階段也會檢查**alpha值**(alpha值是一個物體的透明度值)和物體之間的**混合**(Blend)。所以，即使在片段著色器中計算出來了一個像素所輸出的顏色，最後的像素顏色在渲染多個三角形的時候也可能完全不同。

正如你所見的那樣，圖形渲染管線非常複雜，它包含很多要配置的部分。然而，對於大多數場合，我們必須做的只是頂點和片段著色器。幾何著色器和細分著色器是可選的，通常使用默認的著色器就行了。

在現代OpenGL中，我們**必須**定義至少一個頂點著色器和一個片段著色器(因為GPU中沒有默認的頂點/片段著色器)。出於這個原因，開始學習現代OpenGL的時候非常困難，因為在你能夠渲染自己的第一個三角形之前需要一大堆知識。本節結束就是你可以最終渲染出你的三角形的時候，你也會了解到很多圖形編程知識。

## 頂點輸入(Vertex Input)

開始繪製一些東西之前，我們必須給OpenGL輸入一些頂點數據。OpenGL是一個3D圖形庫，所以我們在OpenGL中指定的所有座標都是在3D座標裡(x、y和z)。OpenGL不是簡單的把你所有的3D座標變換為你屏幕上的2D像素；OpenGL只是在當它們的3個軸(x、y和z)在特定的-1.0到1.0的範圍內時才處理3D座標。所有在這個範圍內的座標叫做**標準化設備座標**(Normalized Device Coordinates，NDC)會最終顯示在你的屏幕上(所有出了這個範圍的都不會顯示)。

由於我們希望渲染一個三角形，我們指定所有的這三個頂點都有一個3D位置。我們把它們以`GLfloat`數組的方式定義為標準化設備座標(也就是在OpenGL的可見區域)中。

```c++
GLfloat vertices[] = {
    -0.5f, -0.5f, 0.0f,
     0.5f, -0.5f, 0.0f,
     0.0f,  0.5f, 0.0f
};
```

由於OpenGL是在3D空間中工作的，我們渲染一個2D三角形，它的每個頂點都要有同一個z座標0.0。在這樣的方式中，三角形的每一處的深度(Depth, [譯註2])都一樣，從而使它看上去就像2D的。

[譯註2]: http://learnopengl-cn.readthedocs.org "通常深度可以理解為z座標，它代表一個像素在空間中和你的距離，如果離你遠就可能被別的像素遮擋，你就看不到它了，它會被丟棄，以節省資源。"

!!! Important

	**標準化設備座標(Normalized Device Coordinates, NDC)**

	一旦你的頂點座標已經在頂點著色器中處理過，它們就應該是**標準化設備座標**了，標準化設備座標是一個x、y和z值在-1.0到1.0的一小段空間。任何落在範圍外的座標都會被丟棄/裁剪，不會顯示在你的屏幕上。下面你會看到我們定義的在標準化設備座標中的三角形(忽略z軸)：

    ![](http://www.learnopengl.com/img/getting-started/ndc.png)

	與通常的屏幕座標不同，y軸正方向上的點和(0,0)座標是這個圖像的中心，而不是左上角。最後你希望所有(變換過的)座標都在這個座標空間中，否則它們就不可見了。

	你的標準化設備座標接著會變換為**屏幕空間座標**(Screen-space Coordinates)，這是使用你通過`glViewport`函數提供的數據，進行**視口變換**(Viewport Transform)完成的。最後的屏幕空間座標被變換為像素輸入到片段著色器。

有了這樣的頂點數據，我們會把它作為輸入數據發送給圖形渲染管線的第一個處理階段：頂點著色器。它會在GPU上創建儲存空間用於儲存我們的頂點數據，還要配置OpenGL如何解釋這些內存，並且指定如何發送給顯卡。頂點著色器接著會處理我們告訴它要處理內存中的頂點的數量。

我們通過**頂點緩衝對象**(Vertex Buffer Objects, VBO)管理這個內存，它會在GPU內存(通常被稱為顯存)中儲存大批頂點。使用這些緩衝對象的好處是我們可以一次性的發送一大批數據到顯卡上，而不是每個頂點發送一次。從CPU把數據發送到顯卡相對較慢，所以無論何處我們都要嘗試儘量一次性發送儘可能多的數據。當數據到了顯卡內存中時，頂點著色器幾乎立即就能獲得頂點，這非常快。

頂點緩衝對象(VBO)是我們在OpenGL教程中第一個出現的OpenGL對象。就像OpenGL中的其他對象一樣，這個緩衝有一個獨一無二的ID，所以我們可以使用`glGenBuffers`函數生成一個緩衝ID：

```c++
GLuint VBO;
glGenBuffers(1, &VBO);  
```

OpenGL有很多緩衝對象類型，`GL_ARRAY_BUFFER`是其中一個頂點緩衝對象的緩衝類型。OpenGL允許我們同時綁定多個緩衝，只要它們是不同的緩衝類型。我們可以使用`glBindBuffer`函數把新創建的緩衝綁定到`GL_ARRAY_BUFFER`上：

```c++
glBindBuffer(GL_ARRAY_BUFFER, VBO);  
```

從這一刻起，我們使用的任何緩衝函數(在`GL_ARRAY_BUFFER`目標上)都會用來配置當前綁定的緩衝(`VBO`)。然後我們可以調用`glBufferData`函數，它會把之前定義的頂點數據複製到緩衝的內存中：

```c++
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
```

`glBufferData`是一個用來把用戶定的義數據複製到當前綁定緩衝的函數。它的第一個參數是我們希望把數據複製到上面的緩衝類型：頂點緩衝對象當前綁定到`GL_ARRAY_BUFFER`目標上。第二個參數指定我們希望傳遞給緩衝的數據的大小(以字節為單位)；用一個簡單的`sizeof`計算出頂點數據就行。第三個參數是我們希望發送的真實數據。

第四個參數指定了我們希望顯卡如何管理給定的數據。有三種形式：

- `GL_STATIC_DRAW` ：數據不會或幾乎不會改變。
- `GL_DYNAMIC_DRAW`：數據會被改變很多。
- `GL_STREAM_DRAW` ：數據每次繪製時都會改變。

三角形的位置數據不會改變，每次渲染調用時都保持原樣，所以它使用的類型最好是`GL_STATIC_DRAW`。如果，比如，一個緩衝中的數據將頻繁被改變，那麼使用的類型就是`GL_DYNAMIC_DRAW`或`GL_STREAM_DRAW`。這樣就能確保圖形卡把數據放在高速寫入的內存部分。

現在我們把頂點數據儲存在顯卡的內存中，用VBO頂點緩衝對象管理。下面我們會創建一個頂點和片段著色器，來處理這些數據。現在我們開始著手創建它們吧。

## 頂點著色器(Vertext Shader)

頂點著色器是幾個著色器中的一個，它是可編程的。現代OpenGL需要我們至少設置一個頂點和一個片段著色器，如果我們打算做渲染的話。我們會簡要介紹一下著色器以及配置兩個非常簡單的著色器來繪製我們第一個三角形。下個教程裡我們會更詳細的討論著色器。

我們需要做的第一件事是用著色器語言GLSL寫頂點著色器，然後編譯這個著色器，這樣我們就可以在應用中使用它了。下面你會看到一個非常基礎的頂點著色器的源代碼，它就是使用GLSL寫的：

```c++
#version 330 core
  
layout (location = 0) in vec3 position;

void main()
{
    gl_Position = vec4(position.x, position.y, position.z, 1.0);
}
```

就像你所看到的那樣，GLSL看起來很像C語言。每個著色器都起始於一個版本聲明。這是因為OpenGL 3.3和更高的GLSL版本號要去匹配OpenGL的版本(GLSL420版本對應於OpenGL 4.2)。我們同樣顯式地表示我們會用核心模式(Core-profile)。

下一步，我們在頂點著色器中聲明所有的輸入頂點屬性，使用in關鍵字。現在我們只關心位置(Position)數據，所以我們只需要一個頂點屬性(Attribute)。GLSL有一個向量數據類型，它包含1到4個`float`元素，包含的數量可以從它的後綴看出來。由於每個頂點都有一個3D座標，我們就創建一個`vec3`輸入變量來表示位置(Position)。我們同樣也指定輸入變量的位置值(Location)，這是用`layout (location = 0)`來完成的，你後面會看到為什麼我們會需要這個位置值。

!!! Important

	**向量(Vector)**

	在圖形編程中我們經常會使用向量這個數學概念，因為它簡明地表達了任意空間中位置和方向，二者是有用的數學屬性。在GLSL中一個向量有最多4個元素，每個元素值都可以從各自代表一個空間座標的`vec.x`、`vec.y`、`vec.z`和`vec.w`來獲取到。注意`vec.w`元素不是用作表達空間中的位置的(我們處理的是3D不是4D)而是用在所謂透視劃分(Perspective Division)上。我們會在後面的教程中更詳細地討論向量。

為了設置頂點著色器的輸出，我們必須把位置數據賦值給預定義的`gl_Position`變量，這個位置數據是一個`vec4`類型的。在main函數的最後，無論我們給`gl_Position`設置成什麼，它都會成為著色器的輸出。由於我們的輸入是一個3元素的向量，我們必須把它轉換為4元素。我們可以通過把`vec3`數據作為`vec4`初始化構造器的參數，同時把`w`元素設置為`1.0f`(我們會在後面解釋為什麼)。

這個頂點著色器可能是能想到的最簡單的了，因為我們什麼都沒有處理就把輸入數據輸出了。在真實的應用裡輸入數據通常都沒有在標準化設備座標中，所以我們首先就必須把它們放進OpenGL的可視區域內。

## 編譯一個著色器

我們已經寫了一個頂點著色器源碼，但是為了OpenGL能夠使用它，我們必須在運行時動態編譯它的源碼。

我們要做的第一件事是創建一個著色器對象，再次引用它的ID。所以我們儲存這個頂點著色器為`GLuint`，然後用`glCreateShader`創建著色器：

```c++
GLuint vertexShader;
vertexShader = glCreateShader(GL_VERTEX_SHADER);
```

我們把著色器的類型提供`glCreateShader`作為它的參數。這裡我們傳遞的參數是`GL_VERTEX_SHADER`這樣就創建了一個頂點著色器。

下一步我們把這個著色器源碼附加到著色器對象，然後編譯它：

```c++
glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
glCompileShader(vertexShader);
```

`glShaderSource`函數把著色器對象作為第一個參數來編譯它。第二參數指定了源碼中有多少個**字符串**，這裡只有一個。第三個參數是頂點著色器真正的源碼，我們可以把第四個參數先設置為`NULL`。

!!! Important

	你可能會希望檢測調用`glCompileShader`後是否編譯成功了，是否要去修正錯誤。檢測編譯時錯誤的方法是：
	
		GLint success;
		GLchar infoLog[512];
		glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);

	首先我們定義一個整型來表示是否成功編譯，還需要一個儲存錯誤消息的容器(如果有的話)。然後我們用`glGetShaderiv`檢查是否編譯成功了。如果編譯失敗，我們應該用`glGetShaderInfoLog`獲取錯誤消息，然後打印它。

		if(!success)
		{
	    	glGetShaderInfoLog(vertexShader, 512, NULL, infoLog);
	    	std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
		}

如果編譯的時候沒有任何錯誤，頂點著色器就被編譯成功了。

## 片段著色器(Fragment Shader)

片段著色器是第二個也是最終我們打算創建的用於渲染三角形的著色器。片段著色器的全部，都是用來計算你的像素的最後顏色輸出。為了讓事情比較簡單，我們的片段著色器只輸出橘黃色。

!!! Important

	在計算機圖形中顏色被表示為有4個元素的數組：紅色、綠色、藍色和alpha(透明度)元素，通常縮寫為RGBA。當定義一個OpenGL或GLSL的顏色的時候，我們就把每個顏色的強度設置在0.0到1.0之間。比如，我們設置紅色為1.0f，綠色為1.0f，這樣這個混合色就是黃色了。這三種顏色元素的不同調配可以生成1600萬不同顏色！

```c++
#version 330 core

out vec4 color;

void main()
{
    color = vec4(1.0f, 0.5f, 0.2f, 1.0f);
}
```

片段著色器只需要一個輸出變量，這個變量是一個4元素表示的最終輸出顏色的向量，我們可以自己計算出來。我們可以用`out`關鍵字聲明輸出變量，這裡我們命名為`color`。下面，我們簡單的把一個帶有alpha值為1.0(1.0代表完全不透明)的橘黃的`vec4`賦值給`color`作為輸出。

編譯片段著色器的過程與頂點著色器相似，儘管這次我們使用`GL_FRAGMENT_SHADER`作為著色器類型：

```c++
GLuint fragmentShader;
fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
glShaderSource(fragmentShader, 1, &fragmentShaderSource, null);
glCompileShader(fragmentShader);
```

每個著色器現在都編譯了，剩下的事情是把兩個著色器對象鏈接到一個著色器程序中(Shader Program)，它是用來渲染的。

### 著色器程序

著色器程序對象(Shader Program Object)是多個著色器最後鏈接的版本。如果要使用剛才編譯的著色器我們必須把它們鏈接為一個著色器程序對象，然後當渲染物體的時候激活這個著色器程序。激活了的著色器程序的著色器，在調用渲染函數時才可用。

把著色器鏈接為一個程序就等於把每個著色器的輸出鏈接到下一個著色器的輸入。如果你的輸出和輸入不匹配那麼就會得到一個鏈接錯誤。

創建一個程序對象很簡單：

```c++
GLuint shaderProgram;
shaderProgram = glCreateProgram();
```

`glCreateProgram`函數創建一個程序，返回新創建的程序對象的ID引用。現在我們需要把前面編譯的著色器附加到程序對象上，然後用`glLinkProgram`鏈接它們：

```c++
glAttachShader(shaderProgram, vertexShader);
glAttachShader(shaderProgram, fragmentShader);
glLinkProgram(shaderProgram);
```

代碼不言自明，我們把著色器附加到程序上，然後用`glLinkProgram`鏈接。

!!! Important

	就像著色器的編譯一樣，我們也可以檢驗鏈接著色器程序是否失敗，獲得相應的日誌。與glGetShaderiv和glGetShaderInfoLog不同，現在我們使用：

		glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
		if(!success) {
	    	glGetProgramInfoLog(shaderProgram, 512, NULL, infoLog);
	  	  ...
		}

我們可以調用`glUseProgram`函數，用新創建的程序對象作為它的參數，這樣就能激活這個程序對象：

```c++
glUseProgram(shaderProgram);
```

現在在`glUseProgram`函數調用之後的每個著色器和渲染函數都會用到這個程序對象(當然還有這些鏈接的著色器)了。

在我們把著色器對象鏈接到程序對象以後，不要忘記刪除著色器對象；我們不再需要它們了：

```c++
glDeleteShader(vertexShader);
glDeleteShader(fragmentShader);
```

現在，我們把輸入頂點數據發送給GPU，指示GPU如何在頂點和片段著色器中處理它。還沒結束，OpenGL還不知道如何解釋內存中的頂點數據，以及怎樣把頂點數據鏈接到頂點著色器的屬性上。我們需要告訴OpenGL怎麼做。

## 鏈接頂點屬性

頂點著色器允許我們以任何我們想要的形式作為頂點屬性(Vertex Attribute)的輸入，同樣它也具有很強的靈活性，這意味著我們必須手動指定我們的輸入數據的哪一個部分對應頂點著色器的哪一個頂點屬性。這意味著我們必須在渲染前指定OpenGL如何解釋頂點數據。

我們的頂點緩衝數據被格式化為下面的形式：

![](http://learnopengl.com/img/getting-started/vertex_attribute_pointer.png)

- 位置數據被儲存為32-bit(4 byte)浮點值。
- 每個位置包含3個這樣的值。
- 在這3個值之間沒有空隙(或其他值)。這幾個值緊密排列為一個數組。
- 數據中第一個值是緩衝的開始位置。

有了這些信息我們就可以告訴OpenGL如何解釋頂點數據了(每一個頂點屬性)，我們使用`glVertexAttribPointer`這個函數：

```c++
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid*)0);
glEnableVertexAttribArray(0);
```

`glVertexAttribPointer`函數有很多參數，所以我們仔細來了解它們：

- 第一個參數指定我們要配置哪一個頂點屬性。記住，我們在頂點著色器中使用`layout(location = 0)`定義了頂點屬性——位置(Position)的位置值(Location)。這樣要把頂點屬性的位置值(Location)設置為0，因為我們希望把數據傳遞到這個頂點屬性中，所以我們在這裡填0。
- 第二個參數指定頂點屬性的大小。頂點屬性是`vec3`類型，它由3個數值組成。
- 第三個參數指定數據的類型，這裡是`GL_FLOAT`(GLSL中`vec*`是由浮點數組成的)。
- 下個參數定義我們是否希望數據被標準化。如果我們設置為`GL_TRUE`，所有數據都會被映射到0(對於有符號型signed數據是-1)到1之間。我們把它設置為`GL_FALSE`。
- 第五個參數叫做步長(Stride)，它告訴我們在連續的頂點屬性之間間隔有多少。由於下個位置數據在3個`GLfloat`後面的位置，我們把步長設置為`3 * sizeof(GLfloat)`。要注意的是由於我們知道這個數組是緊密排列的(在兩個頂點屬性之間沒有空隙)我們也可以設置為0來讓OpenGL決定具體步長是多少(只有當數值是緊密排列時才可用)。每當我們有更多的頂點屬性，我們就必須小心地定義每個頂點屬性之間的空間，我們在後面會看到更多的例子(譯註: 這個參數的意思簡單說就是從這個屬性第二次出現的地方到整個數組0位置之間有多少字節)。
- 最後一個參數有奇怪的`GLvoid*`的強制類型轉換。它表示我們的位置數據在緩衝中起始位置的偏移量。由於位置數據是數組的開始，所以這裡是0。我們會在後面詳細解釋這個參數。

!!! Important

	每個頂點屬性從VBO管理的內存中獲得它的數據，它所獲取數據的那個VBO，就是當調用`glVetexAttribPointer`的時候，最近綁定到`GL_ARRAY_BUFFER`的那個VBO。由於在調用`glVertexAttribPointer`之前綁定了VBO，頂點屬性0現在鏈接到了它的頂點數據。

現在我們定義OpenGL如何解釋頂點數據，我們也要開啟頂點屬性，使用`glEnableVertexAttribArray`，把頂點屬性位置值作為它的參數；頂點屬性默認是關閉的。自此，我們把每件事都做好了：我們使用一個頂點緩衝對象初始化了一個緩衝中的頂點數據，設置了一個頂點和片段著色器，告訴了OpenGL如何把頂點數據鏈接到頂點著色器的頂點屬性上。在OpenGL繪製一個物體，看起來會像是這樣：

```c++
// 0. 複製頂點數組到緩衝中提供給OpenGL使用
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 1. 設置頂點屬性指針
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid*)0);
glEnableVertexAttribArray(0);  
// 2. 當我們打算渲染一個物體時要使用著色器程序
glUseProgram(shaderProgram);
// 3. 繪製物體
someOpenGLFunctionThatDrawsOurTriangle();
```

我們繪製一個物體的時候必須重複這件事。這看起來也不多，但是如果有超過5個頂點屬性，100多個不同物體呢(這其實並不罕見)。綁定合適的緩衝對象，為每個物體配置所有頂點屬性很快就變成一件麻煩事。有沒有一些方法可以使我們把所有的配置儲存在一個對象中，並且可以通過綁定這個對象來恢復狀態？

### 頂點數組對象(Vertex Array Object, VAO)

**頂點數組對象(Vertex Array Object, VAO)**可以像頂點緩衝對象一樣綁定，任何隨後的頂點屬性調用都會儲存在這個VAO中。這有一個好處，當配置頂點屬性指針時，你只用做一次，每次繪製一個物體的時候，我們綁定相應VAO就行了。切換不同頂點數據和屬性配置就像綁定一個不同的VAO一樣簡單。所有狀態我們都放到了VAO裡。

!!! Attention

	OpenGL核心模式版要求我們使用VAO，這樣它就能知道對我們的頂點輸入做些什麼。如果我們綁定VAO失敗，OpenGL會拒絕繪製任何東西。

一個頂點數組對象儲存下面的內容：

- 調用`glEnableVertexAttribArray`和`glDisableVertexAttribArray`。
- 使用`glVertexAttribPointer`的頂點屬性配置。
- 使用`glVertexAttribPointer`進行的頂點緩衝對象與頂點屬性鏈接。

![](http://learnopengl.com/img/getting-started/vertex_array_objects.png)

生成一個VAO和生成VBO類似：

```c++
GLuint VAO;
glGenVertexArrays(1, &VAO);  
```

使用VAO要做的全部就是使用`glBindVertexArray`綁定VAO。自此我們就應該綁定/配置相應的VBO和屬性指針，然後解綁VAO以備後用。當我們打算繪製一個物體的時候，我們只要在繪製物體前簡單地把VAO綁定到希望用到的配置就行了。這段代碼應該看起來像這樣：

```c++
// ..:: 初始化代碼 (一次完成 (除非你的物體頻繁改變)) :: ..
 
// 1. 綁定VAO
glBindVertexArray(VAO);
 
// 2. 把頂點數組複製到緩衝中提供給OpenGL使用
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
 
// 3. 設置頂點屬性指針
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid * )0);
glEnableVertexAttribArray(0);
 
//4. 解綁VAO
glBindVertexArray(0);
 
[...]
 
// ..:: 繪製代碼 (in Game loop) :: ..
 
// 5. 繪製物體
glUseProgram(shaderProgram);
glBindVertexArray(VAO);
someOpenGLFunctionThatDrawsOurTriangle();
glBindVertexArray(0);
```

!!! Attention

	通常情況下當我們配置好它們以後要解綁OpenGL對象，這樣我們才不會在某處錯誤地配置它們。

就是現在！前面做的一切都是等待這一刻，我們已經把我們的頂點屬性配置和打算使用的VBO儲存到一個VAO中。一般當你有多個物體打算繪製時，你首先要生成/配置所有的VAO(它需要VBO和屬性指針)，然後儲存它們準備後面使用。當我們打算繪製物體的時候就拿出相應的VAO，綁定它，繪製完物體後，再解綁VAO。

### 我們一直期待的三角形

OpenGL的`glDrawArrays`函數為我們提供了繪製物體的能力，它使用當前激活的著色器、前面定義的頂點屬性配置和VBO的頂點數據(通過VAO間接綁定)來繪製基本圖形。

```c++
glUseProgram(shaderProgram);
glBindVertexArray(VAO);
glDrawArrays(GL_TRIANGLES, 0, 3);
glBindVertexArray(0);  
```

`glDrawArrays`函數第一個參數是我們打算繪製的OpenGL基本圖形的類型。由於我們在一開始時說過，我們希望繪製三角形，我們傳遞`GL_TRIANGLES`給它。第二個參數定義了我們打算繪製的那個頂點數組的起始位置的索引；我們這裡填0。最後一個參數指定我們打算繪製多少個頂點，這裡是3(我們只從我們的數據渲染一個三角形，它只有3個頂點)。

現在嘗試編譯代碼，如果彈出了任何錯誤，回頭檢查你的代碼。如果你編譯通過了，你應該看到下面的結果：

![](http://learnopengl.com/img/getting-started/hellotriangle.png)

完整的程序源碼可以在[這裡](http://learnopengl.com/code_viewer.php?code=getting-started/hellotriangle)找到。

如果你的輸出和這個不一樣，你可能做錯了什麼，去看源碼，看看是否遺漏了什麼東西或者在評論部分提問。

## 索引緩衝對象(Element Buffer Objects，EBO)

這是我們最後一件在渲染頂點這個問題上要討論的事——索引緩衝對象簡稱EBO(或IBO)。解釋索引緩衝對象的工作方式最好是舉例子：假設我們不再繪製一個三角形而是矩形。我們就可以繪製兩個三角形來組成一個矩形(OpenGL主要就是繪製三角形)。這會生成下面的頂點的集合：

```c++
GLfloat vertices[] = {
 
    // 第一個三角形
    0.5f, 0.5f, 0.0f,   // 右上角
    0.5f, -0.5f, 0.0f,  // 右下角
    -0.5f, 0.5f, 0.0f,  // 左上角
 
    // 第二個三角形
    0.5f, -0.5f, 0.0f,  // 右下角
    -0.5f, -0.5f, 0.0f, // 左下角
    -0.5f, 0.5f, 0.0f   // 左上角
};
```

就像你所看到的那樣，有幾個頂點疊加了。我們指定右下角和左上角兩次！一個矩形只有4個而不是6個頂點，這樣就產生50%的額外開銷。當我們有超過1000個三角的模型這個問題會更糟糕，這會產生一大堆浪費。最好的解決方案就是每個頂點只儲存一次，當我們打算繪製這些頂點的時候只調用頂點的索引。這種情況我們只要儲存4個頂點就能繪製矩形了，我們只要指定我們打算繪製的那個頂點的索引就行了。如果OpenGL提供這個功能就好了，對吧？

很幸運，索引緩衝的工作方式正是這樣的。一個EBO是一個像頂點緩衝對象(VBO)一樣的緩衝，它專門儲存索引，OpenGL調用這些頂點的索引來繪製。索引繪製正是這個問題的解決方案。我們先要定義(獨一無二的)頂點，和繪製出矩形的索引：

```c++
GLfloat vertices[] = {
 
    0.5f, 0.5f, 0.0f,   // 右上角
    0.5f, -0.5f, 0.0f,  // 右下角
    -0.5f, -0.5f, 0.0f, // 左下角
    -0.5f, 0.5f, 0.0f   // 左上角
};
 
GLuint indices[] = { // 起始於0!
 
    0, 1, 3, // 第一個三角形
    1, 2, 3  // 第二個三角形
};
```

你可以看到，當時用索引的時候，我們只定義了4個頂點，而不是6個。下一步我們需要創建索引緩衝對象：

```c++
GLuint EBO;
glGenBuffers(1, &EBO);
```

與VBO相似，我們綁定EBO然後用`glBufferData`把索引複製到緩衝裡。同樣，和VBO相似，我們會把這些函數調用放在綁定和解綁函數調用之間，這次我們把緩衝的類型定義為`GL_ELEMENT_ARRAY_BUFFER`。

```c++
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW); 
```

要注意的是，我們現在用`GL_ELEMENT_ARRAY_BUFFER`當作緩衝目標。最後一件要做的事是用`glDrawElements`來替換`glDrawArrays`函數，來指明我們從索引緩衝渲染。當時用`glDrawElements`的時候，我們就會用當前綁定的索引緩衝進行繪製：

```c++
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
```

第一個參數指定了我們繪製的模式，這個和`glDrawArrays`的一樣。第二個參數是我們打算繪製頂點的次數。我們填6，說明我們總共想繪製6個頂點。第三個參數是索引的類型，這裡是`GL_UNSIGNED_INT`。最後一個參數裡我們可以指定EBO中的偏移量(或者傳遞一個索引數組，但是這只是當你不是在使用索引緩衝對象的時候)，但是我們只打算在這裡填寫0。

`glDrawElements`函數從當前綁定到`GL_ELEMENT_ARRAY_BUFFER`目標的EBO獲取索引。這意味著我們必須在每次要用索引渲染一個物體時綁定相應的EBO，這還是有點麻煩。不過頂點數組對象仍可以保存索引緩衝對象的綁定狀態。VAO綁定之後可以索引緩衝對象，EBO就成為了VAO的索引緩衝對象。再次綁定VAO的同時也會自動綁定EBO。

![](http://learnopengl.com/img/getting-started/vertex_array_objects_ebo.png)

!!! Attention

	當目標是`GL_ELEMENT_ARRAY_BUFFER`的時候，VAO儲存了`glBindBuffer`的函數調用。這也意味著它也會儲存解綁調用，所以確保你沒有在解綁VAO之前解綁索引數組緩衝，否則就沒有這個EBO配置了。

最後的初始化和繪製代碼現在看起來像這樣：

```c++
// ..:: 初始化代碼 :: ..
// 1. 綁定VAO
glBindVertexArray(VAO);
 
// 2. 把我們的頂點數組複製到一個頂點緩衝中，提供給OpenGL使用
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
 
// 3. 複製我們的索引數組到一個索引緩衝中，提供給OpenGL使用
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices),indices, GL_STATIC_DRAW);
 
// 3. 設置頂點屬性指針
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid * )0);
glEnableVertexAttribArray(0);
 
// 4. 解綁VAO，不解綁EBO(譯註：解綁緩衝相當於沒有綁定緩衝，可以在解綁VAO之後解綁緩衝)
glBindVertexArray(0);
 
[...]
 
// ..:: 繪製代碼(在遊戲循環中) :: ..
 
glUseProgram(shaderProgram);
glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0)
glBindVertexArray(0);
```

運行程序會獲得下面這樣的圖片的結果。左側圖片看起來很熟悉，而右側的則是使用線框模式(Wireframe Mode)繪製的。線框矩形可以顯示出矩形的確是由兩個三角形組成的。

![](http://learnopengl.com/img/getting-started/hellotriangle2.png)

!!! Important

	**線框模式(Wireframe Mode)**

	如果用線框模式繪製你的三角，你可以配置OpenGL繪製用的基本圖形，調用`glPolygonMode(GL_FRONT_AND_BACK, GL_LINE)`。第一個參數說：我們打算應用到所有的三角形的前面和背面，第二個參數告訴我們用線來繪製。在隨後的繪製函數調用後會一直以線框模式繪製三角形，直到我們用`glPolygonMode(GL_FRONT_AND_BACK, GL_FILL)`設置回了默認模式。

如果你遇到任何錯誤，回頭檢查代碼，看看是否遺漏了什麼。同時，你可以[在這裡獲得全部源碼](http://learnopengl.com/code_viewer.php?code=getting-started/hellotriangle2)，也可以在評論區自由提問。

如果你繪製出了這個三角形或矩形，那麼恭喜你，你成功地通過了現代OpenGL最難部分之一：繪製你自己的第一個三角形。這部分很難，因為在可以繪製第一個三角形之前需要很多知識。幸運的是我們現在已經越過了這個障礙，接下來的教程會比較容易理解一些。

## 附加資源

- [antongerdelan.net/hellotriangle](http://antongerdelan.net/opengl/hellotriangle.html): 一個渲染第一個三角形的教程。
- [open.gl/drawing](https://open.gl/drawing): Alexander Overvoorde的關於渲染第一個三角形的教程。
- [antongerdelan.net/vertexbuffers](http://antongerdelan.net/opengl/vertexbuffers.html): 頂點緩衝對象的一些深入探討。

# 練習

為了更好的理解討論的那些概念最好做點練習。建議在繼續下面的主題之前做完這些練習，確保你對這些有比較好的理解。

- 嘗試使用`glDrawArrays`以在你的數據中添加更多頂點的方式，繪製兩個彼此相連的三角形：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/hello-triangle-exercise1)
- 現在，使用不同的VAO(和VBO)創建同樣的2個三角形，每個三角形的數據要不同(提示：創建2個頂點數據數組，而不是1個)：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/hello-triangle-exercise2)
- 創建連個著色器程序(Shader Program)，第二個程序使用不同的片段著色器，它輸出黃色；繪製這兩個三角形，其中一個輸出為黃色：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/hello-triangle-exercise3)