# 混合（Blending）

原文     | [Blending](http://learnopengl.com/#!Advanced-OpenGL/Blending)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)


在OpenGL中，物體透明技術通常被叫做混合(Blending)。透明是物體（或物體的一部分）非純色而是混合色，這種顏色來自於不同濃度的自身顏色和它後面的物體顏色。一個有色玻璃窗就是一種透明物體，玻璃有自身的顏色，但是最終的顏色包含了所有玻璃後面的顏色。這也正是混合這名稱的出處，因為我們將多種（來自於不同物體）顏色混合為一個顏色，透明使得我們可以看穿物體。

![](http://learnopengl.com/img/advanced/blending_transparency.png)

透明物體可以是完全透明（它使顏色完全穿透）或者半透明的（它使顏色穿透的同時也顯示自身顏色）。一個物體的透明度，被定義為它的顏色的alpha值。alpha顏色值是一個顏色向量的第四個元素，你可能已經看到很多了。在這個教程前，我們一直把這個元素設置為1.0，這樣物體的透明度就是0.0，同樣的，當alpha值是0.0時就表示物體是完全透明的，alpha值為0.5時表示物體的顏色由50%的自身的顏色和50%的後面的顏色組成。

我們之前所使用的紋理都是由3個顏色元素組成的：紅、綠、藍，但是有些紋理同樣有一個內嵌的aloha通道，它為每個紋理像素(Texel)包含著一個alpha值。這個alpha值告訴我們紋理的哪個部分有透明度，以及這個透明度有多少。例如，下面的窗子紋理的玻璃部分的alpha值為0.25(它的顏色是完全紅色，但是由於它有75的透明度，它會很大程度上反映出網站的背景色，看起來就不那麼紅了)，角落部分alpha是0.0。

![](http://learnopengl.com/img/advanced/blending_transparent_window.png)

我們很快就會把這個窗子紋理加到場景中，但是首先，我們將討論一點簡單的技術來實現紋理的半透明，也就是完全透明和完全不透明。

## 忽略片段

有些圖像並不關心半透明度，但也想基於紋理的顏色值顯示一部分。例如，創建像草這種物體你不需要花費很大力氣，通常把一個草的紋理貼到2D四邊形上，然後把這個四邊形放置到你的場景中。可是，草並不是像2D四邊形這樣的形狀，而只需要顯示草紋理的一部分而忽略其他部分。

下面的紋理正是這樣的紋理，它既有完全不透明的部分（alpha值為1.0）也有完全透明的部分（alpha值為0.0），而沒有半透明的部分。你可以看到沒有草的部分，圖片顯示了網站的背景色，而不是它自身的那部分顏色。

![](http://learnopengl.com/img/textures/grass.png)

所以，當向場景中添加像這樣的紋理時，我們不希望看到一個方塊圖像，而是隻顯示實際的紋理像素，剩下的部分可以被看穿。我們要忽略(丟棄)紋理透明部分的像素，不必將這些片段儲存到顏色緩衝中。在此之前，我們還要學一下如何加載一個帶有透明像素的紋理。

加載帶有alpha值的紋理我們需要告訴SOIL，去加載RGBA元素圖像，而不再是RGB元素的。SOIL能以RGBA的方式加載大多數沒有alpha值的紋理，它會將這些像素的alpha值設為了1.0。

```c++
unsigned char * image = SOIL_load_image(path, &width, &height, 0, SOIL_LOAD_RGBA);
```

不要忘記還要改變OpenGL生成的紋理：

```c++
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, image);
```

保證你在片段著色器中獲取了紋理的所有4個顏色元素，而不僅僅是RGB元素：

```c++
void main()
{
    // color = vec4(vec3(texture(texture1, TexCoords)), 1.0);
    color = texture(texture1, TexCoords);
}
```

現在我們知道了如何加載透明紋理，是時候試試在深度測試教程裡那個場景中添加幾根草了。

我們創建一個`std::vector`，並向裡面添加幾個`glm::vec3`變量，來表示草的位置：

```c++
vector<glm::vec3> vegetation;
vegetation.push_back(glm::vec3(-1.5f,  0.0f, -0.48f));
vegetation.push_back(glm::vec3( 1.5f,  0.0f,  0.51f));
vegetation.push_back(glm::vec3( 0.0f,  0.0f,  0.7f));
vegetation.push_back(glm::vec3(-0.3f,  0.0f, -2.3f));
vegetation.push_back(glm::vec3( 0.5f,  0.0f, -0.6f));
```

一個單獨的四邊形被貼上草的紋理，這並不能完美的表現出真實的草，但是比起加載複雜的模型還是要高效很多，利用一些小技巧，比如在同一個地方添加多個不同朝向的草，還是能獲得比較好的效果的。

由於草紋理被添加到四邊形物體上，我們需要再次創建另一個VAO，向裡面填充VBO，以及設置合理的頂點屬性指針。在我們繪製完地面和兩個立方體後，我們就來繪製草葉：

```c++
glBindVertexArray(vegetationVAO);
glBindTexture(GL_TEXTURE_2D, grassTexture);  
for(GLuint i = 0; i < vegetation.size(); i++)
{
    model = glm::mat4();
    model = glm::translate(model, vegetation[i]);
    glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
    glDrawArrays(GL_TRIANGLES, 0, 6);
}  
glBindVertexArray(0);
```

運行程序你將看到：
![](http://learnopengl.com/img/advanced/blending_no_discard.png)

出現這種情況是因為OpenGL默認是不知道如何處理alpha值的，不知道何時忽略(丟棄)它們。我們不得不手動做這件事。幸運的是這很簡單，感謝著色器，GLSL為我們提供了discard命令，它保證了片段不會被進一步處理，這樣就不會進入顏色緩衝。有了這個命令我們就可以在片段著色器中檢查一個片段是否有在一定的閾限下的alpha值，如果有，那麼丟棄這個片段，就好像它不存在一樣：

```c++
#version 330 core
in vec2 TexCoords;

out vec4 color;

uniform sampler2D texture1;

void main()
{
    vec4 texColor = texture(texture1, TexCoords);
    if(texColor.a < 0.1)
        discard;
    color = texColor;
}
```

在這兒我們檢查被採樣紋理顏色包含著一個低於0.1這個閾限的alpha值，如果有，就丟棄這個片段。這個片段著色器能夠保證我們只渲染哪些不是完全透明的片段。現在我們來看看效果：

![](http://learnopengl.com/img/advanced/blending_discard.png)

!!! Important

    需要注意的是，當採樣紋理邊緣的時候，OpenGL在邊界值和下一個重複的紋理的值之間進行插值（因為我們把它的放置方式設置成了GL_REPEAT）。這樣就行了，但是由於我們使用的是透明值，紋理圖片的上部獲得了它的透明值是與底邊的純色值進行插值的。結果就是一個有點半透明的邊，你可以從我們的紋理四邊形的四周看到。為了防止它的出現，當你使用alpha紋理的時候要把紋理環繞方式設置為`GL_CLAMP_TO_EDGE`：

    `glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);`

    `glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);`

你可以[在這裡得到源碼](http://learnopengl.com/code_viewer.php?code=advanced/blending_discard)。


## 混合

上述丟棄片段的方式，不能使我們獲得渲染半透明圖像，我們要麼渲染出像素，要麼完全地丟棄它。為了渲染出不同的透明度級別，我們需要開啟**混合**(Blending)。像大多數OpenGL的功能一樣，我們可以開啟`GL_BLEND`來啟用混合功能：

```c++
glEnable(GL_BLEND);
```

開啟混合後，我們還需要告訴OpenGL它該如何混合。

OpenGL以下面的方程進行混合：

C¯result = C¯source ∗ Fsource + C¯destination ∗ Fdestination

* C¯source：源顏色向量。這是來自紋理的本來的顏色向量。
* C¯destination：目標顏色向量。這是儲存在顏色緩衝中當前位置的顏色向量。
* Fsource：源因子。設置了對源顏色的alpha值影響。
* Fdestination：目標因子。設置了對目標顏色的alpha影響。

片段著色器運行完成並且所有的測試都通過以後，混合方程才能自由執行片段的顏色輸出，當前它在顏色緩衝中（前面片段的顏色在當前片段之前儲存）。源和目標顏色會自動被OpenGL設置，而源和目標因子可以讓我們自由設置。我們來看一個簡單的例子：

![](http://learnopengl.com/img/advanced/blending_equation.png)

我們有兩個方塊，我們希望在紅色方塊上繪製綠色方塊。紅色方塊會成為源顏色（它會先進入顏色緩衝），我們將在紅色方塊上繪製綠色方塊。

那麼問題來了：我們怎樣來設置因子呢？我們起碼要把綠色方塊乘以它的alpha值，所以我們打算把Fsource設置為源顏色向量的alpha值：0.6。接著，讓目標方塊的濃度等於剩下的alpha值。如果最終的顏色中綠色方塊的濃度為60%，我們就把紅色的濃度設為40%（1.0 – 0.6）。所以我們把Fdestination設置為1減去源顏色向量的alpha值。方程將變成：

![](../img/blending_C_result.png)

最終方塊結合部分包含了60%的綠色和40%的紅色，得到一種髒兮兮的顏色：

![](http://learnopengl.com/img/advanced/blending_equation_mixed.png)

最後的顏色被儲存到顏色緩衝中，取代先前的顏色。

這個方案不錯，但我們怎樣告訴OpenGL來使用這樣的因子呢？恰好有一個叫做`glBlendFunc`的函數。

`void glBlendFunc(GLenum sfactor, GLenum dfactor)`接收兩個參數，來設置源（source）和目標（destination）因子。OpenGL為我們定義了很多選項，我們把最常用的列在下面。注意，顏色常數向量C¯constant可以用`glBlendColor`函數分開來設置。


Option |	Value
---|---
GL_ZERO  |	0
GL_ONE	 |  1
GL_SRC_COLOR |	顏色C¯source.
GL_ONE_MINUS_SRC_COLOR | 1 − C¯source.
GL_DST_COLOR |	 C¯destination
GL_ONE_MINUS_DST_COLOR | 1 − C¯destination.
GL_SRC_ALPHA |	C¯source的alpha值
GL_ONE_MINUS_SRC_ALPHA | 1 - C¯source的alpha值
GL_DST_ALPHA |	C¯destination的alpha值
GL_ONE_MINUS_DST_ALPHA | 1 - C¯destination的alpha值
GL_CONSTANT_COLOR	   | C¯constant.
GL_ONE_MINUS_CONSTANT_COLOR |	1 - C¯constant
GL_CONSTANT_ALPHA	   | C¯constant的alpha值
GL_ONE_MINUS_CONSTANT_ALPHA |	1 − C¯constant的alpha值

為從兩個方塊獲得混合結果，我們打算把源顏色的alpha給源因子，1-alpha給目標因子。調整到`glBlendFunc`之後就像這樣：

```c++
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
```

也可以為RGB和alpha通道各自設置不同的選項，使用`glBlendFuncSeperate`：

```c++
glBlendFuncSeperate(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA,GL_ONE, GL_ZERO);
```

這個方程就像我們之前設置的那樣，設置了RGB元素，但是隻讓最終的alpha元素被源alpha值影響到。

OpenGL給了我們更多的自由，我們可以改變方程源和目標部分的操作符。現在，源和目標元素已經相加了。如果我們願意的話，我們還可以把它們相減。

`void glBlendEquation(GLenum mode)`允許我們設置這個操作，有3種可行的選項：

* GL_FUNC_ADD：默認的，彼此元素相加：C¯result = Src + Dst.
* GL_FUNC_SUBTRACT：彼此元素相減： C¯result = Src – Dst.
* GL_FUNC_REVERSE_SUBTRACT：彼此元素相減，但順序相反：C¯result = Dst – Src.

通常我們可以簡單地省略`glBlendEquation`因為GL_FUNC_ADD在大多數時候就是我們想要的，但是如果你如果你真想嘗試努力打破主流常規，其他的方程或許符合你的要求。

### 渲染半透明紋理

現在我們知道OpenGL如何處理混合，是時候把我們的知識運用起來了，我們來添加幾個半透明窗子。我們會使用本教程開始時用的那個場景，但是不再渲染草紋理，取而代之的是來自教程開始處半透明窗子紋理。

首先，初始化時我們需要開啟混合，設置合適和混合方程：

```c++
glEnable(GL_BLEND);
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
```

由於我們開啟了混合，就不需要丟棄片段了，所以我們把片段著色器設置為原來的那個版本：

```c++
#version 330 core
in vec2 TexCoords;

out vec4 color;

uniform sampler2D texture1;

void main()
{
    color = texture(texture1, TexCoords);
}
```

這一次（無論OpenGL什麼時候去渲染一個片段），它都根據alpha值，把當前片段的顏色和顏色緩衝中的顏色進行混合。因為窗子的玻璃部分的紋理是半透明的，我們應該可以透過玻璃看到整個場景。

![](http://learnopengl.com/img/advanced/blending_incorrect_order.png)

如果你仔細看看，就會注意到有些不對勁。前面的窗子透明部分阻塞了後面的。為什麼會這樣？

原因是深度測試在與混合的一同工作時出現了點狀況。當寫入深度緩衝的時候，深度測試不關心片段是否有透明度，所以透明部分被寫入深度緩衝，就和其他值沒什麼區別。結果是整個四邊形的窗子被檢查時都忽視了透明度。即便透明部分應該顯示出後面的窗子，深度緩衝還是丟棄了它們。

所以我們不能簡簡單單地去渲染窗子，我們期待著深度緩衝為我們解決這所有問題；這也正是混合之處代碼不怎麼好看的原因。為保證前面窗子顯示了它後面的窗子，我們必須首先繪製後面的窗子。這意味著我們必須手工調整窗子的順序，從遠到近地逐個渲染。

!!! Important

    對於全透明物體，比如草葉，我們選擇簡單的丟棄透明像素而不是混合，這樣就減少了令我們頭疼的問題（沒有深度測試問題）。

### 別打亂順序

要讓混合在多物體上有效，我們必須先繪製最遠的物體，最後繪製最近的物體。普通的無混合物體仍然可以使用深度緩衝正常繪製，所以不必給它們排序。我們一定要保證它們在透明物體前繪製好。當無透明度物體和透明物體一起繪製的時候，通常要遵循以下原則：

先繪製所有不透明物體。
為所有透明物體排序。
按順序繪製透明物體。
一種排序透明物體的方式是，獲取一個物體到觀察者透視圖的距離。這可以通過獲取攝像機的位置向量和物體的位置向量來得到。接著我們就可以把它和相應的位置向量一起儲存到一個map數據結構（STL庫）中。map會自動基於它的鍵排序它的值，所以當我們把它們的距離作為鍵添加到所有位置中後，它們就自動按照距離值排序了：

```c++
std::map<float, glm::vec3> sorted;
for (GLuint i = 0; i < windows.size(); i++) // windows contains all window positions
{
    GLfloat distance = glm::length(camera.Position - windows[i]);
    sorted[distance] = windows[i];
}
```

最後產生了一個容器對象，基於它們距離從低到高儲存了每個窗子的位置。

隨後當渲染的時候，我們逆序獲取到每個map的值（從遠到近），然後以正確的繪製相應的窗子：

```c++
for(std::map<float,glm::vec3>::reverse_iterator it = sorted.rbegin(); it != sorted.rend(); ++it)
{
    model = glm::mat4();
    model = glm::translate(model, it->second);
    glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
    glDrawArrays(GL_TRIANGLES, 0, 6);
}
```

我們從map得來一個逆序的迭代器，迭代出每個逆序的條目，然後把每個窗子的四邊形平移到相應的位置。這個相對簡單的方法對透明物體進行了排序，修正了前面的問題，現在場景看起來像這樣：

![](http://learnopengl.com/img/advanced/blending_sorted.png)

你可以[從這裡得到完整的帶有排序的源碼](http://learnopengl.com/code_viewer.php?code=advanced/blending_sorted)。

雖然這個按照它們的距離對物體進行排序的方法在這個特定的場景中能夠良好工作，但它不能進行旋轉、縮放或者進行其他的變換，奇怪形狀的物體需要一種不同的方式，而不能簡單的使用位置向量。

在場景中排序物體是個有難度的技術，它很大程度上取決於你場景的類型，更不必說會耗費額外的處理能力了。完美地渲染帶有透明和不透明的物體的場景並不那麼容易。有更高級的技術例如次序無關透明度（order independent transparency），但是這超出了本教程的範圍。現在你不得不採用普通的混合你的物體，但是如果你小心謹慎，並知道這個侷限，你仍可以得到頗為合適的混合實現。