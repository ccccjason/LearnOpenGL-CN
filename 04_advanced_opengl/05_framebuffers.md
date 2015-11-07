# 幀緩衝（Framebuffer）

原文     | [Framebuffers](http://learnopengl.com/#!Advanced-OpenGL/Framebuffers)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

到目前為止，我們使用了幾種不同類型的屏幕緩衝：用於寫入顏色值的顏色緩衝，用於寫入深度信息的深度緩衝，以及允許我們基於一些條件丟棄指定片段的模板緩衝。把這幾種緩衝結合起來叫做幀緩衝(Framebuffer)，它被儲存於內存中。OpenGL給了我們自己定義幀緩衝的自由，我們可以選擇性的定義自己的顏色緩衝、深度和模板緩衝。

[譯註1]: http://learnopengl-cn.readthedocs.org "framebuffer，在維基百科有framebuffer的詳細介紹能夠幫助你更好的理解"

我們目前所做的渲染操作都是是在默認的幀緩衝之上進行的。當你創建了你的窗口的時候默認幀緩衝就被創建和配置好了（GLFW為我們做了這件事）。通過創建我們自己的幀緩衝我們能夠獲得一種額外的渲染方式。

你也許不能立刻理解應用程序的幀緩衝的含義，通過幀緩衝可以將你的場景渲染到一個不同的幀緩衝中，可以使我們能夠在場景中創建鏡子這樣的效果，或者做出一些炫酷的特效。首先我們會討論它們是如何工作的，然後我們將利用幀緩衝來實現一些炫酷的效果。

## 創建一個幀緩衝

就像OpenGL中其他對象一樣，我們可以使用一個叫做`glGenFramebuffers`的函數來創建一個幀緩衝對象（簡稱FBO）：

```c++
GLuint fbo;
glGenFramebuffers(1, &fbo);
```

這種對象的創建和使用的方式我們已經見過不少了，因此它們的使用方式也和之前我們見過的其他對象的使用方式相似。首先我們要創建一個幀緩衝對象，把它綁定到當前幀緩衝，做一些操作，然後解綁幀緩衝。我們使用`glBindFramebuffer`來綁定幀緩衝：

```c++
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
```

綁定到`GL_FRAMEBUFFER`目標後，接下來所有的讀、寫幀緩衝的操作都會影響到當前綁定的幀緩衝。也可以把幀緩衝分開綁定到讀或寫目標上，分別使用`GL_READ_FRAMEBUFFER`或`GL_DRAW_FRAMEBUFFER`來做這件事。如果綁定到了`GL_READ_FRAMEBUFFER`，就能執行所有讀取操作，像`glReadPixels`這樣的函數使用了；綁定到`GL_DRAW_FRAMEBUFFER`上，就允許進行渲染、清空和其他的寫入操作。大多數時候你不必分開用，通常把兩個都綁定到`GL_FRAMEBUFFER`上就行。

很遺憾，現在我們還不能使用自己的幀緩衝，因為還沒做完呢。建構一個完整的幀緩衝必須滿足以下條件：

* 我們必須往裡面加入至少一個附件（顏色、深度、模板緩衝）。
* 其中至少有一個是顏色附件。
* 所有的附件都應該是已經完全做好的（已經存儲在內存之中）。
* 每個緩衝都應該有同樣數目的樣本。

如果你不知道什麼是樣本也不用擔心，我們會在後面的教程中講到。

從上面的需求中你可以看到，我們需要為幀緩衝創建一些附件，還需要把這些附件附加到幀緩衝上。當我們做完所有上面提到的條件的時候我們就可以用  `glCheckFramebufferStatus` 帶上 `GL_FRAMEBUFFER` 這個參數來檢查是否真的成功做到了。然後檢查當前綁定的幀緩衝，返回了這些規範中的哪個值。如果返回的是 `GL_FRAMEBUFFER_COMPLETE`就對了：

```c++
if(glCheckFramebufferStatus(GL_FRAMEBUFFER) == GL_FRAMEBUFFER_COMPLETE)
  // Execute victory dance
```

後續所有渲染操作將渲染到當前綁定的幀緩衝的附加緩衝中，由於我們的幀緩衝不是默認的幀緩衝，渲染命令對窗口的視頻輸出不會產生任何影響。出於這個原因，它被稱為離屏渲染（off-screen rendering），就是渲染到一個另外的緩衝中。為了讓所有的渲染操作對主窗口產生影響我們必須通過綁定為0來使默認幀緩衝被激活：

```c++
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

當我們做完所有幀緩衝操作，不要忘記刪除幀緩衝對象：

```c++
glDeleteFramebuffers(1, &fbo);
```

現在在執行完成檢測前，我們需要把一個或更多的附件附加到幀緩衝上。一個附件就是一個內存地址，這個內存地址裡面包含一個為幀緩衝準備的緩衝，它可以是個圖像。當創建一個附件的時候我們有兩種方式可以採用：紋理或渲染緩衝（renderbuffer）對象。

## 紋理附件

當把一個紋理附加到幀緩衝上的時候，所有渲染命令會寫入到紋理上，就像它是一個普通的顏色/深度或者模板緩衝一樣。使用紋理的好處是，所有渲染操作的結果都會被儲存為一個紋理圖像，這樣我們就可以簡單的在著色器中使用了。

創建一個幀緩衝的紋理和創建普通紋理差不多：

```c++
GLuint texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);

glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 800, 600, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```

這裡主要的區別是我們把紋理的維度設置為屏幕大小（儘管不是必須的），我們還傳遞NULL作為紋理的data參數。對於這個紋理，我們只分配內存，而不去填充它。紋理填充會在渲染到幀緩衝的時候去做。同樣，要注意，我們不用關心環繞方式或者Mipmap，因為在大多數時候都不會需要它們的。

如果你打算把整個屏幕渲染到一個或大或小的紋理上，你需要用新的紋理的尺寸作為參數再次調用`glViewport`（要在渲染到你的幀緩衝之前做好），否則只有一小部分紋理或屏幕能夠繪製到紋理上。

現在我們已經創建了一個紋理，最後一件要做的事情是把它附加到幀緩衝上：

```c++
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,GL_TEXTURE_2D, texture, 0);
```

`glFramebufferTexture2D`函數需要傳入下列參數：

* target：我們所創建的幀緩衝類型的目標（繪製、讀取或兩者都有）。
* attachment：我們所附加的附件的類型。現在我們附加的是一個顏色附件。需要注意，最後的那個0是暗示我們可以附加1個以上顏色的附件。我們會在後面的教程中談到。
* textarget：你希望附加的紋理類型。
* texture：附加的實際紋理。
* level：Mipmap level。我們設置為0。

除顏色附件以外，我們還可以附加一個深度和一個模板紋理到幀緩衝對象上。為了附加一個深度緩衝，我們可以知道那個`GL_DEPTH_ATTACHMENT`作為附件類型。記住，這時紋理格式和內部格式類型（internalformat）就成了 `GL_DEPTH_COMPONENT`去反應深度緩衝的存儲格式。附加一個模板緩衝，你要使用 `GL_STENCIL_ATTACHMENT`作為第二個參數，把紋理格式指定為 `GL_STENCIL_INDEX`。

也可以同時附加一個深度緩衝和一個模板緩衝為一個單獨的紋理。這樣紋理的每32位數值就包含了24位的深度信息和8位的模板信息。為了把一個深度和模板緩衝附加到一個單獨紋理上，我們使用`GL_DEPTH_STENCIL_ATTACHMENT`類型配置紋理格式以包含深度值和模板值的結合物。下面是一個附加了深度和模板緩衝為單一紋理的例子：

```c++
glTexImage2D( GL_TEXTURE_2D, 0, GL_DEPTH24_STENCIL8, 800, 600, 0, GL_DEPTH_STENCIL, GL_UNSIGNED_INT_24_8, NULL );

glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_TEXTURE_2D, texture, 0);
```

### 渲染緩衝對象附件（Renderbuffer object attachments）

在介紹了幀緩衝的可行附件類型——紋理後，OpenGL引進了渲染緩衝對象（Renderbuffer objects），所以在過去那些美好時光裡紋理是附件的唯一可用的類型。和紋理圖像一樣，渲染緩衝對象也是一個緩衝，它可以是一堆字節、整數、像素或者其他東西。渲染緩衝對象的一大優點是，它以OpenGL原生渲染格式儲存它的數據，因此在離屏渲染到幀緩衝的時候，這些數據就相當於被優化過的了。

渲染緩衝對象將所有渲染數據直接儲存到它們的緩衝裡，而不會進行鍼對特定紋理格式的任何轉換，這樣它們就成了一種快速可寫的存儲介質了。然而，渲染緩衝對象通常是隻寫的，不能修改它們（就像獲取紋理，不能寫入紋理一樣）。可以用`glReadPixels`函數去讀取，函數返回一個當前綁定的幀緩衝的特定像素區域，而不是直接返回附件本身。

因為它們的數據已經是原生格式了，在寫入或把它們的數據簡單地到其他緩衝的時候非常快。當使用渲染緩衝對象時，像切換緩衝這種操作變得異常高速。我們在每個渲染迭代末尾使用的那個`glfwSwapBuffers`函數，同樣以渲染緩衝對象實現：我們簡單地寫入到一個渲染緩衝圖像，最後交換到另一個裡。渲染緩衝對象對於這種操作來說很完美。

創建一個渲染緩衝對象和創建幀緩衝代碼差不多：

```c++
GLuint rbo;
glGenRenderbuffers(1, &rbo);
```

相似地，我們打算把渲染緩衝對象綁定，這樣所有後續渲染緩衝操作都會影響到當前的渲染緩衝對象：

```c++
glBindRenderbuffer(GL_RENDERBUFFER, rbo);
```

由於渲染緩衝對象通常是隻寫的，它們經常作為深度和模板附件來使用，由於大多數時候，我們不需要從深度和模板緩衝中讀取數據，但仍關心深度和模板測試。我們就需要有深度和模板值提供給測試，但不需要對這些值進行採樣（sample），所以深度緩衝對象是完全符合的。當我們不去從這些緩衝中採樣的時候，渲染緩衝對象通常很合適，因為它們等於是被優化過的。

調用`glRenderbufferStorage`函數可以創建一個深度和模板渲染緩衝對象：

```c
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 800, 600);
```

創建一個渲染緩衝對象與創建紋理對象相似，不同之處在於這個對象是專門被設計用於圖像的，而不是通用目的的數據緩衝，比如紋理。這裡我們選擇`GL_DEPTH24_STENCIL8`作為內部格式，它同時代表24位的深度和8位的模板緩衝。

最後一件還要做的事情是把幀緩衝對象附加上：

```c
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);
```

在幀緩衝項目中，渲染緩衝對象可以提供一些優化，但更重要的是知道何時使用渲染緩衝對象，何時使用紋理。通常的規則是，如果你永遠都不需要從特定的緩衝中進行採樣，渲染緩衝對象對特定緩衝是更明智的選擇。如果哪天需要從比如顏色或深度值這樣的特定緩衝採樣數據的話，你最好還是使用紋理附件。從執行效率角度考慮，它不會對效率有太大影響。

### 渲染到紋理

現在我們知道了（一些）幀緩衝如何工作的，是時候把它們用起來了。我們會把場景渲染到一個顏色紋理上，這個紋理附加到一個我們創建的幀緩衝上，然後把紋理繪製到一個簡單的四邊形上，這個四邊形鋪滿整個屏幕。輸出的圖像看似和沒用幀緩衝一樣，但是這次，它其實是直接打印到了一個單獨的四邊形上面。為什麼這很有用呢？下一部分我們會看到原因。

第一件要做的事情是創建一個幀緩衝對象，並綁定它，這比較明瞭：

```c++
GLuint framebuffer;
glGenFramebuffers(1, &framebuffer);
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
```

下一步我們創建一個紋理圖像，這是我們將要附加到幀緩衝的顏色附件。我們把紋理的尺寸設置為窗口的寬度和高度，並保持數據未初始化：

```c++
// Generate texture
GLuint texColorBuffer;
glGenTextures(1, &texColorBuffer);
glBindTexture(GL_TEXTURE_2D, texColorBuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 800, 600, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR );
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
glBindTexture(GL_TEXTURE_2D, 0);

