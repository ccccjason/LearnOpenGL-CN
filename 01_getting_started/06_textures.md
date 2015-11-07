# 紋理(Textures)

原文     | [Textures](http://learnopengl.com/#!Getting-started/Textures)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | Geequlim

我們已經瞭解到，我們可以為每個頂點使用顏色來增加圖形的細節，從而創建出有趣的圖像。但是，如果想讓圖形看起來更真實我們就必須有足夠多的頂點，從而指定足夠多的顏色。這將會產生很多額外開銷，因為每個模型都會需求更多的頂點和頂點顏色。

藝術家和程序員更喜歡使用**紋理(Texture)**。紋理是一個2D圖片(也有1D和3D)，它用來添加物體的細節；這就像有一張繪有磚塊的圖片貼到你的3D的房子上，你的房子看起來就像一堵磚牆。因為我們可以在一張圖片上插入足夠多的細節，這樣物體就會擁有很多細節而不用增加額外的頂點。

!!! Important

    除了圖像以外，紋理也可以儲存大量的數據，這些數據用來發送到著色器上，但是這不是我們現在的主題。
         
下面你會看到之前教程的那個三角形貼上了一張[磚牆](http://learnopengl.com/img/textures/wall.jpg)圖片。

![](http://learnopengl.com/img/getting-started/textures.png)

為了能夠把紋理映射到三角形上，我們需要指定三角形的每個頂點各自對應紋理的哪個部分。這樣每個頂點就會有一個**紋理座標(Texture Coordinate)**，它指明從紋理圖像的哪個地方採樣(採集像素顏色)。之後在所有的其他的片段上進行片段插值(Fragment Interpolation)。

紋理座標是x和y軸上0到1之間的範圍(注意我們使用的是2D紋理圖片)。使用紋理座標獲取紋理顏色叫做**採樣(Sampling)**。紋理座標起始於(0,0)也就是紋理圖片的左下角，終結於紋理圖片的右上角(1,1)。下面的圖片展示了我們是如何把紋理座標映射到三角形上的。

![](http://learnopengl.com/img/getting-started/tex_coords.png)

我們為三角形準備了3個紋理座標點。如上圖所示，我們希望三角形的左下角對應紋理的左下角，因此我們把三角左下角的頂點的紋理座標設置為(0,0)；三角形的上頂點對應於圖片的中間所以我們把它的紋理座標設置為(0.5,1.0)；同理右下方的頂點設置為(1.0,0)。我們只要傳遞這三個紋理座標給頂點著色器就行了，接著片段著色器會為每個片段生成紋理座標的插值。

紋理座標看起來就像這樣：

```c++
GLfloat texCoords[] = {
    0.0f, 0.0f, // 左下角
    1.0f, 0.0f, // 右下角
    0.5f, 1.0f // 頂部位置
};
```

紋理採樣有幾種不同的插值方式。我們需要自己告訴OpenGL在紋理中採用哪種採樣方式。

### 紋理環繞方式(Texture Wrapping)

紋理座標通常的範圍是從(0, 0)到(1, 1)，如果我們把紋理座標設置為範圍以外會發生什麼？OpenGL默認的行為是重複這個紋理圖像(我們簡單地忽略浮點紋理座標的整數部分)，但OpenGL提供了更多的選擇：

環繞方式            | 描述
                 ---|---
GL_REPEAT           | 紋理的默認行為，重複紋理圖像
GL_MIRRORED_REPEAET |和`GL_REPEAT`一樣，除了重複的圖片是鏡像放置的
GL_CLAMP_TO_EDGE    | 紋理座標會在0到1之間，超出的部分會重複紋理座標的邊緣，就是邊緣被拉伸
GL_CLAMP_TO_BORDER  | 超出的部分是用戶指定的邊緣的顏色

當紋理座標超出默認範圍時，每個值都有不同的視覺效果輸出。我們來看看這些紋理圖像的例子：

![](http://learnopengl.com/img/getting-started/texture_wrapping.png)

前面提到的選項都可以使用`glTexParameter`函數單獨設置每個座標軸`s`、`t`(如果是使用3D紋理那麼還有一個`r`)它們和`x`、`y`(`z`)是相等的：

`glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_MIRRORED_REPEAT);`
`glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_MIRRORED_REPEAT);`

第一個參數指定了紋理目標；我們使用的是2D紋理，因此紋理目標是`GL_TEXTURE_2D`。

第二個參數需要我們去告知我們希望去設置哪個紋理軸。

我們打算設置的是`WRAP`選項，並且指定S和T軸。最後一個參數需要我們傳遞放置方式，在這個例子裡我們在當前激活紋理上應用`GL_MIRRORED_REPEAT`。

如果我們選擇`GL_CLAMP_TO_BORDER`選項，我們還要指定一個邊緣的顏色。這次使用`glTexParameter`函數的`fv`後綴形式，加上`GL_TEXTURE_BORDER_COLOR`作為選項，這個函數需要我們傳遞一個邊緣顏色的float數組作為顏色值：

```c++
float borderColor[] = { 1.0f, 1.0f, 0.0f, 1.0f };
glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);
```

### 紋理過濾(Texture Filtering)

紋理座標不依賴於解析度，它可以是任何浮點數值，這樣OpenGL需要描述出哪個紋理像素對應哪個紋理座標(Texture Pixel，也叫Texel，[譯註1])。當你有一個很大的物體但是紋理解析度很低的時候這就變得很重要了。你可能已經猜到了，OpenGL也有一個叫做紋理過濾的選項。有多種不同的選項可用，但是現在我們只討論最重要的兩種：`GL_NEAREST`和`GL_LINEAR`。

[譯註1]: http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/06%20Textures/ "Texture Pixel也叫Texel，你可以想象你打開一張.jpg格式圖片不斷放大你會發現它是由無數像素點組成的，這個點就是紋理像素；注意不要和紋理座標搞混，紋理座標是你給模型頂點設置的那個數組，OpenGL以這個頂點的紋理座標數據去查找紋理圖像上的像素，然後進行採樣提取紋理像素的顏色"

**GL_NEAREST(Nearest Neighbor Filtering，鄰近過濾)** 是一種OpenGL默認的紋理過濾方式。當設置為`GL_NEAREST`的時候，OpenGL選擇最接近紋理座標中心點的那個像素。下圖你會看到四個像素，加號代表紋理座標。左上角的紋理像素是距離紋理座標最近的那個，這樣它就會選擇這個作為採樣顏色：

![](http://learnopengl.com/img/getting-started/filter_nearest.png)

**GL_LINEAR((Bi)linear Filtering，線性過濾)** 它會從紋理座標的臨近紋理像素進行計算，返回一個多個紋理像素的近似值。一個紋理像素距離紋理座標越近，那麼這個紋理像素對最終的採樣顏色的影響越大。下面你會看到臨近像素返回的混合顏色：

![](http://learnopengl.com/img/getting-started/filter_linear.png)

不同的紋理過濾方式有怎樣的視覺效果呢？讓我們看看當在一個很大的物體上應用一張地解析度的紋理會發生什麼吧(紋理被放大了，紋理像素也能看到)：

![](http://learnopengl.com/img/getting-started/texture_filtering.png)

如上面兩張圖片所示，`GL_NEAREST`返回了格子一樣的樣式，我們能夠清晰看到紋理由像素組成，而`GL_LINEAR`產生出更平滑的樣式，看不出紋理像素。`GL_LINEAR`是一種更真實的輸出，但有些開發者更喜歡8-bit風格，所以他們還是用`GL_NEAREST`選項。

紋理過濾可以為放大和縮小設置不同的選項，這樣你可以在紋理被縮小的時候使用最臨近過濾，被放大時使用線性過濾。我們必須通過`glTexParameter`為放大和縮小指定過濾方式。這段代碼看起來和紋理環繞方式(Texture Wrapping)的設置相似：

```c++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```

#### 多級漸遠紋理(Mipmaps)

想象一下，如果我們在一個有著上千物體的大房間，每個物體上都有紋理。距離觀察者遠的與距離近的物體的紋理的解析度是相同的。由於遠處的物體可能只產生很少的片段，OpenGL從高解析度紋理中為這些片段獲取正確的顏色值就很困難。這是因為它不得不拾為一個紋理跨度很大的片段取紋理顏色。在小物體上這會產生人工感，更不用說在小物體上使用高解析度紋理浪費內存的問題了。

OpenGL使用一種叫做 **多級漸遠紋理(Mipmap)** 的概念解決這個問題，大概來說就是一系列紋理，每個後面的一個紋理是前一個的二分之一。多級漸遠紋理背後的思想很簡單：距離觀察者更遠的距離的一段確定的閾值，OpenGL會把最適合這個距離的物體的不同的多級漸遠紋理紋理應用其上。由於距離遠，解析度不高也不會被使用者注意到。同時，多級漸遠紋理另一加分之處是，執行效率不錯。讓我們近距離看一看多級漸遠紋理紋理：

![](http://learnopengl.com/img/getting-started/mipmaps.png)

手工為每個紋理圖像創建一系列多級漸遠紋理很麻煩，幸好OpenGL有一個`glGenerateMipmaps`函數，它可以在我們創建完一個紋理後幫我們做所有的多級漸遠紋理創建工作。後面的紋理教程中你會看到如何使用它。

OpenGL渲染的時候，兩個不同級別的多級漸遠紋理之間會產生不真實感的生硬的邊界。就像普通的紋理過濾一樣，也可以在兩個不同多級漸遠紋理級別之間使用`NEAREST`和`LINEAR`過濾。指定不同多級漸遠紋理級別之間的過濾方式可以使用下面四種選項代替原來的過濾方式：


過濾方式                    | 描述
                         ---|---
GL_NEAREST_MIPMAP_NEAREST   | 接收最近的多級漸遠紋理來匹配像素大小，並使用最臨近插值進行紋理採樣
GL_LINEAR_MIPMAP_NEAREST    | 接收最近的多級漸遠紋理級別，並使用線性插值採樣
GL_NEAREST_MIPMAP_LINEAR    | 在兩個多級漸遠紋理之間進行線性插值，通過最鄰近插值採樣
GL_LINEAR_MIPMAP_LINEAR     | 在兩個相鄰的多級漸遠紋理進行線性插值，並通過線性插值進行採樣

就像紋理過濾一樣，前面提到的四種方法也可以使用`glTexParameteri`設置過濾方式：

```c++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```

常見的錯誤是，為多級漸遠紋理過濾選項設置放大過濾。這樣沒有任何效果，因為多級漸遠紋理主要使用在紋理被縮小的情況下的：紋理放大不會使用多級漸遠紋理，為多級漸遠紋理設置放大過濾選項會產生一個`GL_INVALID_ENUM`錯誤。

## 加載和創建紋理

使用紋理之前要做的第一件事是把它們加載到應用中。紋理圖像可能儲存為各種各樣的格式，每種都有自己的數據結構和排列，所以我們如何才能把這些圖像加載到應用中呢？一個解決方案是寫一個我們自己的某種圖像格式加載器比如.PNG，用它來把圖像轉化為byte序列。寫自己的圖像加載器雖然不難，但是仍然挺煩人的，而且如果要支持更多文件格式呢？你就不得不為每種你希望支持的格式寫加載器了。

另一個解決方案是，也許是更好的一種選擇，就是使用一個支持多種流行格式的圖像加載庫，來為我們解決這個問題。就像SOIL這種庫①。

### SOIL

SOIL是Simple OpenGL Image Library(簡易OpenGL圖像庫)的縮寫，它支持大多數流行的圖像格式，使用起來也很簡單，你可以從他們的主頁下載。像大多數其他庫一樣，你必須自己生成**.lib**。你可以使用**/projects**文件夾裡的解決方案(Solution)文件之一(不用擔心他們的Visual Studio版本太老，你可以把它們轉變為新的版本；這總是可行的。譯註：用VS2010的時候，你要用VC8而不是VC9的解決方案，想必更高版本的情況亦是如此)，你也可以使用CMake自己生成。你還要添加**src**文件夾裡面的文件到你的**includes**文件夾；對了，不要忘記添加**SOIL.lib**到你的連接器選項，並在你代碼文件的開頭加上`#include <SOIL.h>`。

下面的紋理部分，我們會使用一張木箱的圖片。使用SOIL加載圖片，我們會使用它的`SOIL_load_image`函數：

```c++
int width, height;
unsigned char* image = SOIL_load_image(“container..jpg”, &width, &height, 0, SOIL_LOAD_RGB);
```

函數首先需要輸入圖片文件的路徑。然後需要兩個int指針作為第二個和第三個參數，SOIL會返回圖片的寬度和高度到其中。之後，我們需要圖片的寬度和高度來生成紋理。第四個參數指定圖片的通道(Channel)數量，但是這裡我們只需留`0`。最後一個參數告訴SOIL如何來加載圖片：我們只對圖片的RGB感興趣。結果儲存為一個大的char/byte數組。

### 生成紋理

和之前生成的OpenGL對象一樣，紋理也是使用ID引用的。

```c++
GLuint texture;
glGenTextures(1, &texture);
```

`glGenTextures`函數首先需要輸入紋理生成的數量，然後把它們儲存在第二個參數的`GLuint`數組中(我們的例子裡只有一個`GLuint`)，就像其他對象一樣，我們需要綁定它，所以下面的紋理命令會配置當前綁定的紋理：

```c++
glBindTexture(GL_TEXTURE_2D, texture);
```

現在紋理綁定了，我們可以使用前面載入的圖片數據生成紋理了，紋理通過`glTexImage2D`來生成：

```c++
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, image);
glGenerateMipmap(GL_TEXTURE_2D);
```

函數很長，參數也不少，所以我們一個一個地講解：

- 第一個參數指定紋理目標(環境)；設置為`GL_TEXTURE_2D`意味著會生成與當前綁定的紋理對象在同一個目標(Target)上的紋理(任何綁定到`GL_TEXTURE_1D`和`GL_TEXTURE_3D`的紋理不會受到影響)。
- 第二個參數為我們打算創建的紋理指定多級漸遠紋理的層級，如果你希望單獨手工設置每個多級漸遠紋理的層級的話。這裡我們填0基本級。
- 第三個參數告訴OpenGL，我們希望把紋理儲存為何種格式。我們的圖像只有RGB值，因此我們把紋理儲存為`GL_RGB`值。
- 第四個和第五個參數設置最終的紋理的寬度和高度。我們加載圖像的時候提前儲存它們這樣我們就能使用相應變量了。
下個參數應該一直被設為`0`(遺留問題)。
- 第七第八個參數定義了源圖的格式和數據類型。我們使用RGB值加載這個圖像，並把它們儲存在char(byte)，我們將會傳入相應值。
- 最後一個參數是真實的圖像數據。

當調用`glTexImage2D`，當前綁定的紋理對象就會被附加上紋理圖像。然而，當前只有基本級別(Base-level)紋理圖像加載了，如果要使用多級漸遠紋理，我們必須手工設置不同的圖像(通過不斷把第二個參數增加的方式)或者，在生成紋理之後調用`glGenerateMipmap`。這會為當前綁定的紋理自動生成所有需要的多級漸遠紋理。

生成了紋理和相應的多級漸遠紋理後，解綁紋理對象、釋放圖像的內存很重要。

```c++
SOIL_free_image_data(image);
glBindTexture(GL_TEXTURE_2D, 0);
```

生成一個紋理的過程應該看起來像這樣：

```c++
GLuint texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
//為當前綁定的紋理對象設置環繞、過濾方式
...
//加載並生成紋理
int width, height;
unsigned char * image = SOIL_load_image("container.jpg", &width, &eight, 0, SOIL_LOAD_RGB);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, image);
glGenerateMipmap(GL_TEXTURE_2D);
SOIL_free_image_data(image);
glBindTexture(GL_TEXTURE_2D, 0);
```

### 應用紋理

後面的部分我們會使用`glDrawElements`繪製[Hello Triangle](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/04%20Hello%20Triangle/)教程的最後一部分的矩形。我們需要告知OpenGL如何採樣紋理，這樣我們必須更新頂點紋理座標數據：

```c++
GLfloat vertices[] = {
//  ---- 位置 ----     ---- 顏色 ----  ---- 紋理座標 ----
    0.5f, 0.5f, 0.0f, 1.0f, 0.0f, 0.0f, 1.0f, 1.0f,  // 右上
    0.5f, -0.5f, 0.0f, 0.0f, 1.0f, 0.0f, 1.0f, 0.0f, // 右下
    -0.5f, -0.5f, 0.0f, 0.0f, 0.0f, 1.0f, 0.0f, 0.0f,// 左下
    -0.5f, 0.5f, 0.0f, 1.0f, 1.0f, 0.0f, 0.0f, 1.0f  // 左上
};
```

由於我們添加了一個額外的頂點屬性，我們必須通知OpenGL新的頂點格式：

![](http://learnopengl.com/img/getting-started/vertex_attribute_pointer_interleaved_textures.png)

```c++
glVertexAttribPointer(2, 2, GL_FLOAT,GL_FALSE, 8 * sizeof(GLfloat), (GLvoid*)(6 * sizeof(GLfloat)));
glEnableVertexAttribArray(2);  
```

注意，我們必須修正前面兩個頂點屬性的步長參數為`8 * sizeof(GLfloat)`。

接著我們需要讓頂點著色器把紋理座標作為一個頂點屬性，把座標傳給片段著色器：

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 color;
layout (location = 2) in vec2 texCoord;
out vec3 ourColor;
out vec2 TexCoord;
void main()
{
    gl_Position = vec4(position, 1.0f);
    ourColor = color;
    TexCoord = texCoord;
}
```
片段著色器應該把輸出變量`TexCoord`作為輸入變量。

片段著色器應該也獲取紋理對象，但是我們怎樣把紋理對象傳給片段著色器？GLSL有一個內建數據類型，供紋理對象使用，叫做採樣器(Sampler)，它以紋理類型作為後綴，比如`sampler1D`、`sampler3D`，在我們的例子中它是`sampler2D`。我們可以簡單的聲明一個`uniform sampler2D`把一個紋理傳到片段著色器中，稍後我們把我們的紋理賦值給這個uniform。

```c++
#version 330 core
in vec3 ourColor;
in vec2 TexCoord;
out vec4 color;
uniform sampler2D ourTexture;
void main()
{
    color = texture(ourTexture, TexCoord);
}
```

我們使用GLSL的內建`texture`函數來採樣紋理的顏色，它第一個參數是紋理採樣器，第二個參數是相應的紋理座標。`texture`函數使用前面設置的紋理參數對相應顏色值進行採樣。這個片段著色器的輸出就是紋理的(插值)紋理座標上的(過濾)顏色。

現在要做的就是在調用`glDrawElements`之前綁定紋理，它會自動把紋理賦值給片段著色器的採樣器：

```c++
glBindTexture(GL_TEXTURE_2D, texture);
glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
glBindVertexArray(0);
```

如果你跟著這個教程正確的做完了，你會看到下面的圖像：

![](http://learnopengl.com/img/getting-started/textures2.png)

如果你的矩形是全黑或全白的你可能在哪兒做錯了什麼。檢查你的著色器日誌，或者嘗試對比一下[源碼](http://learnopengl.com/code_viewer.php?code=getting-started/textures)。

我們還可以把紋理顏色和頂點顏色混合，來獲得有趣的效果。我們簡單的把紋理顏色與頂點顏色在片段著色器中相乘來混合二者的顏色：

```c++
color = texture(ourTexture, TexCoord) * vec4(ourColor, 1.0f);
```

最終的效果應該是頂點顏色和紋理顏色的混合色：

![](http://learnopengl.com/img/getting-started/textures_funky.png)

這個箱子看起來有點70年代迪斯科風格。

### 紋理單元(Texture Units)

你可能感到奇怪為什麼`sampler2D`是個uniform變量，你卻不用`glUniform`給它賦值，使用`glUniform1i`我們就可以給紋理採樣器確定一個位置，這樣的話我們能夠一次在一個片段著色器中設置多紋理。一個紋理的位置通常稱為一個紋理單元。一個紋理的默認紋理單元是0，它是默認激活的紋理單元，所以教程前面部分我們不用給它確定一個位置。

紋理單元的主要目的是讓我們在著色器中可以使用多於一個的紋理。通過把紋理單元賦值給採樣器，我們可以一次綁定多紋理，只要我們首先激活相應的紋理單元。就像`glBindTexture`一樣，我們可以使用`glActiveTexture`激活紋理單元，傳入我們需要使用的紋理單元：

```c++
glActiveTexture(GL_TEXTURE0); //在綁定紋理之前，先激活紋理單元
glBindTexture(GL_TEXTURE_2D, texture);
```

激活紋理單元之後，接下來`glBindTexture`調用函數，會綁定這個紋理到當前激活的紋理單元，紋理單元`GL_TEXTURE0`總是默認被激活，所以我們在前面的例子裡當我們使用`glBindTexture`的時候，無需激活任何紋理單元。

!!! Important

        OpenGL至少提供16個紋理單元供你使用，也就是說你可以激活`GL_TEXTURE0`到`GL_TEXTRUE15`。它們都是順序定義的，所以我們也可以通過`GL_TEXTURE0+8`的方式獲得`GL_TEXTURE8`，這個例子在當我們不得不循環幾個紋理的時候變得很有用。
        
我們仍然要編輯片段著色器來接收另一個採樣器。方法現在相對簡單了：

```c++
#version 330 core
...
uniform sampler2D ourTexture1;
uniform sampler2D ourTexture2;
void main()
{
    color = mix(texture(ourTexture1, TexCoord), texture(ourTexture2, TexCoord), 0.2);
}
```

最終輸出顏色現在結合了兩個紋理查找。GLSL的內建`mix`函數需要兩個參數將根據第三個參數為前兩者作為輸入，並在之間進行線性插值。如果第三個值是0.0，它返回第一個輸入；如果是1.0，就返回第二個輸入值。0.2返回80%的第一個輸入顏色和20%的第二個輸入顏色，返回兩個紋理的混合。

我們現在需要載入和創建另一個紋理；我們應該對這些步驟感到熟悉了。確保創建另一個紋理對象，載入圖片，使用`glTexImage2D`生成最終紋理。對於第二個紋理我們使用一張你學習OpenGL時的表情圖片。

為了使用第二個紋理(也包括第一個)，我們必須改變一點渲染流程，先綁定兩個紋理到相應的紋理單元，然後定義哪個uniform採樣器對應哪個紋理單元：

```c++
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, texture1);
glUniform1i(glGetUniformLocation(ourShader.Program, “ourTexture1”), 0);
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, texture2);
glUniform1i(glGetUniformLocation(ourShader.Program, “ourTexture2”), 1);
 
glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_IN, 0);
glBindVertexArray(0);
```

注意，我們使用了`glUniform1i`設置uniform採樣器的位置或曰紋理單元。通過`glUniform1i`的設置，我們保證了每個uniform採樣器對應於合適的紋理單元。可以獲得下面的結果：

![](http://learnopengl.com/img/getting-started/textures_combined.png)

你可能注意到紋理上下顛倒了！這是因為OpenGL要求y軸0.0座標是在圖片的下面的，但是圖片通常y軸0.0座標在上面。一些圖片加載器比如DevIL在加載的時候有選項重置y原點，但是SOIL沒有。SOIL有一個叫做`SOIL_load_OGL_texture`函數可以使用一個叫做`SOIL_FLAG_INVERT_Y`的標記加載和生成紋理，它用來解決我們的問題。不過這個函數在現代OpenGL中的這個特性失效了，所以現在我們必須堅持使用`SOIL_load_image`，自己做紋理生成。

所以修復我們的小問題，有兩個選擇：

1. 我們切換頂點數據的紋理座標，翻轉`y`值(用1減去y座標)。
2. 我們可以編輯頂點著色器來翻轉`y`座標，自動替換`TexCoord`賦值：`TexCoord = vec2(texCoord.x, 1 – texCoord.y);`

!!! Attention

        這個提供的解決方案對圖片翻轉進行了一點hack。大多數情況都能工作，但是仍然執行起來以來紋理，所以最好的解決方案是換一個圖片加載器，或者以一種y原點符合OpenGL需求的方式編輯你的紋理圖像。
        
如果你編輯了頂點數據，在頂點著色器中翻轉了縱座標，你會得到下面的結果：

![](http://learnopengl.com/img/getting-started/textures_combined2.png)

### 練習

為了更熟練地使用紋理，建議在繼續之後的學習之前做完這些練習：

- 使用片段著色器**僅**對笑臉圖案進行翻轉，[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/textures-exercise1)
- 嘗試用不同的紋理環繞方式，並將紋理座標的範圍設定為從`0.0f`到`2.0f`而不是原來的`0.0f`到`1.0f`，在木箱子的角落放置4個笑臉：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/textures-exercise2)，[結果](http://learnopengl.com/img/getting-started/textures_exercise2.png)。記得一定要試試其他的環繞方式。
- 嘗試在矩形範圍內只顯示紋理圖的中間一部分，並通過修改紋理座標來設置顯示效果。嘗試使用`GL_NEAREST`的紋理過濾方式讓圖像顯示得更清晰：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/textures-exercise3)
- 使用一個uniform變量作為`mix`函數的第三個參數來改變兩個紋理可見度，使用上和下鍵來改變容器的大小和笑臉是否可見：[參考解答](http://learnopengl.com/code_viewer.php?code=getting-started/textures-exercise4)，[片段著色器](http://learnopengl.com/code_viewer.php?code=getting-started/textures-exercise4_fragment)。
