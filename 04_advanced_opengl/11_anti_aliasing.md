## 抗鋸齒(Anti Aliasing)

原文     | [Anti Aliasing](http://learnopengl.com/#!Advanced-OpenGL/Anti-Aliasing)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

在你的渲染大冒險中，你可能會遇到模型邊緣有鋸齒的問題。鋸齒邊出現的原因是由頂點數據像素化之後成為片段的方式所引起的。下面是一個簡單的立方體，它體現了鋸齒邊的效果：

![](http://learnopengl.com/img/advanced/anti_aliasing_aliasing.png)

也許不是立即可見的，如果你更近的看看立方體的邊，你就會發現鋸齒了。如果我們放大就會看到下面的情境：

![](http://learnopengl.com/img/advanced/anti_aliasing_zoomed.png)

這當然不是我們在最終版本的應用裡想要的效果。這個效果，很明顯能看到邊是由像素所構成的，這種現象叫做走樣（aliasing）。有很多技術能夠減少走樣，產生更平滑的邊緣，這些技術叫做反走樣技術(anti-aliasing,也被稱為抗鋸齒技術)。

首先，我們有一個叫做超級採樣抗鋸齒技術（super sample anti-aliasing SSAA），它暫時使用一個更高的解析度（以超級採樣方式）來渲染場景，當視頻輸出在幀緩衝中被更新時，解析度便降回原來的普通解析度。這個額外的解析度被用來防止鋸齒邊。雖然它確實為我們提供了一種解決走樣問題的方案，但卻由於必須繪製比平時更多的片段而降低了性能。所以這個技術只流行了一段時間。

這個技術的基礎上誕生了更為現代的技術，叫做多采樣抗鋸齒（multisample anti-aliasing）或叫MSAA，雖然它借用了SSAA的理念，但卻以更加高效的方式實現了它。這節教程我們會展開討論這個MSAA技術，它是OpenGL內建的。

## 多重採樣(Multisampling)

為了理解什麼是多重採樣，以及它是如何解決鋸齒問題的，我們先要更深入瞭解一個OpenGL光柵化的工作方式。

光柵化是你的最終的經處理的頂點和片段著色器之間的所有算法和處理的集合。光柵化將屬於一個基本圖形的所有頂點轉化為一系列片段。頂點座標理論上可以含有任何座標，但片段卻不是這樣，這是因為它們與你的窗口的解析度有關。幾乎永遠都不會有頂點座標和片段的一對一映射，所以光柵化必須以某種方式決定每個特定頂點最終結束於哪個片段/屏幕座標上。

![](http://learnopengl.com/img/advanced/anti_aliasing_rasterization.png)

這裡我們看到一個屏幕像素網格，每個像素中心包含一個採樣點（sample point），它被用來決定一個像素是否被三角形所覆蓋。紅色的採樣點如果被三角形覆蓋，那麼就會為這個被覆蓋像（屏幕）素生成一個片段。即使三角形覆蓋了部分屏幕像素，但是採樣點沒被覆蓋，這個像素仍然不會受到任何片段著色器影響到。

你可能已經明白走樣的原因來自何處了。三角形渲染後的版本最後在你的屏幕上是這樣的：

![](http://learnopengl.com/img/advanced/anti_aliasing_rasterization_filled.png)

由於屏幕像素總量的限制，有些邊上的像素能被渲染出來，而有些則不會。結果就是我們渲染出的基本圖形的非光滑邊緣產生了上圖的鋸齒邊。

多采樣所做的正是不再使用單一採樣點來決定三角形的覆蓋範圍，而是採用多個採樣點。我們不再使用每個像素中心的採樣點，取而代之的是4個子樣本（subsample），用它們來決定像素的覆蓋率。這意味著顏色緩衝的大小也由於每個像素的子樣本的增加而增加了。

![](http://learnopengl.com/img/advanced/anti_aliasing_sample_points.png)

左側的圖顯示了我們普通決定一個三角形的覆蓋範圍的方式。這個像素並不會運行一個片段著色器（這就仍保持空白），因為它的採樣點沒有被三角形所覆蓋。右邊的圖展示了多采樣的版本，每個像素包含4個採樣點。這裡我們可以看到只有2個採樣點被三角形覆蓋。

!!! Important

    採樣點的數量是任意的，更多的採樣點能帶來更精確的覆蓋率。

多采樣開始變得有趣了。2個子樣本被三角覆蓋，下一步是決定這個像素的顏色。我們原來猜測，我們會為每個被覆蓋的子樣本運行片段著色器，然後對每個像素的子樣本的顏色進行平均化。例子的那種情況，我們在插值的頂點數據的每個子樣本上運行片段著色器，然後將這些採樣點的最終顏色儲存起來。幸好，它不是這麼運作的，因為這等於說我們必須運行更多的片段著色器，會明顯降低性能。

MSAA的真正工作方式是，每個像素只運行一次片段著色器，無論多少子樣本被三角形所覆蓋。片段著色器運行著插值到像素中心的頂點數據，最後顏色被儲存近每個被覆蓋的子樣本中，每個像素的所有顏色接著將平均化，每個像素最終有了一個唯一顏色。在前面的圖片中4個樣本中只有2個被覆蓋，像素的顏色將以三角形的顏色進行平均化，顏色同時也被儲存到其他2個採樣點，最後生成的是一種淺藍色。

結果是，顏色緩衝中所有基本圖形的邊都生成了更加平滑的樣式。讓我們看看當再次決定前面的三角形覆蓋範圍時多樣本看起來是這樣的：

![](http://learnopengl.com/img/advanced/anti_aliasing_rasterization_samples.png)

這裡每個像素包含著4個子樣本（不相關的已被隱藏）藍色的子樣本是被三角形覆蓋了的，灰色的沒有被覆蓋。三角形內部區域中的所有像素都會運行一次片段著色器，它輸出的顏色被儲存到所有4個子樣本中。三角形的邊緣並不是所有的子樣本都會被覆蓋，所以片段著色器的結果僅儲存在部分子樣本中。根據被覆蓋子樣本的數量，最終的像素顏色由三角形顏色和其他子樣本所儲存的顏色所決定。

大致上來說，如果更多的採樣點被覆蓋，那麼像素的顏色就會更接近於三角形。如果我們用早期使用的三角形的顏色填充像素，我們會獲得這樣的結果：

![](http://learnopengl.com/img/advanced/anti_aliasing_rasterization_samples_filled.png)

對於每個像素來說，被三角形覆蓋的子樣本越少，像素受到三角形的顏色的影響也越少。現在三角形的硬邊被比實際顏色淺一些的顏色所包圍，因此觀察者從遠處看上去就比較平滑了。

不僅顏色值被多采樣影響，深度和模板測試也同樣使用了多采樣點。比如深度測試，頂點的深度值在運行深度測試前被插值到每個子樣本中，對於模板測試，我們為每個子樣本儲存模板值，而不是每個像素。這意味著深度和模板緩衝的大小隨著像素子樣本的增加也增加了。

到目前為止我們所討論的不過是多采樣發走樣工作的方式。光柵化背後實際的邏輯要比我們討論的複雜，但你現在可以理解多采樣抗鋸齒背後的概念和邏輯了。

## OpenGL中的MSAA

如果我們打算在OpenGL中使用MSAA，那麼我們必須使用一個可以為每個像素儲存一個以上的顏色值的顏色緩衝(因為多采樣需要我們為每個採樣點儲存一個顏色)。我們這就需要一個新的緩衝類型，它可以儲存要求數量的多重採樣樣本，它叫做**多樣本緩衝(multisample buffer)**。

多數窗口系統可以為我們提供一個多樣本緩衝，以代替默認的顏色緩衝。GLFW同樣給了我們這個功能，我們所要作的就是提示GLFW，我們希望使用一個帶有N個樣本的多樣本緩衝，而不是普通的顏色緩衝，這要在創建窗口前調用`glfwWindowHint`來完成：

```c++
glfwWindowHint(GLFW_SAMPLES, 4);
```

當我們現在調用`glfwCreateWindow`，用於渲染的窗口就被創建了，這次每個屏幕座標使用一個包含4個子樣本的顏色緩衝。這意味著所有緩衝的大小都增長4倍。

現在我們請求GLFW提供了多樣本緩衝，我們還要調用`glEnable`來開啟多采樣，參數是 `GL_MULTISAMPLE`。大多數OpenGL驅動，多采樣默認是開啟的，所以這個調用有點多餘，但通常記得開啟它是個好主意。這樣所有OpenGL實現的多采樣都開啟了。

```c++
glEnable(GL_MULTISAMPLE);
```

當默認幀緩衝有了多采樣緩衝附件的時候，我們所要做的全部就是調用 `glEnable`開啟多采樣。因為實際的多采樣算法在OpenGL驅動光柵化裡已經實現了，所以我們無需再做什麼了。如果我們現在來渲染教程開頭的那個綠色立方體，我們會看到邊緣變得平滑了：

![](http://learnopengl.com/img/advanced/anti_aliasing_multisampled.png)

這個箱子看起來平滑多了，在場景中繪製任何物體都可以利用這個技術。可以[從這裡找到](http://learnopengl.com/code_viewer.php?code=advanced/anti_aliasing_multisampling)這個簡單的例子。

## 離屏MSAA

因為GLFW負責創建多采樣緩衝，開啟MSAA非常簡單。如果我們打算使用我們自己的幀緩衝，來進行離屏渲染，那麼我們就必須自己生成多采樣緩衝了；現在我們需要自己負責創建多采樣緩衝。

有兩種方式可以創建多采樣緩衝，並使其成為幀緩衝的附件：紋理附件和渲染緩衝附件，和[幀緩衝教程](http://learnopengl-cn.readthedocs.org/zh/latest/04%20Advanced%20OpenGL/05%20Framebuffers/)裡討論過的普通的附件很相似。

### 多采樣紋理附件

為了創建一個支持儲存多采樣點的紋理，我們使用 `glTexImage2DMultisample`來替代 `glTexImage2D`，它的紋理目標是**`GL_TEXTURE_2D_MULTISAMPLE`**：

```c++
glBindTexture(GL_TEXTURE_2D_MULTISAMPLE, tex);
glTexImage2DMultisample(GL_TEXTURE_2D_MULTISAMPLE, samples, GL_RGB, width, height, GL_TRUE);
glBindTexture(GL_TEXTURE_2D_MULTISAMPLE, 0);
```

第二個參數現在設置了我們打算讓紋理擁有的樣本數。如果最後一個參數等於 **`GL_TRUE`**，圖像上的每一個紋理像素（texel）將會使用相同的樣本位置，以及同樣的子樣本數量。

為將多采樣紋理附加到幀緩衝上，我們使用`glFramebufferTexture2D`，不過這次紋理類型是**`GL_TEXTURE_2D_MULTISAMPLE`**：

```c++
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D_MULTISAMPLE, tex, 0);
```

當前綁定的幀緩衝現在有了一個紋理圖像形式的多采樣顏色緩衝。

### 多采樣渲染緩衝對象（Multisampled renderbuffer objects）

和紋理一樣，創建一個多采樣渲染緩衝對象不難。而且還很簡單，因為我們所要做的全部就是當我們指定渲染緩衝的內存的時候將`glRenderbuffeStorage`改為`glRenderbufferStorageMuiltisample`：

```c++
glRenderbufferStorageMultisample(GL_RENDERBUFFER, 4, GL_DEPTH24_STENCIL8, width, height);
```

有一樣東西在這裡有變化，就是緩衝目標後面那個額外的參數，我們將其設置為樣本數量，當前的例子中應該是4.

### 渲染到多采樣幀緩衝

渲染到多采樣幀緩衝對象是自動的。當我們繪製任何東西時，幀緩衝對象就綁定了，光柵化會對負責所有多采樣操作。我們接著得到了一個多采樣顏色緩衝，以及深度和模板緩衝。因為多采樣緩衝有點特別，我們不能為其他操作直接使用它們的緩衝圖像，比如在著色器中進行採樣。

一個多采樣圖像包含了比普通圖像更多的信息，所以我們需要做的是壓縮或還原圖像。還原一個多采樣幀緩衝，通常用`glBlitFramebuffer`來完成，它從一個幀緩衝中複製一個區域粘貼另一個裡面，同時也將任何多采樣緩衝還原。

`glBlitFramebuffer`把一個4屏幕座標源區域傳遞到一個也是4空間座標的目標區域。你可能還記得幀緩衝教程中，如果我們綁定到`GL_FRAMEBUFFER`，我們實際上就同時綁定到了讀和寫的幀緩衝目標。我們還可以通過`GL_READ_FRAMEBUFFER`和`GL_DRAW_FRAMEBUFFER`綁定到各自的目標上。`glBlitFramebuffer`函數從這兩個目標讀取，並決定哪一個是源哪一個是目標幀緩衝。接著我們就可以通過把圖像位塊傳送(Blitting)到默認幀緩衝裡，將多采樣幀緩衝輸出傳遞到實際的屏幕了：

```c++
glBindFramebuffer(GL_READ_FRAMEBUFFER, multisampledFBO);
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0);
glBlitFramebuffer(0, 0, width, height, 0, 0, width, height, GL_COLOR_BUFFER_BIT, GL_NEAREST);
```

如果我們渲染應用，我們將得到和沒用幀緩衝一樣的結果：一個綠色立方體，它使用MSAA顯示出來，但邊緣鋸齒明顯少了：

![](http://learnopengl.com/img/advanced/anti_aliasing_multisampled.png)

你可以[在這裡找到源代碼](http://learnopengl.com/code_viewer.php?code=advanced/anti_aliasing_framebuffers)。

但是如果我們打算使用一個多采樣幀緩衝的紋理結果來做這件事，就像後處理一樣會怎樣？我們不能在片段著色器中直接使用多采樣紋理。我們可以做的事情是把多緩衝位塊傳送(Blit)到另一個帶有非多采樣紋理附件的FBO中。之後我們使用這個普通的顏色附件紋理進行後處理，通過多采樣來對一個圖像渲染進行後處理效率很高。這意味著我們必須生成一個新的FBO，它僅作為一個將多采樣緩衝還原為一個我們可以在片段著色器中使用的普通2D紋理中介。偽代碼是這樣的：

```c++
GLuint msFBO = CreateFBOWithMultiSampledAttachments();
// Then create another FBO with a normal texture color attachment
...
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, screenTexture, 0);
...
while(!glfwWindowShouldClose(window))
{
    ...

    glBindFramebuffer(msFBO);
    ClearFrameBuffer();
    DrawScene();
    // Now resolve multisampled buffer(s) into intermediate FBO
    glBindFramebuffer(GL_READ_FRAMEBUFFER, msFBO);
    glBindFramebuffer(GL_DRAW_FRAMEBUFFER, intermediateFBO);
    glBlitFramebuffer(0, 0, width, height, 0, 0, width, height, GL_COLOR_BUFFER_BIT, GL_NEAREST);
    // Now scene is stored as 2D texture image, so use that image for post-processing
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    ClearFramebuffer();
    glBindTexture(GL_TEXTURE_2D, screenTexture);
    DrawPostProcessingQuad();  

    ...
}
```

如果我們實現幀緩衝教程中講的後處理代碼，我們就能創造出沒有鋸齒邊的所有效果很酷的後處理特效。使用模糊kernel過濾器，看起來會像這樣：

![](http://learnopengl.com/img/advanced/anti_aliasing_post_processing.png)

你可以[在這裡找到所有MSAA版本的後處理源碼](http://learnopengl.com/code_viewer.php?code=advanced/anti_aliasing_post_processing)。

!!! Important

    因為屏幕紋理重新變回了只有一個採樣點的普通紋理，有些後處理過濾器，比如邊檢測（edge-detection）將會再次導致鋸齒邊問題。為了修正此問題，之後你應該對紋理進行模糊處理，或者創建你自己的抗鋸齒算法。

當我們希望將多采樣和離屏渲染結合起來時，我們需要自己負責一些細節。所有細節都是值得付出這些額外努力的，因為多采樣可以明顯提升場景視頻輸出的質量。要注意，開啟多采樣會明顯降低性能，樣本越多越明顯。本文寫作時，MSAA4樣本很常用。

## 自定義抗鋸齒算法

可以直接把一個多采樣紋理圖像傳遞到著色器中，以取代必須先還原的方式。GLSL給我們一個選項來為每個子樣本進行紋理圖像採樣，所以我們可以創建自己的抗鋸齒算法，在比較大的圖形應用中，通常這麼做。

為獲取每個子樣本的顏色值，你必須將紋理uniform採樣器定義為sampler2DMS，而不是使用sampler2D：

```c++
uniform sampler2DMS screenTextureMS;
```

使用texelFetch函數，就可以獲取每個樣本的顏色值了：

```c++
vec4 colorSample = texelFetch(screenTextureMS, TexCoords, 3);  // 4th subsample
```

我們不會深究自定義抗鋸齒技術的創建細節，但是會給你自己去實現它提供一些提示。
