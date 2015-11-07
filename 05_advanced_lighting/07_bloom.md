本文作者JoeyDeVries，由Django翻譯自[http://learnopengl.com](http://learnopengl.com)

## 泛光(Bloom)

明亮的光源和區域經常很難向觀察者表達出來，因為監視器的亮度範圍是有限的。一種區分明亮光源的方式是使它們在監視器上發出光芒，光源的的光芒向四周發散。這樣觀察者就會產生光源或亮區的確是強光區。（譯註：這個問題的提出簡單來說是為了解決這樣的問題：例如有一張在陽光下的白紙，白紙在監視器上顯示出是出白色，而前方的太陽也是純白色的，所以基本上白紙和太陽就是一樣的了，給太陽加一個光暈，這樣太陽看起來似乎就比白紙更亮了）

光暈效果可以使用一個後處理特效bloom來實現。bloom使所有明亮區域產生光暈效果。下面是一個使用了和沒有使用光暈的對比（圖片生成自虛幻引擎）：

![](http://learnopengl.com/img/advanced-lighting/bloom_example.png)

Bloom是我們能夠注意到一個明亮的物體真的有種明亮的感覺。bloom可以極大提升場景中的光照效果，並提供了極大的效果提升，儘管做到這一切只需一點改變。

Bloom和HDR結合使用效果很好。常見的一個誤解是HDR和bloom是一樣的，很多人認為兩種技術是可以互換的。但是它們是兩種不同的技術，用於各自不同的目的上。可以使用默認的8位精確度的幀緩衝，也可以在不使用bloom效果的時候，使用HDR。只不過在有了HDR之後再實現bloom就更簡單了。

為實現bloom，我們像平時那樣渲染一個有光場景，提取出場景的HDR顏色緩衝以及只有這個場景明亮區域可見的圖片。被提取的帶有亮度的圖片接著被模糊，結果被添加到HDR場景上面。

我們來一步一步解釋這個處理過程。我們在場景中渲染一個帶有4個立方體形式不同顏色的明亮的光源。帶有顏色的發光立方體的亮度在1.5到15.0之間。如果我們將其渲染至HDR顏色緩衝，場景看起來會是這樣的：

![](http://learnopengl.com/img/advanced-lighting/bloom_scene.png)

我們得到這個HDR顏色緩衝紋理，提取所有超出一定亮度的fragment。這樣我們就會獲得一個只有fragment超過了一定閾限的顏色區域：

![](http://learnopengl.com/img/advanced-lighting/bloom_extracted.png)

我們將這個超過一定亮度閾限的紋理進行模糊。bloom效果的強度很大程度上被模糊過濾器的範圍和強度所決定。

![](http://learnopengl.com/img/advanced-lighting/bloom_blurred.png)

最終的被模糊化的紋理就是我們用來獲得發出光暈效果的東西。這個已模糊的紋理要添加到原來的HDR場景紋理的上部。因為模糊過濾器的應用明亮區域發出光暈，所以明亮區域在長和寬上都有所擴展。

![](http://learnopengl.com/img/advanced-lighting/bloom_small.png)

bloom本身並不是個複雜的技術，但很難獲得正確的效果。它的品質很大程度上取決於所用的模糊過濾器的質量和類型。簡單的改改模糊過濾器就會極大的改變bloom效果的品質。

下面這幾步就是bloom後處理特效的過程，它總結了實現bloom所需的步驟。

![](http://learnopengl.com/img/advanced-lighting/bloom_steps.png)

首先我們需要根據一定的閾限提取所有明亮的顏色。我們先來做這件事。

 

### 提取亮色

第一步我們要從渲染出來的場景中提取兩張圖片。我們可以渲染場景兩次，每次使用一個不同的不同的著色器渲染到不同的幀緩衝中，但我們可以使用一個叫做MRT（Multiple Render Targets多渲染目標）的小技巧，這樣我們就能定義多個像素著色器了；有了它我們還能夠在一個單獨渲染處理中提取頭兩個圖片。在像素著色器的輸出前，我們指定一個佈局location標識符，這樣我們便可控制一個像素著色器寫入到哪個顏色緩衝：

```c++
layout (location = 0) out vec4 FragColor;
layout (location = 1) out vec4 BrightColor;
```

只有我們真的具有多個地方可寫的時候這才能工作。使用多個像素著色器輸出的必要條件是，有多個顏色緩衝附加到了當前綁定的幀緩衝對象上。你可能從幀緩衝教程那裡回憶起，當把一個紋理鏈接到幀緩衝的顏色緩衝上時，我們可以指定一個顏色附件。直到現在，我們一直使用著GL_COLOR_ATTACHMENT0，但通過使用GL_COLOR_ATTACHMENT1，我們可以得到一個附加了兩個顏色緩衝的幀緩衝對象：

```c++
// Set up floating point framebuffer to render scene to
GLuint hdrFBO;
glGenFramebuffers(1, &hdrFBO);
glBindFramebuffer(GL_FRAMEBUFFER, hdrFBO);
GLuint colorBuffers[2];
glGenTextures(2, colorBuffers);
for (GLuint i = 0; i < 2; i++)
{
    glBindTexture(GL_TEXTURE_2D, colorBuffers[i]);
    glTexImage2D(
        GL_TEXTURE_2D, 0, GL_RGB16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_FLOAT, NULL
    );
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    // attach texture to framebuffer
    glFramebufferTexture2D(
        GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0 + i, GL_TEXTURE_2D, colorBuffers[i], 0
    );
}
```

我們需要顯式告知OpenGL我們正在通過glDrawBuffers渲染到多個顏色緩衝，否則OpenGL只會渲染到幀緩衝的第一個顏色附件，而忽略所有其他的。我們可以通過傳遞多個顏色附件的枚舉來做這件事，我們以下面的操作進行渲染：

```c++
GLuint attachments[2] = { GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1 };
glDrawBuffers(2, attachments);
```

當渲染到這個幀緩衝中的時候，一個著色器使用一個佈局location修飾符，那麼fragment就會用相應的顏色緩衝就會被用來渲染。這很棒，因為這樣省去了我們為提取明亮區域的額外渲染步驟，因為我們現在可以直接從將被渲染的fragment提取出它們：

```c++
#version 330 core
layout (location = 0) out vec4 FragColor;
layout (location = 1) out vec4 BrightColor;
 
[...]
 
void main()
{            
    [...] // first do normal lighting calculations and output results
    FragColor = vec4(lighting, 1.0f);
    // Check whether fragment output is higher than threshold, if so output as brightness color
    float brightness = dot(FragColor.rgb, vec3(0.2126, 0.7152, 0.0722));
    if(brightness > 1.0)
        BrightColor = vec4(FragColor.rgb, 1.0);
}
```

這裡我們先正常計算光照，將其傳遞給第一個像素著色器的輸出變量FragColor。然後我們使用當前儲存在FragColor的東西來決定它的亮度是否超過了一定閾限。我們通過恰當地將其轉為灰度的方式計算一個fragment的亮度，如果它超過了一定閾限，我們就把顏色輸出到第二個顏色緩衝，那裡保存著所有亮部；渲染髮光的立方體也是一樣的。

這也說明了為什麼bloom在HDR基礎上能夠運行得很好。因為HDR中，我們可以將顏色值指定超過1.0這個默認的範圍，我們能夠得到對一個圖像中的亮度的更好的控制權。沒有HDR我們必須將閾限設置為小於1.0的數，雖然可行，但是亮部很容易變得很多，這就導致光暈效果過重。

有了兩個顏色緩衝，我們就有了一個正常場景的圖像和一個提取出的亮區的圖像；這些都在一個渲染步驟中完成。

![](http://learnopengl.com/img/advanced-lighting/bloom_attachments.png)

有了一個提取出的亮區圖像，我們現在就要把這個圖像進行模糊處理。我們可以使用幀緩衝教程後處理部分的那個簡單的盒子過濾器，但不過我們最好還是使用一個更高級的更漂亮的模糊過濾器：高斯模糊。

### 高斯模糊

在後處理教程那裡，我們採用的模糊是一個圖像中所有周圍像素的均值，它的確為我們提供了一個簡易實現的模糊，但是效果並不好。高斯模糊基於高斯曲線，高斯曲線通常被描述為一個鐘形曲線，中間的值達到最大化，隨著距離的增加，兩邊的值不斷減少。高斯曲線在數學上有不同的形式，但是通常是這樣的形狀：

![](http://learnopengl.com/img/advanced-lighting/bloom_gaussian.png)

高斯曲線在它的中間處的面積最大，使用它的值作為權重使得近處的樣本擁有最大的優先權。比如，如果我們從fragment的32×32的四方形區域採樣，這個權重隨著和fragment的距離變大逐漸減小；通常這會得到更好更真實的模糊效果，這種模糊叫做高斯模糊。

要實現高斯模糊過濾我們需要一個二維四方形作為權重，從這個二維高斯曲線方程中去獲取它。然而這個過程有個問題，就是很快會消耗極大的性能。以一個32×32的模糊kernel為例，我們必須對每個fragment從一個紋理中採樣1024次！

幸運的是，高斯方程有個非常巧妙的特性，它允許我們把二維方程分解為兩個更小的方程：一個描述水平權重，另一個描述垂直權重。我們首先用水平權重在整個紋理上進行水平模糊，然後在經改變的紋理上進行垂直模糊。利用這個特性，結果是一樣的，但是可以節省難以置信的性能，因為我們現在只需做32+32次採樣，不再是1024了！這叫做兩步高斯模糊。

![](http://learnopengl.com/img/advanced-lighting/bloom_gaussian_two_pass.png)

這意味著我們如果對一個圖像進行模糊處理，至少需要兩步，最好使用幀緩衝對象做這件事。具體來說，我們將實現像乒乓球一樣的幀緩衝來實現高斯模糊。它的意思是，有一對兒幀緩衝，我們把另一個幀緩衝的顏色緩衝放進當前的幀緩衝的顏色緩衝中，使用不同的著色效果渲染指定的次數。基本上就是不斷地切換幀緩衝和紋理去繪製。這樣我們先在場景紋理的第一個緩衝中進行模糊，然後在把第一個幀緩衝的顏色緩衝放進第二個幀緩衝進行模糊，接著，將第二個幀緩衝的顏色緩衝放進第一個，循環往復。

在我們研究幀緩衝之前，先討論高斯模糊的像素著色器：

```c++
#version 330 core
out vec4 FragColor;
in vec2 TexCoords;
 
uniform sampler2D image;
 
uniform bool horizontal;
 
uniform float weight[5] = float[] (0.227027, 0.1945946, 0.1216216, 0.054054, 0.016216);
 
void main()
{             
    vec2 tex_offset = 1.0 / textureSize(image, 0); // gets size of single texel
    vec3 result = texture(image, TexCoords).rgb * weight[0]; // current fragment's contribution
    if(horizontal)
    {
        for(int i = 1; i < 5; ++i)
        {
            result += texture(image, TexCoords + vec2(tex_offset.x * i, 0.0)).rgb * weight[i];
            result += texture(image, TexCoords - vec2(tex_offset.x * i, 0.0)).rgb * weight[i];
        }
    }
    else
    {
        for(int i = 1; i < 5; ++i)
        {
            result += texture(image, TexCoords + vec2(0.0, tex_offset.y * i)).rgb * weight[i];
            result += texture(image, TexCoords - vec2(0.0, tex_offset.y * i)).rgb * weight[i];
        }
    }
    FragColor = vec4(result, 1.0);
}
```

這裡我們使用一個比較小的高斯權重做例子，每次我們用它來指定當前fragment的水平或垂直樣本的特定權重。你會發現我們基本上是將模糊過濾器根據我們在uniform變量horizontal設置的值分割為一個水平和一個垂直部分。通過用1.0除以紋理的大小（從textureSize得到一個vec2）得到一個紋理像素的實際大小，以此作為偏移距離的根據。

我們為圖像的模糊處理創建兩個基本的幀緩衝，每個只有一個顏色緩衝紋理：

```c++
GLuint pingpongFBO[2];
GLuint pingpongBuffer[2];
glGenFramebuffers(2, pingpongFBO);
glGenTextures(2, pingpongColorbuffers);
for (GLuint i = 0; i < 2; i++)
{
    glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[i]);
    glBindTexture(GL_TEXTURE_2D, pingpongBuffer[i]);
    glTexImage2D(
        GL_TEXTURE_2D, 0, GL_RGB16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_FLOAT, NULL
    );
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glFramebufferTexture2D(
        GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, pingpongBuffer[i], 0
    );
}
```

得到一個HDR紋理後，我們用提取出來的亮區紋理填充一個幀緩衝，然後對其模糊處理10次（5次垂直5次水平）：

```c++
GLboolean horizontal = true, first_iteration = true;
GLuint amount = 10;
shaderBlur.Use();
for (GLuint i = 0; i < amount; i++)
{
    glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[horizontal]); 
    glUniform1i(glGetUniformLocation(shaderBlur.Program, "horizontal"), horizontal);
    glBindTexture(
        GL_TEXTURE_2D, first_iteration ? colorBuffers[1] : pingpongBuffers[!horizontal]
    ); 
    RenderQuad();
    horizontal = !horizontal;
    if (first_iteration)
        first_iteration = false;
}
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

每次循環我們根據我們打算渲染的是水平還是垂直來綁定兩個緩衝其中之一，而將另一個綁定為紋理進行模糊。第一次迭代，因為兩個顏色緩衝都是空的所以我們隨意綁定一個去進行模糊處理。重複這個步驟10次，亮區圖像就進行一個重複5次的高斯模糊了。這樣我們可以對任意圖像進行任意次模糊處理；高斯模糊循環次數越多，模糊的強度越大。

通過對提取亮區紋理進行5次模糊，我們就得到了一個正確的模糊的場景亮區圖像。

![](http://learnopengl.com/img/advanced-lighting/bloom_blurred_large.png)

bloom的最後一步是把模糊處理的圖像和場景原來的HDR紋理進行結合。

 

### 把兩個紋理混合

有了場景的HDR紋理和模糊處理的亮區紋理，我們只需把它們結合起來就能實現bloom或稱光暈效果了。最終的像素著色器（大部分和HDR教程用的差不多）要把兩個紋理混合：

```c++
#version 330 core
out vec4 FragColor;
in vec2 TexCoords;
 
uniform sampler2D scene;
uniform sampler2D bloomBlur;
uniform float exposure;
 
void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(scene, TexCoords).rgb;      
    vec3 bloomColor = texture(bloomBlur, TexCoords).rgb;
    hdrColor += bloomColor; // additive blending
    // tone mapping
    vec3 result = vec3(1.0) - exp(-hdrColor * exposure);
    // also gamma correct while we're at it       
    result = pow(result, vec3(1.0 / gamma));
    FragColor = vec4(result, 1.0f);
}
```

要注意的是我們要在應用色調映射之前添加bloom效果。這樣添加的亮區的bloom，也會柔和轉換為LDR，光照效果相對會更好。

把兩個紋理結合以後，場景亮區便有了合適的光暈特效：

![](http://learnopengl.com/img/advanced-lighting/bloom.png)

有顏色的立方體看起來彷彿更亮，它向外發射光芒，的確是一個更好的視覺效果。這個場景比較簡單，所以bloom效果不算十分令人矚目，但在更好的場景中合理配置之後效果會有巨大的不同。你可以在這裡找到這個簡單的例子的源碼，以及模糊的頂點和像素著色器、立方體的像素著色器、後處理的頂點和像素著色器。

這個教程我們只是用了一個相對簡單的高斯模糊過濾器，它在每個方向上只有5個樣本。通過沿著更大的半徑或重複更多次數的模糊，進行採樣我們就可以提升模糊的效果。因為模糊的質量與bloom效果的質量正相關，提升模糊效果就能夠提升bloom效果。有些提升將模糊過濾器與不同大小的模糊kernel或採用多個高斯曲線來選擇性地結合權重結合起來使用。來自Kalogirou和EpicGames的附加資源討論瞭如何通過提升高斯模糊來顯著提升bloom效果。

 

### 附加資源

* [Efficient Gaussian Blur with linear sampling](http://rastergrid.com/blog/2010/09/efficient-gaussian-blur-with-linear-sampling/)：非常詳細地描述了高斯模糊，以及如何使用OpenGL的雙線性紋理採樣提升性能。
* [Bloom Post Process Effect](https://udn.epicgames.com/Three/Bloom.html)：來自Epic Games關於通過對權重的多個高斯曲線結合來提升bloom效果的文章。
* [How to do good bloom for HDR rendering](http://kalogirou.net/2006/05/20/how-to-do-good-bloom-for-hdr-rendering/)：Kalogirou的文章描述瞭如何使用更好的高斯模糊算法來提升bloom效果。