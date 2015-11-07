# 文字渲染

原文     | [Text Rendering](http://learnopengl.com/#!In-Practice/Text-Rendering)
      ---|---
作者     | JoeyDeVries
翻譯     | [Geequlim](http://geequlim.com)
校對     | gjy_1992

當你在圖形計算領域冒險到了一定階段以後你可能會想使用OpenGL來繪製文字。然而，可能與你想象的並不一樣，使用像OpenGL這樣的底層庫來把文字渲染到屏幕上並不是一件簡單的事情。如果你你只需要繪製128種字符，那麼事情可能會簡單一些。但是當我們要繪製的字符有著不同的寬、高和邊距；如果你使用的語言中不止包含128個字符；當你要繪製音樂符、數學符號；以及考慮把如何處理文本自動轉行等等情況考慮進來的時候...事情馬上就會變得複雜得多，你甚至覺得這些工作並不屬於像OpenGL這樣的底層圖形庫該討論的範疇。

由於OpenGL本身並沒有定義如何渲染文字到屏幕，也沒有用於表示文字的基本圖形，我們必須自己定義一套全新的方式才能讓OpenGL來繪製文字。目前一些技術包括：通過**`GL_LINES`**來繪製字形、創建文字的3D網格、將帶有文字的紋理渲染到一個2D方塊中。

開發者最常用的一種方式是將字符紋理繪製到矩形方塊上。繪製這些紋理方塊其實並不是很複雜，然而檢索要繪製的文字的紋理卻變成了一項挑戰性的工作。本教程將探索多種文字渲染的實現方法，並且著重對更加現代而且靈活的渲染技術(使用FreeType庫)進行講解。

## 經典文字渲染：使用位圖字體

在早期渲染文字時，選擇你應用程序的字體(或者創建你自己的字體)來繪製文字是通過將所有用到的文字加載在一張大紋理圖中來實現的。這張紋理貼圖我們把它叫做位圖字體(Bitmap Font)，它包含了所有我們想要使用的字符。這些字符被稱為**字形(glyph)**。每個字形根據他們的編號被放到位圖字體中的確切位置，在渲染這些字形的時候根據這些排列規則將他們取出並貼到指定的2D方塊中。

![](http://learnopengl.com/img/in-practice/bitmapfont.png)

上圖展示了我們如何從一張位圖字體的紋理中通過對字形的合理取樣(通過小心地選擇字形的紋理座標)來實現繪製文字“OpenGL”到2D方塊中的原理。通過對OpenGL啟用混合並讓位圖字體的紋理背景保持透明，這樣就能實現使用OpenGL繪製你想要文字到屏幕的功能。上圖的這張位圖字體是使用[Codehead的位圖字體生成器](http://www.codehead.co.uk/cbfg/)生成的。

使用這種途徑來繪製文字有許多優勢也有很多缺點。首先，它相對來說很容易實現，並且因為位圖字體被預渲染好，使得這種方法效率很高。然而，這種途徑並不夠靈活。當你想要使用不同的字體時，你不得不重新生成位圖字體，以及你的程序會被限制在一個固定的分辨率：如果你對這些文字進行放大的話你會看到文字的像素邊緣。此外，這種方式僅侷限於用來繪製很少的字符，如果你想讓它來擴展支持Unicode文字的話就很不現實了。

這種繪製文字的方式曾經得益於它的高速和可移植性而非常流行，然而現在已經存在更加靈活的方式了。其中一個是我們即將展開討論的使用FreeType庫來加載TrueType字體的方式。

## 現代文字渲染：使用FreeType

FreeType是一個能夠用於加載字體並將他們渲染到位圖以及提供多種字體相關的操作的軟件開發庫。它是一個非常受歡迎的跨平臺字體庫，被用於 Mac OSX、Java、PlayStation主機、Linux、Android等。FreeType的真正吸引力在於它能夠加載TrueType字體。

TrueType字體不採用像素或其他不可縮放的方式來定義，而是一些通過數學公式(曲線的組合)。這些字形，類似於矢量圖像，可以根據你需要的字體大小來生成像素圖像。通過使用TrueType字體可以輕易呈現不同大小的字符符號並且沒有任何質量損失。

FreeType可以在他們的[官方網站](http://www.freetype.org/)中下載到。你可以選擇自己編譯他們提供的源代碼或者使用他們已編譯好的針對你的平臺的鏈接庫。請確認你將freetype.lib添加到你項目的鏈接庫中，同時你還要添加它的頭文件目錄到項目的搜索目錄中。

然後請確認包含合適的頭文件：

```c++
#include <ft2build.h>
#include FT_FREETYPE_H  
```

!!! Attention
    
    由於FreeType開發得比較早（至少在我寫這篇文章以前就已經開發好了），你不能將它們的頭文件放到一個新的目錄下，它們應該保存在你包含目錄的根目錄下。通過使用像 `#include <FreeType/ft2build.h>` 這樣的方式導入FreeType可能會出現各種頭文件衝突的問題。

FreeType要做的事就是加載TrueType字體併為每一個字形生成位圖和幾個度量值。我們可以取出它生成的位圖作為字形的紋理，將這些度量值用作字形紋理的位置、偏移等描述。

要加載一個字體，我們需要做的是初始化FreeType並且將這個字體加載為FreeType稱之為面(Face)的東西。這裡為我們加載一個從Windows/Fonts目錄中拷貝來的TrueType字體文件arial.ttf。

```c++
FT_Library ft;
if (FT_Init_FreeType(&ft))
    std::cout << "ERROR::FREETYPE: Could not init FreeType Library" << std::endl;

FT_Face face;
if (FT_New_Face(ft, "fonts/arial.ttf", 0, &face))
    std::cout << "ERROR::FREETYPE: Failed to load font" << std::endl;  
```

這些FreeType函數在出現錯誤的情況下返回一個非零整數值。

一旦我們加載字體面完成，我們還要定義文字大小，這表示著我們要從字體面中生成多大的字形：

```c++
FT_Set_Pixel_Sizes(face, 0, 48);  
```
此函數設置了字體面的寬度和高度，將寬度值設為0表示我們要從字體面通過給出的高度中動態計算出字形的寬度。

一個字體面中包含了所有字形的集合。我們可以通過調用`FT_Load_Char`函數來激活當前要表示的字形。這裡我們選在加載字母字形'X'：

```c++
if (FT_Load_Char(face, 'X', FT_LOAD_RENDER))
    std::cout << "ERROR::FREETYTPE: Failed to load Glyph" << std::endl;  
```

通過將**`FT_LOAD_RENDER`**設為一個加載標識，我們告訴FreeType去創建一個8位的灰度位圖，我們可以通過`face->glyph->bitmap`來取得這個位圖。

使用FreeType加載的字形位圖並不像我們使用位圖字體那樣持有相同的尺寸大小。使用FreeType生產的字形位圖的大小是恰好能包含這個字形的尺寸。例如生產用於表示'.'的位圖的尺寸要比表示'X'的小得多。因此，FreeType在加載字形的時候還生產了幾個度量值來描述生成的字形位圖的大小和位置。下圖展示了FreeType的所有度量值的涵義。

![](http://learnopengl.com/img/in-practice/glyph.png)

每一個字形都放在一個水平的基線(Baseline)上，上圖中被描黑的水平箭頭表示該字形的基線。這條基線類似於拼音四格線中的第二根水平線，一些字形被放在基線以上(如'x'或'a')，而另一些則會超過基線以下(如'g'或'p')。FreeType的這些度量值中包含了字形在相對於基線上的偏移量用來描述字形相對於此基線的位置，字形的大小，以及與下一個字符之間的距離。下面列出了我們在渲染字形時所需要的度量值的屬性：



屬性|獲取方式|生成位圖描述
---|---
width   | face->glyph->bitmap.width | 寬度，單位:像素
height  | face->glyph->bitmap.rows  | 高度，單位:像素
bearingX| face->glyph->bitmap_left| 水平位置(相對於起點origin)，單位:像素
bearingY| face->glyph->bitmap_top | 垂直位置(相對於基線Baseline)，單位:像素
advance | face->glyph->advance.x  | 起點到下一個字形的起點間的距離(單位:1/64像素)

在我們想要渲染字符到屏幕的時候就能根據這些度量值來生成對應字形的紋理了，然而每次渲染都這樣做顯然不是高效的。我們應該將這些生成的數據儲存在應用程序中，在渲染過程中再去取。方便起見，我們需要定義一個用來儲存這些屬性的結構體,並創建一個字符表來存儲這些字形屬性。

```c++
struct Character {
    GLuint     TextureID;  // 字形紋理ID
    glm::ivec2 Size;       // 字形大大小
    glm::ivec2 Bearing;    // 字形基於基線和起點的位置
    GLuint     Advance;    // 起點到下一個字形起點的距離
};

std::map<GLchar, Character> Characters;
```

本教程本著讓一切簡單的目的，我們只生成表示128個ASCII字符的字符表。併為每一個字符儲存紋理和一些度量值。這樣，所有需要的字符就被存下來備用了。

```c++
glPixelStorei(GL_UNPACK_ALIGNMENT, 1); //禁用byte-alignment限制
for (GLubyte c = 0; c < 128; c++)
{
    // 加載字符的字形 
    if (FT_Load_Char(face, c, FT_LOAD_RENDER))
    {
        std::cout << "ERROR::FREETYTPE: Failed to load Glyph" << std::endl;
        continue;
    }
    // 生成字形紋理
    GLuint texture;
    glGenTextures(1, &texture);
    glBindTexture(GL_TEXTURE_2D, texture);
    glTexImage2D(
        GL_TEXTURE_2D,
        0,
        GL_RED,
        face->glyph->bitmap.width,
        face->glyph->bitmap.rows,
        0,
        GL_RED,
        GL_UNSIGNED_BYTE,
        face->glyph->bitmap.buffer
    );
    // 設置紋理選項
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    // 將字符存儲到字符表中備用
    Character character = {
        texture, 
        glm::ivec2(face->glyph->bitmap.width, face->glyph->bitmap.rows),
        glm::ivec2(face->glyph->bitmap_left, face->glyph->bitmap_top),
        face->glyph->advance.x
    };
    Characters.insert(std::pair<GLchar, Character>(c, character));
}
```

在這個for循環中我們將所有ASCII中的128個字符設置合適的字形。為每一個字符創建紋理並存儲它們的部分度量值。有趣的是我們這裡將紋理的格式設置為**GL_RED**。通過字形生成的位圖是8位灰度圖，他的每一個顏色表示為一個字節。因此我們需要將每一位都作為紋理的顏色值。我們通過創建每一個字節表示一個顏色的紅色分量(顏色分量的第一個字節)來創建字形紋理。我們想用一個字節來表示紋理顏色，這需要提前通知OpenGL

```c++
glPixelStorei(GL_UNPACK_ALIGNMENT, 1);   
```

OpenGL要求所有的紋理顏都必須是4字節隊列，這樣紋理的內存大小就一定是4字節的整數倍。通常這並不會出現什麼問題，因為通常的紋理都有4或者4的整數被的儲存大小，但是現在我們只用一位來表示每一個像素顏色，此時的紋理可能有任意內存長度。通過將紋理解壓參數設為1，這樣才能確保不會有內存對齊的解析問題。

當你處理完字形後同樣不要忘記清理FreeType的資源。

```c++
FT_Done_Face(face);
FT_Done_FreeType(ft);
```

### 著色器

我們需要使用下面的頂點著色器來渲染字形：

```c++
#version 330 core
layout (location = 0) in vec4 vertex; // <vec2 pos, vec2 tex>
out vec2 TexCoords;

uniform mat4 projection;

void main()
{
    gl_Position = projection * vec4(vertex.xy, 0.0, 1.0);
    TexCoords = vertex.zw;
}  
```

我們將位置和紋理紋理座標的數據合起來存在一個`vec4`中。頂點著色器將會將位置座標與投影矩陣相乘，並將紋理座標轉發給片段著色器:

```c++
#version 330 core
in vec2 TexCoords;
out vec4 color;

uniform sampler2D text;
uniform vec3 textColor;

void main()
{    
    vec4 sampled = vec4(1.0, 1.0, 1.0, texture(text, TexCoords).r);
    color = vec4(textColor, 1.0) * sampled;
}  
```

片段著色器有兩個uniform變量：一個是單顏色通道的字形位圖紋理，另一個是文字的顏色，我們可以同調整它來改變最終輸出的字體顏色。我們首先從位圖紋理中採樣，由於紋理數據中僅存儲著紅色分量，我們就通過**`r`**分量來作為取樣顏色的aplpha值。結果值是一個字形背景為純透明，而字符部分為不透明的白色的顏色。我們將此顏色與字體顏色uniform值相乘就得到了要輸出的字符顏色了。

當然我們必需開啟混合才能讓這一切行之有效：

```c++
glEnable(GL_BLEND);
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);  
```
至於投影矩陣我們將使用一個正交投影矩陣。對於文字渲染我們通常都不需要進行透視，使用正交投影也能保證我們所有的頂點座標設置有效：

```c++
glm::mat4 projection = glm::ortho(0.0f, 800.0f, 0.0f, 600.0f);
```

我們設置投影矩陣的底部參數為0.0f並將頂部參數設置為窗口的高度。這樣做的結果是我們可以通過設置0.0f~600.0f的縱座標來表示頂點在窗口中的垂直位置。這意味著現在點(0.0,0.0)表示左下角而不再是窗口正中間。

最後要做的事是創建一個VBO和VAO用來渲染方塊。現在我們分配足夠的內存來初始化VBO然後在我們渲染字符的時候再來更新VBO的內存。

```c++
GLuint VAO, VBO;
glGenVertexArrays(1, &VAO);
glGenBuffers(1, &VBO);
glBindVertexArray(VAO);
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(GLfloat) * 6 * 4, NULL, GL_DYNAMIC_DRAW);
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 4 * sizeof(GLfloat), 0);
glBindBuffer(GL_ARRAY_BUFFER, 0);
glBindVertexArray(0);      
```

每個2D方塊需要6個頂點，每個頂點又是由一個4維向量（一個紋理座標和一個頂點座標）組成，因此我們將VBO的內存分配為6*4個float的大小。由於我們會在繪製字符時更新這斷內存，所以我們將內存類型設置為`GL_DYNAMIC_DRAW`

### 渲染一行文字

要渲染一個字符，我們從之前創建的字符表中取出一個字符結構體，根據字符的度量值來計算方塊的大小。根據方塊的大小我們就能計算出6個描述方塊的頂點，並使用`glBufferSubData`函數將其更新到VBO所在內內存中。

我們創建一個函數用來渲染一行文字:

```c++
void RenderText(Shader &s, std::string text, GLfloat x, GLfloat y, GLfloat scale, glm::vec3 color)
{
    // 激活合適的渲染狀態
    s.Use();
    glUniform3f(glGetUniformLocation(s.Program, "textColor"), color.x, color.y, color.z);
    glActiveTexture(GL_TEXTURE0);
    glBindVertexArray(VAO);

    // 對文本中的所有字符迭代
    std::string::const_iterator c;
    for (c = text.begin(); c != text.end(); c++)
    {
        Character ch = Characters[*c];

        GLfloat xpos = x + ch.Bearing.x * scale;
        GLfloat ypos = y - (ch.Size.y - ch.Bearing.y) * scale;

        GLfloat w = ch.Size.x * scale;
        GLfloat h = ch.Size.y * scale;
        // 當前字符的VBO
        GLfloat vertices[6][4] = {
            { xpos,     ypos + h,   0.0, 0.0 },            
            { xpos,     ypos,       0.0, 1.0 },
            { xpos + w, ypos,       1.0, 1.0 },

            { xpos,     ypos + h,   0.0, 0.0 },
            { xpos + w, ypos,       1.0, 1.0 },
            { xpos + w, ypos + h,   1.0, 0.0 }           
        };
        // 在方塊上繪製字形紋理
        glBindTexture(GL_TEXTURE_2D, ch.textureID);
        // 更新當前字符的VBO
        glBindBuffer(GL_ARRAY_BUFFER, VBO);
        glBufferSubData(GL_ARRAY_BUFFER, 0, sizeof(vertices), vertices); 
        glBindBuffer(GL_ARRAY_BUFFER, 0);
        // 繪製方塊
        glDrawArrays(GL_TRIANGLES, 0, 6);
        //  更新位置到下一個字形的原點，注意單位是1/64像素
        x += (ch.Advance >> 6) * scale; //(2^6 = 64)
    }
    glBindVertexArray(0);
    glBindTexture(GL_TEXTURE_2D, 0);
}
```

這個函數的內容註釋的比較詳細了：我們首先計算出方塊的起點座標和它的大小，併為該方塊生成一個6個頂點；注意我們在縮放的同時會將部分度量值也進行縮放。接著我們更新方塊的VBO、綁定字形紋理來渲染它。

其中這行代碼需要加倍留意：

```c++
GLfloat ypos = y - (ch.Size.y - ch.Bearing.y);   
```

一些字符(諸如'p'或'q')需要被渲染到基線一下，因此字形方塊也應該講y位置往下調整。調整的量就是便是字形度量值中字形的高度和BearingY的差：

![](http://learnopengl.com/img/in-practice/glyph_offset.png)

要計算這段偏移量的距離，我們需要指出是字形在基線以下的部分最底斷到基線的距離。在上圖中這段距離用紅色雙向箭頭標出。如你所見，在字形度量值中，我們可以通過用字形的高度減去bearingY來計算這段向量的長度。這段距離有可能是0.0f(如'x'字符)
，或是超出基線底部的距離的長度(如'g'或'j')。

如果你每件事都做對了，那麼你現在已經可以使用下面的句子成功地渲染字符串了：

```c++
RenderText(shader, "This is sample text", 25.0f, 25.0f, 1.0f, glm::vec3(0.5, 0.8f, 0.2f));
RenderText(shader, "(C) LearnOpenGL.com", 540.0f, 570.0f, 0.5f, glm::vec3(0.3, 0.7f, 0.9f));
```

渲染效果看上去像這樣：

![](http://learnopengl.com/img/in-practice/text_rendering.png)

你可以從這裡獲取這個例子的[源代碼](http://learnopengl.com/code_viewer.php?code=in-practice/text_rendering)。

通過關閉字形紋理的綁定，能夠給你對文字方塊的頂點計算更好的理解，它看上去像這樣：

![](http://learnopengl.com/img/in-practice/text_rendering_quads.png)

這樣你就能清楚地看到那條傳說中的基線了。

## 關於未來

本教程演示瞭如何使用FreeType繪製TrueType文字。這種方式靈活、可縮放並支持多種字符編碼。然而，你的應用程序可能並不需要這麼強大的功能，性能更好的點陣字體也許是更可取的。當然你可以結合這兩種方法通過動態生成位圖字體中所有字符字形。這節省了從大量的紋理渲染器開關和基於每個字形緊密包裝可以節省相當的一些性能。

另一個使用FreeType字體的問題是字形紋理是對應著一個固定的字體大小的，因此直接對其放大就會出現鋸齒邊緣。此外，對字形進行旋轉還會使它們看上去變得模糊。可以通過將每個像素設為最近的字形輪廓的像素，而不是直接設為實際柵格化的像素，可以減輕這些問題。這項技術被稱為**signed distance fields**，Valve在幾年前發表過一篇了[論文](http://www.valvesoftware.com/publications/2007/SIGGRAPH2007_AlphaTestedMagnification.pdf)，探討了他們通過這項技術來獲得好得驚人的3D渲染效果。