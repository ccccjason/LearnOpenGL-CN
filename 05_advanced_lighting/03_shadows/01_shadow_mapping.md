## 陰影映射(Shadow Mapping)

本文作者JoeyDeVries，由Django翻譯自[http://learnopengl.com](http://learnopengl.com)

原文     | [Shadow Mapping](http://learnopengl.com/#!Advanced-Lighting/Shadows/Shadow-Mapping)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | gjy_1992



陰影是光線被阻擋的結果；當一個光源的光線由於其他物體的阻擋不能夠達到一個物體的表面的時候，那麼這個物體就在陰影中了。陰影能夠使場景看起來真實得多，並且可以讓觀察者獲得物體之間的空間位置關係。場景和物體的深度感因此能夠得到極大提升，下圖展示了有陰影和沒有陰影的情況下的不同：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_with_without.png)

你可以看到，有陰影的時候你能更容易地區分出物體之間的位置關係，例如，當使用陰影的時候浮在地板上的立方體的事實更加清晰。

陰影還是比較不好實現的，因為當前實時渲染領域還沒找到一種完美的陰影算法。目前有幾種近似陰影技術，但它們都有自己的弱點和不足，這點我們必須要考慮到。

視頻遊戲中較多使用的一種技術是陰影貼圖（shadow mapping），效果不錯，而且相對容易實現。陰影貼圖並不難以理解，性能也不會太低，而且非常容易擴展成更高級的算法（比如 [Omnidirectional Shadow Maps](http://learnopengl.com/#!Advanced-Lighting/Shadows/Point-Shadows)和 [Cascaded Shadow Maps](http://learnopengl.com/#!Advanced-Lighting/Shadows/CSM)）。

### 陰影映射

陰影映射背後的思路非常簡單：我們以光的位置為視角進行渲染，我們能看到的東西都將被點亮，看不見的一定是在陰影之中了。假設有一個地板，在光源和它之間有一個大盒子。由於光源處向光線方向看去，可以看到這個盒子，但看不到地板的一部分，這部分就應該在陰影中了。

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_theory.png)

這裡的所有藍線代表光源可以看到的fragment。黑線代表被遮擋的fragment：它們應該渲染為帶陰影的。如果我們繪製一條從光源出發，到達最右邊盒子上的一個片元上的線段或射線，那麼射線將先擊中懸浮的盒子，隨後才會到達最右側的盒子。結果就是懸浮的盒子被照亮，而最右側的盒子將處於陰影之中。

我們希望得到射線第一次擊中的那個物體，然後用這個最近點和射線上其他點進行對比。然後我們將測試一下看看射線上的其他點是否比最近點更遠，如果是的話，這個點就在陰影中。對從光源發出的射線上的成千上萬個點進行遍歷是個極端消耗性能的舉措，實時渲染上基本不可取。我們可以採取相似舉措，不用投射出光的射線。我們所使用的是非常熟悉的東西：深度緩衝。

你可能記得在[深度測試](http://learnopengl.com/#!Advanced-OpenGL/Depth-testing)教程中，在深度緩衝裡的一個值是攝像機視角下，對應於一個片元的一個0到1之間的深度值。如果我們從光源的透視圖來渲染場景，並把深度值的結果儲存到紋理中會怎樣？通過這種方式，我們就能對光源的透視圖所見的最近的深度值進行採樣。最終，深度值就會顯示從光源的透視圖下見到的第一個片元了。我們管儲存在紋理中的所有這些深度值，叫做深度貼圖（depth map）或陰影貼圖。

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_theory_spaces.png)

左側的圖片展示了一個定向光源（所有光線都是平行的）在立方體下的表面投射的陰影。通過儲存到深度貼圖中的深度值，我們就能找到最近點，用以決定片元是否在陰影中。我們使用一個來自光源的視圖和投影矩陣來渲染場景就能創建一個深度貼圖。這個投影和視圖矩陣結合在一起成為一個T變換，它可以將任何三維位置轉變到光源的可見座標空間。

!!! Important

    定向光並沒有位置，因為它被規定為無窮遠。然而，為了實現陰影貼圖，我們得從一個光的透視圖渲染場景，這樣就得在光的方向的某一點上渲染場景。

在右邊的圖中我們顯示出同樣的平行光和觀察者。我們渲染一個點P處的片元，需要決定它是否在陰影中。我們先得使用T把P變換到光源的座標空間裡。既然點P是從光的透視圖中看到的，它的z座標就對應於它的深度，例子中這個值是0.9。使用點P在光源的座標空間的座標，我們可以索引深度貼圖，來獲得從光的視角中最近的可見深度，結果是點C，最近的深度是0.4。因為索引深度貼圖的結果是一個小於點P的深度，我們可以斷定P被擋住了，它在陰影中了。

深度映射由兩個步驟組成：首先，我們渲染深度貼圖，然後我們像往常一樣渲染場景，使用生成的深度貼圖來計算片元是否在陰影之中。聽起來有點複雜，但隨著我們一步一步地講解這個技術，就能理解了。

### 深度貼圖（depth map）

第一步我們需要生成一張深度貼圖。深度貼圖是從光的透視圖裡渲染的深度紋理，用它計算陰影。因為我們需要將場景的渲染結果儲存到一個紋理中，我們將再次需要幀緩衝。

首先，我們要為渲染的深度貼圖創建一個幀緩衝對象：

```c++
GLuint depthMapFBO;
glGenFramebuffers(1, &depthMapFBO);
```

然後，創建一個2D紋理，提供給幀緩衝的深度緩衝使用：

```c++
const GLuint SHADOW_WIDTH = 1024, SHADOW_HEIGHT = 1024;
 
GLuint depthMap;
glGenTextures(1, &depthMap);
glBindTexture(GL_TEXTURE_2D, depthMap);
glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT, 
             SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT); 
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
```

生成深度貼圖不太複雜。因為我們只關心深度值，我們要把紋理格式指定為GL_DEPTH_COMPONENT。我們還要把紋理的高寬設置為1024：這是深度貼圖的解析度。

把我們把生成的深度紋理作為幀緩衝的深度緩衝：

```c++
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, depthMap, 0);
glDrawBuffer(GL_NONE);
glReadBuffer(GL_NONE);
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

我們需要的只是在從光的透視圖下渲染場景的時候深度信息，所以顏色緩衝沒有用。然而幀緩衝對象不是完全不包含顏色緩衝的，所以我們需要顯式告訴OpenGL我們不適用任何顏色數據進行渲染。我們通過將調用glDrawBuffer和glReadBuffer把讀和繪製緩衝設置為GL_NONE來做這件事。

合理配置將深度值渲染到紋理的幀緩衝後，我們就可以開始第一步了：生成深度貼圖。兩個步驟的完整的渲染階段，看起來有點像這樣：

```c++
// 1. 首選渲染深度貼圖
glViewport(0, 0, SHADOW_WIDTH, SHADOW_HEIGHT);
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
    glClear(GL_DEPTH_BUFFER_BIT);
    ConfigureShaderAndMatrices();
    RenderScene();
glBindFramebuffer(GL_FRAMEBUFFER, 0);
// 2. 像往常一樣渲染場景，但這次使用深度貼圖
glViewport(0, 0, SCR_WIDTH, SCR_HEIGHT);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
ConfigureShaderAndMatrices();
glBindTexture(GL_TEXTURE_2D, depthMap);
RenderScene();
```

這段代碼隱去了一些細節，但它表達了陰影映射的基本思路。這裡一定要記得調用glViewport。因為陰影貼圖經常和我們原來渲染的場景（通常是窗口解析度）有著不同的解析度，我們需要改變視口（viewport）的參數以適應陰影貼圖的尺寸。如果我們忘了更新視口參數，最後的深度貼圖要麼太小要麼就不完整。

### 光源空間的變換（light spacce transform）

前面那段代碼中一個不清楚的函數是COnfigureShaderAndMatrices。它是用來在第二個步驟確保為每個物體設置了合適的投影和視圖矩陣，以及相關的模型矩陣。然而，第一個步驟中，我們從光的位置的視野下使用了不同的投影和視圖矩陣來渲染的場景。

因為我們使用的是一個所有光線都平行的定向光。出於這個原因，我們將為光源使用正交投影矩陣，透視圖將沒有任何變形：

```c++
GLfloat near_plane = 1.0f, far_plane = 7.5f;
glm::mat4 lightProjection = glm::ortho(-10.0f, 10.0f, -10.0f, 10.0f, near_plane, far_plane);
```

這裡有個本節教程的demo場景中使用的正交投影矩陣的例子。因為投影矩陣間接決定可視區域的範圍，以及哪些東西不會被裁切，你需要保證投影視錐（frustum）的大小，以包含打算在深度貼圖中包含的物體。當物體和片元不在深度貼圖中時，它們就不會產生陰影。

為了創建一個視圖矩陣來變換每個物體，把它們變換到從光源視角可見的空間中，我們將使用glm::lookAt函數；這次從光源的位置看向場景中央。

```c++
glm::mat4 lightView = glm::lookAt(glm::vec(-2.0f, 4.0f, -1.0f), glm::vec3(0.0f), glm::vec3(1.0));
```

二者相結合為我們提供了一個光空間的變換矩陣，它將每個世界空間座標變換到光源處所見到的那個空間；這正是我們渲染深度貼圖所需要的。

```c++
glm::mat4 lightSpaceMatrix = lightProjection * lightView;
```

這個lightSpaceMatrix正是前面我們稱為T的那個變換矩陣。有了lightSpaceMatrix只要給shader提供光空間的投影和視圖矩陣，我們就能像往常那樣渲染場景了。然而，我們只關心深度值，並非所有片元計算都在我們的著色器中進行。為了提升性能，我們將使用一個與之不同但更為簡單的著色器來渲染出深度貼圖。

### 渲染出深度貼圖

當我們以光的透視圖進行場景渲染的時候，我們會用一個比較簡單的著色器，這個著色器除了把頂點變換到光空間以外，不會做得更多了。這個簡單的著色器叫做simpleDepthShader，就是使用下面的這個著色器：

```c++
#version 330 core
layout (location = 0) in vec3 position;
 
uniform mat4 lightSpaceMatrix;
uniform mat4 model;
 
void main()
{
    gl_Position = lightSpaceMatrix * model * vec4(position, 1.0f);
}
```

這個頂點著色器將一個單獨模型的一個頂點，使用lightSpaceMatrix變換到光空間中。

由於我們沒有顏色緩衝，最後的片元不需要任何處理，所以我們可以簡單地使用一個空像素著色器：

```c++
#version 330 core
 
void main()
{             
    // gl_FragDepth = gl_FragCoord.z;
}
```

這個空像素著色器什麼也不幹，運行完後，深度緩衝會被更新。我們可以取消那行的註釋，來顯式設置深度，但是這個（指註釋掉那行之後）是更有效率的，因為底層無論如何都會默認去設置深度緩衝。

渲染深度緩衝現在成了：

```c++
simpleDepthShader.Use();
glUniformMatrix4fv(lightSpaceMatrixLocation, 1, GL_FALSE, glm::value_ptr(lightSpaceMatrix));
 
glViewport(0, 0, SHADOW_WIDTH, SHADOW_HEIGHT);
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
    glClear(GL_DEPTH_BUFFER_BIT);
    RenderScene(simpleDepthShader);
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

這裡的RenderScene函數的參數是一個著色器程序（shader program），它調用所有相關的繪製函數，並在需要的地方設置相應的模型矩陣。

最後，在光的透視圖視角下，很完美地用每個可見片元的最近深度填充了深度緩衝。通過將這個紋理投射到一個2D四邊形上（和我們在幀緩衝一節做的後處理過程類似），就能在屏幕上顯示出來，我們會獲得這樣的東西：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_depth_map.png)

