本文作者JoeyDeVries，由Django翻譯自[http://learnopengl.com](http://learnopengl.com)

## 視差貼圖(Parallax Mapping)

視差貼圖技術和法線貼圖差不多，但它有著不同的原則。和法線貼圖一樣視差貼圖能夠極大提升表面細節，使之具有深度感。它也是利用了視錯覺，然而對深度有著更好的表達，與法線貼圖一起用能夠產生難以置信的效果。視差貼圖和光照無關，我在這裡是作為法線貼圖的技術延續來討論它的。需要注意的是在開始學習視差貼圖之前強烈建議先對法線貼圖，特別是切線空間有較好的理解。

視差貼圖屬於位移貼圖（譯註：displacement mapping也叫置換貼圖）技術的一種，它對根據儲存在紋理中的幾何信息對頂點進行位移或偏移。一種實現的方式是比如有1000個頂點，更具紋理中的數據對平面特定區域的頂點的高度進行位移。這樣的每個紋理像素包含了高度值紋理叫做高度貼圖。一張簡單的磚塊表面的告訴貼圖如下所示：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_height_map.png)

整個平面上的每個頂點都根據從高度貼圖採樣出來的高度值進行位移，根據材質的幾何屬性平坦的平面變換成凹凸不平的表面。例如一個平坦的平面利用上面的高度貼圖進行置換能得到以下結果：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_plane_heightmap.png)

置換頂點有一個問題就是平面必須由很多頂點組成才能獲得具有真實感的效果，否則看起來效果並不會很好。一個平坦的表面上有1000個頂點計算量太大了。我們能否不用這麼多的頂點就能取得相似的效果呢？事實上，上面的表面就是用6個頂點渲染出來的（兩個三角形）。上面的那個表面使用視差貼圖技術渲染，位移貼圖技術不需要額外的頂點數據來表達深度，它像法線貼圖一樣採用一種聰明的手段欺騙用戶的眼睛。

視差貼圖背後的思想是修改紋理座標使一個fragment的表面看起來比實際的更高或者更低，所有這些都根據觀察方向和高度貼圖。為了理解它如何工作，看看下面磚塊表面的圖片：

[](http://learnopengl.com/img/advanced-lighting/parallax_mapping_plane_height.png)

這裡粗糙的紅線代表高度貼圖中的數值的立體表達，向量V代表觀察方向。如果平面進行實際位移，觀察者會在點B看到表面。然而我們的平面沒有實際上進行位移，觀察方向將在點A與平面接觸。視差貼圖的目的是，在A位置上的fragment不再使用點A的紋理座標而是使用點B的。隨後我們用點B的紋理座標採樣，觀察者就像看到了點B一樣。

這個技巧就是描述如何從點A得到點B的紋理座標。視差貼圖嘗試通過對從fragment到觀察者的方向向量V進行縮放的方式解決這個問題，縮放的大小是A處fragment的高度。所以我們將V的長度縮放為高度貼圖在點A處H（A）採樣得來的值。下圖展示了經縮放得到的向量P：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_scaled_height.png)

我們隨後選出P以及這個向量與平面對齊的座標作為紋理座標的偏移量。這能工作是因為向量P是使用從高度貼圖得到的高度值計算出來的，所以一個fragment的高度越高位移的量越大。

這個技巧在大多數時候都沒問題，但點B是粗略估算得到的。當表面的高度變化很快的時候，看起來就不會真實，因為向量P最終不會和B接近，就像下圖這樣：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_incorrect_p.png)

視差貼圖的另一個問題是，當表面被任意旋轉以後很難指出從P獲取哪一個座標。我們在視差貼圖中使用了另一個座標空間，這個空間P向量的x和y元素總是與紋理表面對齊。如果你看了法線貼圖教程，你也許猜到了，我們實現它的方法，是的，我們還是在切線空間中實現視差貼圖。