// Attach it to currently bound framebuffer object
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texColorBuffer, 0);
```

我們同樣打算要讓OpenGL確定可以進行深度測試（模板測試，如果你用的話）所以我們必須還要確保向幀緩衝中添加一個深度（和模板）附件。由於我們只採樣顏色緩衝，並不採樣其他緩衝，我們可以創建一個渲染緩衝對象來達到這個目的。記住，當你不打算從指定緩衝採樣的的時候，它們是一個不錯的選擇。

創建一個渲染緩衝對象不太難。唯一一件要記住的事情是，我們正在創建的是一個渲染緩衝對象的深度和模板附件。我們把它的內部給事設置為`GL_DEPTH24_STENCIL8`，對於我們的目的來說這個精確度已經足夠了。

```c++
GLuint rbo;
glGenRenderbuffers(1, &rbo);
glBindRenderbuffer(GL_RENDERBUFFER, rbo);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 800, 600);  
glBindRenderbuffer(GL_RENDERBUFFER, 0);
```

我們為渲染緩衝對象分配了足夠的內存空間以後，我們可以解綁渲染緩衝。

接著，在做好幀緩衝之前，還有最後一步，我們把渲染緩衝對象附加到幀緩衝的深度和模板附件上：

```c++
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);
```

然後我們要檢查幀緩衝是否真的做好了，如果沒有，我們就打印一個錯誤消息。

```c++
if(glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
 cout << "ERROR::FRAMEBUFFER:: Framebuffer is not complete!" << endl;
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

還要保證解綁幀緩衝，這樣我們才不會意外渲染到錯誤的幀緩衝上。

現在幀緩衝做好了，我們要做的全部就是渲染到幀緩衝上，而不是綁定到幀緩衝對象的默認緩衝。餘下所有命令會影響到當前綁定的幀緩衝上。所有深度和模板操作同樣會從當前綁定的幀緩衝的深度和模板附件中讀取，當然，得是在它們可用的情況下。如果你遺漏了比如深度緩衝，所有深度測試就不會工作，因為當前綁定的幀緩衝裡沒有深度緩衝。

所以，為把場景繪製到一個單獨的紋理，我們必須以下面步驟來做：

1. 使用新的綁定為激活幀緩衝的幀緩衝，像往常那樣渲染場景。
2. 綁定到默認幀緩衝。
3. 繪製一個四邊形，讓它平鋪到整個屏幕上，用新的幀緩衝的顏色緩衝作為他的紋理。

我們使用在深度測試教程中同一個場景進行繪製，但是這次使用老氣橫秋的[箱子紋理](http://learnopengl.com/img/textures/container.jpg)。

為了繪製四邊形我們將會創建新的著色器。我們不打算引入任何花哨的變換矩陣，因為我們只提供已經是標準化設備座標的[頂點座標](http://learnopengl.com/code_viewer.php?code=advanced/framebuffers_quad_vertices)，所以我們可以直接把它們作為頂點著色器的輸出。頂點著色器看起來像這樣：

```c++
#version 330 core
layout (location = 0) in vec2 position;
layout (location = 1) in vec2 texCoords;

out vec2 TexCoords;

void main()
{
    gl_Position = vec4(position.x, position.y, 0.0f, 1.0f);
    TexCoords = texCoords;
}
```

沒有花哨的地方。片段著色器更簡潔，因為我們做的唯一一件事是從紋理採樣：

```c++
#version 330 core
in vec2 TexCoords;
out vec4 color;

uniform sampler2D screenTexture;

void main()
{
    color = texture(screenTexture, TexCoords);
}
```

接著需要你為屏幕上的四邊形創建和配置一個VAO。渲染迭代中幀緩衝處理會有下面的結構：

```c++
// First pass
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // We're not using stencil buffer now
glEnable(GL_DEPTH_TEST);
DrawScene();

// Second pass
glBindFramebuffer(GL_FRAMEBUFFER, 0); // back to default
glClearColor(1.0f, 1.0f, 1.0f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT);

screenShader.Use();  
glBindVertexArray(quadVAO);
glDisable(GL_DEPTH_TEST);
glBindTexture(GL_TEXTURE_2D, textureColorbuffer);
glDrawArrays(GL_TRIANGLES, 0, 6);
glBindVertexArray(0);
```

只有很少的事情要說明。第一，由於我們用的每個幀緩衝都有自己的一系列緩衝，我們打算使用`glClear`設置的合適的位（bits）來清空這些緩衝。第二，當渲染四邊形的時候，我們關閉深度測試，因為我們不關係深度測試，我們繪製的是一個簡單的四邊形；當我們繪製普通場景時我們必須再次開啟深度測試。

這裡的確有很多地方會做錯，所以如果你沒有獲得任何輸出，嘗試排查任何可能出現錯誤的地方，再次閱讀教程中相關章節。如果每件事都做對了就一定能成功，你將會得到這樣的輸出：

![](http://learnopengl.com/img/advanced/framebuffers_screen_texture.png)

左側展示了和深度測試教程中一樣的輸出結果，但是這次卻是渲染到一個簡單的四邊形上的。如果我們以線框方式顯示的話，那麼顯然，我們只是繪製了一個默認幀緩衝中單調的四邊形。

你可以[從這裡得到應用的源碼](http://learnopengl.com/code_viewer.php?code=advanced/framebuffers_screen_texture)。

然而這有什麼好處呢？好處就是我們現在可以自由的獲取已經渲染場景中的任何像素，然後把它當作一個紋理圖像了，我們可以在片段著色器中創建一些有意思的效果。所有這些有意思的效果統稱為後處理特效。


### 後處理

現在，整個場景渲染到了一個單獨的紋理上，我們可以創建一些有趣的效果，只要簡單操縱紋理數據就能做到。這部分，我們會向你展示一些流行的後處理特效，以及怎樣添加一些創造性去創建出你自己的特效。

### 反相

我們已經取得了渲染輸出的每個顏色，所以在片段著色器裡返回這些顏色的反色並不難。我們得到屏幕紋理的顏色，然後用1.0減去它：

```c++
void main()
{
    color = vec4(vec3(1.0 - texture(screenTexture, TexCoords)), 1.0);
}
```

雖然反相是一種相對簡單的後處理特效，但是已經很有趣了：

![image description](http://learnopengl.com/img/advanced/framebuffers_grayscale.png)

整個場景現在的顏色都反轉了，只需在著色器中寫一行代碼就能做到，酷吧？

### 灰度

另一個有意思的效果是移除所有除了黑白灰以外的顏色作用，是整個圖像成為黑白的。實現它的簡單的方式是獲得所有顏色元素，然後將它們平均化：

```c++
void main()
{
    color = texture(screenTexture, TexCoords);
    float average = (color.r + color.g + color.b) / 3.0;
    color = vec4(average, average, average, 1.0);
}
```
這已經創造出很讚的效果了，但是人眼趨向於對綠色更敏感，對藍色感知比較弱，所以為了獲得更精確的符合人體物理的結果，我們需要使用加權通道：

```c++
void main()
{
    color = texture(screenTexture, TexCoords);
    float average = 0.2126 * color.r + 0.7152 * color.g + 0.0722 * color.b;
    color = vec4(average, average, average, 1.0);
}
```

![](http://learnopengl.com/img/advanced/framebuffers_grayscale.png)

### Kernel effects

在單獨紋理圖像上進行後處理的另一個好處是我們可以從紋理的其他部分進行採樣。比如我們可以從當前紋理值的周圍採樣多個紋理值。創造性地把它們結合起來就能創造出有趣的效果了。

kernel是一個長得有點像一個小矩陣的數值數組，它中間的值中心可以映射到一個像素上，這個像素和這個像素周圍的值再乘以kernel，最後再把結果相加就能得到一個值。所以，我們基本上就是給當前紋理座標加上一個它四周的偏移量，然後基於kernel把它們結合起來。下面是一個kernel的例子：

![](http://learnopengl-cn.readthedocs.org/zh/latest/img/05_05framebuffers_ kernel_sample.png)

這個kernel表示一個像素周圍八個像素乘以2，它自己乘以-15。這個例子基本上就是把周圍像素乘上2，中間像素去乘以一個比較大的負數來進行平衡。

!!! Important

    你在網上能找到的kernel的例子大多數都是所有值加起來等於1，如果加起來不等於1就意味著這個紋理值比原來更大或者更小了。

kernel對於後處理來說非常管用，因為用起來簡單。網上能找到有很多實例，為了能用上kernel我們還得改改片段著色器。這裡假設每個kernel都是3×3（實際上大多數都是3×3）：

```c++
const float offset = 1.0 / 300;  

void main()
{
    vec2 offsets[9] = vec2[](
        vec2(-offset, offset),  // top-left
        vec2(0.0f,    offset),  // top-center
        vec2(offset,  offset),  // top-right
        vec2(-offset, 0.0f),    // center-left
        vec2(0.0f,    0.0f),    // center-center
        vec2(offset,  0.0f),    // center-right
        vec2(-offset, -offset), // bottom-left
        vec2(0.0f,    -offset), // bottom-center
        vec2(offset,  -offset)  // bottom-right
    );

    float kernel[9] = float[](
        -1, -1, -1,
        -1,  9, -1,
        -1, -1, -1
    );

    vec3 sampleTex[9];
    for(int i = 0; i < 9; i++)
    {
        sampleTex[i] = vec3(texture(screenTexture, TexCoords.st + offsets[i]));
    }
    vec3 col;
    for(int i = 0; i < 9; i++)
        col += sampleTex[i] * kernel[i];

    color = vec4(col, 1.0);
}
```

在片段著色器中我們先為每個四周的紋理座標創建一個9個vec2偏移量的數組。偏移量是一個簡單的常數，你可以設置為自己喜歡的。接著我們定義kernel，這裡應該是一個銳化kernel，它通過一種有趣的方式從所有周邊的像素採樣，對每個顏色值進行銳化。最後，在採樣的時候我們把每個偏移量加到當前紋理座標上，然後用加在一起的kernel的值乘以這些紋理值。

這個銳化的kernel看起來像這樣：

![](http://learnopengl.com/img/advanced/framebuffers_sharpen.png)

這裡創建的有趣的效果就好像你的玩家吞了某種麻醉劑產生的幻覺一樣。

### Blur

創建模糊效果的kernel定義如下：

![](http://learnopengl-cn.readthedocs.org/zh/latest/img/05_05_blur_sample.png)

由於所有數值加起來的總和為16,簡單返回結合起來的採樣顏色是非常亮的,所以我們必須將kernel的每個值除以16.最終的kernel數組會是這樣的:

```c++
float kernel[9] = float[](
    1.0 / 16, 2.0 / 16, 1.0 / 16,
    2.0 / 16, 4.0 / 16, 2.0 / 16,
    1.0 / 16, 2.0 / 16, 1.0 / 16  
);
```

通過在像素著色器中改變kernel的float數組,我們就完全改變了之後的後處理效果.現在看起來會像是這樣:

![](http://learnopengl.com/img/advanced/framebuffers_blur.png)

這樣的模糊效果具有創建許多有趣效果的潛力.例如,我們可以隨著時間的變化改變模糊量,創建出類似於某人喝醉酒的效果,或者,當我們的主角摘掉眼鏡的時候增加模糊.模糊也能為我們在後面的教程中提供都顏色值進行平滑處理的能力.

你可以看到我們一旦擁有了這個kernel的實現以後,創建一個後處理特效就不再是一件難事.最後,我們再來討論一個流行的特效,以結束本節內容.

### 邊檢測

下面的邊檢測kernel與銳化kernel類似:

![](http://learnopengl-cn.readthedocs.org/zh/latest/img/05_05_Edge_detection.png)

這個kernel將所有的邊提高亮度,而對其他部分進行暗化處理,當我們值關心一副圖像的邊緣的時候,它非常有用.

![](http://learnopengl.com/img/advanced/framebuffers_edge_detection.png)

在一些像Photoshop這樣的軟件中使用這些kernel作為圖像操作工具/過濾器一點都不奇怪.因為掀開可以具有很強的平行處理能力，我們以實時進行鍼對每個像素的圖像操作便相對容易，圖像編輯工具因而更經常使用顯卡來進行圖像處理。

## 練習

* 你可以使用幀緩衝來創建一個後視鏡嗎?做到它,你必須繪製場景兩次:一次正常繪製,另一次攝像機旋轉180度後繪製.嘗試在你的顯示器頂端創建一個小四邊形,在上面應用後視鏡的鏡面紋理:[解決方案](http://learnopengl.com/code_viewer.php?code=advanced/framebuffers-exercise1),[視覺效果](http://learnopengl.com/img/advanced/framebuffers_mirror.png)
* 自己隨意調整一下kernel值,創建出你自己後處理特效.嘗試在網上搜索其他有趣的kernel.
