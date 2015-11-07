本文作者JoeyDeVries，由Meow J翻譯自[http://learnopengl.com](http://learnopengl.com/#!Advanced-Lighting/HDR)

## HDR

一般來說，當存儲在幀緩衝(Framebuffer)中時，亮度和顏色的值是默認被限制在0.0到1.0之間的. 這個看起來無辜的語句使我們一直將亮度與顏色的值設置在這個範圍內，嘗試著與場景契合. 這樣是能夠運行的，也能給出還不錯的效果. 但是如果我們遇上了一個特定的區域，其中有多個亮光源使這些數值總和超過了1.0，又會發生什麼呢? 答案是這些片段中超過1.0的亮度或者顏色值會被約束在1.0, 從而導致場景混成一片，難以分辨:

![](http://learnopengl.com/img/advanced-lighting/hdr_clamped.png)

這是由於大量片段的顏色值都非常接近1.0，在很大一個區域內每一個亮的片段都有相同的白色. 這損失了很多的細節，使場景看起來非常假.

解決這個問題的一個方案是減小光源的強度從而保證場景內沒有一個片段亮於1.0. 然而這並不是一個好的方案，因為你需要使用不切實際的光照參數. 一個更好的方案是讓顏色暫時超過1.0，然後將其轉換至0.0到1.0的區間內，從而防止損失細節.

顯示器被限制為只能顯示值為0.0到1.0間的顏色，但是在光照方程中卻沒有這個限制. 通過使片段的顏色超過1.0，我們有了一個更大的顏色範圍，這也被稱作HDR(High Dynamic Range, 高動態範圍). 有了HDR，亮的東西可以變得非常亮，暗的東西可以變得非常暗，而且充滿細節.

HDR原本只是被運用在攝影上，攝影師對同一個場景採取不同曝光拍多張照片，捕捉大範圍的色彩值. 這些圖片被合成為HDR圖片，從而綜合不同的曝光等級使得大範圍的細節可見. 看下面這個例子，左邊這張圖片在被光照亮的區域充滿細節，但是在黑暗的區域就什麼都看不見了；但是右邊這張圖的高曝光卻可以讓之前看不出來的黑暗區域顯現出來.

![](http://learnopengl.com/img/advanced-lighting/hdr_image.png)

這與我們眼睛工作的原理非常相似，也是HDR渲染的基礎. 當光線很弱的啥時候，人眼會自動調整從而使過暗和過亮的部分變得更清晰，就像人眼有一個能自動根據場景亮度調整的自動曝光滑塊.

HDR渲染和其很相似，我們允許用更大範圍的顏色值渲染從而獲取大範圍的黑暗與明亮的場景細節，最後將所有HDR值轉換成在[0.0, 1.0]範圍的LDR(Low Dynamic Range,低動態範圍). 轉換HDR值到LDR值得過程叫做色調映射(Tone Mapping)，現在現存有很多的色調映射算法，這些算法致力於在轉換過程中保留儘可能多的HDR細節. 這些色調映射算法經常會包含一個選擇性傾向黑暗或者明亮區域的參數.

在實時渲染中，HDR不僅允許我們超過LDR的範圍[0.0, 1.0]與保留更多的細節，同時還讓我們能夠根據光源的**真實**強度指定它的強度. 比如太陽有比閃光燈之類的東西更高的強度，那麼我們為什麼不這樣子設置呢?(比如說設置一個10.0的漫亮度) 這允許我們用更現實的光照參數恰當地配置一個場景的光照，而這在LDR渲染中是不能實現的，因為他們會被上限約束在1.0.

因為顯示器只能顯示在0.0到1.0範圍之內的顏色，我們肯定要做一些轉換從而使得當前的HDR顏色值符合顯示器的範圍. 簡單地取平均值重新轉換這些顏色值並不能很好的解決這個問題，因為明亮的地方會顯得更加顯著. 我們能做的是用一個不同的方程與/或曲線來轉換這些HDR值到LDR值，從而給我們對於場景的亮度完全掌控，這就是之前說的色調變換，也是HDR渲染的最終步驟.

### 浮點幀緩衝(Floating Point Framebuffers)

在實現HDR渲染之前，我們首先需要一些防止顏色值在每一個片段著色器運行後被限制約束的方法. 當幀緩衝使用了一個標準化的定點格式(像`GL_RGB`)為其顏色緩衝的內部格式，OpenGL會在將這些值存入幀緩衝前自動將其約束到0.0到1.0之間. 這一操作對大部分幀緩衝格式都是成立的，除了專門用來存放被拓展範圍值的浮點格式.

當一個幀緩衝的顏色緩衝的內部格式被設定成了`GL_RGB16F`, `GL_RGBA16F`, `GL_RGB32F` 或者`GL_RGBA32F`時，這些幀緩衝被叫做浮點幀緩衝(Floating Point Framebuffer)，浮點幀緩衝可以存儲超過0.0到1.0範圍的浮點值，所以非常適合HDR渲染.

想要創建一個浮點幀緩衝，我們只需要改變顏色緩衝的內部格式參數就行了（注意`GL_FLOAT`參數):

```c++
glBindTexture(GL_TEXTURE_2D, colorBuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_FLOAT, NULL);  
```

默認的幀緩衝默認一個顏色分量只佔用8位(bits). 當使用一個使用32位每顏色分量的浮點幀緩衝時(使用`GL_RGB32F` 或者`GL_RGBA32F`)，我們需要四倍的內存來存儲這些顏色. 所以除非你需要一個非常高的精確度，32位不是必須的，使用`GLRGB16F`就足夠了.

有了一個帶有浮點顏色緩衝的幀緩衝，我們可以放心渲染場景到這個幀緩衝中. 在這個教程的例子當中，我們先渲染一個光照的場景到浮點幀緩衝中，之後再在一個填充屏幕的四方形上應用這個幀緩衝的顏色緩衝，代碼會是這樣子:

```c++
glBindFramebuffer(GL_FRAMEBUFFER, hdrFBO);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);  
    // [...] 渲染(光照的)場景
glBindFramebuffer(GL_FRAMEBUFFER, 0);

// 現在使用一個不同的著色器將HDR顏色緩衝渲染至2D填充屏幕的四方形上
hdrShader.Use();
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, hdrColorBufferTexture);
RenderQuad();
```

這裡場景的顏色值存在一個可以包含任意顏色值的浮點顏色緩衝中，值可能是超過1.0的. 這個簡單的演示中，場景被創建為一個被拉伸的立方體通道和四個點光源，其中一個非常亮的在隧道的盡頭:

```c++
std::vector<glm::vec3> lightColors;
lightColors.push_back(glm::vec3(200.0f, 200.0f, 200.0f));
lightColors.push_back(glm::vec3(0.1f, 0.0f, 0.0f));
lightColors.push_back(glm::vec3(0.0f, 0.0f, 0.2f));
lightColors.push_back(glm::vec3(0.0f, 0.1f, 0.0f));  
```

渲染至浮點幀緩衝和渲染至一個普通的幀緩衝是一樣的. 新的東西就是這個的`hdrShader`的片段著色器，用來渲染最終擁有浮點顏色緩衝紋理的2D四方形. 我們來定義一個簡單的直通片段著色器(Pass-through Fragment Shader):

```c++
#version 330 core
out vec4 color;
in vec2 TexCoords;

uniform sampler2D hdrBuffer;

void main()
{             
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
    color = vec4(hdrColor, 1.0);
}  
```

這裡我們直接採樣了浮點顏色緩衝並將其作為片段著色器的輸出. 然而，這個2D四方形的輸出是被直接渲染到默認的幀緩衝中，導致所有片段著色器的輸出值被約束在0.0到1.0間，儘管我們已經有了一些存在浮點顏色紋理的值超過了1.0.

![](http://learnopengl.com/img/advanced-lighting/hdr_direct.png)

很明顯，在隧道盡頭的強光的值被約束在1.0，因為一大塊區域都是白色的，過程中超過1.0的地方損失了所有細節. 因為我們直接轉換HDR值到LDR值，這就像我們根本就沒有應用HDR一樣. 為了修復這個問題我們需要做的是無損轉化所有浮點顏色值回0.0-1.0範圍中. 我們需要應用到色調映射.

### 色調映射(Tone Mapping)

色調映射是一個損失很小的轉換浮點顏色值至我們所需的LDR[0.0, 1.0]範圍內的過程，通常會伴有特定的風格的色平衡(Stylistic Color Balance).

最簡單的色調映射算法是Reinhard色調映射，它涉及到分散整個HDR顏色值到LDR顏色值上，所有的值都有對應. Reinhard色調映射算法平均得將所有亮度值分散到LDR上. 我們將Reinhard色調映射應用到之前的片段著色器上，並且為了更好的測量加上一個Gamma校正過濾(包括SRGB紋理的使用):

```c++
void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
  
    // Reinhard色調映射
    vec3 mapped = hdrColor / (hdrColor + vec3(1.0));
    // Gamma校正
    mapped = pow(mapped, vec3(1.0 / gamma));
  
    color = vec4(mapped, 1.0);
}   
```

有了Reinhard色調映射的應用，我們不再會在場景明亮的地方損失細節. 當然，這個算法是傾向明亮的區域的，暗的區域會不那麼精細也不那麼有區分度.

![](http://learnopengl.com/img/advanced-lighting/hdr_reinhard.png)

現在你可以看到在隧道的盡頭木頭紋理變得可見了. 用了這個非常簡單地色調映射算法，我們可以合適的看到存在浮點幀緩衝中整個範圍的HDR值，給我們對於無損場景光照精確的控制.

另一個有趣的色調映射應用是曝光(Exposure)參數的使用. 你可能還記得之前我們在介紹裡講到的，HDR圖片包含在不同曝光等級的細節. 如果我們有一個場景要展現日夜交替，我們當然會在白天使用低曝光，在夜間使用高曝光，就像人眼調節方式一樣. 有了這個曝光參數，我們可以去設置可以同時在白天和夜晚不同光照條件工作的光照參數，我們只需要調整曝光參數就行了.

一個簡單的曝光色調映射算法會像這樣:

```c++
uniform float exposure;

void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
  
    // 曝光色調映射
    vec3 mapped = vec3(1.0) - exp(-hdrColor * exposure);
    // Gamma校正 
    mapped = pow(mapped, vec3(1.0 / gamma));
  
    color = vec4(mapped, 1.0);
}  
```

在這裡我們將`exposure`定義為默認為1.0的`uniform`，從而允許我們更加精確設定我們是要注重黑暗還是明亮的區域的HDR顏色值. 舉例來說，高曝光值會使隧道的黑暗部分顯示更多的細節，然而低曝光值會顯著減少黑暗區域的細節，但允許我們看到更多明亮區域的細節. 下面這組圖片展示了在不同曝光值下的通道:

![](http://learnopengl.com/img/advanced-lighting/hdr_exposure.png)

這個圖片清晰地展示了HDR渲染的優點. 通過改變曝光等級，我們可以看見場景的很多細節，而這些細節可能在LDR渲染中都被丟失了. 比如說隧道盡頭，在正常曝光下木頭結構隱約可見，但用低曝光木頭的花紋就可以清晰看見了. 對於近處的木頭花紋來說，在高曝光下會能更好的看見.

你可以在[這裡](http://learnopengl.com/code_viewer.php?code=advanced-lighting/hdr "這裡")找到這個演示的源碼和HDR的[頂點](http://learnopengl.com/code_viewer.php?code=advanced-lighting/hdr&type=vertex "頂點")和[片段](http://learnopengl.com/code_viewer.php?code=advanced-lighting/hdr&type=fragment "片段")著色器.

### HDR拓展

在這裡展示的兩個色調映射算法僅僅是大量(更先進)的色調映射算法中的一小部分，這些算法各有長短.一些色調映射算法傾向於特定的某種顏色/強度，也有一些算法同時顯示低於高曝光顏色從而能夠顯示更加多彩和精細的圖像. 也有一些技巧被稱作自動曝光調整(Automatic Exposure Adjustment)或者叫人眼適應(Eye Adaptation)技術，它能夠檢測前一幀場景的亮度並且緩慢調整曝光參數模仿人眼使得場景在黑暗區域逐漸變亮或者在明亮區域逐漸變暗.

HDR渲染的真正優點在龐大和複雜的場景中應用複雜光照算法會被顯示出來，但是出於教學目的創建這樣複雜的演示場景是很困難的，這個教程用的場景是很小的，而且缺乏細節. 但是如此簡單的演示也是能夠顯示出HDR渲染的一些優點: 在明亮和黑暗區域無細節損失，因為它們可以由色調映射重新獲取；多個光照的疊加不會導致亮度被約束的區域；光照可以被設定為他們原來的亮度而不是被LDR值限定. 而且，HDR渲染也使一些有趣的效果更加可行和真實; 其中一個效果叫做泛光(Bloom)，我們將在下一節討論他.

### 附加資源

- [如果泛光效果不被應用HDR渲染還有好處嗎?](http://gamedev.stackexchange.com/questions/62836/does-hdr-rendering-have-any-benefits-if-bloom-wont-be-applied): 一個StackExchange問題，其中有一個答案非常詳細地解釋HDR渲染的好處.
- [什麼是色調映射? 它與HDR有什麼聯繫?](http://photo.stackexchange.com/questions/7630/what-is-tone-mapping-how-does-it-relate-to-hdr): 另一個非常有趣的答案，用了大量圖片解釋色調映射.