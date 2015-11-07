# 投光物

原文     | [Light casters](http://www.learnopengl.com/#!Lighting/Light-casters)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

我們目前使用的所有光照都來自於一個單獨的光源，這是空間中的一個點。它的效果不錯，但是在真實世界，我們有多種類型的光，它們每個表現都不同。一個光源把光投射到物體上，叫做投光。這個教程裡我們討論幾種不同的投光類型。學習模擬不同的光源是你未來豐富你的場景的另一個工具。

我們首先討論定向光(directional light)，接著是作為之前學到知識的擴展的點光(point light)，最後我們討論聚光燈(Spotlight)。下面的教程我們會把這幾種不同的光類型整合到一個場景中。

## 定向光(Directional Light)

當一個光源很遠的時候，來自光源的每條光線接近於平行。這看起來就像所有的光線來自於同一個方向，無論物體和觀察者在哪兒。當一個光源被設置為無限遠時，它被稱為定向光(也被成為平行光)，因為所有的光線都有著同一個方向；它會獨立於光源的位置。

我們知道的定向光源的一個好例子是，太陽。太陽和我們不是無限遠，但它也足夠遠了，在計算光照的時候，我們感覺它就像無限遠。在下面的圖片裡，來自於太陽的所有的光線都被定義為平行光：

![](http://www.learnopengl.com/img/lighting/light_casters_directional.png)

因為所有的光線都是平行的，對於場景中的每個物體光的方向都保持一致，物體和光源的位置保持怎樣的關係都無所謂。由於光的方向向量保持一致，光照計算會和場景中的其他物體相似。

我們可以通過定義一個光的方向向量，來模擬這樣一個定向光，而不是使用光的位置向量。著色器計算保持大致相同的要求，這次我們直接使用光的方向向量來代替用`lightDir`向量和`position`向量的計算：

```c++
struct Light
{
    // vec3 position; // 現在不在需要光源位置了，因為它是無限遠的
    vec3 direction;
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};
...
void main()
{
    vec3 lightDir = normalize(-light.direction);
    ...
}
```

注意，我們首先忽略light.direction向量。目前我們使用的光照計算需要光的方向作為一個來自片段朝向的光源的方向，但是人們通常更習慣定義一個定向光作為一個全局方向，它從光源發出。所以我們必須忽略全局光的方向向量來改變它的方向；它現在是一個方向向量指向光源。同時，確保對向量進行標準化處理，因為假定輸入的向量就是一個單位向量是不明智的。

作為結果的`lightDir`向量被使用在`diffuse`和`specular`計算之前。

為了清晰地強調一個定向光對所有物體都有同樣的影響，我們再次訪問[座標系教程](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/08%20Coordinate%20Systems/)結尾部分的箱子場景。例子裡我們先定義10個不同的箱子位置，為每個箱子生成不同的模型矩陣，每個模型矩陣包含相應的本地到世界變換：

```c++
for(GLuint i = 0; i < 10; i++)
{
    model = glm::mat4();
    model = glm::translate(model, cubePositions[i]);
    GLfloat angle = 20.0f * i;
    model = glm::rotate(model, angle, glm::vec3(1.0f, 0.3f, 0.5f));
    glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
    glDrawArrays(GL_TRIANGLES, 0, 36);
}
```

同時，不要忘記定義光源的方向（注意，我們把方向定義為：從光源處發出的方向；在下面，你可以快速看到光的方向的指向）：

```c++
GLint lightDirPos = glGetUniformLocation(lightingShader.Program, "light.direction");
glUniform3f(lightDirPos, -0.2f, -1.0f, -0.3f);
```

!!! Important
    
    我們已經把光的位置和方向向量傳遞為vec3，但是有些人去想更喜歡把所有的向量設置為vec4.當定義位置向量為vec4的時候，把w元素設置為1.0非常重要，這樣平移和投影才會合理的被應用。然而，當定義一個方向向量為vec4時，我們並不想讓平移發揮作用（因為它們除了代表方向，其他什麼也不是）所以我們把w元素設置為0.0。

    方向向量被表示為：vec4(0.2f, 1.0f, 0.3f, 0.0f)。這可以作為簡單檢查光的類型的方法：你可以檢查w元素是否等於1.0，查看我們現在所擁有的光的位置向量，w是否等於0.0，我們有一個光的方向向量，所以根據那個調整計算方法：
    
    ```c++
    if(lightVector.w == 0.0) // 請留意浮點數錯誤
        // 執行定向光照計算
    
    else if(lightVector.w == 1.0)
        // 像上一個教程一樣執行頂點光照計算
    ```
    
    有趣的事實：這就是舊OpenGL（固定函數式）決定一個光源是一個定向光還是位置光源，更具這個修改它的光照。
    
如果你現在編譯應用，飛躍場景，它看起來像有一個太陽一樣的光源，把光拋到物體身上。你可以看到`diffuse`和`specular`元素都對該光源進行反射了，就像天空上有一個光源嗎？看起來就像這樣：

![](http://www.learnopengl.com/img/lighting/light_casters_directional_light.png)

你可以在這裡獲得[應用的所有代碼](http://learnopengl.com/code_viewer.php?code=lighting/light_casters_directional)，這裡是[頂點](http://learnopengl.com/code_viewer.php?code=lighting/lighting_maps&type=vertex)和[片段](http://learnopengl.com/code_viewer.php?code=lighting/light_casters_directional&type=fragment)著色器代碼。



## 定點光(Point Light)

定向光作為全局光可以照亮整個場景，這非常棒，但是另一方面除了定向光，我們通常也需要幾個定點光，在場景裡發亮。點光是一個在時間裡有位置的光源，它向所有方向發光，光線隨距離增加逐漸變暗。想象燈泡和火炬作為投光物，它們可以扮演點光的角色。

![](http://www.learnopengl.com/img/lighting/light_casters_point.png)

之前的教程我們已經使用了（最簡單的）點光。我們有一個有位置的光源，它從自身的位置向所有方向發出光線。然而，這個我們定義的光源所模擬光線的從不會衰減，這使得它看起來光源亮度極強。在大多數3D模擬中，我們喜歡模擬一個能照亮一個周圍確定區域光源，但它不會照亮整個場景。

如果你把10個箱子添加到之前教程的光照場景中，你會注意到黑暗中的每個箱子都會有同樣的亮度，就像箱子在光照的前面；沒有公式定義光的距離衰減。我們想讓黑暗中與光源比較近的箱子被輕微地照亮。

### 衰減(Attenuation)

隨著光線穿越更遠的距離相應地減少亮度，通常被稱為衰減(Attenuation)。一種隨著距離減少亮度的方式是使用線性等式。這樣的一個隨著距離減少亮度的線性方程，可以使遠處的物體更暗。然而，這樣的線性方程效果會有點假。在真實世界，通常光在近處時非常亮，但是一個光源的亮度，開始的時候減少的非常快，之後隨著距離的增加，減少的速度會慢下來。我們需要一種不同的方程來減少光的亮度。

幸運的是一些聰明人已經早就把它想到了。下面的方程把一個片段的光的亮度除以一個已經計算出來的衰減值，這個值根據光源的遠近得到：

![Latex Formula](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Light_casters1.png)

在這裡![I](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/I.png)是當前片段的光的亮度，![d](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/d.png)代表片段到光源的距離。為了計算衰減值，我們定義3個項：常數項![Kc](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Kc.png)，一次項![Kl](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Kl.png)和二次項![Kq](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Kq.png)。

常數項通常是1.0，它的作用是保證墳墓永遠不會比1小，因為它可以利用一定的距離增加亮度，這個結果不會影響到我們所尋找的。
一次項用於與距離值相稱，這回以線性的方式減少亮度。
二次項用於與距離的平方相乘，為光源設置一個亮度的二次遞減。二次項在距離比較近的時候相比一次項會比一次項更小，但是當距離更遠的時候比一次項更大。
由於二次項的光會以線性方式減少，指導距離足夠大的時候，就會超過一次項，之後，光的亮度會減少的更快。最後的效果就是光在近距離時，非常量，但是距離變遠亮度迅速降低，最後亮度降低速度再次變慢。下面的圖展示了在100以內的範圍，這樣的衰減效果。

![](http://www.learnopengl.com/img/lighting/attenuation.png)

你可以看到當距離很近的時候光有最強的亮度，但是隨著距離增大，亮度明顯減弱，大約接近100的時候，就會慢下來。這就是我們想要的。



#### 選擇正確的值

但是，我們把這三個項設置為什麼值呢？正確的值的設置由很多因素決定：環境、你希望光所覆蓋的距離範圍、光的類型等。大多數場合，這是經驗的問題，也要適度調整。下面的表格展示一些各項的值，它們模擬現實（某種類型的）光源，覆蓋特定的半徑（距離）。第一藍定義一個光的距離，它覆蓋所給定的項。這些值是大多數光的良好開始，它是來自Ogre3D的維基的禮物：

Distance|Constant|Linear|Quadratic
-------|------|-----|------
7|1.0|0.7|1.8
13|1.0|0.35|0.44
20|1.0|0.22|0.20
32|1.0|0.14|0.07
50|1.0|0.09|0.032
65|1.0|0.07|0.017
100|1.0|0.045|0.0075
160|1.0|0.027|0.0028
200|1.0|0.022|0.0019
325|1.0|0.014|0.0007
600|1.0|0.007|0.0002
3250|1.0|0.0014|0.000007

就像你所看到的，常數項![Kc](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Kc.png)一直都是1.0。一次項![Kl](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Kl.png)為了覆蓋更遠的距離通常很小，二次項![Kq](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Kq.png)就更小了。嘗試用這些值進行實驗，看看它們在你的實現中各自的效果。我們的環境中，32到100的距離對大多數光通常就足夠了。

#### 實現衰減

為了實現衰減，在著色器中我們會需要三個額外數值：也就是公式的常量、一次項和二次項。最好把它們儲存在之前定義的Light結構體中。要注意的是我們計算`lightDir`，就是在前面的教程中我們所做的，不是像之前的定向光的那部分。

```c++
struct Light
{
    vec3 position;
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
    float constant;
    float linear;
    float quadratic;
};
```

然後，我們在OpenGL中設置這些項：我們希望光覆蓋50的距離，所以我們會使用上面的表格中合適的常數項、一次項和二次項：

```c++
glUniform1f(glGetUniformLocation(lightingShader.Program, "light.constant"), 1.0f);
glUniform1f(glGetUniformLocation(lightingShader.Program, "light.linear"), 0.09);
glUniform1f(glGetUniformLocation(lightingShader.Program, "light.quadratic"), 0.032);
```

在片段著色器中實現衰減很直接：我們根據公式簡單的計算衰減值，在乘以`ambient`、`diffuse`和`specular`元素。

我們需要光源的距離提供給公式；記得我們能夠計算向量的長度嗎？我們可以通過獲取片段和光源之間的不同向量把向量的長度結果作為距離項。我們可以使用GLSL的內建`length`函數做這件事：

```c++
float distance = length(light.position - Position);
float attenuation = 1.0f / (light.constant + light.linear*distance +light.quadratic*(distance*distance));
```

然後，我們在光照計算中，通過把衰減值乘以`ambient`、`diffuse`和`specular`顏色，包含這個衰減值。

!!! Important

    我們可以可以把`ambient`元素留著不變，這樣`amient`光照就不會隨著距離減少，但是如果我們使用多餘1個的光源，所有的`ambient`元素會開始疊加，因此這種情況，我們希望`ambient`光照也衰減。簡單的調試出對於你的環境來說最好的效果。

```c++
ambient *= attenuation;
diffuse *= attenuation;
specular *= attenuation;
```

如果你運行應用後獲得這樣的效果：

![](http://learnopengl.com/img/lighting/light_casters_point_light.png)

你可以看到現在只有最近處的箱子的前面被照得最亮。後面的箱子一點都沒被照亮，因為它們距離光源太遠了。你可以在這裡找到[應用源碼](http://learnopengl.com/code_viewer.php?code=lighting/light_casters_point)和[片段著色器](http://learnopengl.com/code_viewer.php?code=lighting/light_casters_point&type=fragment)的代碼。

定點光就是一個可配的置位置和衰減值應用到光照計算中。還有另一種類型光可用於我們照明庫當中。


## 聚光燈(Spotlight)

我們要討論的最後一種類型光是聚光燈(Spotlight)。聚光燈是一種位於環境中某處的光源，它不是向所有方向照射，而是隻朝某個方向照射。結果是隻有一個聚光燈照射方向的確定半徑內的物體才會被照亮，其他的都保持黑暗。聚光燈的好例子是路燈或手電筒。

OpenGL中的聚光燈用世界空間位置，一個方向和一個指定了聚光燈半徑的切光角來表示。我們計算的每個片段，如果片段在聚光燈的切光方向之間（就是在圓錐體內），我們就會把片段照亮。下面的圖可以讓你明白聚光燈是如何工作的：

![](http://www.learnopengl.com/img/lighting/light_casters_spotlight_angles.png)

* `LightDir`：從片段指向光源的向量。
* `SpotDir`：聚光燈所指向的方向。
* `Phiφ`：定義聚光燈半徑的切光角。每個落在這個角度之外的，聚光燈都不會照亮。
* `Thetaθ`：`LightDir`向量和`SpotDir向`量之間的角度。θ值應該比φ值小，這樣才會在聚光燈內。

所以我們大致要做的是，計算`LightDir`向量和`SpotDir`向量的點乘（返回兩個單位向量的點乘，還記得嗎？），然後在和遮光角φ對比。現在你應該明白聚光燈是我們下面將創建的手電筒的範例。



### 手電筒

手電筒是一個坐落在觀察者位置的聚光燈，通常瞄準玩家透視圖的前面。基本上說，一個手電筒是一個普通的聚光燈，但是根據玩家的位置和方向持續的更新它的位置和方向。

所以我們需要為片段著色器提供的值，是聚光燈的位置向量（來計算光的方向座標），聚光燈的方向向量和遮光角。我們可以把這些值儲存在`Light`結構體中：

```c++
struct Light
{
    vec3 position;
    vec3 direction;
    float cutOff;
    ...
};
```

下面我們把這些適當的值傳給著色器：

```c++
glUniform3f(lightPosLoc, camera.Position.x, camera.Position.y, camera.Position.z);
glUniform3f(lightSpotdirLoc, camera.Front.x, camera.Front.y, camera.Front.z);
glUniform1f(lightSpotCutOffLoc, glm::cos(glm::radians(12.5f)));
```

你可以看到，我們為遮光角設置一個角度，但是我們根據一個角度計算了餘弦值，把這個餘弦結果傳給了片段著色器。這麼做的原因是在片段著色器中，我們計算`LightDir`和`SpotDir`向量的點乘，而點乘返回一個餘弦值，不是一個角度，所以我們不能直接把一個角度和餘弦值對比。為了獲得這個角度，我們必須計算點乘結果的反餘弦，這個操作開銷是很大的。所以為了節約一些性能，我們先計算給定切光角的餘弦值，然後把結果傳遞給片段著色器。由於每個角度都被表示為餘弦了，我們可以直接對比它們，而不用進行任何開銷高昂的操作。

現在剩下要做的是計算θ值，用它和φ值對比，以決定我們是否在或不在聚光燈的內部：

```c++
float theta = dot(lightDir, normalize(-light.direction));
if(theta > light.cutOff)
{
    // 執行光照計算
}
else // 否則使用環境光，使得場景不至於完全黑暗
color = vec4(light.ambient*vec3(texture(material.diffuse,TexCoords)), 1.0f);
```

我們首先計算`lightDir`和負方向向量的點乘（它負的是因為我們想要向量指向光源，而不是從光源作為指向出發點。譯註：前面的`specular`教程中作者卻用了相反的表示方法，這裡讀者可以選擇喜歡的表達方式）。確保對所有相關向量進行了標準化處理。

!!! Important

    你可能奇怪為什麼if條件中使用>符號而不是<符號。為了在聚光燈以內，θ不是應該比光的遮光值更小嗎？這沒錯，但是不要忘了，角度值是以餘弦值來表示的，一個0度的角表示為1.0的餘弦值，當一個角是90度的時候被表示為0.0的餘弦值，你可以在這裡看到：

    ![](http://www.learnopengl.com/img/lighting/light_casters_cos.png)

    現在你可以看到，餘弦越是接近1.0，角度就越小。這就解釋了為什麼θ需要比切光值更大了。切光值當前被設置為12.5的餘弦，它等於0.9978，所以θ的餘弦值在0.9979和1.0之間，片段會在聚光燈內，被照亮。

運行應用，在聚光燈內的片段才會被照亮。這看起來像這樣：

![](http://www.learnopengl.com/img/lighting/light_casters_spotlight_hard.png)

你可以在這裡獲得[全部源碼](http://learnopengl.com/code_viewer.php?code=lighting/light_casters_spotlight_hard)和[片段著色器的源碼](http://learnopengl.com/code_viewer.php?code=lighting/light_casters_spotlight_hard&type=fragment)。

它看起來仍然有點假，原因是聚光燈有了一個硬邊。片段著色器一旦到達了聚光燈的圓錐邊緣，它就立刻黑了下來，卻沒有任何平滑減弱的過度。一個真實的聚光燈的光會在它的邊界處平滑減弱的。

## 平滑/軟化邊緣

為創建聚光燈的平滑邊，我們希望去模擬的聚光燈有一個內圓錐和外圓錐。我們可以把內圓錐設置為前面部分定義的圓錐，我們希望外圓錐從內邊到外邊逐步的變暗。

為創建外圓錐，我們簡單定義另一個餘弦值，它代表聚光燈的方向向量和外圓錐的向量（等於它的半徑）的角度。然後，如果片段在內圓錐和外圓錐之間，就會給它計算出一個0.0到1.0之間的亮度。如果片段在內圓錐以內這個亮度就等於1.0，如果在外面就是0.0。

我們可以使用下面的公式計算這樣的值：
![Latex Formula](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Light_casters1.png)
這裡![Epsilon](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/epsilon.png)是內部（![Phi](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/phi.png)）和外部圓錐(![Gamma](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/gamma.png))![Latex Formula](https://raw.githubusercontent.com/LearnOpenGL-CN/LearnOpenGL-CN/master/img/Light_casters3.png)的差。結果I的值是聚光燈在當前片段的亮度。

很難用圖畫描述出這個公式是怎樣工作的，所以我們嘗試使用一個例子：


θ|θ in degrees|φ (inner cutoff)|φ in degrees|γ (outer cutoff)|γ in degrees|ε|l
--|---|---|---|---|---|---|---
0.87|30|0.91|25|0.82|35|0.91 - 0.82 = 0.09|0.87 - 0.82 / 0.09 = 0.56
0.9|26|0.91|25|0.82|35|0.91 - 0.82 = 0.09|0.9 - 0.82 / 0.09 = 0.89
0.97|14|0.91|25|0.82|35|0.91 - 0.82 = 0.09|0.97 - 0.82 / 0.09 = 1.67
0.97|14|0.91|25|0.82|35|0.91 - 0.82 = 0.09|0.97 - 0.82 / 0.09 = 1.67
0.83|34|0.91|25|0.82|35|0.91 - 0.82 = 0.09|0.83 - 0.82 / 0.09 = 0.11
0.64|50|0.91|25|0.82|35|0.91 - 0.82 = 0.09|0.64 - 0.82 / 0.09 = -2.0
0.966|15|0.9978|12.5|0.953|17.5|0.966 - 0.953 = 0.0448|0.966 - 0.953 / 0.0448 = 0.29

就像你看到的那樣我們基本是根據θ在外餘弦和內餘弦之間插值。如果你仍然不明白怎麼繼續，不要擔心。你可以簡單的使用這個公式計算，當你更加老道和明白的時候再來看。

由於我們現在有了一個亮度值，當在聚光燈外的時候是個負的，當在內部圓錐以內大於1。如果我們適當地把這個值固定，我們在片段著色器中就再不需要if-else了，我們可以簡單地用計算出的亮度值乘以光的元素：

```c++
float theta = dot(lightDir, normalize(-light.direction));
float epsilon = light.cutOff - light.outerCutOff;
float intensity = clamp((theta - light.outerCutOff) / epsilon,0.0, 1.0);
...
// We’ll leave ambient unaffected so we always have a little
light.diffuse* = intensity;
specular* = intensity;
...
```

注意，我們使用了`clamp`函數，它把第一個參數固定在0.0和1.0之間。這保證了亮度值不會超出[0, 1]以外。

確定你把`outerCutOff`值添加到了`Light`結構體，並在應用中設置了它的uniform值。對於下面的圖片，內部遮光角`12.5f`，外部遮光角是`17.5f`：

![](http://www.learnopengl.com/img/lighting/light_casters_spotlight.png)

看起來好多了。仔細看看內部和外部遮光角，嘗試創建一個符合你求的聚光燈。可以在這裡找到應用源碼，以及片段的源代碼。

這樣的一個手電筒/聚光燈類型的燈光非常適合恐怖遊戲，結合定向和點光，環境會真的開始被照亮了。[下一個教程](http://learnopengl-cn.readthedocs.org/zh/latest/02%20Lighting/06%20Multiple%20lights/)，我們會結合所有我們目前討論了的光和技巧。
