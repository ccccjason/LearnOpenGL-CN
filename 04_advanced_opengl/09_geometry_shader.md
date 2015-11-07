# 幾何著色器

原文     | [Geometry Shader](http://learnopengl.com/#!Advanced-OpenGL/Geometry-Shader)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

## 幾何著色器(Geometry Shader)

在頂點和片段著色器之間有一個可選的著色器，叫做幾何著色器（geometry shader）。幾何著色器以一個或多個表示為一個單獨基本圖形（primitive）的頂點作為輸入，比如可以是一個點或者三角形。幾何著色器在將這些頂點發送到下一個著色階段之前，可以將這些頂點轉變為它認為合適的內容。幾何著色器有意思的地方在於它可以把（一個或多個）頂點轉變為完全不同的基本圖形（primitive），從而生成比原來多得多的頂點。

我們直接用一個例子深入瞭解一下：

```c++
#version 330 core
layout (points) in;
layout (line_strip, max_vertices = 2) out;

void main() {
    gl_Position = gl_in[0].gl_Position + vec4(-0.1, 0.0, 0.0, 0.0);
    EmitVertex();

    gl_Position = gl_in[0].gl_Position + vec4(0.1, 0.0, 0.0, 0.0);
    EmitVertex();

    EndPrimitive();
}
```

每個幾何著色器開始位置我們需要聲明輸入的基本圖形(primitive)類型，這個輸入是我們從頂點著色器中接收到的。我們在in關鍵字前面聲明一個layout標識符。這個輸入layout修飾符可以從一個頂點著色器接收以下基本圖形值：


 基本圖形|描述
---|---
points |繪製GL_POINTS基本圖形的時候（1）
lines  |當繪製GL_LINES或GL_LINE_STRIP（2）時
lines_adjacency | GL_LINES_ADJACENCY或GL_LINE_STRIP_ADJACENCY（4）
triangles |GL_TRIANGLES, GL_TRIANGLE_STRIP或GL_TRIANGLE_FAN（3）
triangles_adjacency |GL_TRIANGLES_ADJACENCY或GL_TRIANGLE_STRIP_ADJACENCY（6）

這是我們能夠給渲染函數的幾乎所有的基本圖形。如果我們選擇以GL_TRIANGLES繪製頂點，我們要把輸入修飾符設置為triangles。括號裡的數字代表一個基本圖形所能包含的最少的頂點數。

當我們需要指定一個幾何著色器所輸出的基本圖形類型時，我們就在out關鍵字前面加一個layout修飾符。和輸入layout標識符一樣，輸出的layout標識符也可以接受以下基本圖形值：

* points
* line_strip
* triangle_strip

使用這3個輸出修飾符我們可以從輸入的基本圖形創建任何我們想要的形狀。為了生成一個三角形，我們定義一個triangle_strip作為輸出，然後輸出3個頂點。

幾何著色器同時希望我們設置一個它能輸出的頂點數量的最大值（如果你超出了這個數值，OpenGL就會忽略剩下的頂點），我們可以在out關鍵字的layout標識符上做這件事。在這個特殊的情況中，我們將使用最大值為2個頂點，來輸出一個line_strip。

這種情況，你會奇怪什麼是線條：一個線條是把多個點鏈接起來表示出一個連續的線，它最少有兩個點來組成。每個後一個點在前一個新渲染的點後面渲染，你可以看看下面的圖，其中包含5個頂點：

![](http://learnopengl.com/img/advanced/geometry_shader_line_strip.png)

上面的著色器，我們只能輸出一個線段，因為頂點的最大值設置為2。

為生成更有意義的結果，我們需要某種方式從前一個著色階段獲得輸出。GLSL為我們提供了一個內建變量，它叫做**gl_in**，它的內部看起來可能像這樣：

```c++
in gl_Vertex
{
    vec4 gl_Position;
    float gl_PointSize;
    float gl_ClipDistance[];
} gl_in[];

```

這裡它被聲明為一個接口塊(interface block，前面的教程已經討論過)，它包含幾個有意思的變量，其中最有意思的是`gl_Position`，它包含著和我們設置的頂點著色器的輸出相似的向量。

要注意的是，它被聲明為一個數組，因為大多數渲染基本圖形由一個以上頂點組成，幾何著色器接收一個基本圖形的所有頂點作為它的輸入。

使用從前一個頂點著色階段的頂點數據，我們就可以開始生成新的數據了，這是通過2個幾何著色器函數`EmitVertex`和`EndPrimitive`來完成的。幾何著色器需要你去生成/輸出至少一個你定義為輸出的基本圖形。在我們的例子裡我們打算至少生成一個線條（line strip）基本圖形。

```c++
void main() {
    gl_Position = gl_in[0].gl_Position + vec4(-0.1, 0.0, 0.0, 0.0);
    EmitVertex();

    gl_Position = gl_in[0].gl_Position + vec4(0.1, 0.0, 0.0, 0.0);
    EmitVertex();

    EndPrimitive();
}
```

每次我們調用`EmitVertex`，當前設置到`gl_Position`的向量就會被添加到基本圖形上。無論何時調用`EndPrimitive`，所有為這個基本圖形發射出去的頂點都將結合為一個特定的輸出渲染基本圖形。一個或多個`EmitVertex`函數調用後，重複調用`EndPrimitive`就能生成多個基本圖形。這個特殊的例子裡，發射了兩個頂點，它們被從頂點原來的位置平移了一段距離，然後調用`EndPrimitive`將這兩個頂點結合為一個單獨的有兩個頂點的線條。

現在你瞭解了幾何著色器的工作方式，你就可能猜出這個幾何著色器做了什麼。這個幾何著色器接收一個基本圖形——點，作為它的輸入，使用輸入點作為它的中心，創建了一個水平線基本圖形。如果我們渲染它，結果就會像這樣：

![](http://bullteacher.com/wp-content/uploads/2015/06/geometry_shader_lines.png)

並不是非常引人注目，但是考慮到它的輸出是使用下面的渲染命令生成的就很有意思了：

```c++
glDrawArrays(GL_POINTS, 0, 4);
```

這是個相對簡單的例子，它向你展示了我們如何使用幾何著色器來動態地在運行時生成新的形狀。本章的後面，我們會討論一些可以使用幾何著色器獲得有趣的效果，但是現在我們將以創建一個簡單的幾何著色器開始。

## 使用幾何著色器

為了展示幾何著色器的使用，我們將渲染一個簡單的場景，在場景中我們只繪製4個點，這4個點在標準化設備座標的z平面上。這些點的座標是：

```c++
GLfloat points[] = {
 -0.5f,  0.5f, // 左上方
 0.5f,  0.5f,  // 右上方
 0.5f, -0.5f,  // 右下方
 -0.5f, -0.5f  // 左下方
};
```

頂點著色器只在z平面繪製點，所以我們只需要一個基本頂點著色器：

```c++
#version 330 core
layout (location = 0) in vec2 position;

void main()
{
    gl_Position = vec4(position.x, position.y, 0.0f, 1.0f);
}
```

我們會簡單地為所有點輸出綠色，我們直接在片段著色器裡進行硬編碼：

```c++
#version 330 core
out vec4 color;

void main()
{
    color = vec4(0.0f, 1.0f, 0.0f, 1.0f);
}
```

為點的頂點生成一個VAO和VBO，然後使用`glDrawArrays`進行繪製：

```c++
shader.Use();
glBindVertexArray(VAO);
glDrawArrays(GL_POINTS, 0, 4);
glBindVertexArray(0);
```

效果是黑色場景中有四個綠點（雖然很難看到）：

![](http://learnopengl.com/img/advanced/geometry_shader_points.png)

但我們不是已經學到了所有內容了嗎？對，現在我們將通過為場景添加一個幾何著色器來為這個小場景增加點活力。

出於學習的目的我們將創建一個叫pass-through的幾何著色器，它用一個point基本圖形作為它的輸入，並把它無修改地傳（pass）到下一個著色器。

```c++
#version 330 core
layout (points) in;
layout (points, max_vertices = 1) out;

void main() {
    gl_Position = gl_in[0].gl_Position;
    EmitVertex();
    EndPrimitive();
}
```

現在這個幾何著色器應該很容易理解了。它簡單地將它接收到的輸入的無修改的頂點位置發射出去，然後生成一個point基本圖形。

一個幾何著色器需要像頂點和片段著色器一樣被編譯和鏈接，但是這次我們將使用`GL_GEOMETRY_SHADER`作為著色器的類型來創建這個著色器：

```c++
geometryShader = glCreateShader(GL_GEOMETRY_SHADER);
glShaderSource(geometryShader, 1, &gShaderCode, NULL);
glCompileShader(geometryShader);  
...
glAttachShader(program, geometryShader);
glLinkProgram(program);
```

編譯著色器的代碼和頂點、片段著色器的基本一樣。要記得檢查編譯和鏈接錯誤！

如果你現在編譯和運行，就會看到和下面相似的結果：

![](http://learnopengl.com/img/advanced/geometry_shader_points.png)

它和沒用幾何著色器一樣！我承認有點無聊，但是事實上，我們仍能繪製證明幾何著色器工作了的點，所以現在是時候來做點更有意思的事了！


### 創建幾個房子

繪製點和線沒什麼意思，所以我們將在每個點上使用幾何著色器繪製一個房子。我們可以通過把幾何著色器的輸出設置為`triangle_strip`來達到這個目的，總共要繪製3個三角形：兩個用來組成方形和另表示一個屋頂。

在OpenGL中三角形帶(triangle strip)繪製起來更高效，因為它所使用的頂點更少。第一個三角形繪製完以後，每個後續的頂點會生成一個毗連前一個三角形的新三角形：每3個毗連的頂點都能構成一個三角形。如果我們有6個頂點，它們以三角形帶的方式組合起來，那麼我們會得到這些三角形：（1, 2, 3）、（2, 3, 4）、（3, 4, 5）、（4,5,6）因此總共可以表示出4個三角形。一個三角形帶至少要用3個頂點才行，它能生曾N-2個三角形；6個頂點我們就能創建6-2=4個三角形。下面的圖片表達了這點：

![](http://learnopengl.com/img/advanced/geometry_shader_triangle_strip.png)

使用一個三角形帶作為一個幾何著色器的輸出，我們可以輕鬆創建房子的形狀，只要以正確的順序來生成3個毗連的三角形。下面的圖像顯示，我們需要以何種順序來繪製點，才能獲得我們需要的三角形，圖上的藍點代表輸入點：

![](http://learnopengl.com/img/advanced/geometry_shader_house.png)

上圖的內容轉變為幾何著色器：

```c++
#version 330 core
layout (points) in;
layout (triangle_strip, max_vertices = 5) out;

void build_house(vec4 position)
{
    gl_Position = position + vec4(-0.2f, -0.2f, 0.0f, 0.0f);// 1:左下角
    EmitVertex();
    gl_Position = position + vec4( 0.2f, -0.2f, 0.0f, 0.0f);// 2:右下角
    EmitVertex();
    gl_Position = position + vec4(-0.2f,  0.2f, 0.0f, 0.0f);// 3:左上
    EmitVertex();
    gl_Position = position + vec4( 0.2f,  0.2f, 0.0f, 0.0f);// 4:右上
    EmitVertex();
    gl_Position = position + vec4( 0.0f,  0.4f, 0.0f, 0.0f);// 5:屋頂
    EmitVertex();
    EndPrimitive();
}

void main()
{
    build_house(gl_in[0].gl_Position);
}
```

這個幾何著色器生成5個頂點，每個頂點是點（point）的位置加上一個偏移量，來組成一個大三角形帶。接著最後的基本圖形被像素化，片段著色器處理整三角形帶，結果是為我們繪製的每個點生成一個綠房子：

![](http://learnopengl.com/img/advanced/geometry_shader_houses.png)

可以看到，每個房子實則是由3個三角形組成，都是僅僅使用空間中一點來繪製的。綠房子看起來還是不夠漂亮，所以我們再給每個房子加一個不同的顏色。我們將在頂點著色器中為每個頂點增加一個額外的代表顏色信息的頂點屬性。

下面是更新了的頂點數據：

```c++
GLfloat points[] = {
    -0.5f,  0.5f, 1.0f, 0.0f, 0.0f, // 左上
     0.5f,  0.5f, 0.0f, 1.0f, 0.0f, // 右上
     0.5f, -0.5f, 0.0f, 0.0f, 1.0f, // 右下
    -0.5f, -0.5f, 1.0f, 1.0f, 0.0f  // 左下
};
```

然後我們更新頂點著色器，使用一個接口塊來項幾何著色器發送顏色屬性：

```c++
#version 330 core
layout (location = 0) in vec2 position;
layout (location = 1) in vec3 color;

out VS_OUT {
    vec3 color;
} vs_out;

void main()
{
    gl_Position = vec4(position.x, position.y, 0.0f, 1.0f);
    vs_out.color = color;
}
```

接著我們還需要在幾何著色器中聲明同樣的接口塊(使用一個不同的接口名)：

```c++
in VS_OUT {
    vec3 color;
} gs_in[];
```

因為幾何著色器把多個頂點作為它的輸入，從頂點著色器來的輸入數據總是被以數組的形式表示出來，即使現在我們只有一個頂點。

!!! Important

        我們不是必須使用接口塊來把數據發送到幾何著色器中。我們還可以這麼寫：

        in vec3 vColor[];

        如果頂點著色器發送的顏色向量是out vec3 vColor那麼接口塊就會在比如幾何著色器這樣的著色器中更輕鬆地完成工作。事實上，幾何著色器的輸入可以非常大，把它們組成一個大的接口塊數組會更有意義。


然後我們還要為下一個像素著色階段聲明一個輸出顏色向量：

```c++
out vec3 fColor;
```

因為片段著色器只需要一個（已進行了插值的）顏色，傳送多個顏色沒有意義。fColor向量這樣就不是一個數組，而是一個單一的向量。當發射一個頂點時，為了它的片段著色器運行，每個頂點都會儲存最後在fColor中儲存的值。對於這些房子來說，我們可以在第一個頂點被髮射，對整個房子上色前，只使用來自頂點著色器的顏色填充fColor一次：

```c++
fColor = gs_in[0].color; //只有一個輸出顏色，所以直接設置為gs_in[0]
gl_Position = position + vec4(-0.2f, -0.2f, 0.0f, 0.0f);    // 1:左下
EmitVertex();
gl_Position = position + vec4( 0.2f, -0.2f, 0.0f, 0.0f);    // 2:右下
EmitVertex();
gl_Position = position + vec4(-0.2f,  0.2f, 0.0f, 0.0f);    // 3:左上
EmitVertex();
gl_Position = position + vec4( 0.2f,  0.2f, 0.0f, 0.0f);    // 4:右上
EmitVertex();
gl_Position = position + vec4( 0.0f,  0.4f, 0.0f, 0.0f);    // 5:屋頂
EmitVertex();
EndPrimitive();
```

所有發射出去的頂點都把最後儲存在fColor中的值嵌入到他們的數據中，和我們在他們的屬性中定義的頂點顏色相同。所有的分房子便都有了自己的顏色：

![](http://learnopengl.com/img/advanced/geometry_shader_houses_colored.png)

為了好玩兒，我們還可以假裝這是在冬天，給最後一個頂點一個自己的白色，就像在屋頂上落了一些雪。

```c++
fColor = gs_in[0].color;
gl_Position = position + vec4(-0.2f, -0.2f, 0.0f, 0.0f); 
EmitVertex();
gl_Position = position + vec4( 0.2f, -0.2f, 0.0f, 0.0f);
EmitVertex();
gl_Position = position + vec4(-0.2f,  0.2f, 0.0f, 0.0f); 
EmitVertex();
gl_Position = position + vec4( 0.2f,  0.2f, 0.0f, 0.0f);  
EmitVertex();
gl_Position = position + vec4( 0.0f,  0.4f, 0.0f, 0.0f); 
fColor = vec3(1.0f, 1.0f, 1.0f);
EmitVertex();
EndPrimitive();

```

結果就像這樣：

![](http://learnopengl.com/img/advanced/geometry_shader_houses_snow.png)

你可以對比一下你的[源碼](http://learnopengl.com/code_viewer.php?code=advanced/geometry_shader_houses)和[著色器](http://learnopengl.com/code_viewer.php?code=advanced/geometry_shader_houses_shaders)。

你可以看到，使用幾何著色器，你可以使用最簡單的基本圖形就能獲得漂亮的新玩意。因為這些形狀是在你的GPU超快硬件上動態生成的，這要比使用頂點緩衝自己定義這些形狀更為高效。幾何緩衝在簡單的經常被重複的形狀比如體素（voxel）的世界和室外的草地上，是一種非常強大的優化工具。

### 爆炸式物體

繪製房子的確很有趣，但我們不會經常這麼用。這就是為什麼現在我們將撬起物體缺口，形成爆炸式物體的原因！雖然這個我們也不會經常用到，但是它能向你展示一些幾何著色器的強大之處。

當我們說對一個物體進行爆破的時候並不是說我們將要把之前的那堆頂點炸掉，但是我們打算把每個三角形沿著它們的法線向量移動一小段距離。效果是整個物體上的三角形看起來就像沿著它們的法線向量爆炸了一樣。納米服上的三角形的爆炸式效果看起來是這樣的：

![](http://learnopengl.com/img/advanced/geometry_shader_explosion.png)

這樣一個幾何著色器效果的一大好處是，它可以用到任何物體上，無論它們多複雜。

因為我們打算沿著三角形的法線向量移動三角形的每個頂點，我們需要先計算它的法線向量。我們要做的是計算出一個向量，它垂直於三角形的表面，使用這三個我們已經的到的頂點就能做到。你可能記得變換教程中，我們可以使用叉乘獲取一個垂直於兩個其他向量的向量。如果我們有兩個向量a和b，它們平行於三角形的表面，我們就可以對這兩個向量進行叉乘得到法線向量了。下面的幾何著色器函數做的正是這件事，它使用3個輸入頂點座標獲取法線向量：

```c++
vec3 GetNormal()
{
   vec3 a = vec3(gl_in[0].gl_Position) - vec3(gl_in[1].gl_Position);
   vec3 b = vec3(gl_in[2].gl_Position) - vec3(gl_in[1].gl_Position);
   return normalize(cross(a, b));
}
```

這裡我們使用減法獲取了兩個向量a和b，它們平行於三角形的表面。兩個向量相減得到一個兩個向量的差值，由於所有3個點都在三角形平面上，任何向量相減都會得到一個平行於平面的向量。一定要注意，如果我們調換了a和b的叉乘順序，我們得到的法線向量就會使反的，順序很重要！

知道了如何計算法線向量，我們就能創建一個explode函數，函數返回的是一個新向量，它把位置向量沿著法線向量方向平移：

```c++
vec4 explode(vec4 position, vec3 normal)
{
    float magnitude = 2.0f;
    vec3 direction = normal * ((sin(time) + 1.0f) / 2.0f) * magnitude;
    return position + vec4(direction, 0.0f);
}
```

函數本身並不複雜，sin（正弦）函數把一個time變量作為它的參數，它根據時間來返回一個-1.0到1.0之間的值。因為我們不想讓物體坍縮，所以我們把sin返回的值做成0到1的範圍。最後的值去乘以法線向量，direction向量被添加到位置向量上。

爆炸效果的完整的幾何著色器是這樣的，它使用我們的模型加載器，繪製出一個模型：

```c++
#version 330 core
layout (triangles) in;
layout (triangle_strip, max_vertices = 3) out;

in VS_OUT {
    vec2 texCoords;
} gs_in[];

out vec2 TexCoords;

uniform float time;

vec4 explode(vec4 position, vec3 normal) { ... }

vec3 GetNormal() { ... }

void main() {
    vec3 normal = GetNormal();

    gl_Position = explode(gl_in[0].gl_Position, normal);
    TexCoords = gs_in[0].texCoords;
    EmitVertex();
    gl_Position = explode(gl_in[1].gl_Position, normal);
    TexCoords = gs_in[1].texCoords;
    EmitVertex();
    gl_Position = explode(gl_in[2].gl_Position, normal);
    TexCoords = gs_in[2].texCoords;
    EmitVertex();
    EndPrimitive();
}
```

注意我們同樣在發射一個頂點前輸出了合適的紋理座標。

也不要忘記在OpenGL代碼中設置time變量：

```c++
glUniform1f(glGetUniformLocation(shader.Program, "time"), glfwGetTime());
 ```

最後的結果是一個隨著時間持續不斷地爆炸的3D模型（不斷爆炸不斷回到正常狀態）。儘管沒什麼大用處，它卻向你展示出很多幾何著色器的高級用法。你可以用[完整的源碼](http://learnopengl.com/code_viewer.php?code=advanced/geometry_shader_explode)和[著色器](http://learnopengl.com/code_viewer.php?code=advanced/geometry_shader_explode_shaders)對比一下你自己的。

### 把法線向量顯示出來

在這部分我們將使用幾何著色器寫一個例子，非常有用：顯示一個法線向量。當編寫光照著色器的時候，你最終會遇到奇怪的視頻輸出問題，你很難決定是什麼導致了這個問題。通常導致光照錯誤的是，不正確的加載頂點數據，以及給它們指定了不合理的頂點屬性，又或是在著色器中不合理的管理，導致產生了不正確的法線向量。我們所希望的是有某種方式可以檢測出法線向量是否正確。把法線向量顯示出來正是這樣一種方法，恰好幾何著色器能夠完美地達成這個目的。

思路是這樣的：我們先不用幾何著色器，正常繪製場景，然後我們再次繪製一遍場景，但這次只顯示我們通過幾何著色器生成的法線向量。幾何著色器把一個三角形基本圖形作為輸入類型，用它們生成3條和法線向量同向的線段，每個頂點一條。偽代碼應該是這樣的：

```c++
shader.Use();
DrawScene();
normalDisplayShader.Use();
DrawScene();
```

這次我們會創建一個使用模型提供的頂點法線，而不是自己去生成。為了適應縮放和旋轉我們會在把它變換到裁切空間座標前，使用法線矩陣來法線（幾何著色器用他的位置向量做為裁切空間座標，所以我們還要把法線向量變換到同一個空間）。這些都能在頂點著色器中完成：

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;

out VS_OUT {
    vec3 normal;
} vs_out;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;

void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
    mat3 normalMatrix = mat3(transpose(inverse(view * model)));
    vs_out.normal = normalize(vec3(projection * vec4(normalMatrix * normal, 1.0)));
}
```

經過變換的裁切空間法線向量接著通過一個接口塊被傳遞到下個著色階段。幾何著色器接收每個頂點（帶有位置和法線向量），從每個位置向量繪製出一個法線向量：

```c++
#version 330 core
layout (triangles) in;
layout (line_strip, max_vertices = 6) out;

in VS_OUT {
    vec3 normal;
} gs_in[];

const float MAGNITUDE = 0.4f;

void GenerateLine(int index)
{
    gl_Position = gl_in[index].gl_Position;
    EmitVertex();
    gl_Position = gl_in[index].gl_Position + vec4(gs_in[index].normal, 0.0f) * MAGNITUDE;
    EmitVertex();
    EndPrimitive();
}

void main()
{
    GenerateLine(0); // First vertex normal
    GenerateLine(1); // Second vertex normal
    GenerateLine(2); // Third vertex normal
}
```

到現在為止，像這樣的幾何著色器的內容就不言自明瞭。需要注意的是我們我們把法線向量乘以一個MAGNITUDE向量來限制顯示出的法線向量的大小（否則它們就太大了）。

由於把法線顯示出來通常用於調試的目的，我們可以在片段著色器的幫助下把它們顯示為單色的線（如果你願意也可以更炫一點）。

```c++
#version 330 core
out vec4 color;

void main()
{
    color = vec4(1.0f, 1.0f, 0.0f, 1.0f);
}
```

現在先使用普通著色器來渲染你的模型，然後使用特製的法線可視著色器，你會看到這樣的效果：

![](http://learnopengl.com/img/advanced/geometry_shader_normals.png)

除了我們的納米服現在看起來有點像一個帶著隔熱手套的全身多毛的傢伙外，它給了我們一種非常有效的檢查一個模型的法線向量是否有錯誤的方式。你可以想象下這樣的幾何著色器也經常能被用在給物體添加毛髮上。

你可以從這裡找到[源碼](http://learnopengl.com/code_viewer.php?code=advanced/geometry_shader_normals)和可顯示法線的[著色器](http://learnopengl.com/code_viewer.php?code=advanced/geometry_shader_normals_shaders)。
