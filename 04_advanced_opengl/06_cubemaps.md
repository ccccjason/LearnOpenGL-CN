# 立方體貼圖(Cubemap)

原文     | [Cubemaps](http://learnopengl.com/#!Advanced-OpenGL/Cubemaps)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

我們之前一直使用的是2D紋理，還有更多的紋理類型我們沒有探索過，本教程中我們討論的紋理類型是將多個紋理組合起來映射到一個單一紋理，它就是cubemap。

基本上說cubemap它包含6個2D紋理，這每個2D紋理是一個立方體（cube）的一個面，也就是說它是一個有貼圖的立方體。你可能會奇怪這樣的立方體有什麼用？為什麼費事地把6個獨立紋理結合為一個單獨的紋理，只使用6個各自獨立的不行嗎？這是因為cubemap有自己特有的屬性，可以使用方向向量對它們索引和採樣。想象一下，我們有一個1×1×1的單位立方體，有個以原點為起點的方向向量在它的中心。

從cubemap上使用橘黃色向量採樣一個紋理值看起來和下圖有點像：

![](http://learnopengl.com/img/advanced/cubemaps_sampling.png)

!!! Important

        方向向量的大小無關緊要。一旦提供了方向，OpenGL就會獲取方向向量觸碰到立方體表面上的相應的紋理像素（texel），這樣就返回了正確的紋理採樣值。


方向向量觸碰到立方體表面的一點也就是cubemap的紋理位置，這意味著只要立方體的中心位於原點上，我們就可以使用立方體的位置向量來對cubemap進行採樣。然後我們就可以獲取所有頂點的紋理座標，就和立方體上的頂點位置一樣。所獲得的結果是一個紋理座標，通過這個紋理座標就能獲取到cubemap上正確的紋理。

### 創建一個Cubemap

Cubemap和其他紋理一樣，所以要創建一個cubemap，在進行任何紋理操作之前，需要生成一個紋理，激活相應紋理單元然後綁定到合適的紋理目標上。這次要綁定到 `GL_TEXTURE_CUBE_MAP`紋理類型：

```c++
GLuint textureID;
glGenTextures(1, &textureID);
glBindTexture(GL_TEXTURE_CUBE_MAP, textureID);
```

由於cubemap包含6個紋理，立方體的每個面一個紋理，我們必須調用`glTexImage2D`函數6次，函數的參數和前面教程講的相似。然而這次我們必須把紋理目標（target）參數設置為cubemap特定的面，這是告訴OpenGL我們創建的紋理是對應立方體哪個面的。因此我們便需要為cubemap的每個面調用一次 `glTexImage2D`。

由於cubemap有6個面，OpenGL就提供了6個不同的紋理目標，來應對cubemap的各個面。

紋理目標（Texture target）	   | 方位
                            ---|---
GL_TEXTURE_CUBE_MAP_POSITIVE_X |	右
GL_TEXTURE_CUBE_MAP_NEGATIVE_X |	左
GL_TEXTURE_CUBE_MAP_POSITIVE_Y |	上
GL_TEXTURE_CUBE_MAP_NEGATIVE_Y |	下
GL_TEXTURE_CUBE_MAP_POSITIVE_Z |	後
GL_TEXTURE_CUBE_MAP_NEGATIVE_Z |	前

和很多OpenGL其他枚舉一樣，對應的int值都是連續增加的，所以我們有一個紋理位置的數組或vector，就能以 `GL_TEXTURE_CUBE_MAP_POSITIVE_X`為起始來對它們進行遍歷，每次迭代枚舉值加 `1`，這樣循環所有的紋理目標效率較高：

```c++
int width,height;
unsigned char* image;  
for(GLuint i = 0; i < textures_faces.size(); i++)
{
    image = SOIL_load_image(textures_faces[i], &width, &height, 0, SOIL_LOAD_RGB);
    glTexImage2D(
        GL_TEXTURE_CUBE_MAP_POSITIVE_X + i,
        0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, image
    );
}
```

這兒我們有個vector叫`textures_faces`，它包含cubemap所各個紋理的文件路徑，並且以上表所列的順序排列。它將為每個當前綁定的cubemp的每個面生成一個紋理。

由於cubemap和其他紋理沒什麼不同，我們也要定義它的環繞方式和過濾方式：

```c++
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
```

別被 `GL_TEXTURE_WRAP_R`嚇到，它只是簡單的設置了紋理的R座標，R座標對應於紋理的第三個維度（就像位置的z一樣）。我們把放置方式設置為 `GL_CLAMP_TO_EDGE` ，由於紋理座標在兩個面之間，所以可能並不能觸及哪個面（由於硬件限制），因此使用 `GL_CLAMP_TO_EDGE` 後OpenGL會返回它們的邊界的值，儘管我們可能在兩個兩個面中間進行的採樣。

在繪製物體之前，將使用cubemap，而在渲染前我們要激活相應的紋理單元並綁定到cubemap上，這和普通的2D紋理沒什麼區別。

在片段著色器中，我們也必須使用一個不同的採樣器——**samplerCube**，用它來從`texture`函數中採樣，但是這次使用的是一個`vec3`方向向量，取代`vec2`。下面是一個片段著色器使用了cubemap的例子：

```c++
in vec3 textureDir; // 用一個三維方向向量來表示Cubemap紋理的座標

uniform samplerCube cubemap;  // Cubemap紋理採樣器

void main()
{
    color = texture(cubemap, textureDir);
}
```

看起來不錯，但是何必這麼做呢？因為恰巧使用cubemap可以簡單的實現很多有意思的技術。其中之一便是著名的**天空盒(Skybox)**。



## 天空盒(Skybox)

天空盒是一個包裹整個場景的立方體，它由6個圖像構成一個環繞的環境，給玩家一種他所在的場景比實際的要大得多的幻覺。比如有些在視頻遊戲中使用的天空盒的圖像是群山、白雲或者滿天繁星。比如下面的夜空繁星的圖像就來自《上古卷軸》：

![](http://learnopengl.com/img/advanced/cubemaps_morrowind.jpg)

你現在可能已經猜到cubemap完全滿足天空盒的要求：我們有一個立方體，它有6個面，每個面需要一個貼圖。上圖中使用了幾個夜空的圖片給予玩家一種置身廣袤宇宙的感覺，可實際上，他還是在一個小盒子之中。

網上有很多這樣的天空盒的資源。[這個網站](http://www.custommapmakers.org/skyboxes.php)就提供了很多。這些天空盒圖像通常有下面的樣式：

![](http://learnopengl.com/img/advanced/cubemaps_skybox.png)

如果你把這6個面折疊到一個立方體中，你機會獲得模擬了一個巨大的風景的立方體。有些資源所提供的天空盒比如這個例子6個圖是連在一起的，你必須手工它們切割出來，不過大多數情況它們都是6個單獨的紋理圖像。

這個細緻（高精度）的天空盒就是我們將在場景中使用的那個，你可以[在這裡下載](http://learnopengl.com/img/textures/skybox.rar)。

### 加載一個天空盒

由於天空盒實際上就是一個cubemap，加載天空盒和之前我們加載cubemap的沒什麼大的不同。為了加載天空盒我們將使用下面的函數，它接收一個包含6個紋理文件路徑的vector：

```c++
GLuint loadCubemap(vector<const GLchar*> faces)
{
    GLuint textureID;
    glGenTextures(1, &textureID);
    glActiveTexture(GL_TEXTURE0);

    int width,height;
    unsigned char* image;

    glBindTexture(GL_TEXTURE_CUBE_MAP, textureID);
    for(GLuint i = 0; i < faces.size(); i++)
    {
        image = SOIL_load_image(faces[i], &width, &height, 0, SOIL_LOAD_RGB);
        glTexImage2D(
            GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0,
            GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, image
        );
    }
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
    glBindTexture(GL_TEXTURE_CUBE_MAP, 0);

    return textureID;
}
```

這個函數沒什麼特別之處。這就是我們前面已經見過的cubemap代碼，只不過放進了一個可管理的函數中。

然後，在我們調用這個函數之前，我們將把合適的紋理路徑加載到一個vector之中，順序還是按照cubemap枚舉的特定順序：

```c++
vector<const GLchar*> faces;
faces.push_back("right.jpg");
faces.push_back("left.jpg");
faces.push_back("top.jpg");
faces.push_back("bottom.jpg");
faces.push_back("back.jpg");
faces.push_back("front.jpg");
GLuint cubemapTexture = loadCubemap(faces);
```

現在我們已經用`cubemapTexture`作為id把天空盒加載為cubemap。我們現在可以把它綁定到一個立方體來替換不完美的`clear color`，在前面的所有教程中這個東西做背景已經很久了。



### 天空盒的顯示

因為天空盒繪製在了一個立方體上，我們還需要另一個VAO、VBO以及一組全新的頂點，和任何其他物體一樣。你可以[從這裡獲得頂點數據](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps_skybox_data)。

cubemap用於給3D立方體帖上紋理，可以用立方體的位置作為紋理座標進行採樣。當一個立方體的中心位於原點(0，0，0)的時候，它的每一個位置向量也就是以原點為起點的方向向量。這個方向向量就是我們要得到的立方體某個位置的相應紋理值。出於這個理由，我們只需要提供位置向量，而無需紋理座標。為了渲染天空盒，我們需要一組新著色器，它們不會太複雜。因為我們只有一個頂點屬性，頂點著色器非常簡單：

```c++
#version 330 core
layout (location = 0) in vec3 position;
out vec3 TexCoords;

uniform mat4 projection;
uniform mat4 view;

void main()
{
    gl_Position =   projection * view * vec4(position, 1.0);  
    TexCoords = position;
}
```

注意，頂點著色器有意思的地方在於我們把輸入的位置向量作為輸出給片段著色器的紋理座標。片段著色器就會把它們作為輸入去採樣samplerCube：

```c++
#version 330 core
in vec3 TexCoords;
out vec4 color;

uniform samplerCube skybox;

void main()
{
    color = texture(skybox, TexCoords);
}
```

片段著色器比較明瞭，我們把頂點屬性中的位置向量作為紋理的方向向量，使用它們從cubemap採樣紋理值。渲染天空盒現在很簡單，我們有了一個cubemap紋理，我們簡單綁定cubemap紋理，天空盒就自動地用天空盒的cubemap填充了。為了繪製天空盒，我們將把它作為場景中第一個繪製的物體並且關閉深度寫入。這樣天空盒才能成為所有其他物體的背景來繪製出來。

```c++

glDepthMask(GL_FALSE);
skyboxShader.Use();
// ... Set view and projection matrix
glBindVertexArray(skyboxVAO);
glBindTexture(GL_TEXTURE_CUBE_MAP, cubemapTexture);
glDrawArrays(GL_TRIANGLES, 0, 36);
glBindVertexArray(0);
glDepthMask(GL_TRUE);
// ... Draw rest of the scene
```

如果你運行程序就會陷入困境，我們希望天空盒以玩家為中心，這樣無論玩家移動了多遠，天空盒都不會變近，這樣就產生一種四周的環境真的非常大的印象。當前的視圖矩陣對所有天空盒的位置進行了轉轉縮放和平移變換，所以玩家移動，cubemap也會跟著移動！我們打算移除視圖矩陣的平移部分，這樣移動就影響不到天空盒的位置向量了。在基礎光照教程裡我們提到過我們可以只用4X4矩陣的3×3部分去除平移。我們可以簡單地將矩陣轉為33矩陣再轉回來，就能達到目標

```c++
glm::mat4 view = glm::mat4(glm::mat3(camera.GetViewMatrix()));
```

這會移除所有平移，但保留所有旋轉，因此用戶仍然能夠向四面八方看。由於有了天空盒，場景即可變得巨大了。如果你添加些物體然後自由在其中游蕩一會兒你會發現場景的真實度有了極大提升。最後的效果看起來像這樣：

![](http://learnopengl.com/img/advanced/cubemaps_skybox_result.png)

[這裡有全部源碼](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps_skybox)，你可以對比一下你寫的。

嘗試用不同的天空盒實驗，看看它們對場景有多大影響。

### 優化

現在我們在渲染場景中的其他物體之前渲染了天空盒。這麼做沒錯，但是不怎麼高效。如果我們先渲染了天空盒，那麼我們就是在為每一個屏幕上的像素運行片段著色器，即使天空盒只有部分在顯示著；fragment可以使用前置深度測試（early depth testing）簡單地被丟棄，這樣就節省了我們寶貴的帶寬。

所以最後渲染天空盒就能夠給我們帶來輕微的性能提升。採用這種方式，深度緩衝被全部物體的深度值完全填充，所以我們只需要渲染通過前置深度測試的那部分天空的片段就行了，而且能顯著減少片段著色器的調用。問題是天空盒是個1×1×1的立方體，極有可能會渲染失敗，因為極有可能通不過深度測試。簡單地不用深度測試渲染它也不是解決方案，這是因為天空盒會在之後覆蓋所有的場景中其他物體。我們需要耍個花招讓深度緩衝相信天空盒的深度緩衝有著最大深度值1.0，如此只要有個物體存在深度測試就會失敗，看似物體就在它前面了。

在座標系教程中我們說過，透視除法（perspective division）是在頂點著色器運行之後執行的，把`gl_Position`的xyz座標除以w元素。我們從深度測試教程瞭解到除法結果的z元素等於頂點的深度值。利用這個信息，我們可以把輸出位置的z元素設置為它的w元素，這樣就會導致z元素等於1.0了，因為，當透視除法應用後，它的z元素轉換為w/w = 1.0：

```c++
void main()
{
    vec4 pos = projection * view * vec4(position, 1.0);
    gl_Position = pos.xyww;
    TexCoords = position;
}
```

最終，標準化設備座標就總會有個與1.0相等的z值了，1.0就是深度值的最大值。只有在沒有任何物體可見的情況下天空盒才會被渲染（只有通過深度測試才渲染，否則假如有任何物體存在，就不會被渲染，只去渲染物體）。

我們必須改變一下深度方程，把它設置為`GL_LEQUAL`，原來默認的是`GL_LESS`。深度緩衝會為天空盒用1.0這個值填充深度緩衝，所以我們需要保證天空盒是使用小於等於深度緩衝來通過深度測試的，而不是小於。

你可以在這裡找到優化過的版本的[源碼](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps_skybox_optimized)。

### 環境映射

我們現在有了一個把整個環境映射到為一個單獨紋理的對象，我們利用這個信息能做的不僅是天空盒。使用帶有場景環境的cubemap，我們還可以讓物體有一個反射或折射屬性。像這樣使用了環境cubemap的技術叫做**環境貼圖技術**，其中最重要的兩個是**反射(reflection)**和**折射(refraction)**。

#### 反射(reflection)

凡是是一個物體（或物體的某部分）反射他周圍的環境的屬性，比如物體的顏色多少有些等於它周圍的環境，這要基於觀察者的角度。例如一個鏡子是一個反射物體：它會基於觀察者的角度泛著它周圍的環境。

反射的基本思路不難。下圖展示了我們如何計算反射向量，然後使用這個向量去從一個cubemap中採樣：

![](http://learnopengl.com/img/advanced/cubemaps_reflection_theory.png)

我們基於觀察方向向量I和物體的法線向量N計算出反射向量R。我們可以使用GLSL的內建函數reflect來計算這個反射向量。最後向量R作為一個方向向量對cubemap進行索引/採樣，返回一個環境的顏色值。最後的效果看起來就像物體反射了天空盒。

因為我們在場景中已經設置了一個天空盒，創建反射就不難了。我們改變一下箱子使用的那個片段著色器，給箱子一個反射屬性：

```c++
#version 330 core
in vec3 Normal;
in vec3 Position;
out vec4 color;

uniform vec3 cameraPos;
uniform samplerCube skybox;

void main()
{
    vec3 I = normalize(Position - cameraPos);
    vec3 R = reflect(I, normalize(Normal));
    color = texture(skybox, R);
}
```

我們先來計算觀察/攝像機方向向量I，然後使用它來計算反射向量R，接著我們用R從天空盒cubemap採樣。要注意的是，我們有了片段的插值Normal和Position變量，所以我們需要修正頂點著色器適應它。

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;

out vec3 Normal;
out vec3 Position;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
    Normal = mat3(transpose(inverse(model))) * normal;
    Position = vec3(model * vec4(position, 1.0f));
}
```

我們用了法線向量，所以我們打算使用一個**法線矩陣(normal matrix)**變換它們。`Position`輸出的向量是一個世界空間位置向量。頂點著色器輸出的`Position`用來在片段著色器計算觀察方向向量。

因為我們使用法線，你還得更新頂點數據，更新屬性指針。還要確保設置`cameraPos`的uniform。

然後在渲染箱子前我們還得綁定cubemap紋理：

```c++
glBindVertexArray(cubeVAO);
glBindTexture(GL_TEXTURE_CUBE_MAP, skyboxTexture);
glDrawArrays(GL_TRIANGLES, 0, 36);
glBindVertexArray(0);
```

編譯運行你的代碼，你等得到一個鏡子一樣的箱子。箱子完美地反射了周圍的天空盒：

![](http://learnopengl.com/img/advanced/cubemaps_reflection.png)

你可以[從這裡找到全部源代碼](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps_reflection)。

當反射應用於整個物體之上的時候，物體看上去就像有一個像鋼和鉻這種高反射材質。如果我們加載[模型教程](http://learnopengl-cn.readthedocs.org/zh/latest/03%20Model%20Loading/03%20Model/)中的納米鎧甲模型，我們就會獲得一個鉻金屬製鎧甲：

![](http://learnopengl.com/img/advanced/cubemaps_reflection_nanosuit.png)

看起來挺驚豔，但是現實中大多數模型都不是完全反射的。我們可以引進反射貼圖（reflection map）來使模型有另一層細節。和diffuse、specular貼圖一樣，我們可以從反射貼圖上採樣來決定fragment的反射率。使用反射貼圖我們還可以決定模型的哪個部分有反射能力，以及強度是多少。本節的練習中，要由你來在我們早期創建的模型加載器引入反射貼圖，這回極大的提升納米服模型的細節。

#### 折射(refraction)

環境映射的另一個形式叫做折射，它和反射差不多。折射是光線通過特定材質對光線方向的改變。我們通常看到像水一樣的表面，光線並不是直接通過的，而是讓光線彎曲了一點。它看起來像你把半隻手伸進水裡的效果。

折射遵守[斯涅爾定律](http://en.wikipedia.org/wiki/Snell%27s_law)，使用環境貼圖看起來就像這樣：

![](http://learnopengl.com/img/advanced/cubemaps_refraction_theory.png)

我們有個觀察向量I，一個法線向量N，這次折射向量是R。就像你所看到的那樣，觀察向量的方向有輕微彎曲。彎曲的向量R隨後用來從cubemap上採樣。

折射可以通過GLSL的內建函數refract來實現，除此之外還需要一個法線向量，一個觀察方向和一個兩種材質之間的折射指數。

折射指數決定了一個材質上光線扭曲的數量，每個材質都有自己的折射指數。下表是常見的折射指數：

材質  |	折射指數
---|---
空氣 |	1.00
水	 | 1.33
冰	 | 1.309
玻璃 |	1.52
寶石 |	2.42

我們使用這些折射指數來計算光線通過兩個材質的比率。在我們的例子中，光線/視線從空氣進入玻璃（如果我們假設箱子是玻璃做的）所以比率是1.001.52 = 0.658。

我們已經綁定了cubemap，提供了定點數據，設置了攝像機位置的uniform。現在只需要改變片段著色器：

```c++
void main()
{
    float ratio = 1.00 / 1.52;
    vec3 I = normalize(Position - cameraPos);
    vec3 R = refract(I, normalize(Normal), ratio);
    color = texture(skybox, R);
}
```

通過改變折射指數你可以創建出完全不同的視覺效果。編譯運行應用，結果也不是太有趣，因為我們只是用了一個普通箱子，這不能顯示出折射的效果，看起來像個放大鏡。使用同一個著色器，納米服模型卻可以展示出我們期待的效果：玻璃制物體。

![](http://learnopengl.com/img/advanced/cubemaps_refraction.png)

你可以向想象一下，如果將光線、反射、折射和頂點的移動合理的結合起來就能創造出漂亮的水的圖像。一定要注意，出於物理精確的考慮當光線離開物體的時候還要再次進行折射；現在我們簡單的使用了單邊（一次）折射，大多數目的都可以得到滿足。

#### 動態環境貼圖（Dynamic environment maps）

現在，我們已經使用了靜態圖像組合的天空盒，看起來不錯，但是沒有考慮到物體可能移動的實際場景。我們到現在還沒注意到這點，是因為我們目前還只使用了一個物體。如果我們有個鏡子一樣的物體，它周圍有多個物體，只有天空盒在鏡子中可見，和場景中只有這一個物體一樣。

使用幀緩衝可以為提到的物體的所有6個不同角度創建一個場景的紋理，把它們每次渲染迭代儲存為一個cubemap。之後我們可以使用這個（動態生成的）cubemap來創建真實的反射和折射表面，這樣就能包含所有其他物體了。這種方法叫做動態環境映射（dynamic environment mapping）,因為我們動態地創建了一個物體的以其四周為參考的cubemap，並把它用作環境貼圖。

它看起效果很好，但是有一個劣勢：使用環境貼圖我們必須為每個物體渲染場景6次，這需要非常大的開銷。現代應用嘗試儘量使用天空盒子，凡可能預編譯cubemap就創建少量動態環境貼圖。動態環境映射是個非常棒的技術，要想在不降低執行效率的情況下實現它就需要很多巧妙的技巧。



## 練習

嘗試在模型加載中引進反射貼圖，你將再次得到很大視覺效果的提升。這其中有幾點需要注意：

- Assimp並不支持反射貼圖，我們可以使用環境貼圖的方式將反射貼圖從`aiTextureType_AMBIENT`類型中來加載反射貼圖的材質。
- 我匆忙地使用反射貼圖來作為鏡面反射的貼圖，而反射貼圖並沒有很好的映射在模型上:)。
- 由於加載模型已經佔用了3個紋理單元，因此你要綁定天空盒到第4個紋理單元上，這樣才能在同一個著色器內從天空盒紋理中取樣。

You can find the solution source code here together with the updated model and mesh class. The shaders used for rendering the reflection maps can be found here: vertex shader and fragment shader. 

你可以在此獲取解決方案的[源代碼](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps-exercise1)，這其中還包括升級過的[Model](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps-exercise1-model)和[Mesh](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps-exercise1-mesh)類，還有用來繪製反射貼圖的[頂點著色器](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps-exercise1-vertex)和[片段著色器](http://learnopengl.com/code_viewer.php?code=advanced/cubemaps-exercise1-fragment)。

如果你一切都做對了，那你應該看到和下圖類似的效果：

![](http://learnopengl.com/img/advanced/cubemaps_reflection_map.png)