將fragment到觀察者的向量V轉換到切線空間中，經變換的P向量的x和y元素將於表面的切線和副切線向量對齊。由於切線和副切線向量與表面紋理座標的方向相同，我們可以用P的x和y元素作為紋理座標的偏移量，這樣就不用考慮表面的方向了。

理論都有了，下面我們來動手實現視差貼圖。

 

### 視差貼圖

我們將使用一個簡單的2D平面，在把它發送給GPU之前我們先計算它的切線和副切線向量；和法線貼圖教程做的差不多。我們將向平面貼diffuse紋理、法線貼圖以及一個位移貼圖，你可以點擊鏈接下載。這個例子中我們將視差貼圖和法線貼圖連用。因為視差貼圖生成表面位移了的幻覺，當光照不匹配時這種幻覺就被破壞了。法線貼圖通常根據高度貼圖生成，法線貼圖和高度貼圖一起用能保證光照能和位移想匹配。

你可能已經注意到，上面鏈接上的那個位移貼圖和教程一開始的那個高度貼圖相比是顏色是相反的。這是因為使用反色高度貼圖（也叫深度貼圖）去模擬深度比模擬高度更容易。下圖反映了這個輕微的改變：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_depth.png)

我們再次獲得A和B，但是這次我們用向量V減去點A的紋理座標得到P。我們通過在著色器中用1.0減去採樣得到的高度貼圖中的值來取得深度值，而不再是高度值，或者簡單地在圖片編輯軟件中把這個紋理進行反色操作，就像我們對連接中的那個深度貼圖所做的一樣。

位移貼圖是在像素著色器中實現的，因為三角形表面的所有位移效果都不同。在像素著色器中我們將需要計算fragment到觀察者到方向向量V所以我們需要觀察者位置和在切線空間中的fragment位置。法線貼圖教程中我們已經有了一個頂點著色器，它把這些向量發送到切線空間，所以我們可以複製那個頂點著色器：

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 texCoords;
layout (location = 3) in vec3 tangent;
layout (location = 4) in vec3 bitangent;
 
out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} vs_out;
 
uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;
 
uniform vec3 lightPos;
uniform vec3 viewPos;
 
void main()
{
    gl_Position      = projection * view * model * vec4(position, 1.0f);
    vs_out.FragPos   = vec3(model * vec4(position, 1.0));   
    vs_out.TexCoords = texCoords;    
    
    vec3 T   = normalize(mat3(model) * tangent);
    vec3 B   = normalize(mat3(model) * bitangent);
    vec3 N   = normalize(mat3(model) * normal);
    mat3 TBN = transpose(mat3(T, B, N));
 
    vs_out.TangentLightPos = TBN * lightPos;
    vs_out.TangentViewPos  = TBN * viewPos;
    vs_out.TangentFragPos  = TBN * vs_out.FragPos;
}
```

在這裡有件事很重要，我們需要把position和在切線空間中的觀察者的位置viewPos發送給像素著色器。

在像素著色器中，我們實現視差貼圖的邏輯。像素著色器看起來會是這樣的：

```c++
#version 330 core
out vec4 FragColor;
 
in VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} fs_in;
 
uniform sampler2D diffuseMap;
uniform sampler2D normalMap;
uniform sampler2D depthMap;
  
uniform float height_scale;
  
vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir);
  
