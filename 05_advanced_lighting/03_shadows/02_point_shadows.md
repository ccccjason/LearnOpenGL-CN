本文作者JoeyDeVries，由Django翻譯自[http://learnopengl.com](http://learnopengl.com)

## 點光源陰影(Shadow Mapping)

上個教程我們學到了如何使用陰影映射技術創建動態陰影。效果不錯，但它只適合定向光，因為陰影只是在單一定向光源下生成的。所以它也叫定向陰影映射，深度（陰影）貼圖生成自定向光的視角。

!!! Important

    本節我們的焦點是在各種方向生成動態陰影。這個技術可以適用於點光源，生成所有方向上的陰影。

這個技術叫做點光陰影，過去的名字是萬向陰影貼圖（omnidirectional shadow maps）技術。

本節代碼基於前面的陰影映射教程，所以如果你對傳統陰影映射不熟悉，還是建議先讀一讀陰影映射教程。
算法和定向陰影映射差不多：我們從光的透視圖生成一個深度貼圖，基於當前fragment位置來對深度貼圖採樣，然後用儲存的深度值和每個fragment進行對比，看看它是否在陰影中。定向陰影映射和萬向陰影映射的主要不同在於深度貼圖的使用上。

對於深度貼圖，我們需要從一個點光源的所有渲染場景，普通2D深度貼圖不能工作；如果我們使用cubemap會怎樣？因為cubemap可以儲存6個面的環境數據，它可以將整個場景渲染到cubemap的每個面上，把它們當作點光源四周的深度值來採樣。

![](http://learnopengl.com/img/advanced-lighting/point_shadows_diagram.png)

生成後的深度cubemap被傳遞到光照像素著色器，它會用一個方向向量來採樣cubemap，從而得到當前的fragment的深度（從光的透視圖）。大部分複雜的事情已經在陰影映射教程中討論過了。算法只是在深度cubemap生成上稍微複雜一點。

#### 生成深度cubemap

為創建一個光周圍的深度值的cubemap，我們必須渲染場景6次：每次一個面。顯然渲染場景6次需要6個不同的視圖矩陣，每次把一個不同的cubemap面附加到幀緩衝對象上。這看起來是這樣的：

```c++
for(int i = 0; i < 6; i++)
{
    GLuint face = GL_TEXTURE_CUBE_MAP_POSITIVE_X + i;
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, face, depthCubemap, 0);
    BindViewMatrix(lightViewMatrices[i]);
    RenderScene();  
}
```

這會很耗費性能因為一個深度貼圖下需要進行很多渲染調用。這個教程中我們將轉而使用另外的一個小技巧來做這件事，幾何著色器允許我們使用一次渲染過程來建立深度cubemap。

首先，我們需要創建一個cubemap：

```c++
GLuint depthCubemap;
glGenTextures(1, &depthCubemap);
```

然後生成cubemap的每個面，將它們作為2D深度值紋理圖像：

```c++
const GLuint SHADOW_WIDTH = 1024, SHADOW_HEIGHT = 1024;
glBindTexture(GL_TEXTURE_CUBE_MAP, depthCubemap);
for (GLuint i = 0; i < 6; ++i)
        glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_DEPTH_COMPONENT, 
                     SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
```

不要忘記設置合適的紋理參數：

```c++
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
```

正常情況下，我們把cubemap紋理的一個面附加到幀緩衝對象上，渲染場景6次，每次將幀緩衝的深度緩衝目標改成不同cubemap面。由於我們將使用一個幾何著色器，它允許我們把所有面在一個過程渲染，我們可以使用glFramebufferTexture直接把cubemap附加成幀緩衝的深度附件：

```c++
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
glFramebufferTexture(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, depthCubemap, 0);
glDrawBuffer(GL_NONE);
glReadBuffer(GL_NONE);
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

還要記得調用glDrawBuffer和glReadBuffer：當生成一個深度cubemap時我們只關心深度值，所以我們必須顯式告訴OpenGL這個幀緩衝對象不會渲染到一個顏色緩衝裡。

萬向陰影貼圖有兩個渲染階段：首先我們生成深度貼圖，然後我們正常使用深度貼圖渲染，在場景中創建陰影。幀緩衝對象和cubemap的處理看起是這樣的：

```c++
// 1. first render to depth cubemap
glViewport(0, 0, SHADOW_WIDTH, SHADOW_HEIGHT);
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
    glClear(GL_DEPTH_BUFFER_BIT);
    ConfigureShaderAndMatrices();
    RenderScene();
glBindFramebuffer(GL_FRAMEBUFFER, 0);
// 2. then render scene as normal with shadow mapping (using depth cubemap)
glViewport(0, 0, SCR_WIDTH, SCR_HEIGHT);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
ConfigureShaderAndMatrices();
glBindTexture(GL_TEXTURE_CUBE_MAP, depthCubemap);
RenderScene();
```

這個過程和默認的陰影映射一樣，儘管這次我們渲染和使用的是一個cubemap深度紋理，而不是2D深度紋理。在我們實際開始從光的視角的所有方向渲染場景之前，我們先得計算出合適的變換矩陣。

### 光空間的變換

設置了幀緩衝和cubemap，我們需要一些方法來講場景的所有幾何體變換到6個光的方向中相應的光空間。與陰影映射教程類似，我們將需要一個光空間的變換矩陣T，但是這次是每個面都有一個。

每個光空間的變換矩陣包含了投影和視圖矩陣。對於投影矩陣來說，我們將使用一個透視投影矩陣；光源代表一個空間中的點，所以透視投影矩陣更有意義。每個光空間變換矩陣使用同樣的投影矩陣：

```c++
GLfloat aspect = (GLfloat)SHADOW_WIDTH/(GLfloat)SHADOW_HEIGHT;
GLfloat near = 1.0f;
GLfloat far = 25.0f;
glm::mat4 shadowProj = glm::perspective(90.0f, aspect, near, far);
```

非常重要的一點是，這裡glm::perspective的視野參數，設置為90度。90度我們才能保證視野足夠大到可以合適地填滿cubemap的一個面，cubemap的所有面都能與其他面在邊緣對齊。

因為投影矩陣在每個方向上並不會改變，我們可以在6個變換矩陣中重複使用。我們要為每個方向提供一個不同的視圖矩陣。用glm::lookAt創建6個觀察方向，每個都按順序注視著cubemap的的一個方向：右、左、上、下、近、遠：

```c++
std::vector<glm::mat4> shadowTransforms;
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3(1.0,0.0,0.0), glm::vec3(0.0,-1.0,0.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3(-1.0,0.0,0.0), glm::vec3(0.0,-1.0,0.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3(0.0,1.0,0.0), glm::vec3(0.0,0.0,1.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3(0.0,-1.0,0.0), glm::vec3(0.0,0.0,-1.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3(0.0,0.0,1.0), glm::vec3(0.0,-1.0,0.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3(0.0,0.0,-1.0), glm::vec3(0.0,-1.0,0.0));
```

這裡我們創建了6個視圖矩陣，把它們乘以投影矩陣，來得到6個不同的光空間變換矩陣。glm::lookAt的target參數是它注視的cubemap的面的一個方向。

這些變換矩陣發送到著色器渲染到cubemap裡。

 

### 深度著色器

為了把值渲染到深度cubemap，我們將需要3個著色器：頂點和像素著色器，以及一個它們之間的幾何著色器。

幾何著色器是負責將所有世界空間的頂點變換到6個不同的光空間的著色器。因此頂點著色器簡單地將頂點變換到世界空間，然後直接發送到幾何著色器：

```c++
#version 330 core
layout (location = 0) in vec3 position;
 
uniform mat4 model;
 
void main()
{
    gl_Position = model * vec4(position, 1.0);
}
```

緊接著幾何著色器以3個三角形的頂點作為輸入，它還有一個光空間變換矩陣的uniform數組。幾何著色器接下來會負責將頂點變換到光空間；這裡它開始變得有趣了。

幾何著色器有一個內建變量叫做gl_Layer，它指定發散出基本圖形送到cubemap的哪個面。當不管它時，幾何著色器就會像往常一樣把它的基本圖形發送到輸送管道的下一階段，但當我們更新這個變量就能控制每個基本圖形將渲染到cubemap的哪一個面。當然這隻有當我們有了一個附加到激活的幀緩衝的cubemap紋理才有效：

```c++
#version 330 core
layout (triangles) in;
layout (triangle_strip, max_vertices=18) out;
 
uniform mat4 shadowMatrices[6];
 
out vec4 FragPos; // FragPos from GS (output per emitvertex)
 
void main()
{
    for(int face = 0; face < 6; ++face)
    {
        gl_Layer = face; // built-in variable that specifies to which face we render.
        for(int i = 0; i < 3; ++i) // for each triangle's vertices
        {
            FragPos = gl_in[i].gl_Position;
            gl_Position = shadowMatrices[face] * FragPos;
            EmitVertex();
        }    
        EndPrimitive();
    }
}
```

幾何著色器相對簡單。我們輸入一個三角形，輸出總共6個三角形（6*3頂點，所以總共18個頂點）。在main函數中，我們遍歷cubemap的6個面，我們每個面指定為一個輸出面，把這個面的interger（整數）存到gl_Layer。然後，我們通過把面的光空間變換矩陣乘以FragPos，將每個世界空間頂點變換到相關的光空間，生成每個三角形。注意，我們還要將最後的FragPos變量發送給像素著色器，我們需要計算一個深度值。

上個教程，我們使用的是一個空的像素著色器，讓OpenGL配置深度貼圖的深度值。這次我們將計算自己的深度，這個深度就是每個fragment位置和光源位置之間的線性距離。計算自己的深度值使得之後的陰影計算更加直觀。

```c++
#version 330 core
in vec4 FragPos;
 
uniform vec3 lightPos;
uniform float far_plane;
 
void main()
{
    // get distance between fragment and light source
    float lightDistance = length(FragPos.xyz - lightPos);
    
    // map to [0;1] range by dividing by far_plane
    lightDistance = lightDistance / far_plane;
    
    // Write this as modified depth
    gl_FragDepth = gl_FragCoord.z;
}
``` 

像素著色器將來自幾何著色器的FragPos、光的位置向量和視錐的遠平面值作為輸入。這裡我們把fragment和光源之間的距離，映射到0到1的範圍，把它寫入為fragment的深度值。

使用這些著色器渲染場景，cubemap附加的幀緩衝對象激活以後，你會得到一個完全填充的深度cubemap，以便於進行第二階段的陰影計算。

### 萬向陰影貼圖

所有事情都做好了，是時候來渲染萬向陰影了。這個過程和定向陰影映射教程相似，儘管這次我們綁定的深度貼圖是一個cubemap，而不是2D紋理，並且將光的投影的遠平面發送給了著色器。

```c++
glViewport(0, 0, SCR_WIDTH, SCR_HEIGHT);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
shader.Use();  
// ... send uniforms to shader (including light's far_plane value)
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_CUBE_MAP, depthCubemap);
// ... bind other textures
RenderScene();
``` 

這裡的renderScene函數在一個大立方體房間中渲染一些立方體，它們散落在大立方體各處，光源在場景中央。

頂點著色器和像素著色器和原來的陰影映射著色器大部分都一樣：不同之處是在光空間中像素著色器不再需要一個fragment位置，現在我們可以使用一個方向向量採樣深度值。

因為這個頂點著色器不再需要將他的位置向量變換到光空間，所以我們可以去掉FragPosLightSpace變量：

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
} vs_out;
 
uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;
 
void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
    vs_out.FragPos = vec3(model * vec4(position, 1.0));
    vs_out.Normal = transpose(inverse(mat3(model))) * normal;
    vs_out.TexCoords = texCoords;
}
```

片段著色器的Blinn-Phong光照代碼和我們之前陰影相乘的結尾部分一樣：

```c++
#version 330 core
out vec4 FragColor;
 
in VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
} fs_in;
 
uniform sampler2D diffuseTexture;
uniform samplerCube depthMap;
 
uniform vec3 lightPos;
uniform vec3 viewPos;
 
uniform float far_plane;
 
float ShadowCalculation(vec3 fragPos)
{
    [...]
}
 
void main()
{           
    vec3 color = texture(diffuseTexture, fs_in.TexCoords).rgb;
    vec3 normal = normalize(fs_in.Normal);
    vec3 lightColor = vec3(0.3);
    // Ambient
    vec3 ambient = 0.3 * color;
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
    // Calculate shadow
    float shadow = ShadowCalculation(fs_in.FragPos);                      
    vec3 lighting = (ambient + (1.0 - shadow) * (diffuse + specular)) * color;    
    
    FragColor = vec4(lighting, 1.0f);
}
```

有一些細微的不同：光照代碼一樣，但我們現在有了一個uniform變量samplerCube，shadowCalculation函數用fragment的位置作為它的參數，取代了光空間的fragment位置。我們現在還要引入光的視錐的遠平面值，後面我們會需要它。像素著色器的最後，我們計算出陰影元素，當fragment在陰影中時它是1.0，不在陰影中時是0.0。我們使用計算出來的陰影元素去影響光照的diffuse和specular元素。

在ShadowCalculation函數中有很多不同之處，現在是從cubemap中進行採樣，不再使用2D紋理了。我們來一步一步的討論一下的它的內容。

我們需要做的第一件事是獲取cubemap的森都。你可能已經從教程的cubemap部分想到，我們已經將深度儲存為fragment和光位置之間的距離了；我們這裡採用相似的處理方式：

```c++
float ShadowCalculation(vec3 fragPos)
{
    vec3 fragToLight = fragPos - lightPos; 
    float closestDepth = texture(depthMap, fragToLight).r;
}
```

在這裡，我們得到了fragment的位置與光的位置之間的不同的向量，使用這個向量作為一個方向向量去對cubemap進行採樣。方向向量不需要是單位向量，所以無需對它進行標準化。最後的closestDepth是光源和它最接近的可見fragment之間的標準化的深度值。

closestDepth值現在在0到1的範圍內了，所以我們先將其轉換會0到far_plane的範圍，這需要把他乘以far_plane：

```c++
closestDepth *= far_plane;
```

下一步我們獲取當前fragment和光源之間的深度值，我們可以簡單的使用fragToLight的長度來獲取它，這取決於我們如何計算cubemap中的深度值：

```c++
float currentDepth = length(fragToLight);
```

返回的是和closestDepth範圍相同的深度值。

現在我們可以將兩個深度值對比一下，看看哪一個更接近，以此決定當前的fragment是否在陰影當中。我們還要包含一個陰影偏移，所以才能避免陰影失真，這在前面教程中已經討論過了。

```c++
float bias = 0.05; 
float shadow = currentDepth -  bias > closestDepth ? 1.0 : 0.0;
```

完整的ShadowCalculation現在變成了這樣：

```c++
float ShadowCalculation(vec3 fragPos)
{
    // Get vector between fragment position and light position
    vec3 fragToLight = fragPos - lightPos;
    // Use the light to fragment vector to sample from the depth map    
    float closestDepth = texture(depthMap, fragToLight).r;
    // It is currently in linear range between [0,1]. Re-transform back to original value
    closestDepth *= far_plane;
    // Now get current linear depth as the length between the fragment and light position
    float currentDepth = length(fragToLight);
    // Now test for shadows
    float bias = 0.05; 
    float shadow = currentDepth -  bias > closestDepth ? 1.0 : 0.0;
 
    return shadow;
}
```

有了這些著色器，我們已經能得到非常好的陰影效果了，這次從一個點光源所有周圍方向上都有陰影。有一個位於場景中心的點光源，看起來會像這樣：

![](http://learnopengl.com/img/advanced-lighting/point_shadows.png)

你可以從這裡找到這個[demo的源碼](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/point_shadows)、[頂點](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/point_shadows&type=vertex)和[片段](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/point_shadows&type=fragment)著色器。

#### 把cubemap深度緩衝顯示出來

如果你想我一樣第一次並沒有做對，那麼就要進行調試排錯，將深度貼圖顯示出來以檢查其是否正確。因為我們不再用2D深度貼圖紋理，深度貼圖的顯示不會那麼顯而易見。

一個簡單的把深度緩衝顯示出來的技巧是，在ShadowCalculation函數中計算標準化的closestDepth變量，把變量顯示為：

```c++
FragColor = vec4(vec3(closestDepth / far_plane), 1.0);
```

結果是一個灰度場景，每個顏色代表著場景的線性深度值：

![](http://learnopengl.com/img/advanced-lighting/point_shadows_depth_cubemap.png)

你可能也注意到了帶陰影部分在牆外。如果看起來和這個差不多，你就知道深度cubemap生成的沒錯。否則你可能做錯了什麼，也許是closestDepth仍然還在0到far_plane的範圍。

#### PCF

由於萬向陰影貼圖基於傳統陰影映射的原則，它便也繼承了由解析度產生的非真實感。如果你放大就會看到鋸齒邊了。PCF或稱Percentage-closer filtering允許我們通過對fragment位置周圍過濾多個樣本，並對結果平均化。

如果我們用和前面教程同樣的那個簡單的PCF過濾器，並加入第三個維度，就是這樣的：

```c+++
float shadow = 0.0;
float bias = 0.05; 
float samples = 4.0;
float offset = 0.1;
for(float x = -offset; x < offset; x += offset / (samples * 0.5))
{
    for(float y = -offset; y < offset; y += offset / (samples * 0.5))
    {
        for(float z = -offset; z < offset; z += offset / (samples * 0.5))
        {
            float closestDepth = texture(depthMap, fragToLight + vec3(x, y, z)).r; 
            closestDepth *= far_plane;   // Undo mapping [0;1]
            if(currentDepth - bias > closestDepth)
                shadow += 1.0;
        }
    }
}
shadow /= (samples * samples * samples);
```

這段代碼和我們傳統的陰影映射沒有多少不同。這裡我們根據樣本的數量動態計算了紋理偏移量，我們在三個軸向採樣三次，最後對子樣本進行平均化。

現在陰影看起來更加柔和平滑了，由此得到更加真實的效果：

![](http://learnopengl.com/img/advanced-lighting/point_shadows_soft.png)

然而，samples設置為4.0，每個fragment我們會得到總共64個樣本，這太多了！

大多數這些樣本都是多餘的，它們在原始方向向量近處採樣，不如在採樣方向向量的垂直方向進行採樣更有意義。可是，沒有（簡單的）方式能夠指出哪一個子方向是多餘的，這就難了。有個技巧可以使用，用一個偏移量方向數組，它們差不多都是分開的，每一個指向完全不同的方向，剔除彼此接近的那些子方向。下面就是一個有著20個偏移方向的數組：

```c++
vec3 sampleOffsetDirections[20] = vec3[]
(
   vec3( 1,  1,  1), vec3( 1, -1,  1), vec3(-1, -1,  1), vec3(-1,  1,  1), 
   vec3( 1,  1, -1), vec3( 1, -1, -1), vec3(-1, -1, -1), vec3(-1,  1, -1),
   vec3( 1,  1,  0), vec3( 1, -1,  0), vec3(-1, -1,  0), vec3(-1,  1,  0),
   vec3( 1,  0,  1), vec3(-1,  0,  1), vec3( 1,  0, -1), vec3(-1,  0, -1),
   vec3( 0,  1,  1), vec3( 0, -1,  1), vec3( 0, -1, -1), vec3( 0,  1, -1)
);
```

然後我們把PCF算法與從sampleOffsetDirections得到的樣本數量進行適配，使用它們從cubemap裡採樣。這麼做的好處是與之前的PCF算法相比，我們需要的樣本數量變少了。

```c++
float shadow = 0.0;
float bias = 0.15;
int samples = 20;
float viewDistance = length(viewPos - fragPos);
float diskRadius = 0.05;
for(int i = 0; i < samples; ++i)
{
    float closestDepth = texture(depthMap, fragToLight + sampleOffsetDirections[i] * diskRadius).r;
    closestDepth *= far_plane;   // Undo mapping [0;1]
    if(currentDepth - bias > closestDepth)
        shadow += 1.0;
}
shadow /= float(samples);
```

這裡我們把一個偏移量添加到指定的diskRadius中，它在fragToLight方向向量周圍從cubemap裡採樣。

另一個在這裡可以應用的有意思的技巧是，我們可以基於觀察者裡一個fragment的距離來改變diskRadius；這樣我們就能根據觀察者的距離來增加偏移半徑了，當距離更遠的時候陰影更柔和，更近了就更銳利。

```c++
float diskRadius = (1.0 + (viewDistance / far_plane)) / 25.0;
```

PCF算法的結果如果沒有變得更好，也是非常不錯的，這是柔和的陰影效果：

![](http://learnopengl.com/img/advanced-lighting/point_shadows_soft_better.png)

當然了，我們添加到每個樣本的bias（偏移）高度依賴於上下文，總是要根據場景進行微調的。試試這些值，看看怎樣影響了場景。
這裡是最終版本的頂點和像素著色器。

我還要提醒一下使用幾何著色器來生成深度貼圖不會一定比每個面渲染場景6次更快。使用幾何著色器有它自己的性能侷限，在第一個階段使用它可能獲得更好的性能表現。這取決於環境的類型，以及特定的顯卡驅動等等，所以如果你很關心性能，就要確保對兩種方法有大致瞭解，然後選擇對你場景來說更高效的那個。我個人還是喜歡使用幾何著色器來進行陰影映射，原因很簡單，因為它們使用起來更簡單。

 

### 附加資源

[Shadow Mapping for point light sources in OpenGL](http://www.sunandblackcat.com/tipFullView.php?l=eng&topicid=36)：sunandblackcat的萬向陰影映射教程。

[Multipass Shadow Mapping With Point Lights](http://ogldev.atspace.co.uk/www/tutorial43/tutorial43.html)：ogldev的萬向陰影映射教程。

[Omni-directional Shadows](http://www.cg.tuwien.ac.at/~husky/RTR/OmnidirShadows-whyCaps.pdf)：Peter Houska的關於萬向陰影映射的一組很好的ppt。