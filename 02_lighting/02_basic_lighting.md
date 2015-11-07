# 光照基礎

原文     | [Basic Lighting](http://learnopengl.com/#!Lighting/Basic-Lighting)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | Geequlim

現實世界的光照是極其複雜的，而且會受到諸多因素的影響，這是以目前我們所擁有的處理能力無法模擬的。因此OpenGL的光照僅僅使用了簡化的模型並基於對現實的估計來進行模擬，這樣處理起來會更容易一些，而且看起來也差不多一樣。這些光照模型都是基於我們對光的物理特性的理解。其中一個模型被稱為馮氏光照模型(Phong Lighting Model)。馮氏光照模型的主要結構由3個元素組成：環境(Ambient)、漫反射(Diffuse)和鏡面(Specular)光照。這些光照元素看起來像下面這樣：

![](http://learnopengl.com/img/lighting/basic_lighting_phong.png)

- 環境光照(Ambient Lighting)：即使在黑暗的情況下，世界上也仍然有一些光亮(月亮、一個來自遠處的光)，所以物體永遠不會是完全黑暗的。我們使用環境光照來模擬這種情況，也就是無論如何永遠都給物體一些顏色。
- 漫反射光照(Diffuse Lighting)：模擬一個發光物對物體的方向性影響(Directional Impact)。它是馮氏光照模型最顯著的組成部分。面向光源的一面比其他面會更亮。
- 鏡面光照(Specular Lighting)：模擬有光澤物體上面出現的亮點。鏡面光照的顏色，相比於物體的顏色更傾向於光的顏色。

為了創建有趣的視覺場景，我們希望模擬至少這三種光照元素。我們將以最簡單的一個開始：**環境光照**。

## 環境光照(Ambient Lighting)

光通常都不是來自於同一光源，而是來自散落於我們周圍的很多光源，即使它們可能並不是那麼顯而易見。光的一個屬性是，它可以向很多方向發散和反彈，所以光最後到達的地點可能並不是它所臨近的直射方向；光能夠像這樣**反射(Reflect)**到其他表面，一個物體的光照可能受到來自一個非直射的光源影響。考慮到這種情況的算法叫做**全局照明(Global Illumination)**算法，但是這種算法既開銷高昂又極其複雜。

因為我們不是複雜和昂貴算法的死忠粉絲，所以我們將會使用一種簡化的全局照明模型，叫做環境光照。如你在前面章節所見，我們使用一個(數值)很小的常量(光)顏色添加進物體**片段**(Fragment，指當前討論的光線在物體上的照射點)的最終顏色裡，這看起來就像即使沒有直射光源也始終存在著一些發散的光。

把環境光添加到場景裡非常簡單。我們用光的顏色乘以一個(數值)很小常量環境因子，再乘以物體的顏色，然後使用它作為片段的顏色：

```c++
void main()
{
    float ambientStrength = 0.1f;
    vec3 ambient = ambientStrength * lightColor;
    vec3 result = ambient * objectColor;
    color = vec4(result, 1.0f);
}
```

如果你現在運行你的程序，你會注意到馮氏光照的第一個階段已經應用到你的物體上了。這個物體非常暗，但不是完全的黑暗，因為我們應用了環境光照(注意發光立方體沒被環境光照影響是因為我們對它使用了另一個著色器)。它看起來應該像這樣：

![](http://learnopengl.com/img/lighting/ambient_lighting.png)

## 漫反射光照(Diffuse Lighting)

環境光本身不提供最明顯的光照效果，但是漫反射光照會對物體產生顯著的視覺影響。漫反射光使物體上與光線排布越近的片段越能從光源處獲得更多的亮度。為了更好的理解漫反射光照，請看下圖：

![](http://learnopengl.com/img/lighting/diffuse_light.png)

圖左上方有一個光源，它所發出的光線落在物體的一個片段上。我們需要測量這個光線與它所接觸片段之間的角度。如果光線垂直於物體表面，這束光對物體的影響會最大化(譯註：更亮)。為了測量光線和片段的角度，我們使用一個叫做法向量(Normal Vector)的東西，它是垂直於片段表面的一種向量(這裡以黃色箭頭表示)，我們在後面再講這個東西。兩個向量之間的角度就能夠根據點乘計算出來。

你可能記得在[變換](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/07%20Transformations/)那一節教程裡，我們知道兩個單位向量的角度越小，它們點乘的結果越傾向於1。當兩個向量的角度是90度的時候，點乘會變為0。這同樣適用於θ，θ越大，光對片段顏色的影響越小。

!!! Important

    注意，我們使用的是單位向量(Unit Vector，長度是1的向量)取得兩個向量夾角的餘弦值，所以我們需要確保所有的向量都被標準化，否則點乘返回的值就不僅僅是餘弦值了(如果你不明白，可以複習[變換](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/07%20Transformations/)那一節的點乘部分)。

點乘返回一個標量，我們可以用它計算光線對片段顏色的影響，基於不同片段所朝向光源的方向的不同，這些片段被照亮的情況也不同。

所以，我們需要些什麼來計算漫反射光照？

- 法向量：一個垂直於頂點表面的向量。
- 定向的光線：作為光的位置和片段的位置之間的向量差的方向向量。為了計算這個光線，我們需要光的位置向量和片段的位置向量。

### 法向量(Normal Vector)

法向量是垂直於頂點表面的(單位)向量。由於頂點自身並沒有表面(它只是空間中一個獨立的點)，我們利用頂點周圍的頂點計算出這個頂點的表面。我們能夠使用叉乘這個技巧為立方體所有的頂點計算出法線，但是由於3D立方體不是一個複雜的形狀，所以我們可以簡單的把法線數據手工添加到頂點數據中。更新的頂點數據數組可以在[這裡](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting_vertex_data)找到。試著去想象一下，這些法向量真的是垂直於立方體的各個面的表面的(一個立方體由6個面組成)。

因為我們向頂點數組添加了額外的數據，所以我們應該更新光照的頂點著色器：

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;
...
```

現在我們已經向每個頂點添加了一個法向量，已經更新了頂點著色器，我們還要更新頂點屬性指針(Vertex Attibute Pointer)。注意，發光物使用同樣的頂點數組作為它的頂點數據，然而發光物的著色器沒有使用新添加的法向量。我們不會更新發光物的著色器或者屬性配置，但是我們必須至少修改一下頂點屬性指針來適應新的頂點數組的大小：

```c++
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (GLvoid * )0);
glEnableVertexAttribArray(0);
```

我們只想使用每個頂點的前三個浮點數，並且我們忽略後三個浮點數，所以我們只需要把**步長**參數改成`GLfloat`尺寸的6倍就行了。

!!! Important

    發光物著色器頂點數據的不完全使用看起來有點低效，但是這些頂點數據已經從立方體對象載入到GPU的內存裡了，所以GPU內存不是必須再儲存新數據。相對於重新給發光物分配VBO，實際上卻是更高效了。

所有光照的計算需要在片段著色器裡進行，所以我們需要把法向量由頂點著色器傳遞到片段著色器。我們這麼做：

```c++
out vec3 Normal;
void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
    Normal = normal;
}
```

剩下要做的事情是，在片段著色器中定義相應的輸入值：

```c++
in vec3 Normal;
```

### 計算漫反射光照

每個頂點現在都有了法向量，但是我們仍然需要光的位置向量和片段的位置向量。由於光的位置是一個靜態變量，我們可以簡單的在片段著色器中把它聲明為uniform：

```c++
uniform vec3 lightPos;
```

然後再遊戲循環中(外面也可以，因為它不會變)更新uniform。我們使用在前面教程中聲明的`lightPos`向量作為光源位置：

```c++
GLint lightPosLoc = glGetUniformLocation(lightingShader.Program, “lightPos”);
glUniform3f(lightPosLoc, lightPos.x, lightPos.y, lightPos,z);
```

最後，我們還需要片段的位置(Position)。我們會在世界空間中進行所有的光照計算，因此我們需要一個在世界空間中的頂點位置。我們可以通過把頂點位置屬性乘以模型矩陣(Model Matrix,只用模型矩陣不需要用觀察和投影矩陣)來把它變換到世界空間座標。這個在頂點著色器中很容易完成，所以讓我們就聲明一個輸出(out)變量，然後計算它的世界空間空間座標：

```c++
out vec3 FragPos;
out vec3 Normal;

void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
    FragPos = vec3(model * vec4(position, 1.0f));
    Normal = normal;
}
```

最後，在片段著色器中添加相應的輸入變量。

```c++
in vec3 FragPos;
```

現在，所有需要的變量都設置好了，我們可以在片段著色器中開始光照的計算了。

我們需要做的第一件事是計算光源和片段位置之間的方向向量。前面提到，光的方向向量是光的位置向量與片段的位置向量之間的向量差。你可能記得，在變換教程中，我們簡單的通過兩個向量相減的方式計算向量差。我們同樣希望確保所有相關向量最後都轉換為單位向量，所以我們把法線和方向向量這個結果都進行標準化：

```c++
vec3 norm = normalize(Normal);
vec3 lightDir = normalize(lightPos - FragPos);
```

!!! Important

	當計算光照時我們通常不關心一個向量的“量”或它的位置，我們只關心它們的方向。所有的計算都使用單位向量完成，因為這會簡化了大多數計算(比如點乘)。所以當進行光照計算時，確保你總是對相關向量進行標準化，這樣它們才會保證自身為單位向量。忘記對向量進行標準化是一個十分常見的錯誤。

下一步，我們對`norm`和`lightDir`向量進行點乘，來計算光對當前片段的實際的散射影響。結果值再乘以光的顏色，得到散射因子。兩個向量之間的角度越大，散射因子就會越小：

```c++
float diff = max(dot(norm, lightDir), 0.0);
vec3 diffuse = diff * lightColor;
```

如果兩個向量之間的角度大於90度，點乘的結果就會變成負數，這樣會導致散射因子變為負數。為此，我們使用`max`函數返回兩個參數之間較大的參數，從而保證散射因子不會變成負數。負數的顏色是沒有實際定義的，所以最好避免它，除非你是那種古怪的藝術家。

既然我們有了一個環境光照顏色和一個散射光顏色，我們把它們相加，然後把結果乘以物體的顏色，來獲得片段最後的輸出顏色。

```c++
vec3 result = (ambient + diffuse) * objectColor;
color = vec4(result, 1.0f);
```

如果你的應用(和著色器)編譯成功了，你可能看到類似的輸出：

![](http://learnopengl.com/img/lighting/basic_lighting_diffuse.png)

你可以看到使用了散射光照，立方體看起來就真的像個立方體了。嘗試在你的腦中想象，通過移動正方體，法向量和光的方向向量之間的夾角增大，片段變得更暗。

如果你遇到很多困難，可以對比[完整的源代碼](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting_diffuse)以及[片段著色器](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting_diffuse&type=fragment)代碼。

### 最後一件事

現在我們已經把法向量從頂點著色器傳到了片段著色器。可是，目前片段著色器裡，我們都是在世界空間座標中進行計算的，所以，我們不是應該把法向量轉換為世界空間座標嗎？基本正確，但是這不是簡單地把它乘以一個模型矩陣就能搞定的。

首先，法向量只是一個方向向量，不能表達空間中的特定位置。同時，法向量沒有齊次座標(頂點位置中的w分量)。這意味著，平移不應該影響到法向量。因此，如果我們打算把法向量乘以一個模型矩陣，我們就要把模型矩陣左上角的3×3矩陣從模型矩陣中移除(譯註：所謂移除就是設置為0)，它是模型矩陣的平移部分(注意，我們也可以把法向量的w分量設置為0，再乘以4×4矩陣；同樣可以移除平移)。對於法向量，我們只能對它應用縮放(Scale)和旋轉(Rotation)變換。

其次，如果模型矩陣執行了不等比縮放，法向量就不再垂直於表面了，頂點就會以這種方式被改變了。因此，我們不能用這樣的模型矩陣去乘以法向量。下面的圖展示了應用了不等比縮放的矩陣對法向量的影響：

![](http://learnopengl.com/img/lighting/basic_lighting_normal_transformation.png)

無論何時當我們提交一個不等比縮放(注意：等比縮放不會破壞法線，因為法線的方向沒被改變，而法線的長度很容易通過標準化進行修復)，法向量就不會再垂直於它們的表面了，這樣光照會被扭曲。

修復這個行為的訣竅是使用另一個為法向量專門定製的模型矩陣。這個矩陣稱之為正規矩陣(Normal Matrix)，它是進行了一點線性代數操作移除了對法向量的錯誤縮放效果。如果你想知道這個矩陣是如何計算出來的，我建議看[這個文章](http://www.lighthouse3d.com/tutorials/glsl-tutorial/the-normal-matrix/)。

正規矩陣被定義為“模型矩陣左上角的逆矩陣的轉置矩陣”。真拗口，如果你不明白這是什麼意思，別擔心；我們還沒有討論逆矩陣(Inverse Matrix)和轉置矩陣(Transpose Matrix)。注意，定義正規矩陣的大多資源就像應用到模型觀察矩陣(Model-view Matrix)上的操作一樣，但是由於我們只在世界空間工作(而不是在觀察空間)，我們只使用模型矩陣。

在頂點著色器中，我們可以使用`inverse`和`transpose`函數自己生成正規矩陣，`inverse`和`transpose`函數對所有類型矩陣都有效。注意，我們也要把這個被處理過的矩陣強制轉換為3×3矩陣，這是為了保證它失去了平移屬性，之後它才能乘以法向量。

```c++
Normal = mat3(transpose(inverse(model))) * normal;
```

在環境光照部分，光照表現沒問題，這是因為我們沒有對物體本身執行任何縮放操作，因而不是非得使用正規矩陣不可，用模型矩陣乘以法線也沒錯。可是，如果你進行了不等比縮放，使用正規矩陣去乘以法向量就是必不可少的了。

!!! Attention

    對於著色器來說，逆矩陣也是一種開銷比較大的操作，因此，無論何時，在著色器中只要可能就應該儘量避免逆操作，因為它們必須為你場景中的每個頂點進行這樣的處理。以學習的目的這樣做很好，但是對於一個對於效率有要求的應用來說，在繪製之前，你最好用CPU計算出正規矩陣，然後通過uniform把值傳遞給著色器(和模型矩陣一樣)。

## 鏡面光照(Specular Lighting)

如果你還沒被這些光照計算搞得精疲力盡，我們就再把鏡面高光(Specular Highlight)加進來，這樣馮氏光照才算完整。

和環境光照一樣，鏡面光照同樣依據光的方向向量和物體的法向量，但是這次它也會依據觀察方向，例如玩家是從什麼方向看著這個片段的。鏡面光照根據光的反射特性。如果我們想象物體表面像一面鏡子一樣，那麼，無論我們從哪裡去看那個表面所反射的光，鏡面光照都會達到最大化。你可以從下面的圖片看到效果：

![](http://learnopengl.com/img/lighting/basic_lighting_specular_theory.png)

我們通過反射法向量周圍光的方向計算反射向量。然後我們計算反射向量和視線方向的角度，如果之間的角度越小，那麼鏡面光的作用就會越大。它的作用效果就是，當我們去看光被物體所反射的那個方向的時候，我們會看到一個高光。

觀察向量是鏡面光照的一個附加變量，我們可以使用觀察者世界空間位置(Viewer’s World Space Position)和片段的位置來計算。之後，我們計算鏡面光亮度，用它乘以光的顏色，在用它加上作為之前計算的光照顏色。

!!! Important

    我們選擇在世界空間(World Space)進行光照計算，但是大多數人趨向於在觀察空間(View Space)進行光照計算。在觀察空間計算的好處是，觀察者的位置總是(0, 0, 0)，所以這樣你直接就獲得了觀察者位置。可是，我發現出於學習的目的，在世界空間計算光照更符合直覺。如果你仍然希望在視野空間計算光照的話，那就使用觀察矩陣應用到所有相關的需要變換的向量(不要忘記，也要改變正規矩陣)。

為了得到觀察者的世界空間座標，我們簡單地使用攝像機對象的位置座標代替(它就是觀察者)。所以我們把另一個uniform添加到片段著色器，把相應的攝像機位置座標傳給片段著色器：

```c++
uniform vec3 viewPos;

GLint viewPosLoc = glGetUniformLocation(lightingShader.Program, "viewPos");
glUniform3f(viewPosLoc, camera.Position.x, camera.Position.y, camera.Position.z);
```

現在我們已經獲得所有需要的變量，可以計算高光亮度了。首先，我們定義一個鏡面強度(Specular Intensity)變量`specularStrength`，給鏡面高光一箇中等亮度顏色，這樣就不會產生過度的影響了。

```c++
float specularStrength = 0.5f;
```

如果我們把它設置為1.0f，我們會得到一個對於珊瑚色立方體來說過度明亮的鏡面亮度因子。下一節教程，我們會討論所有這些光照亮度的合理設置，以及它們是如何影響物體的。下一步，我們計算視線方向座標，和沿法線軸的對應的反射座標：

```
vec3 viewDir = normalize(viewPos - FragPos);
vec3 reflectDir = reflect(-lightDir, norm);
```

需要注意的是我們使用了`lightDir`向量的相反數。`reflect`函數要求的第一個是從光源指向片段位置的向量，但是`lightDir`當前是從片段指向光源的向量(由先前我們計算`lightDir`向量時，(減數和被減數)減法的順序決定)。為了保證我們得到正確的`reflect`座標，我們通過`lightDir`向量的相反數獲得它的方向的反向。第二個參數要求是一個法向量，所以我們提供的是已標準化的`norm`向量。

剩下要做的是計算鏡面亮度分量。下面的代碼完成了這件事：

```c++
float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);
vec3 specular = specularStrength * spec * lightColor;
```

我們先計算視線方向與反射方向的點乘(確保它不是負值)，然後得到它的32次冪。這個32是高光的**發光值(Shininess)**。一個物體的發光值越高，反射光的能力越強，散射得越少，高光點越小。在下面的圖片裡，你會看到不同發光值對視覺(效果)的影響：

![](http://learnopengl.com/img/lighting/basic_lighting_specular_shininess.png)

我們不希望鏡面成分過於顯眼，所以我們把指數設置為32。剩下的最後一件事情是把它添加到環境光顏色和散射光顏色裡，然後再乘以物體顏色：

```c++
vec3 result = (ambient + diffuse + specular) * objectColor;
color = vec4(result, 1.0f);
```

我們現在為馮氏光照計算了全部的光照元素。根據你的觀察點，你可以看到類似下面的畫面：

![](http://learnopengl.com/img/lighting/basic_lighting_specular.png)

你可以[在這裡找到完整源碼](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting_specular)，在這裡有[頂點](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting&type=vertex)和[片段](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting&type=fragment)著色器。

!!! Important

    早期的光照著色器，開發者在頂點著色器中實現馮氏光照。在頂點著色器中做這件事的優勢是，相比片段來說，頂點要少得多，因此會更高效，所以(開銷大的)光照計算頻率會更低。然而，頂點著色器中的顏色值是隻是頂點的顏色值，片段的顏色值是它與周圍的顏色值的插值。結果就是這種光照看起來不會非常真實，除非使用了大量頂點。

    ![](http://learnopengl.com/img/lighting/basic_lighting_gouruad.png)

    在頂點著色器中實現的馮氏光照模型叫做Gouraud著色，而不是馮氏著色。記住由於插值，這種光照連起來有點遜色。馮氏著色能產生更平滑的光照效果。

現在你可以看到著色器的強大之處了。只用很少的信息，著色器就能計算出光照，影響到為我們所有物體的片段顏色。[下一個教程](http://learnopengl-cn.readthedocs.org/zh/latest/02%20Lighting/03%20Materials/)，我們會更深入的研究光照模型，看看我們還能做些什麼。

## 練習

- 目前，我們的光源時靜止的，你可以嘗試使用`sin`和`cos`函數讓光源在場景中來回移動，此時再觀察光照效果能讓你更容易理解馮氏光照模型。[參考解答](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting-exercise1)。
- 嘗試使用不同的環境光、散射鏡面強度，觀察光照效果。改變鏡面光照的`shininess`因子試試。
- 在觀察空間中計算而不是世界空間馮氏光照：[參考解答](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting-exercise2)。
- 嘗試實現一個Gouraud光照來模擬馮氏光照，[參考解答](http://learnopengl.com/code_viewer.php?code=lighting/basic_lighting-exercise3)。