void main()
{           
    // Offset texture coordinates with Parallax Mapping
    vec3 viewDir   = normalize(fs_in.TangentViewPos - fs_in.TangentFragPos);
    vec2 texCoords = ParallaxMapping(fs_in.TexCoords,  viewDir);
 
    // then sample textures with new texture coords
    vec3 diffuse = texture(diffuseMap, texCoords);
    vec3 normal  = texture(normalMap, texCoords);
    normal = normalize(normal * 2.0 - 1.0);
    // proceed with lighting code
    [...]    
}
```

我們定義了一個叫做ParallaxMapping的函數，它把fragment的紋理座標作和切線空間中的fragment到觀察者的方向向量為輸入。這個函數返回經位移的紋理座標。然後我們使用這些經位移的紋理座標進行diffuse和法線貼圖的採樣。最後fragment的diffuse顏色和法線向量就正確的對應於表面的經位移的位置上了。

我們來看看ParallaxMapping函數的內部：

```c++
vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir)
{ 
    float height =  texture(depthMap, texCoords).r;    
    vec3 p = viewDir.xy / viewDir.z * (height * height_scale);
    return texCoords - p;    
}
```

這個相對簡單的函數是我們所討論過的內容的直接表述。我們用本來的紋理座標texCoords從高度貼圖中來採樣出當前fragment高度H(A)。然後計算出P，x和y元素在切線空間中，viewDir向量除以它的z元素，用fragment的高度對它進行縮放。我們同時引入額一個height_scale的uniform，來進行一些額外的控制，因為視差效果如果沒有一個縮放參數通常會過於強烈。然後我們用P減去紋理座標來獲得最終的經過位移紋理座標。

有一個地方需要注意，就是viewDir.xy除以viewDir.z那裡。因為viewDir向量是經過了標準化的，viewDir.z會在0.0到1.0之間的某處。當viewDir大致平行於表面時，它的z元素接近於0.0，除法會返回比viewDir垂直於表面的時候更大的P向量。所以基本上我們增加了P的大小，當以一個角度朝向一個表面相比朝向頂部時它對紋理座標會進行更大程度的縮放；這回在角上獲得更大的真實度。

有些人更喜歡在等式中不使用viewDir.z，因為普通的視差貼圖會在角上產生不想要的結果；這個技術叫做有偏移量限制的視差貼圖（Parallax Mapping with Offset Limiting）。選擇哪一個技術是個人偏好問題，但我傾向於普通的視差貼圖。

最後的紋理座標隨後被用來進行採樣（diffuse和法線）貼圖，下圖所展示的位移效果中height_scale等於1：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping.png)

這裡你會看到只用法線貼圖和與視差貼圖相結合的法線貼圖的不同之處。因為視差貼圖嘗試模擬深度，它實際上能夠根據你觀察它們的方向使磚塊疊加到其他磚塊上。

在視差貼圖的那個平面裡你仍然能看到在邊上有古怪的失真。原因是在平面的邊緣上，紋理座標超出了0到1的範圍進行採樣，根據紋理的環繞方式導致了不真實的結果。解決的方法是當它超出默認紋理座標範圍進行採樣的時候就丟棄這個fragment：

```c++
texCoords = ParallaxMapping(fs_in.TexCoords,  viewDir);
if(texCoords.x > 1.0 || texCoords.y > 1.0 || texCoords.x < 0.0 || texCoords.y < 0.0)
    discard;
