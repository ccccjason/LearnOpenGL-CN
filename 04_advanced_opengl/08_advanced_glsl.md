# 高級GLSL

原文     | [Advanced GLSL](http://learnopengl.com/#!Advanced-OpenGL/Advanced-GLSL)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)


這章不會向你展示什麼新的功能，也不會對你的場景的視覺效果有較大提升。本文多多少少地深入探討了一些GLSL有趣的知識，它們可能在將來能幫助你。基本來說有些不可不知的內容和功能在你去使用GLSL創建OpenGL應用的時候能讓你的生活更輕鬆。

我們會討論一些內建變量、組織著色器輸入和輸出的新方式以及一個叫做uniform緩衝對象的非常有用的工具。

## GLSL的內建變量

著色器是很小的，如果我們需要從當前著色器以外的別的資源裡的數據，那麼我們就不得不傳給它。我們學過了使用頂點屬性、uniform和採樣器可以實現這個目標。GLSL有幾個以**gl\_**為前綴的變量，使我們有一個額外的手段來獲取和寫入數據。其中兩個我們已經打過交道了：`gl_Position`和`gl_FragCoord`，前一個是頂點著色器的輸出向量，後一個是片段著色器的變量。

我們會討論幾個有趣的GLSL內建變量，並向你解釋為什麼它們對我們來說很有好處。注意，我們不會討論到GLSL中所有的內建變量，因此如果你想看到所有的內建變量還是最好去查看[OpenGL的wiki](http://www.opengl.org/wiki/Built-in_Variable_(GLSL)。

### 頂點著色器變量

#### gl_Position

我們已經瞭解`gl_Position`是頂點著色器裁切空間輸出的位置向量。如果你想讓屏幕上渲染出東西`gl_Position`必須使用。否則我們什麼都看不到。

#### gl_PointSize

我們可以使用的另一個可用於渲染的基本圖形(primitive)是**GL\_POINTS**，使用它每個頂點作為一個基本圖形，被渲染為一個點(point)。可以使用`glPointSize`函數來設置這個點的大小，但我們還可以在頂點著色器裡修改點的大小。

GLSL有另一個輸出變量叫做`gl_PointSize`，他是一個`float`變量，你可以以像素的方式設置點的高度和寬度。它在著色器中描述每個頂點做為點被繪製出來的大小。

在著色器中影響點的大小默認是關閉的，但是如果你打算開啟它，你需要開啟OpenGL的`GL_PROGRAM_POINT_SIZE`：

```c++
glEnable(GL_PROGRAM_POINT_SIZE);
```

把點的大小設置為裁切空間的z值，這樣點的大小就等於頂點距離觀察者的距離，這是一種影響點的大小的方式。當頂點距離觀察者更遠的時候，它就會變得更大。

```c++
void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
    gl_PointSize = gl_Position.z;
}
```

結果是我們繪製的點距離我們越遠就越大：

![](http://learnopengl.com/img/advanced/advanced_glsl_pointsize.png)

想象一下，每個頂點表示出來的點的大小的不同，如果用在像粒子生成之類的技術裡會挺有意思的。

 #### gl_VertexID

`gl_Position`和`gl_PointSize`都是輸出變量，因為它們的值是作為頂點著色器的輸出被讀取的；我們可以向它們寫入數據來影響結果。頂點著色器為我們提供了一個有趣的輸入變量，我們只能從它那裡讀取，這個變量叫做`gl_VertexID`。

`gl_VertexID`是個整型變量，它儲存著我們繪製的當前頂點的ID。當進行索引渲染（indexed rendering，使用`glDrawElements`渲染）時，這個變量保存著當前繪製的頂點的索引。當用的不是索引繪製（`glDrawArrays`）時，這個變量保存的是從渲染開始起直到當前處理的這個頂點的（當前頂點）編號。

儘管目前看似沒用，但是我們最好知道我們能獲取這樣的信息。

### 片段著色器的變量

在片段著色器中也有一些有趣的變量。GLSL給我們提供了兩個有意思的輸入變量，它們是`gl_FragCoord`和`gl_FrontFacing`。

#### gl_FragCoord

在討論深度測試的時候，我們已經看過`gl_FragCoord`好幾次了，因為`gl_FragCoord`向量的z元素和特定的fragment的深度值相等。然而，我們也可以使用這個向量的x和y元素來實現一些有趣的效果。

`gl_FragCoord`的x和y元素是當前片段的窗口空間座標（window-space coordinate）。它們的起始處是窗口的左下角。如果我們的窗口是800×600的，那麼一個片段的窗口空間座標x的範圍就在0到800之間，y在0到600之間。

我們可以使用片段著色器基於片段的窗口座標計算出一個不同的顏色。`gl_FragCoord`變量的一個常用的方式是與一個不同的片段計算出來的視頻輸出進行對比，通常在技術演示中常見。比如我們可以把屏幕分為兩個部分，窗口的左側渲染一個輸出，窗口的右邊渲染另一個輸出。下面是一個基於片段的窗口座標的位置的不同輸出不同的顏色的片段著色器：


```c++
void main()
{
    if(gl_FragCoord.x < 400)
        color = vec4(1.0f, 0.0f, 0.0f, 1.0f);
    else
        color = vec4(0.0f, 1.0f, 0.0f, 1.0f);
}
```

因為窗口的寬是800，當一個像素的x座標小於400，那麼它一定在窗口的左邊，這樣我們就讓物體有個不同的顏色。

![](http://learnopengl.com/img/advanced/advanced_glsl_frontfacing.png)

我們現在可以計算出兩個完全不同的片段著色器結果，每個顯示在窗口的一端。這對於測試不同的光照技術很有好處。



#### gl_FrontFacing

片段著色器另一個有意思的輸入變量是`gl_FrontFacing`變量。在面剔除教程中，我們提到過OpenGL可以根據頂點繪製順序弄清楚一個面是正面還是背面。如果我們不適用面剔除，那麼`gl_FrontFacing`變量能告訴我們當前片段是某個正面的一部分還是背面的一部分。然後我們可以決定做一些事情，比如為正面計算出不同的顏色。

`gl_FrontFacing`變量是一個布爾值，如果當前片段是正面的一部分那麼就是true，否則就是false。這樣我們可以創建一個立方體，裡面和外面使用不同的紋理：

```c++
#version 330 core
out vec4 color;
in vec2 TexCoords;

uniform sampler2D frontTexture;
uniform sampler2D backTexture;

void main()
{
    if(gl_FrontFacing)
        color = texture(frontTexture, TexCoords);
    else
        color = texture(backTexture, TexCoords);
}
```

如果我們從箱子的一角往裡看，就能看到裡面用的是另一個紋理。

![](http://bullteacher.com/wp-content/uploads/2015/06/advanced_glsl_frontfacing.png)

注意，如果你開啟了面剔除，你就看不到箱子裡面有任何東西了，所以此時使用`gl_FrontFacing`毫無意義。

#### gl_FragDepth

輸入變量`gl_FragCoord`讓我們可以讀得當前片段的窗口空間座標和深度值，但是它是隻讀的。我們不能影響到這個片段的窗口屏幕座標，但是可以設置這個像素的深度值。GLSL給我們提供了一個叫做`gl_FragDepth`的變量，我們可以用它在著色器中遂舍之像素的深度值。

為了在著色器中設置深度值，我們簡單的寫一個0.0到1.0之間的float數，賦值給這個輸出變量：

```c++
gl_FragDepth = 0.0f; //現在片段的深度值被設為0
```

如果著色器中沒有像`gl_FragDepth`變量寫入，它就會自動採用`gl_FragCoord.z`的值。

我們自己設置深度值有一個顯著缺點，因為只要我們在片段著色器中對`gl_FragDepth`寫入什麼，OpenGL就會關閉所有的前置深度測試。它被關閉的原因是，在我們運行片段著色器之前OpenGL搞不清像素的深度值，因為片段著色器可能會完全改變這個深度值。

因此，你需要考慮到`gl_FragDepth`寫入所帶來的性能的下降。然而從OpenGL4.2起，我們仍然可以對二者進行一定的調和，這需要在片段著色器的頂部使用深度條件（depth condition）來重新聲明`gl_FragDepth`：

```c++
layout (depth_<condition>) out float gl_FragDepth;
```

condition可以使用下面的值：

Condition	| 描述
         ---|---
any	        | 默認值. 前置深度測試是關閉的，你失去了很多性能表現
greater	    |深度值只能比gl_FragCoord.z大
less	    |深度值只能設置得比gl_FragCoord.z小
unchanged	|如果寫入gl_FragDepth, 你就會寫gl_FragCoord.z

如果把深度條件定義為greater或less，OpenGL會假定你只寫入比當前的像素深度值的深度值大或小的。

下面是一個在片段著色器裡增加深度值的例子，不過仍可開啟前置深度測試：

```c++
#version 330 core
layout (depth_greater) out float gl_FragDepth;
out vec4 color;

void main()
{
    color = vec4(1.0f);
    gl_FragDepth = gl_FragCoord.z + 0.1f;
}
```

一定要記住這個功能只在OpenGL4.2以上版本才有。



## 接口塊(Interface blocks)

到目前位置，每次我們打算從頂點向片段著色器發送數據，我們都會聲明一個相互匹配的輸出/輸入變量。從一個著色器向另一個著色器發送數據，一次將它們聲明好是最簡單的方式，但是隨著應用變得越來越大，你也許會打算髮送的不僅僅是變量，最好還可以包括數組和結構體。

為了幫助我們組織這些變量，GLSL為我們提供了一些叫做接口塊(Interface blocks)的東西，好讓我們能夠組織這些變量。聲明接口塊和聲明struct有點像，不同之處是它現在基於塊（block），使用in和out關鍵字來聲明，最後它將成為一個輸入或輸出塊（block）。

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec2 texCoords;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

out VS_OUT
{
    vec2 TexCoords;
} vs_out;

void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
    vs_out.TexCoords = texCoords;
}
```

這次我們聲明一個叫做vs_out的接口塊，它把我們需要發送給下個階段著色器的所有輸出變量組合起來。雖然這是一個微不足道的例子，但是你可以想象一下，它的確能夠幫助我們組織著色器的輸入和輸出。當我們希望把著色器的輸入和輸出組織成數組的時候它就變得很有用，我們會在下節幾何著色器(geometry)中見到。

然後，我們還需要在下一個著色器——片段著色器中聲明一個輸入interface block。塊名（block name）應該是一樣的，但是實例名可以是任意的。

```c++
#version 330 core
out vec4 color;

in VS_OUT
{
    vec2 TexCoords;
} fs_in;

uniform sampler2D texture;

void main()
{
    color = texture(texture, fs_in.TexCoords);
}
```

如果兩個interface block名一致，它們對應的輸入和輸出就會匹配起來。這是另一個可以幫助我們組織代碼的有用功能，特別是在跨著色階段的情況，比如幾何著色器。

## uniform緩衝對象 (Uniform buffer objects)

我們使用OpenGL很長時間了，也學到了一些很酷的技巧，但是產生了一些煩惱。比如說，當時用一個以上的著色器的時候，我們必須一次次設置uniform變量，儘管對於每個著色器來說它們都是一樣的，所以為什麼還麻煩地多次設置它們呢？

OpenGL為我們提供了一個叫做uniform緩衝對象的工具，使我們能夠聲明一系列的全局uniform變量， 它們會在幾個著色器程序中保持一致。當時用uniform緩衝的對象時相關的uniform只能設置一次。我們仍需為每個著色器手工設置唯一的uniform。創建和配置一個uniform緩衝對象需要費點功夫。

因為uniform緩衝對象是一個緩衝，因此我們可以使用`glGenBuffers`創建一個，然後綁定到`GL_UNIFORM_BUFFER`緩衝目標上，然後把所有相關uniform數據存入緩衝。有一些原則，像uniform緩衝對象如何儲存數據，我們會在稍後討論。首先我們我們在一個簡單的頂點著色器中，用uniform塊(uniform block)儲存投影和視圖矩陣：

```c++
#version 330 core
layout (location = 0) in vec3 position;

layout (std140) uniform Matrices
{
    mat4 projection;
    mat4 view;
};

uniform mat4 model;

void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0);
}
```

前面，大多數例子裡我們在每次渲染迭代，都為projection和view矩陣設置uniform。這個例子裡使用了uniform緩衝對象，這非常有用，因為這些矩陣我們設置一次就行了。

在這裡我們聲明瞭一個叫做Matrices的uniform塊，它儲存兩個4×4矩陣。在uniform塊中的變量可以直接獲取，而不用使用block名作為前綴。接著我們在緩衝中儲存這些矩陣的值，每個聲明瞭這個uniform塊的著色器都能夠獲取矩陣。

現在你可能會奇怪layout(std140)是什麼意思。它的意思是說當前定義的uniform塊為它的內容使用特定的內存佈局，這個聲明實際上是設置uniform塊佈局(uniform block layout)。



### uniform塊佈局(uniform block layout)

一個uniform塊的內容被儲存到一個緩衝對象中，實際上就是在一塊內存中。因為這塊內存也不清楚它保存著什麼類型的數據，我們就必須告訴OpenGL哪一塊內存對應著色器中哪一個uniform變量。

假想下面的uniform塊在一個著色器中：

```c++
layout (std140) uniform ExampleBlock
{
    float value;
    vec3 vector;
    mat4 matrix;
    float values[3];
    bool boolean;
    int integer;
};
```

我們所希望知道的是每個變量的大小（以字節為單位）和偏移量（從block的起始處），所以我們可以以各自的順序把它們放進一個緩衝裡。每個元素的大小在OpenGL中都很清楚，直接與C++數據類型呼應，向量和矩陣是一個float序列（數組）。OpenGL沒有澄清的是變量之間的間距。這讓硬件能以它認為合適的位置方式變量。比如有些硬件可以在float旁邊放置一個vec3。不是所有硬件都能這樣做，在vec3旁邊附加一個float之前，給vec3加一個邊距使之成為4個（空間連續的）float數組。功能很好，但對於我們來說用起來不方便。

GLSL 默認使用的uniform內存佈局叫做共享佈局(shared layout)，叫共享是因為一旦偏移量被硬件定義，它們就會持續地被多個程序所共享。使用共享佈局，GLSL可以為了優化而重新放置uniform變量，只要變量的順序保持完整。因為我們不知道每個uniform變量的偏移量是多少，所以我們也就不知道如何精確地填充uniform緩衝。我們可以使用像`glGetUniformIndices`這樣的函數來查詢這個信息，但是這超出了本節教程的範圍。

由於共享佈局給我們做了一些空間優化。通常在實踐中並不適用分享佈局，而是使用std140佈局。std140通過一系列的規則的規範聲明瞭它們各自的偏移量，std140佈局為每個變量類型顯式地聲明瞭內存的佈局。由於被顯式的提及，我們就可以手工算出每個變量的偏移量。

每個變量都有一個基線對齊(base alignment)，它等於在一個uniform塊中這個變量所佔的空間（包含邊距），這個基線對齊是使用std140佈局原則計算出來的。然後，我們為每個變量計算出它的對齊偏移(aligned offset)，這是一個變量從塊（block）開始處的字節偏移量。變量對齊的字節偏移一定等於它的基線對齊的倍數。

準確的佈局規則可以[在OpenGL的uniform緩衝規範](http://www.opengl.org/registry/specs/ARB/uniform_buffer_object.txt)中得到，但我們會列出最常見的規範。GLSL中每個變量類型比如int、float和bool被定義為4字節，每4字節被表示為N。

類型	    | 佈局規範
         ---|---
像int和bool這樣的標量 |	每個標量的基線為N
向量      |	每個向量的基線是2N或4N大小。這意味著vec3的基線為4N
標量與向量數組	| 每個元素的基線與vec4的相同
矩陣  | 被看做是存儲著大量向量的數組，每個元素的基數與vec4相同
結構體 | 根據以上規則計算其各個元素，並且間距必須是vec4基線的倍數

像OpenGL大多數規範一樣，舉個例子就很容易理解。再次利用之前介紹的uniform塊`ExampleBlock`，我們用std140佈局，計算它的每個成員的aligned offset（對齊偏移）：

```c++
layout (std140) uniform ExampleBlock
{
                     // base alignment ----------  // aligned offset
    float value;     // 4                          // 0
    vec3 vector;     // 16                         // 16 (必須是16的倍數，因此 4->16)
    mat4 matrix;     // 16                         // 32  (第 0 行)
                     // 16                         // 48  (第 1 行)
                     // 16                         // 64  (第 2 行)
                     // 16                         // 80  (第 3 行)
    float values[3]; // 16 (數組中的標量與vec4相同)//96 (values[0])
                     // 16                        // 112 (values[1])
                     // 16                        // 128 (values[2])
    bool boolean;    // 4                         // 144
    int integer;     // 4                         // 148
};
```

嘗試自己計算出偏移量，把它們和表格對比，你可以把這件事當作一個練習。使用計算出來的偏移量，根據std140佈局規則，我們可以用`glBufferSubData`這樣的函數，使用變量數據填充緩衝。雖然不是很高效，但std140佈局可以保證在每個程序中聲明的這個uniform塊的佈局保持一致。

在定義uniform塊前面添加layout (std140)聲明，我們就能告訴OpenGL這個uniform塊使用了std140佈局。另外還有兩種其他的佈局可以選擇，它們需要我們在填充緩衝之前查詢每個偏移量。我們已經瞭解了分享佈局（shared layout）和其他的佈局都將被封裝（packed）。當使用封裝（packed）佈局的時候，不能保證佈局在別的程序中能夠保持一致，因為它允許編譯器從uniform塊中優化出去uniform變量，這在每個著色器中都可能不同。

### 使用uniform緩衝

我們討論了uniform塊在著色器中的定義和如何定義它們的內存佈局，但是我們還沒有討論如何使用它們。

首先我們需要創建一個uniform緩衝對象，這要使用`glGenBuffers`來完成。當我們擁有了一個緩衝對象，我們就把它綁定到`GL_UNIFORM_BUFFER`目標上，調用`glBufferData`來給它分配足夠的空間。

```c++
GLuint uboExampleBlock;
glGenBuffers(1, &uboExampleBlock);
glBindBuffer(GL_UNIFORM_BUFFER, uboExampleBlock);
glBufferData(GL_UNIFORM_BUFFER, 150, NULL, GL_STATIC_DRAW); // 分配150個字節的內存空間
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```

現在任何時候當我們打算往緩衝中更新或插入數據，我們就綁定到`uboExampleBlock`上，並使用`glBufferSubData`來更新它的內存。我們只需要更新這個uniform緩衝一次，所有的使用這個緩衝著色器就都會使用它更新的數據了。但是，OpenGL是如何知道哪個uniform緩衝對應哪個uniform塊呢？

在OpenGL環境（context）中，定義了若干綁定點（binding points），在哪兒我們可以把一個uniform緩衝鏈接上去。當我們創建了一個uniform緩衝，我們把它鏈接到一個這個綁定點上，我們也把著色器中uniform塊鏈接到同一個綁定點上，這樣就把它們鏈接到一起了。下面的圖標表示了這點：

![](http://learnopengl.com/img/advanced/advanced_glsl_binding_points.png)

你可以看到，我們可以將多個uniform緩衝綁定到不同綁定點上。因為著色器A和著色器B都有一個鏈接到同一個綁定點0的uniform塊，它們的uniform塊分享同樣的uniform數據—`uboMatrices`有一個前提條件是兩個著色器必須都定義了Matrices這個uniform塊。

我們調用`glUniformBlockBinding`函數來把uniform塊設置到一個特定的綁定點上。函數的第一個參數是一個程序對象，接著是一個uniform塊索引（uniform block index）和打算鏈接的綁定點。uniform塊索引是一個著色器中定義的uniform塊的索引位置，可以調用`glGetUniformBlockIndex`來獲取這個值，這個函數接收一個程序對象和uniform塊的名字。我們可以從圖表設置Lights這個uniform塊鏈接到綁定點2：

```c++
GLuint lights_index = glGetUniformBlockIndex(shaderA.Program, "Lights");
glUniformBlockBinding(shaderA.Program, lights_index, 2);
```

注意，我們必須在每個著色器中重複做這件事。

從OpenGL4.2起，也可以在著色器中通過添加另一個佈局標識符來儲存一個uniform塊的綁定點，就不用我們調用`glGetUniformBlockIndex`和`glUniformBlockBinding`了。下面的代表顯式設置了Lights這個uniform塊的綁定點：


```c++
layout(std140, binding = 2) uniform Lights { ... };
```

然後我們還需要把uniform緩衝對象綁定到同樣的綁定點上，這個可以使用`glBindBufferBase`或`glBindBufferRange`來完成。

```c++
glBindBufferBase(GL_UNIFORM_BUFFER, 2, uboExampleBlock);
// 或者
glBindBufferRange(GL_UNIFORM_BUFFER, 2, uboExampleBlock, 0, 150);
```

函數`glBindBufferBase`接收一個目標、一個綁定點索引和一個uniform緩衝對象作為它的參數。這個函數把`uboExampleBlock`鏈接到綁定點2上面，自此綁定點所鏈接的兩端都鏈接在一起了。你還可以使用`glBindBufferRange`函數，這個函數還需要一個偏移量和大小作為參數，這樣你就可以只把一定範圍的uniform緩衝綁定到一個綁定點上了。使用`glBindBufferRage`函數，你能夠將多個不同的uniform塊鏈接到同一個uniform緩衝對象上。

現在所有事情都做好了，我們可以開始向uniform緩衝添加數據了。我們可以使用`glBufferSubData`將所有數據添加為一個單獨的字節數組或者更新緩衝的部分內容，只要我們願意。為了更新uniform變量boolean，我們可以這樣更新uniform緩衝對象：

```c++
glBindBuffer(GL_UNIFORM_BUFFER, uboExampleBlock);
GLint b = true; // GLSL中的布爾值是4個字節，因此我們將它創建為一個4字節的整數
glBufferSubData(GL_UNIFORM_BUFFER, 142, 4, &b);
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```

同樣的處理也能夠應用到uniform塊中其他uniform變量上。

### 一個簡單的例子

我們來師範一個真實的使用uniform緩衝對象的例子。如果我們回頭看看前面所有演示的代碼，我們一直使用了3個矩陣：投影、視圖和模型矩陣。所有這些矩陣中，只有模型矩陣是頻繁變化的。如果我們有多個著色器使用了這些矩陣，我們可能最好還是使用uniform緩衝對象。

我們將把投影和視圖矩陣儲存到一個uniform塊中，它被取名為Matrices。我們不打算儲存模型矩陣，因為模型矩陣會頻繁在著色器間更改，所以使用uniform緩衝對象真的不會帶來什麼好處。

```c++
#version 330 core
layout (location = 0) in vec3 position;

layout (std140) uniform Matrices
{
    mat4 projection;
    mat4 view;
};
uniform mat4 model;

void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0);
}
```

這兒沒什麼特別的，除了我們現在使用了一個帶有std140佈局的uniform塊。我們在例程中將顯示4個立方體，每個立方體都使用一個不同的著色器程序。4個著色器程序使用同樣的頂點著色器，但是它們將使用各自的片段著色器，每個片段著色器輸出一個單色。

首先，我們把頂點著色器的uniform塊設置為綁定點0。注意，我們必須為每個著色器做這件事。

```c++
GLuint uniformBlockIndexRed = glGetUniformBlockIndex(shaderRed.Program, "Matrices");
GLuint uniformBlockIndexGreen = glGetUniformBlockIndex(shaderGreen.Program, "Matrices");
GLuint uniformBlockIndexBlue = glGetUniformBlockIndex(shaderBlue.Program, "Matrices");
GLuint uniformBlockIndexYellow = glGetUniformBlockIndex(shaderYellow.Program, "Matrices");  