將深度貼圖渲染到四邊形上的像素著色器：

```c++
#version 330 core
out vec4 color;
in vec2 TexCoords;
 
uniform sampler2D depthMap;
 
void main()
{             
    float depthValue = texture(depthMap, TexCoords).r;
    color = vec4(vec3(depthValue), 1.0);
}
```

要注意的是當用透視投影矩陣取代正交投影矩陣來顯示深度時，有一些輕微的改動，因為使用透視投影時，深度是非線性的。本節教程的最後，我們會討論這些不同之處。

你可以在[這裡](http://learnopengl.com/code_viewer.php?code=advanced-lighting/shadow_mapping_depth_map)獲得把場景渲染成深度貼圖的源碼。

### 渲染陰影

正確地生成深度貼圖以後我們就可以開始生成陰影了。這段代碼在像素著色器中執行，用來檢驗一個片元是否在陰影之中，不過我們在頂點著色器中進行光空間的變換：

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 texCoords;
 
out vec2 TexCoords;
 
out VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
    vec4 FragPosLightSpace;
} vs_out;
 
uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;
uniform mat4 lightSpaceMatrix;
 
void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
    vs_out.FragPos = vec3(model * vec4(position, 1.0));
    vs_out.Normal = transpose(inverse(mat3(model))) * normal;
    vs_out.TexCoords = texCoords;
    vs_out.FragPosLightSpace = lightSpaceMatrix * vec4(vs_out.FragPos, 1.0);
}
``` 

這兒的新的地方是FragPosLightSpace這個輸出向量。我們用同一個lightSpaceMatrix，把世界空間頂點位置轉換為光空間。頂點著色器傳遞一個普通的經變換的世界空間頂點位置vs_out.FragPos和一個光空間的vs_out.FragPosLightSpace給像素著色器。

像素著色器使用Blinn-Phong光照模型渲染場景。我們接著計算出一個shadow值，當fragment在陰影中時是1.0，在陰影外是0.0。然後，diffuse和specular顏色會乘以這個陰影元素。由於陰影不會是全黑的（由於散射），我們把ambient分量從乘法中剔除。

```c++
#version 330 core
out vec4 FragColor;
 
in VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
    vec4 FragPosLightSpace;
} fs_in;
 
uniform sampler2D diffuseTexture;
uniform sampler2D shadowMap;
 
uniform vec3 lightPos;
uniform vec3 viewPos;
 
float ShadowCalculation(vec4 fragPosLightSpace)
{
    [...]
}
 
void main()
{           
    vec3 color = texture(diffuseTexture, fs_in.TexCoords).rgb;
    vec3 normal = normalize(fs_in.Normal);
    vec3 lightColor = vec3(1.0);
    // Ambient
    vec3 ambient = 0.15 * color;
    // Diffuse
    vec3 lightDir = normalize(lightPos - fs_in.FragPos);
    float diff = max(dot(lightDir, normal), 0.0);
    vec3 diffuse = diff * lightColor;
    // Specular
    vec3 viewDir = normalize(viewPos - fs_in.FragPos);
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = 0.0;
    vec3 halfwayDir = normalize(lightDir + viewDir);  
    spec = pow(max(dot(normal, halfwayDir), 0.0), 64.0);
    vec3 specular = spec * lightColor;    
    // 計算陰影
    float shadow = ShadowCalculation(fs_in.FragPosLightSpace);       
    vec3 lighting = (ambient + (1.0 - shadow) * (diffuse + specular)) * color;    
    
    FragColor = vec4(lighting, 1.0f);
}
```

像素著色器大部分是從高級光照教程中複製過來，只不過加上了個陰影計算。我們聲明一個shadowCalculation函數，用它計算陰影。像素著色器的最後，我們我們把diffuse和specular乘以(1-陰影元素)，這表示這個片元有多大成分不在陰影中。這個像素著色器還需要兩個額外輸入，一個是光空間的片元位置和第一個渲染階段得到的深度貼圖。

首先要檢查一個片元是否在陰影中，把光空間片元位置轉換為裁切空間的標準化設備座標。當我們在頂點著色器輸出一個裁切空間頂點位置到gl_Position時，OpenGL自動進行一個透視除法，將裁切空間座標的範圍-w到w轉為-1到1，這要將x、y、z元素除以向量的w元素來實現。由於裁切空間的FragPosLightSpace並不會通過gl_Position傳到像素著色器裡，我們必須自己做透視除法：

```c++
float ShadowCalculation(vec4 fragPosLightSpace)
{
    // 執行透視除法
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    [...]
}
```

返回了片元在光空間的-1到1的範圍。

!!! Important

    當使用正交投影矩陣，頂點w元素仍保持不變，所以這一步實際上毫無意義。可是，當使用透視投影的時候就是必須的了，所以為了保證在兩種投影矩陣下都有效就得留著這行。

因為來自深度貼圖的深度在0到1的範圍，我們也打算使用projCoords從深度貼圖中去採樣，所以我們將NDC座標變換為0到1的範圍：
（譯者注：這裡的意思是，上面的projCoords的xyz分量都是[-1,1]（下面會指出這對於遠平面之類的點才成立），而為了和深度貼圖的深度相比較，z分量需要變換到[0,1]；為了作為從深度貼圖中採樣的座標，xy分量也需要變換到[0,1]。所以整個projCoords向量都需要變換到[0,1]範圍。）

```c++
projCoords = projCoords * 0.5 + 0.5;
```

有了這些投影座標，我們就能從深度貼圖中採樣得到0到1的結果，從第一個渲染階段的projCoords座標直接對應於變換過的NDC座標。我們將得到光的位置視野下最近的深度：

```c++
float closestDepth = texture(shadowMap, projCoords.xy).r;
```

為了得到片元的當前深度，我們簡單獲取投影向量的z座標，它等於來自光的透視視角的片元的深度。

```c++
float currentDepth = projCoords.z;
```

實際的對比就是簡單檢查currentDepth是否高於closetDepth，如果是，那麼片元就在陰影中。

```c++
float shadow = currentDepth > closestDepth  ? 1.0 : 0.0;
```

完整的shadowCalculation函數是這樣的：

```c++
float ShadowCalculation(vec4 fragPosLightSpace)
{
    // 執行透視除法
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    // 變換到[0,1]的範圍
    projCoords = projCoords * 0.5 + 0.5;
    // 取得最近點的深度(使用[0,1]範圍下的fragPosLight當座標)
    float closestDepth = texture(shadowMap, projCoords.xy).r; 
    // 取得當前片元在光源視角下的深度
    float currentDepth = projCoords.z;
    // 檢查當前片元是否在陰影中
    float shadow = currentDepth > closestDepth  ? 1.0 : 0.0;
 
    return shadow;
}
```

激活這個著色器，綁定合適的紋理，激活第二個渲染階段默認的投影以及視圖矩陣，結果如下圖所示：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_shadows.png)

如果你做對了，你會看到地板和上有立方體的陰影。你可以從這裡找到demo程序的[源碼](http://learnopengl.com/code_viewer.php?code=advanced-lighting/shadow_mapping_shadows)。

### 改進陰影貼圖

我們試圖讓陰影映射工作，但是你也看到了，陰影映射還是有點不真實，我們修復它才能獲得更好的效果，這是下面的部分所關注的焦點。

#### 陰影失真（shadow acne）

前面的圖片中明顯有不對的地方。放大看會發現明顯的線條樣式：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_acne.png)

我們可以看到地板四邊形渲染出很大一塊交替黑線。這種陰影貼圖的不真實感叫做陰影失真，下圖解釋了成因：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_acne_diagram.png)

因為陰影貼圖受限於解析度，在距離光源比較遠的情況下，多個片元可能從深度貼圖的同一個值中去採樣。圖片每個斜坡代表深度貼圖一個單獨的紋理像素。你可以看到，多個片元從同一個深度值進行採樣。

雖然很多時候沒問題，但是當光源以一個角度朝向表面的時候就會出問題，這種情況下深度貼圖也是從一個角度下進行渲染的。多個片元就會從同一個斜坡的深度紋理像素中採樣，有些在地板上面，有些在地板下面；這樣我們所得到的陰影就有了差異。因為這個，有些片元被認為是在陰影之中，有些不在，由此產生了圖片中的條紋樣式。

我們可以用一個叫做**陰影偏移**（shadow bias）的技巧來解決這個問題，我們簡單的對錶面的深度（或深度貼圖）應用一個偏移量，這樣片元就不會被錯誤地認為在表面之下了。

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_acne_bias.png)

使用了偏移量後，所有采樣點都獲得了比表面深度更小的深度值，這樣整個表面就正確地被照亮，沒有任何陰影。我們可以這樣實現這個偏移：

```c++
float bias = 0.005;
float shadow = currentDepth - bias > closestDepth  ? 1.0 : 0.0;
```

一個0.005的偏移就能幫到很大的忙，但是有些表面坡度很大，仍然會產生陰影失真。有一個更加可靠的辦法能夠根據表面朝向光線的角度更改偏移量：使用點乘：

```c++
float bias = max(0.05 * (1.0 - dot(normal, lightDir)), 0.005);
```

這裡我們有一個偏移量的最大值0.05，和一個最小值0.005，它們是基於表面法線和光照方向的。這樣像地板這樣的表面幾乎與光源垂直，得到的偏移就很小，而比如立方體的側面這種表面得到的偏移就更大。下圖展示了同一個場景，但使用了陰影偏移，效果的確更好：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_with_bias.png)

選用正確的偏移數值，在不同的場景中需要一些像這樣的輕微調校，但大多情況下，實際上就是增加偏移量直到所有失真都被移除的問題。

#### 懸浮

使用陰影偏移的一個缺點是你對物體的實際深度應用了平移。偏移有可能足夠大，以至於可以看出陰影相對實際物體位置的偏移，你可以從下圖看到這個現象（這是一個誇張的偏移值）：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_peter_panning.png)

這個陰影失真叫做Peter panning，因為物體看起來輕輕懸浮在表面之上（譯註Peter Pan就是童話彼得潘，而panning有平移、懸浮之意，而且彼得潘是個會飛的男孩…）。我們可以使用一個叫技巧解決大部分的Peter panning問題：當渲染深度貼圖時候使用正面剔除（front face culling）你也許記得在面剔除教程中OpenGL默認是背面剔除。我們要告訴OpenGL我們要剔除正面。

因為我們只需要深度貼圖的深度值，對於實體物體無論我們用它們的正面還是背面都沒問題。使用背面深度不會有錯誤，因為陰影在物體內部有錯誤我們也看不見。

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_culling.png)

為了修復peter遊移，我們要進行正面剔除，先必須開啟GL_CULL_FACE：

```c++
glCullFace(GL_FRONT);
RenderSceneToDepthMap();
glCullFace(GL_BACK); // 不要忘記設回原先的culling face
```

這十分有效地解決了peter panning的問題，但只針對實體物體，內部不會對外開口。我們的場景中，在立方體上工作的很好，但在地板上無效，因為正面剔除完全移除了地板。地面是一個單獨的平面，不會被完全剔除。如果有人打算使用這個技巧解決peter panning必須考慮到只有剔除物體的正面才有意義。

另一個要考慮到的地方是接近陰影的物體仍然會出現不正確的效果。必須考慮到何時使用正面剔除對物體才有意義。不過使用普通的偏移值通常就能避免peter panning。

#### 採樣超出

無論你喜不喜歡還有一個視覺差異，就是光的視錐不可見的區域一律被認為是處於陰影中，不管它真的處於陰影之中。出現這個狀況是因為超出光的視錐的投影座標比1.0大，這樣採樣的深度紋理就會超出他默認的0到1的範圍。根據紋理環繞方式，我們將會得到不正確的深度結果，它不是基於真實的來自光源的深度值。

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_outside_frustum.png)

你可以在圖中看到，光照有一個區域，超出該區域就成為了陰影；這個區域實際上代表著深度貼圖的大小，這個貼圖投影到了地板上。發生這種情況的原因是我們之前將深度貼圖的環繞方式設置成了GL_REPEAT。

我們寧可讓所有超出深度貼圖的座標的深度範圍是1.0，這樣超出的座標將永遠不在陰影之中。我們可以儲存一個邊框顏色，然後把深度貼圖的紋理環繞選項設置為GL_CLAMP_TO_BORDER：

```c++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER);
GLfloat borderColor[] = { 1.0, 1.0, 1.0, 1.0 };
glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);
```

現在如果我們採樣深度貼圖0到1座標範圍以外的區域，紋理函數總會返回一個1.0的深度值，陰影值為0.0。結果看起來會更真實：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_clamp_edge.png)

仍有一部分是黑暗區域。那裡的座標超出了光的正交視錐的遠平面。你可以看到這片黑色區域總是出現在光源視錐的極遠處。

當一個點比光的遠平面還要遠時，它的投影座標的z座標大於1.0。這種情況下，GL_CLAMP_TO_BORDER環繞方式不起作用，因為我們把座標的z元素和深度貼圖的值進行了對比；它總是為大於1.0的z返回true。

解決這個問題也很簡單，我們簡單的強制把shadow的值設為0.0，不管投影向量的z座標是否大於1.0：

```c++
float ShadowCalculation(vec4 fragPosLightSpace)
{
    [...]
    if(projCoords.z > 1.0)
        shadow = 0.0;
    
    return shadow;
}
```

檢查遠平面，並將深度貼圖限制為一個手工指定的邊界顏色，就能解決深度貼圖採樣超出的問題，我們最終會得到下面我們所追求的效果：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_over_sampling_fixed.png)

這些結果意味著，只有在深度貼圖範圍以內的被投影的fragment座標才有陰影，所以任何超出範圍的都將會沒有陰影。由於在遊戲中通常這隻發生在遠處，就會比我們之前的那個明顯的黑色區域效果更真實。

#### PCF

陰影現在已經附著到場景中了，不過這仍不是我們想要的。如果你放大看陰影，陰影映射對解析度的依賴很快變得很明顯。

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_zoom.png)

因為深度貼圖有一個固定的解析度，多個片元對應於一個紋理像素。結果就是多個片元會從深度貼圖的同一個深度值進行採樣，這幾個片元便得到的是同一個陰影，這就會產生鋸齒邊。

你可以通過增加深度貼圖解析度的方式來降低鋸齒塊，也可以嘗試儘可能的讓光的視錐接近場景。

另一個（並不完整的）解決方案叫做PCF（percentage-closer filtering），這是一種多個不同過濾方式的組合，它產生柔和陰影，使它們出現更少的鋸齒塊和硬邊。核心思想是從深度貼圖中多次採樣，每一次採樣的紋理座標都稍有不同。每個獨立的樣本可能在也可能不再陰影中。所有的次生結果接著結合在一起，進行平均化，我們就得到了柔和陰影。

一個簡單的PCF的實現是簡單的從紋理像素四周對深度貼圖採樣，然後把結果平均起來：

```c++
float shadow = 0.0;
vec2 texelSize = 1.0 / textureSize(shadowMap, 0);
for(int x = -1; x <= 1; ++x)
{
    for(int y = -1; y <= 1; ++y)
    {
        float pcfDepth = texture(shadowMap, projCoords.xy + vec2(x, y) * texelSize).r; 
        shadow += currentDepth - bias > pcfDepth ? 1.0 : 0.0;        
    }    
}
shadow /= 9.0;
```

這個textureSize返回一個給定採樣器紋理的0級mipmap的vec2類型的寬和高。用1除以它返回一個單獨紋理像素的大小，我們用以對紋理座標進行偏移，確保每個新樣本，來自不同的深度值。這裡我們採樣得到9個值，它們在投影座標的x和y值的周圍，為陰影阻擋進行測試，並最終通過樣本的總數目將結果平均化。

使用更多的樣本，更改texelSize變量，你就可以增加陰影的柔和程度。下面你可以看到應用了PCF的陰影：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_soft_shadows.png)

從稍微遠一點的距離看去，陰影效果好多了，也不那麼生硬了。如果你放大，仍會看到陰影貼圖解析度的不真實感，但通常對於大多數應用來說效果已經很好了。

你可以從[這裡](http://learnopengl.com/code_viewer.php?code=advanced-lighting/shadow_mapping)找到這個例子的全部源碼和第二個階段的[頂點](http://learnopengl.com/code_viewer.php?code=advanced-lighting/shadow_mapping&type=vertex)和[片段](http://learnopengl.com/code_viewer.php?code=advanced-lighting/shadow_mapping&type=fragment)著色器。

實際上PCF還有更多的內容，以及很多技術要點需要考慮以提升柔和陰影的效果，但處於本章內容長度考慮，我們將留在以後討論。

 

### 正交 vs 投影

在渲染深度貼圖的時候，正交和投影矩陣之間有所不同。正交投影矩陣並不會將場景用透視圖進行變形，所有視線/光線都是平行的，這使它對於定向光來說是個很好的投影矩陣。然而透視投影矩陣，會將所有頂點根據透視關係進行變形，結果因此而不同。下圖展示了兩種投影方式所產生的不同陰影區域：

![](http://learnopengl.com/img/advanced-lighting/shadow_mapping_projection.png)

透視投影對於光源來說更合理，不像定向光，它是有自己的位置的。透視投影因此更經常用在點光源和聚光燈上，而正交投影經常用在定向光上。

另一個細微差別是，透視投影矩陣，將深度緩衝視覺化經常會得到一個幾乎全白的結果。發生這個是因為透視投影下，深度變成了非線性的深度值，它的大多數可辨範圍接近於近平面。為了可以像使用正交投影一樣合適的觀察到深度值，你必須先講過非線性深度值轉變為線性的，我們在深度測試教程中已經討論過。

```c++
#version 330 core
out vec4 color;
in vec2 TexCoords;
 