```

丟棄了超出默認範圍的紋理座標的所有fragment，視差貼圖的表面邊緣給出了正確的結果。注意，這個技巧不能在所有類型的表面上都能工作，但是應用於平面上它還是能夠是平面看起來真的進行位移了：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_edge_fix.png)

你可以在這裡找到[源代碼](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/parallax_mapping)，以及[頂點](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/parallax_mapping&type=vertex)和[像素](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/parallax_mapping&type=fragment)著色器。

看起來不錯，運行起來也很快，因為我們只要給視差貼圖提供一個額外的紋理樣本就能工作。當從一個角度看過去的時候，會有一些問題產生（和法線貼圖相似），陡峭的地方會產生不正確的結果，從下圖你可以看到：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_issues.png)

問題的原因是這只是一個大致近似的視差映射。還有一些技巧讓我們在陡峭的高度上能夠獲得幾乎完美的結果，即使當以一定角度觀看的時候。例如，我們不再使用單一樣本，取而代之使用多樣本來找到最近點B會得到怎樣的結果？

 

### 陡峭視差映射（Steep Parallax Mapping）

陡峭視差映射是視差映射的擴展，原則是一樣的，但不是使用一個樣本而是多個樣本來確定向量P到B。它能得到更好的結果，它將總深度範圍分佈到同一個深度/高度的多個層中。從每個層中我們沿著P方向移動採樣紋理座標，直到我們找到了一個採樣得到的低於當前層的深度值的深度值。看看下面的圖片：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_steep_parallax_mapping_diagram.png)

我們從上到下遍歷深度層，我們把每個深度層和儲存在深度貼圖中的它的深度值進行對比。如果這個層的深度值小於深度貼圖的值，就意味著這一層的P向量部分在表面之下。我們繼續這個處理過程直到有一層的深度高於儲存在深度貼圖中的值：這個點就在（經過位移的）表面下方。

這個例子中我們可以看到第二層(D(2) = 0.73)的深度貼圖的值仍低於第二層的深度值0.4，所以我們繼續。下一次迭代，這一層的深度值0.6大於深度貼圖中採樣的深度值(D(3) = 0.37)。我們便可以假設第三層向量P是可用的位移幾何位置。我們可以用從向量P3的紋理座標偏移T3來對fragment的紋理座標進行位移。你可以看到隨著深度曾的增加精確度也在提高。

為實現這個技術，我們只需要改變ParallaxMapping函數，因為所有需要的變量都有了：

```c++
vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir)
{ 
    // number of depth layers
    const float numLayers = 10;
    // calculate the size of each layer
    float layerDepth = 1.0 / numLayers;
    // depth of current layer
    float currentLayerDepth = 0.0;
    // the amount to shift the texture coordinates per layer (from vector P)
    vec2 P = viewDir.xy * height_scale; 
    float deltaTexCoords = P / numLayers;
  
    [...]     
}
```

我們先定義層的數量，計算每一層的深度，最後計算紋理座標偏移，每一層我們必須沿著P的方向進行移動。

然後我們遍歷所有層，從上開始，知道找到小於這一層的深度值的深度貼圖值：

```c++
// get initial values
vec2  currentTexCoords     = texCoords;
float currentDepthMapValue = texture(depthMap, currentTexCoords).r;
  
while(currentLayerDepth < currentDepthMapValue)
{
    // shift texture coordinates along direction of P
    currentTexCoords -= deltaTexCoords;
    // get depthmap value at current texture coordinates
    currentDepthMapValue = texture(depthMap, currentTexCoords).r;  
    // get depth of next layer
    currentLayerDepth += layerDepth;  
}
 
return texCoords - currentTexCoords;