glUniformBlockBinding(shaderRed.Program, uniformBlockIndexRed, 0);
glUniformBlockBinding(shaderGreen.Program, uniformBlockIndexGreen, 0);
glUniformBlockBinding(shaderBlue.Program, uniformBlockIndexBlue, 0);
glUniformBlockBinding(shaderYellow.Program, uniformBlockIndexYellow, 0);
```

然後，我們創建真正的uniform緩衝對象，並把緩衝綁定到綁定點0：

```c++
GLuint uboMatrices
glGenBuffers(1, &uboMatrices);

glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferData(GL_UNIFORM_BUFFER, 2 * sizeof(glm::mat4), NULL, GL_STATIC_DRAW);
glBindBuffer(GL_UNIFORM_BUFFER, 0);

glBindBufferRange(GL_UNIFORM_BUFFER, 0, uboMatrices, 0, 2 * sizeof(glm::mat4));
```

我們先為緩衝分配足夠的內存，它等於glm::mat4的2倍。GLM的矩陣類型的大小直接對應於GLSL的mat4。然後我們把一個特定範圍的緩衝鏈接到綁定點0，這個例子中應該是整個緩衝。

現在所有要做的事只剩下填充緩衝了。如果我們把視野（ field of view）值保持為恆定的投影矩陣（這樣就不會有攝像機縮放），我們只要在程序中定義它一次就行了，這也意味著我們只需向緩衝中把它插入一次。因為我們已經在緩衝對象中分配了足夠的內存，我們可以在我們進入遊戲循環之前使用`glBufferSubData`來儲存投影矩陣：

```c++
glm::mat4 projection = glm::perspective(45.0f, (float)width/(float)height, 0.1f, 100.0f);
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferSubData(GL_UNIFORM_BUFFER, 0, sizeof(glm::mat4), glm::value_ptr(projection));
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```

這裡我們用投影矩陣儲存了uniform緩衝的前半部分。在我們在每次渲染迭代繪製物體前，我們用視圖矩陣更新緩衝的第二個部分：

```c++
glm::mat4 view = camera.GetViewMatrix();
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferSubData(
  GL_UNIFORM_BUFFER, sizeof(glm::mat4), sizeof(glm::mat4), glm::value_ptr(view));
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```

這就是uniform緩衝對象。每個包含著`Matrices`這個uniform塊的頂點著色器都將對應`uboMatrices`所儲存的數據。所以如果我們現在使用4個不同的著色器繪製4個立方體，它們的投影和視圖矩陣都是一樣的：

```c++
glBindVertexArray(cubeVAO);
shaderRed.Use();
glm::mat4 model;
model = glm::translate(model, glm::vec3(-0.75f, 0.75f, 0.0f)); // 移動到左上方
glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
glDrawArrays(GL_TRIANGLES, 0, 36);
// ... 繪製綠色立方體
// ... 繪製藍色立方體
// ... 繪製黃色立方體
glBindVertexArray(0);
```

我們只需要在去設置一個`model`的uniform即可。在一個像這樣的場景中使用uniform緩衝對象在每個著色器中可以減少uniform的調用。最後效果看起來像這樣：

![](http://learnopengl.com/img/advanced/advanced_glsl_uniform_buffer_objects.png)

通過改變模型矩陣，每個立方體都移動到窗口的一邊，由於片段著色器不同，物體的顏色也不同。這是一個相對簡單的場景，我們可以使用uniform緩衝對象，但是任何大型渲染程序有成百上千的活動著色程序，彼時uniform緩衝對象就會閃閃發光了。

你可以[在這裡獲得例程的完整源碼](http://www.learnopengl.com/code_viewer.php?code=advanced/advanced_glsl_uniform_buffer_objects)。

uniform緩衝對象比單獨的uniform有很多好處。第一，一次設置多個uniform比一次設置一個速度快。第二，如果你打算改變一個橫跨多個著色器的uniform，在uniform緩衝中只需更改一次。最後一個好處可能不是很明顯，使用uniform緩衝對象你可以在著色器中使用更多的uniform。OpenGL有一個對可使用uniform數據的數量的限制，可以用`GL_MAX_VERTEX_UNIFORM_COMPONENTS`來獲取。當使用uniform緩衝對象中，這個限制的閾限會更高。所以無論何時，你達到了uniform的最大使用數量（比如做骨骼動畫的時候），你可以使用uniform緩衝對象。