uniform sampler2D depthMap;
uniform float near_plane;
uniform float far_plane;
 
float LinearizeDepth(float depth)
{
    float z = depth * 2.0 - 1.0; // Back to NDC 
    return (2.0 * near_plane * far_plane) / (far_plane + near_plane - z * (far_plane - near_plane));
}
 
void main()
{             
    float depthValue = texture(depthMap, TexCoords).r;
    color = vec4(vec3(LinearizeDepth(depthValue) / far_plane), 1.0); // perspective
    // color = vec4(vec3(depthValue), 1.0); // orthographic
}
```

這個深度值與我們見到的用正交投影的很相似。需要注意的是，這個只適用於調試；正交或投影矩陣的深度檢查仍然保持原樣，因為相關的深度並沒有改變。

### 附加資源

[Tutorial 16 : Shadow](http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-16-shadow-mapping/)

[mapping：opengl-tutorial.org](http://ogldev.atspace.co.uk/www/tutorial23/tutorial23.html) 提供的類似的陰影映射教程，裡面有一些額外的解釋。

[Shadow Mapping – Part 1：ogldev](http://ogldev.atspace.co.uk/www/tutorial23/tutorial23.html)提供的另一個陰影映射教程。

[How Shadow Mapping Works](https://www.youtube.com/watch?v=EsccgeUpdsM)：的一個第三方YouTube視頻教程，裡面解釋了陰影映射及其實現。

[Common Techniques to Improve Shadow Depth Maps](https://msdn.microsoft.com/en-us/library/windows/desktop/ee416324%28v=vs.85%29.aspx)：微軟的一篇好文章，其中理出了很多提升陰影貼圖質量的技術。