```

這裡我們循環每一層深度，直到沿著P向量找到第一個返回低於（位移）表面的深度的紋理座標偏移量。從fragment的紋理座標減去最後的偏移量，來得到最終的經過位移的紋理座標向量，這次就比傳統的視差映射更精確了。

有10個樣本磚牆從一個角度看上去就已經很好了，但是當有一個強前面展示的木製表面一樣陡峭的表面時，陡峭的視差映射的威力就顯示出來了：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_steep_parallax_mapping.png)

我們可以通過對視差貼圖的一個屬性的利用，對算法進行一點提升。當垂直看一個表面的時候紋理時位移比以一定角度看時的小。我們可以在垂直看時使用更少的樣本，以一定角度看時增加樣本數量：

```c++
const float minLayers = 8;
const float maxLayers = 32;
float numLayers = mix(maxLayers, minLayers, abs(dot(vec3(0.0, 0.0, 1.0), viewDir)));
```

這裡我們得到viewDir和正z方向的點乘，使用它的結果根據我們看向表面的角度調整樣本數量（注意正z方向等於切線空間中的表面的法線）。如果我們所看的方向平行於表面，我們就是用32層。

你可以在這裡找到最新的像素著色器代碼。這裡也提供木製玩具箱的表面貼圖：diffuse、法線、深度。

陡峭視差貼圖同樣有自己的問題。因為這個技術是基於有限的樣本數量的，我們會遇到鋸齒效果以及圖層之間有明顯的斷層：

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_steep_artifact.png)

我們可以通過增加樣本的方式減少這個問題，但是很快就會花費很多性能。有些旨在修復這個問題的方法：不適用低於表面的第一個位置，而是在兩個接近的深度層進行插值找出更匹配B的。

兩種最流行的解決方法叫做Relief Parallax Mapping和Parallax Occlusion Mapping，Relief Parallax Mapping更精確一些，但是比Parallax Occlusion Mapping性能開銷更多。因為Parallax Occlusion Mapping的效果和前者差不多但是效率更高，因此這種方式更經常使用，所以我們將在下面討論一下。

 

### Parallax Occlusion Mapping

Parallax Occlusion Mapping和陡峭視差映射的原則相同，但不是用觸碰的第一個深度層的紋理座標，而是在觸碰之前和之後，在深度層之間進行線性插值。我們根據表面的高度距離啷個深度層的深度層值的距離來確定線性插值的大小。看看下面的圖pain就能瞭解它是如何工作的：

[](http://learnopengl.com/img/advanced-lighting/parallax_mapping_parallax_occlusion_mapping_diagram.png)

你可以看到大部分和陡峭視差映射一樣，不一樣的地方是有個額外的步驟，兩個深度層的紋理座標圍繞著交叉點的線性插值。這也是近似的，但是比陡峭視差映射更精確。

Parallax Occlusion Mapping的代碼基於陡峭視差映射，所以並不難：

```c++
[...] // steep parallax mapping code here
  
// get texture coordinates before collision (reverse operations)
vec2 prevTexCoords = currentTexCoords + deltaTexCoords;
 
// get depth after and before collision for linear interpolation
float afterDepth  = currentDepthMapValue - currentLayerDepth;
float beforeDepth = texture(depthMap, prevTexCoords).r - currentLayerDepth + layerDepth;
 
// interpolation of texture coordinates
float weight = afterDepth / (afterDepth - beforeDepth);
vec2 finalTexCoords = prevTexCoords * weight + currentTexCoords * (1.0 - weight);
 
return finalTexCoords;
```

在對（位移的）表面幾何進行交叉，找到深度層之後，我們獲取交叉前的紋理座標。然後我們計算來自相應深度層的幾何之間的深度之間的距離，並在兩個值之間進行插值。線性插值的方式是在兩個層的紋理座標之間進行的基礎插值。函數最後返回最終的經過插值的紋理座標。

Parallax Occlusion Mapping的效果非常好，儘管有一些可以看到的輕微的不真實和鋸齒的問題，這仍是一個好交易，因為除非是放得非常大或者觀察角度特別陡，否則也看不到。

![](http://learnopengl.com/img/advanced-lighting/parallax_mapping_parallax_occlusion_mapping.png)

你可以在這裡找到[源代碼](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/parallax_mapping)，及其[頂點](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/parallax_mapping&type=vertex)和[像素](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/parallax_mapping_occlusion&type=fragment)著色器。

視差貼圖是提升場景細節非常好的技術，但是使用的時候還是要考慮到它會帶來一點不自然。大多數時候視差貼圖用在地面和牆壁表面，這種情況下查明表面的輪廓並不容易，同時觀察角度往往趨向於垂直於表面。這樣視差貼圖的不自然也就很難能被注意到了，對於提升物體的細節可以祈禱難以置信的效果。

 

### 附加資源

[Parallax Occlusion Mapping in GLSL](http://sunandblackcat.com/tipFullView.php?topicid=28)：sunandblackcat.com上的視差貼圖教程。

[How Parallax Displacement Mapping Works](https://www.youtube.com/watch?v=xvOT62L-fQI)：TheBennyBox的關於視差貼圖原理的視頻教程。