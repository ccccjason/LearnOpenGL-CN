# 實例化(Instancing)

原文     | [Instancing](http://learnopengl.com/#!Advanced-OpenGL/Instancing)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

假如你有一個有許多模型的場景，而這些模型的頂點數據都一樣，只是進行了不同的世界空間的變換。想象一下，有一個場景中充滿了草葉：每根草都是幾個三角形組成的。你可能需要繪製很多的草葉，最終一次渲染循環中就肯能有成千上萬個草需要繪製了。因為每個草葉只是由幾個三角形組成，繪製一個幾乎是即刻完成，但是數量巨大以後，執行起來就很慢了。

如果我們渲染這樣多的物體的時候，也許代碼會寫成這樣：

```c++
for(GLuint i = 0; i < amount_of_models_to_draw; i++)
{
    DoSomePreparations(); //在這裡綁定VAO、綁定紋理、設置uniform變量等
    glDrawArrays(GL_TRIANGLES, 0, amount_of_vertices);
}
```


像這樣繪製出你模型的其他實例，多次繪製之後，很快將達到一個瓶頸，這是因為你`glDrawArrays`或`glDrawElements`這樣的函數(Draw call)過多。這樣渲染頂點數據，會明顯降低執行效率，這是因為OpenGL在它可以繪製你的頂點數據之前必須做一些準備工作(比如告訴GPU從哪個緩衝讀取數據，以及在哪裡找到頂點屬性，所有這些都會使CPU到GPU的總線變慢)。所以即使渲染頂點超快，而多次給你的GPU下達這樣的渲染命令卻未必。

如果我們能夠將數據一次發送給GPU，就會更方便，然後告訴OpenGL使用一個繪製函數，將這些數據繪製為多個物體。這就是我們將要展開討論的**實例化(instancing)**。

**實例化(instancing)**是一種只調用一次渲染函數卻能繪製出很多物體的技術，它節省渲染物體時從CPU到GPU的通信時間，而且只需做一次即可。要使用實例化渲染，我們必須將`glDrawArrays`和`glDrawElements`各自改為`glDrawArraysInstanced`和`glDrawElementsInstanced`。這些用於實例化的函數版本需要設置一個額外的參數，叫做**實例數量(instance count)**，它設置我們打算渲染實例的數量。這樣我們就只需要把所有需要的數據發送給GPU一次就行了，然後告訴GPU它該如何使用一個函數來繪製所有這些實例。

就其本身而言，這個函數用處不大。渲染同一個物體一千次對我們來說沒用，因為每個渲染出的物體不僅相同而且還在同一個位置；我們只能看到一個物體！出於這個原因GLSL在著色器中嵌入了另一個內建變量，叫做**`gl_InstanceID`**。

在通過實例化繪製時，`gl_InstanceID`的初值是0，它在每個實例渲染時都會增加1。如果我們渲染43個實例，那麼在頂點著色器`gl_InstanceID`的值最後就是42。每個實例都擁有唯一的值意味著我們可以索引到一個位置數組，並將每個實例擺放在世界空間的不同的位置上。

我們調用一個實例化渲染函數，在標準化設備座標中繪製一百個2D四邊形來看看實例化繪製的效果是怎樣的。通過對一個儲存著100個偏移量向量的索引，我們為每個實例四邊形添加一個偏移量。最後，窗口被排列精美的四邊形網格填滿：

![](http://learnopengl.com/img/advanced/instancing_quads.png)

每個四邊形是2個三角形所組成的，因此總共有6個頂點。每個頂點包含一個2D標準設備座標位置向量和一個顏色向量。下面是例子中所使用的頂點數據，每個三角形為了適應屏幕都很小：

```c++
GLfloat quadVertices[] = {
    //  ---位置---   ------顏色-------
    -0.05f,  0.05f,  1.0f, 0.0f, 0.0f,
     0.05f, -0.05f,  0.0f, 1.0f, 0.0f,
    -0.05f, -0.05f,  0.0f, 0.0f, 1.0f,

    -0.05f,  0.05f,  1.0f, 0.0f, 0.0f,
     0.05f, -0.05f,  0.0f, 1.0f, 0.0f,
     0.05f,  0.05f,  0.0f, 1.0f, 1.0f
};
```

片段著色器接收從頂點著色器發送來的顏色向量，設置為它的顏色輸出，從而為四邊形上色：

```c++
#version 330 core
in vec3 fColor;
out vec4 color;

void main()
{
    color = vec4(fColor, 1.0f);
}
```

到目前為止沒有什麼新內容，但頂點著色器開始變得有意思了：

```c++
#version 330 core
layout (location = 0) in vec2 position;
layout (location = 1) in vec3 color;

out vec3 fColor;

uniform vec2 offsets[100];

void main()
{
    vec2 offset = offsets[gl_InstanceID];
    gl_Position = vec4(position + offset, 0.0f, 1.0f);
    fColor = color;
}
```

這裡我們定義了一個uniform數組，叫`offsets`，它包含100個偏移量向量。在頂點著色器裡，我們接收一個對應著當前實例的偏移量，這是通過使用 `gl_InstanceID`來索引offsets得到的。如果我們使用實例化繪製100個四邊形，使用這個頂點著色器，我們就能得到100位於不同位置的四邊形。

我們一定要設置偏移位置，在遊戲循環之前我們用一個嵌套for循環計算出它來：

```c++
glm::vec2 translations[100];
int index = 0;
GLfloat offset = 0.1f;
for(GLint y = -10; y < 10; y += 2)
{
    for(GLint x = -10; x < 10; x += 2)
    {
        glm::vec2 translation;
        translation.x = (GLfloat)x / 10.0f + offset;
        translation.y = (GLfloat)y / 10.0f + offset;
        translations[index++] = translation;
    }
}
```

這裡我們創建100個平移向量，它包含著10×10格子所有位置。除了生成`translations`數組外，我們還需要把這些位移數據發送到頂點著色器的uniform數組：

```c++
shader.Use();
for(GLuint i = 0; i < 100; i++)
{
    stringstream ss;
    string index;
    ss << i;
    index = ss.str();
    GLint location = glGetUniformLocation(shader.Program, ("offsets[" + index + "]").c_str())
    glUniform2f(location, translations[i].x, translations[i].y);
}
```

這一小段代碼中，我們將for循環計數器i變為string，接著就能動態創建一個為請求的uniform的`location`創建一個`location`字符串。將offsets數組中的每個條目，我們都設置為相應的平移向量。

現在所有的準備工作都結束了，我們可以開始渲染四邊形了。用實例化渲染來繪製四邊形，我們需要調用`glDrawArraysInstanced`或`glDrawElementsInstanced`，由於我們使用的不是索引繪製緩衝，所以我們用的是`glDrawArrays`對應的那個版本`glDrawArraysInstanced`：

```c++
glBindVertexArray(quadVAO);
glDrawArraysInstanced(GL_TRIANGLES, 0, 6, 100);  
glBindVertexArray(0);
```

`glDrawArraysInstanced`的參數和`glDrawArrays`一樣，除了最後一個參數設置了我們打算繪製實例的數量。我們想展示100個四邊形，它們以10×10網格形式展現，所以這兒就是100.運行代碼，你會得到100個相似的有色四邊形。

## 實例化數組(instanced arrays)

在這種特定條件下，前面的實現很好，但是當我們有100個實例的時候（這很正常），最終我們將碰到uniform數據數量的上線。為避免這個問題另一個可替代方案是實例化數組，它使用頂點屬性來定義，這樣就允許我們使用更多的數據了，當頂點著色器渲染一個新實例時它才會被更新。

使用頂點屬性，每次運行頂點著色器都將讓GLSL獲取到下個頂點屬性集合，它們屬於當前頂點。當把頂點屬性定義為實例數組時，頂點著色器只更新每個實例的頂點屬性的內容而不是頂點的內容。這使我們在每個頂點數據上使用標準頂點屬性，用實例數組來儲存唯一的實例數據。

為了展示一個實例化數組的例子，我們將採用前面的例子，把偏移uniform表示為一個實例數組。我們不得不增加另一個頂點屬性，來更新頂點著色器。

```c++
#version 330 core
layout (location = 0) in vec2 position;
layout (location = 1) in vec3 color;
layout (location = 2) in vec2 offset;

out vec3 fColor;

void main()
{
    gl_Position = vec4(position + offset, 0.0f, 1.0f);
    fColor = color;
}
```

我們不再使用`gl_InstanceID`，可以直接用`offset`屬性，不用先在一個大uniform數組裡進行索引。

因為一個實例化數組實際上就是一個和位置和顏色一樣的頂點屬性，我們還需要把它的內容儲存為一個頂點緩衝對象裡，並把它配置為一個屬性指針。我們首先將平移變換數組貯存到一個新的緩衝對象上：

```c++
GLuint instanceVBO;
glGenBuffers(1, &instanceVBO);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(glm::vec2) * 100, &translations[0], GL_STATIC_DRAW);
glBindBuffer(GL_ARRAY_BUFFER, 0);
```

之後我們還需要設置它的頂點屬性指針，並開啟頂點屬性：

```c++
glEnableVertexAttribArray(2);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(GLfloat), (GLvoid*)0);
glBindBuffer(GL_ARRAY_BUFFER, 0);
glVertexAttribDivisor(2, 1);
```

代碼中有意思的地方是，最後一行，我們調用了 **`glVertexAttribDivisor`**。這個函數告訴OpenGL什麼時候去更新頂點屬性的內容到下個元素。它的第一個參數是提到的頂點屬性，第二個參數是屬性除數（attribute divisor）。默認屬性除數是0，告訴OpenGL在頂點著色器的每次迭代更新頂點屬性的內容。把這個屬性設置為1，我們就是告訴OpenGL我們打算在開始渲染一個新的實例的時候更新頂點屬性的內容。設置為2代表我們每2個實例更新內容，依此類推。把屬性除數設置為1，我們可以高效地告訴OpenGL，location是2的頂點屬性是一個實例數組（instanced array）。

如果我們現在再次使用`glDrawArraysInstanced`渲染四邊形，我們會得到下面的輸出：

![](http://learnopengl.com/img/advanced/instancing_quads.png)

和前面的一樣，但這次是使用實例數組實現的，它使我們為繪製實例向頂點著色器傳遞更多的數據（內存允許我們存多少就能存多少）。

你還可以使`用gl_InstanceID`從右上向左下縮小每個四邊形。

```c++
void main()
{
    vec2 pos = position * (gl_InstanceID / 100.0f);
    gl_Position = vec4(pos + offset, 0.0f, 1.0f);
    fColor = color;
}
```

結果是第一個實例的四邊形被繪製的非常小，隨著繪製實例的增加，`gl_InstanceID`越來越接近100，這樣更多的四邊形會更接近它們原來的大小。這是一種很好的將`gl_InstanceID`與實例數組結合使用的法則：

![](http://learnopengl.com/img/advanced/instancing_quads_arrays.png)

如果你仍然不確定實例渲染如何工作，或者想看看上面的代碼是如何組合起來的，你可以在[這裡找到應用的源碼](http://learnopengl.com/code_viewer.php?code=advanced/instancing_quads)。

這些例子不是實例的好例子，不過挺有意思的。它們可以讓你對實例的工作方式有一個概括的理解，但是當繪製擁有極大數量的相同物體的時候，它極其有用，現在我們還沒有展示呢。出於這個原因，我們將在接下來的部分進入太空來看看實例渲染的威力。

### 小行星帶

想象一下，在一個場景中有一個很大的行星，行星周圍有一圈小行星帶。這樣一個小行星大可能包含成千上萬的石塊，對於大多數顯卡來說幾乎是難以完成的渲染任務。這個場景對於實例渲染來說卻不再話下，由於所有小行星可以使用一個模型來表示。每個小行星使用一個變換矩陣就是一個經過少量變化的獨一無二的小行星了。

為了展示實例渲染的影響，我們先不使用實例渲染，來渲染一個小行星圍繞行星飛行的場景。這個場景的大天體可以[從這裡下載](http://learnopengl.com/data/models/planet.rar)，此外要把小行星放在合適的位置上。小行星可以[從這裡下載](http://learnopengl.com/data/models/rock.rar)。

為了得到我們理想中的效果，我們將為每個小行星生成一個變換矩陣，作為它們的模型矩陣。變換矩陣先將小行星平移到行星帶上，我們還要添加一個隨機位移值來作為偏移量，這樣才能使行星帶更自然。接著我們應用一個隨機縮放，以及一個隨機旋轉向量。最後，變換矩陣就會將小行星變換到行星的周圍，同時使它們更自然，每個行星都有別於其他的。

```c++
GLuint amount = 1000;
glm::mat4* modelMatrices;
modelMatrices = new glm::mat4[amount];
srand(glfwGetTime()); // initialize random seed
GLfloat radius = 50.0;
GLfloat offset = 2.5f;
for(GLuint i = 0; i < amount; i++)
{
    glm::mat4 model;
    // 1. Translation: displace along circle with 'radius' in range [-offset, offset]
    GLfloat angle = (GLfloat)i / (GLfloat)amount * 360.0f;
    GLfloat displacement = (rand() % (GLint)(2 * offset * 100)) / 100.0f - offset;
    GLfloat x = sin(angle) * radius + displacement;
    displacement = (rand() % (GLint)(2 * offset * 100)) / 100.0f - offset;
    GLfloat y = displacement * 0.4f; // y value has smaller displacement
    displacement = (rand() % (GLint)(2 * offset * 100)) / 100.0f - offset;
    GLfloat z = cos(angle) * radius + displacement;
    model = glm::translate(model, glm::vec3(x, y, z));
    // 2. Scale: Scale between 0.05 and 0.25f
    GLfloat scale = (rand() % 20) / 100.0f + 0.05;
    model = glm::scale(model, glm::vec3(scale));
    // 3. Rotation: add random rotation around a (semi)randomly picked rotation axis vector
    GLfloat rotAngle = (rand() % 360);
    model = glm::rotate(model, rotAngle, glm::vec3(0.4f, 0.6f, 0.8f));
    // 4. Now add to list of matrices
    modelMatrices[i] = model;
}
```

這段代碼看起來還是有點嚇人，但我們基本上是沿著一個半徑為radius的圓圈變換小行星的x和y的值，讓每個小行星在-offset和offset之間隨機生成一個位置。我們讓y變化的更小，這讓這個環帶就會成為扁平的。接著我們縮放和旋轉變換，把結果儲存到一個modelMatrices矩陣裡，它的大小是amount。這裡我們生成1000個模型矩陣，每個小行星一個。

加載完天體和小行星模型後，編譯著色器，渲染代碼是這樣的：

```c++
// 繪製行星
shader.Use();
glm::mat4 model;
model = glm::translate(model, glm::vec3(0.0f, -5.0f, 0.0f));
model = glm::scale(model, glm::vec3(4.0f, 4.0f, 4.0f));
glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
planet.Draw(shader);

// 繪製石頭
for(GLuint i = 0; i < amount; i++)
{
    glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(modelMatrices[i]));
    rock.Draw(shader);
}
```

我們先繪製天體模型，要把它平移和縮放一點以適應場景，接著，我們繪製amount數量的小行星，它們按照我們所計算的結果進行變換。在我們繪製每個小行星之前，我們還得先在著色器中設置相應的模型變換矩陣。

結果是一個太空樣子的場景，我們可以看到有一個自然的小行星帶：

![](http://learnopengl.com/img/advanced/instancing_asteroids.png)

這個場景包含1001次渲染函數調用，每幀渲染1000個小行星模型。你可以在這裡找到[場景的源碼](http://learnopengl.com/code_viewer.php?code=advanced/instancing_asteroids_normal)，以及[頂點](http://learnopengl.com/code_viewer.php?code=advanced/instancing&type=vertex)和[片段](http://learnopengl.com/code_viewer.php?code=advanced/instancing&type=fragment)著色器。

當我們開始增加數量的時候，很快就會注意到幀數的下降，而且下降的厲害。當我們設置為2000的時候，場景慢得已經難以移動了。

我們再次使用實例渲染來渲染同樣的場景。我們先得對頂點著色器進行一點修改：

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 2) in vec2 texCoords;
layout (location = 3) in mat4 instanceMatrix;

out vec2 TexCoords;

uniform mat4 projection;
uniform mat4 view;

void main()
{
    gl_Position = projection * view * instanceMatrix * vec4(position, 1.0f);
    TexCoords = texCoords;
}
```

我們不再使用模型uniform變量，取而代之的是把一個mat4的頂點屬性，送一我們可以將變換矩陣儲存為一個實例數組（instanced array）。然而，當我們聲明一個數據類型為頂點屬性的時候，它比一個vec4更大，是有些不同的。頂點屬性被允許的最大數據量和vec4相等。因為一個mat4大致和4個vec4相等，我們為特定的矩陣必須保留4個頂點屬性。因為我們將它的位置賦值為3個列的矩陣，頂點屬性的位置就會是3、4、5和6。

然後我們必須為這4個頂點屬性設置屬性指針，並將其配置為實例數組：

```c++
for(GLuint i = 0; i < rock.meshes.size(); i++)
{
    GLuint VAO = rock.meshes[i].VAO;
    // Vertex Buffer Object
    GLuint buffer;
    glBindVertexArray(VAO);
    glGenBuffers(1, &buffer);
    glBindBuffer(GL_ARRAY_BUFFER, buffer);
    glBufferData(GL_ARRAY_BUFFER, amount * sizeof(glm::mat4), &modelMatrices[0], GL_STATIC_DRAW);
    // Vertex Attributes
    GLsizei vec4Size = sizeof(glm::vec4);
    glEnableVertexAttribArray(3);
    glVertexAttribPointer(3, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (GLvoid*)0);
    glEnableVertexAttribArray(4);
    glVertexAttribPointer(4, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (GLvoid*)(vec4Size));
    glEnableVertexAttribArray(5);
    glVertexAttribPointer(5, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (GLvoid*)(2 * vec4Size));
    glEnableVertexAttribArray(6);
    glVertexAttribPointer(6, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (GLvoid*)(3 * vec4Size));

    glVertexAttribDivisor(3, 1);
    glVertexAttribDivisor(4, 1);
    glVertexAttribDivisor(5, 1);
    glVertexAttribDivisor(6, 1);

    glBindVertexArray(0);
}
```

要注意的是我們將Mesh的VAO變量聲明為一個public（公有）變量，而不是一個private（私有）變量，所以我們可以獲取它的頂點數組對象。這不是最乾淨的方案，但這能較好的適應本教程。若沒有這點hack，代碼就乾淨了。我們聲明瞭OpenGL該如何為每個矩陣的頂點屬性的緩衝進行解釋，每個頂點屬性都是一個實例數組。

下一步我們再次獲得網格的VAO，這次使用`glDrawElementsInstanced`進行繪製：

```c++
// Draw meteorites
instanceShader.Use();
for(GLuint i = 0; i < rock.meshes.size(); i++)
{
    glBindVertexArray(rock.meshes[i].VAO);
    glDrawElementsInstanced(
        GL_TRIANGLES, rock.meshes[i].vertices.size(), GL_UNSIGNED_INT, 0, amount
    );
    glBindVertexArray(0);
}
```

這裡我們繪製和前面的例子裡一樣數量（amount）的小行星，只不過是使用的實例渲染。結果是相似的，但你會看在開始增加數量以後效果的不同。不實例渲染，我們可以流暢渲染1000到1500個小行星。而使用了實例渲染，我們可以設置為100000，每個模型由576個頂點，這幾乎有5千7百萬個頂點，而且幀率沒有絲毫下降！

![](http://learnopengl.com/img/advanced/instancing_asteroids_quantity.png)

上圖渲染了十萬小行星，半徑為150.0f，偏移等於25.0f。你可以在這裡找到這個演示實例渲染的[源碼](http://learnopengl.com/code_viewer.php?code=advanced/instancing_asteroids_instanced)。

!!! Important

        有些機器渲染十萬可能會有點吃力，所以嘗試修改這個數量知道你能獲得可以接受的幀率。

就像你所看到的，在合適的條件下，實例渲染對於你的顯卡來說和普通渲染有很大不同。處於這個理由，實例渲染通常用來渲染草、草叢、粒子以及像這樣的場景，基本上來講只要場景中有很多重複物體，使用實例渲染都會獲得好處。
