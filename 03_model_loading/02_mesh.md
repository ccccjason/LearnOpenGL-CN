# 網格(Mesh)

原文     | [Mesh](http://learnopengl.com/#!Model-Loading/Mesh)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

使用Assimp可以把多種不同格式的模型加載到程序中，但是一旦載入，它們就都被儲存為Assimp自己的數據結構。我們最終的目的是把這些數據轉變為OpenGL可讀的數據，才能用OpenGL來渲染物體。我們從前面的教程瞭解到，一個網格(Mesh)代表一個可繪製實體，現在我們就定義一個自己的網格類。

先來複習一點目前學到知識，考慮一個網格最少需要哪些數據。一個網格應該至少需要一組頂點，每個頂點包含一個位置向量，一個法線向量，一個紋理座標向量。一個網格也應該包含一個索引繪製用的索引，以紋理（diffuse/specular map）形式表現的材質數據。

為了在OpenGL中定義一個頂點，現在我們設置有最少需求一個網格類:


```c++
struct Vertex
{
    glm::vec3 Position;
    glm::vec3 Normal;
    glm::vec2 TexCoords;
};
```

我們把每個需要的向量儲存到一個叫做`Vertex`的結構體中，它被用來索引每個頂點屬性。另外除了`Vertex`結構體外，我們也希望組織紋理數據，所以我們定義一個`Texture`結構體：


```c++
struct Texture
{
    GLuint id;
    String type;
};
```

我們儲存紋理的id和它的類型，比如`diffuse`紋理或者`specular`紋理。

知道了頂點和紋理的實際表達，我們可以開始定義網格類的結構：


```c++
class Mesh
{
Public:
    vector<Vertex> vertices;
    vector<GLuint> indices;
    vector<Texture> textures;
    Mesh(vector<Vertex> vertices, vector<GLuint> indices, vector<Texture> texture);
    Void Draw(Shader shader);
 
private:
    GLuint VAO, VBO, EBO;
    void setupMesh();
}
```

如你所見這個類一點都不復雜，構造方法裡我們初始化網格所有必須數據。在`setupMesh`函數裡初始化緩衝。最後通過`Draw`函數繪製網格。注意，我們把`shader`傳遞給`Draw`函數。通過把`shader`傳遞給Mesh，在繪製之前我們設置幾個uniform（就像鏈接採樣器到紋理單元）。

構造函數的內容非常直接。我們簡單設置類的公有變量，使用的是構造函數相應的參數。我們在構造函數中也調用`setupMesh`函數：


```c++
Mesh(vector<Vertex> vertices, vector<GLuint> indices, vector<Texture> textures)
{
    this->vertices = vertices;
    this->indices = indices;
    this->textures = textures;
 
    this->setupMesh();
}
```

這裡沒什麼特別的，現在讓我們研究一下`setupMesh`函數。

 
## 初始化

現在我們有一大列的網格數據可用於渲染，這要感謝構造函數。我們確實需要設置合適的緩衝，通過頂點屬性指針（vertex attribute pointers）定義頂點著色器layout。現在你應該對這些概念很熟悉，但是我們我們通過介紹了結構體中使用頂點數據，所以稍微有點不一樣：


```c++
void setupMesh()
{
    glGenVertexArrays(1, &this->VAO);
    glGenBuffers(1, &this->VBO);
    glGenBuffers(1, &this->EBO);
  
    glBindVertexArray(this->VAO);
    glBindBuffer(GL_ARRAY_BUFFER, this->VBO);
 
    glBufferData(GL_ARRAY_BUFFER, this->vertices.size() * sizeof(Vertex), 
                 &this->vertices[0], GL_STATIC_DRAW);  
 
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, this->EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, this->indices.size() * sizeof(GLuint), 
                 &this->indices[0], GL_STATIC_DRAW);
 
    // 設置頂點座標指針
    glEnableVertexAttribArray(0); 
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), 
                         (GLvoid*)0);
    // 設置法線指針
    glEnableVertexAttribArray(1); 
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), 
                         (GLvoid*)offsetof(Vertex, Normal));
    // 設置頂點的紋理座標
    glEnableVertexAttribArray(2); 
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), 
                         (GLvoid*)offsetof(Vertex, TexCoords));
 
    glBindVertexArray(0);
}
```

如你所想代碼沒什麼特別不同的地方，在`Vertex`結構體的幫助下有了一些小把戲。

C++的結構體有一個重要的屬性，那就是在內存中它們是連續的。如果我們用結構體表示一列數據，這個結構體只包含結構體的連續的變量，它就會直接轉變為一個`float`（實際上是byte）數組，我們就能用於一個數組緩衝（array buffer）中了。比如，如果我們填充一個`Vertex`結構體，它在內存中的排布等於：


```c++
Vertex vertex;
vertex.Position = glm::vec3(0.2f, 0.4f, 0.6f);
vertex.Normal = glm::vec3(0.0f, 1.0f, 0.0f);
vertex.TexCoords = glm::vec2(1.0f, 0.0f);
// = [0.2f, 0.4f, 0.6f, 0.0f, 1.0f, 0.0f, 1.0f, 0.0f];
```

感謝這個有用的特性，我們能直接把一個作為緩衝數據的一大列`Vertex`結構體的指針傳遞過去，它們會翻譯成`glBufferData`能用的參數：


```c++
glBufferData(GL_ARRAY_BUFFER, this->vertices.size() * sizeof(Vertex), 
             &this->vertices[0], GL_STATIC_DRAW);
```

自然地，`sizeof`函數也可以使用於結構體來計算字節類型的大小。它應該是32字節（8float * 4）。

一個預處理指令叫做`offsetof(s,m)`把結構體作為它的第一個參數，第二個參數是這個結構體名字的變量。這是結構體另外的一個重要用途。函數返回這個變量從結構體開始的字節偏移量（offset）。這對於定義`glVertexAttribPointer`函數偏移量參數效果很好：


```c++
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), 
                     (GLvoid*)offsetof(Vertex, Normal));
```
偏移量現在使用`offsetof`函數定義了，在這個例子裡，設置法線向量的字節偏移量等於法線向量在結構體的字節偏移量，它是`3float`，也就是12字節（一個float佔4字節）。注意，我們同樣設置步長參數等於`Vertex`結構體的大小。

使用一個像這樣的結構體，不僅能提供可讀性更高的代碼同時也是我們可以輕鬆的擴展結構體。如果我們想要增加另一個頂點屬性，我們把它可以簡單的添加到結構體中，由於它的可擴展性，渲染代碼不會被破壞。

## 渲染

我們需要為`Mesh`類定義的最後一個函數，是它的Draw函數。在真正渲染前我們希望綁定合適的紋理，然後調用`glDrawElements`。可因為我們從一開始不知道這個網格有多少紋理以及它們應該是什麼類型的，所以這件事變得很困難。所以我們該怎樣在著色器中設置紋理單元和採樣器呢？

解決這個問題，我們需要假設一個特定的名稱慣例：每個`diffuse`紋理被命名為`texture_diffuseN`,每個`specular`紋理應該被命名為`texture_specularN`。N是一個從1到紋理才搶其允許使用的最大值之間的數。可以說，在一個網格中我們有3個`diffuse`紋理和2個`specular`紋理，它們的紋理採樣器應該這樣被調用：


```c++
uniform sampler2D texture_diffuse1;
uniform sampler2D texture_diffuse2;
uniform sampler2D texture_diffuse3;
uniform sampler2D texture_specular1;
uniform sampler2D texture_specular2;
```

使用這樣的慣例，我們能定義我們在著色器中需要的紋理採樣器的數量。如果一個網格真的有（這麼多）紋理，我們就知道它們的名字應該是什麼。這個慣例也使我們能夠處理一個網格上的任何數量的紋理，通過定義合適的採樣器開發者可以自由使用希望使用的數量（雖然定義少的話就會有點浪費綁定和uniform調用了）。

像這樣的問題有很多不同的解決方案，如果你不喜歡這個方案，你可以自己創造一個你自己的方案。
最後的繪製代碼：


```c++
void Draw(Shader shader) 
{
    GLuint diffuseNr = 1;
    GLuint specularNr = 1;
    for(GLuint i = 0; i < this->textures.size(); i++)
    {
        glActiveTexture(GL_TEXTURE0 + i); // 在綁定紋理前需要激活適當的紋理單元
        // 檢索紋理序列號 (N in diffuse_textureN)
        stringstream ss;
        string number;
        string name = this->textures[i].type;
        if(name == "texture_diffuse")
            ss << diffuseNr++; // 將GLuin輸入到string stream
        else if(name == "texture_specular")
            ss << specularNr++; // 將GLuin輸入到string stream
        number = ss.str(); 
 
        glUniform1f(glGetUniformLocation(shader.Program, ("material." + name + number).c_str()), i);
        glBindTexture(GL_TEXTURE_2D, this->textures[i].id);
    }
    glActiveTexture(GL_TEXTURE0);
 
    // 繪製Mesh
    glBindVertexArray(this->VAO);
    glDrawElements(GL_TRIANGLES, this->indices.size(), GL_UNSIGNED_INT, 0);
    glBindVertexArray(0);
}
```

這不是最漂亮的代碼，但是這主要歸咎於C++轉換類型時的醜陋，比如`int`轉`string`時。我們首先計算N-元素每個紋理類型，把它鏈接到紋理類型字符串來獲取合適的uniform名。然後查找合適的採樣器位置，給它位置值對應當前激活紋理單元，綁定紋理。這也是我們需要在`Draw`方法是用`shader`的原因。我們添加`material.`到作為結果的uniform名，因為我們通常把紋理儲存進材質結構體（對於每個實現也許會有不同）。

!!! Important

    注意，當我們把`diffuse`和`specular`傳遞到字符串流（`stringstream`）的時候，計數器會增加，在C++自增叫做：變量++，它會先返回自身然後加1，而++變量，先加1再返回自身，我們的例子裡，我們先傳遞原來的計數器值到字符串流，然後再加1，下一輪生效。

你可以從這裡得到[Mesh類的源碼](http://learnopengl.com/code_viewer.php?code=mesh&type=header)。

Mesh類是對我們前面的教程裡討論的很多話題的的簡潔的抽象在下面的教程裡，我們會創建一個模型，它用作乘放多個網格物體的容器，真正的實現Assimp的加載接口。