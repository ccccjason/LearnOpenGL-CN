# 模型(Model)

原文     | [Model](http://learnopengl.com/#!Model-Loading/Model)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

現在是時候著手啟用Assimp，並開始創建實際的加載和轉換代碼了。本教程的目標是創建另一個類，這個類可以表達模型的全部。更確切的說，一個模型包含多個網格(Mesh)，一個網格可能帶有多個對象。一個別墅，包含一個木製陽臺，一個尖頂或許也有一個游泳池，它仍然被加載為一個單一模型。我們通過Assimp加載模型，把它們轉變為多個網格（Mesh）對象，這些對象是是先前教程裡創建的。

閒話少說，我把Model類的結構呈現給你：

```c++
class Model 
{
    public:
        /*  成員函數   */
        Model(GLchar* path)
        {
            this->loadModel(path);
        }
        void Draw(Shader shader); 
    private:
        /*  模型數據  */
        vector<Mesh> meshes;
        string directory;
        
        /*  私有成員函數   */
        void loadModel(string path);
        void processNode(aiNode* node, const aiScene* scene);
        Mesh processMesh(aiMesh* mesh, const aiScene* scene);
        vector<Texture> loadMaterialTextures(aiMaterial* mat, aiTextureType type, string typeName);
};
```

`Model`類包含一個`Mesh`對象的向量，我們需要在構造函數中給出文件的位置。之後，在構造其中，它通過`loadModel`函數加載文件。私有方法都被設計為處理一部分的Assimp導入的常規動作，我們會簡單講講它們。同樣，我們儲存文件路徑的目錄，這樣稍後加載紋理的時候會用到。

函數`Draw`沒有什麼特別之處，基本上是循環每個網格，調用各自的Draw函數。


```c++
void Draw(Shader shader)
{
    for(GLuint i = 0; i < this->meshes.size(); i++)
        this->meshes[i].Draw(shader);
}
```

## 把一個3D模型導入到OpenGL

為了導入一個模型，並把它轉換為我們自己的數據結構，第一件需要做的事是包含合適的Assimp頭文件，這樣編譯器就不會對我們抱怨了。


```c++
#include <assimp/Importer.hpp>
#include <assimp/scene.h>
#include <assimp/postprocess.h>
```

我們將要調用的第一個函數是`loadModel`,它被構造函數直接調用。在`loadModel`函數裡面，我們使用Assimp加載模型到Assimp中被稱為scene對象的數據結構。你可能還記得模型加載系列的第一個教程中，這是Assimp的數據結構的根對象。一旦我們有了場景對象，我們就能從已加載模型中獲取所有所需數據了。

Assimp最大優點是，它簡約的抽象了所加載所有不同格式文件的技術細節，用一行可以做到這一切：


```c++
Assimp::Importer importer;
const aiScene* scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs);
```

我們先來聲明一個`Importer`對象，它的名字空間是`Assimp`，然後調用它的`ReadFile`函數。這個函數需要一個文件路徑，第二個參數是後處理（post-processing）選項。除了可以簡單加載文件外，Assimp允許我們定義幾個選項來強制Assimp去對導入數據做一些額外的計算或操作。通過設置`aiProcess_Triangulate`，我們告訴Assimp如果模型不是（全部）由三角形組成，應該轉換所有的模型的原始幾何形狀為三角形。`aiProcess_FlipUVs`基於y軸翻轉紋理座標，在處理的時候是必須的（你可能記得，我們在紋理教程中，我們說過在OpenGL大多數圖像會被沿著y軸反轉，所以這個小小的後處理選項會為我們修正這個）。一少部分其他有用的選項如下：

* `aiProcess_GenNormals` : 如果模型沒有包含法線向量，就為每個頂點創建法線。
* `aiProcess_SplitLargeMeshes` : 把大的網格成幾個小的的下級網格，當你渲染有一個最大數量頂點的限制時或者只能處理小塊網格時很有用。
* `aiProcess_OptimizeMeshes` : 和上個選項相反，它把幾個網格結合為一個更大的網格。以減少繪製函數調用的次數的方式來優化。

Assimp提供了後處理說明，你可以從這裡找到所有內容。事實上通過Assimp加載一個模型超級簡單。困難的是使用返回的場景對象把加載的數據變換到一個Mesh對象的數組。

完整的`loadModel`函數在這裡列出：


```c++
void loadModel(string path)
{
    Assimp::Importer import;
    const aiScene* scene = import.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs); 
 
    if(!scene || scene->mFlags == AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode) 
    {
        cout << "ERROR::ASSIMP::" << import.GetErrorString() << endl;
        return;
    }
    this->directory = path.substr(0, path.find_last_of('/'));
 
    this->processNode(scene->mRootNode, scene);
}
```

在我們加載了模型之後，我們檢驗是否場景和場景的根節點為空，查看這些標記中的一個來看看返回的數據是否完整。如果發生了任何一個錯誤，我們通過導入器（impoter）的`GetErrorString`函數返回錯誤報告。我們同樣重新獲取文件的目錄路徑。

如果沒什麼錯誤發生，我們希望處理所有的場景節點，所以我們傳遞第一個節點（根節點）到遞歸函數`processNode`。因為每個節點（可能）包含多個子節點，我們希望先處理父節點再處理子節點，以此類推。這符合遞歸結構，所以我們定義一個遞歸函數。遞歸函數就是一個做一些什麼處理之後，用不同的參數調用它自身的函數，此種循環不會停止，直到一個特定條件發生。在我們的例子裡，特定條件是所有的節點都被處理。

也許你記得，Assimp的結構，每個節點包含一個網格集合的索引，每個索引指向一個在場景對象中特定的網格位置。我們希望獲取這些網格索引，獲取每個網格，處理每個網格，然後對其他的節點的子節點做同樣的處理。`processNode`函數的內容如下：


```c++
void processNode(aiNode* node, const aiScene* scene)
{
    // 添加當前節點中的所有Mesh
    for(GLuint i = 0; i < node->mNumMeshes; i++)
    {
        aiMesh* mesh = scene->mMeshes[node->mMeshes[i]]; 
        this->meshes.push_back(this->processMesh(mesh, scene)); 
    }
    // 遞歸處理該節點的子孫節點
    for(GLuint i = 0; i < node->mNumChildren; i++)
    {
        this->processNode(node->mChildren[i], scene);
    }
}
```

我們首先利用場景的`mMeshes`數組來檢查每個節點的網格索引以獲取相應的網格。被返回的網格被傳遞給`processMesh`函數，它返回一個網格對象，我們可以把它儲存在`meshes`的list或vector（STL裡的兩種實現鏈表的數據結構）中。

一旦所有的網格都被處理，我們遍歷所有子節點，同樣調用processNode函數。一旦一個節點不再擁有任何子節點，函數就會停止執行。

!!! Important

    認真的讀者會注意到，我們可能基本忘記處理任何的節點，簡單循環出場景所有的網格，而不是用索引做這件複雜的事。我們這麼做的原因是，使用這種節點的原始的想法是，在網格之間定義一個父-子關係。通過遞歸遍歷這些關係，我們可以真正定義特定的網格作為其他網格的父（節點）。
    
    關於這個系統的一個有用的例子是，當你想要平移一個汽車網格需要確保把它的子（節點）比如，引擎網格，方向盤網格和輪胎網格都進行平移；使用父-子關係這樣的系統很容易被創建出來。
    
    現在我們沒用這種系統，但是無論何時你想要對你的網格數據進行額外的控制，這通常是一種堅持被推薦的做法。這些模型畢竟是那些定義了這些節點風格的關係的藝術家所創建的。

下一步是用上個教程創建的`Mesh`類開始真正處理Assimp的數據。

## 從Assimp到網格

把一個`aiMesh`對象轉換為一個我們自己定義的網格對象並不難。我們所要做的全部是獲取每個網格相關的屬性並把這些屬性儲存到我們自己的對象。通常`processMesh`函數的結構會是這樣：


```c++
Mesh processMesh(aiMesh* mesh, const aiScene* scene)
{
    vector<Vertex> vertices;
    vector<GLuint> indices;
    vector<Texture> textures;
 
    for(GLuint i = 0; i < mesh->mNumVertices; i++)
    {
        Vertex vertex;
        // 處理頂點座標、法線和紋理座標
        ...
        vertices.push_back(vertex);
    }
    // 處理頂點索引
    ...
    // 處理材質
    if(mesh->mMaterialIndex >= 0)
    {
        ...
    }
 
    return Mesh(vertices, indices, textures);
}
```

處理一個網格基本由三部分組成：獲取所有頂點數據，獲取網格的索引，獲取相關材質數據。處理過的數據被儲存在3個向量其中之一里面，一個Mesh被以這些數據創建，返回到函數的調用者。

獲取頂點數據很簡單：我們定義一個`Vertex`結構體，在每次遍歷後我們把這個結構體添加到`Vertices`數組。我們為存在於網格中的眾多頂點循環（通過`mesh->mNumVertices`獲取）。在遍歷的過程中，我們希望用所有相關數據填充這個結構體。每個頂點位置會像這樣被處理：


```c++
glm::vec3 vector; 
vector.x = mesh->mVertices[i].x;
vector.y = mesh->mVertices[i].y;
vector.z = mesh->mVertices[i].z; 
vertex.Position = vector;
```

注意，為了傳輸Assimp的數據，我們定義一個`vec3`的宿主，我們需要它是因為Assimp維持它自己的數據類型，這些類型用於向量、材質、字符串等。這些數據類型轉換到glm的數據類型時通常效果不佳。

!!! Important

    Assimp調用他們的頂點位置數組`mVertices`真有點違反直覺。

對應法線的步驟毫無疑問是這樣的：


```c++
vector.x = mesh->mNormals[i].x;
vector.y = mesh->mNormals[i].y;
vector.z = mesh->mNormals[i].z;
vertex.Normal = vector;
```

紋理座標也基本一樣，但是Assimp允許一個模型的每個頂點有8個不同的紋理座標，我們可能用不到，所以我們只關係第一組紋理座標。我們也希望檢查網格是否真的包含紋理座標（可能並不總是如此）:


```c++
if(mesh->mTextureCoords[0]) // Does the mesh contain texture coordinates?
{
    glm::vec2 vec;
    vec.x = mesh->mTextureCoords[0][i].x; 
    vec.y = mesh->mTextureCoords[0][i].y;
    vertex.TexCoords = vec;
}
else
    vertex.TexCoords = glm::vec2(0.0f, 0.0f);
```

`Vertex`結構體現在完全被所需的頂點屬性填充了，我們能把它添加到`vertices`向量的尾部。要對每個網格的頂點做相同的處理。

## 頂點

Assimp的接口定義每個網格有一個以面（faces）為單位的數組，每個面代表一個單獨的圖元，在我們的例子中（由於`aiProcess_Triangulate`選項）總是三角形，一個麵包含索引，這些索引定義我們需要繪製的頂點以在那樣的順序提供給每個圖元，所以如果我們遍歷所有面，把所有面的索引儲存到`indices`向量，我們需要這麼做：


```c++
for(GLuint i = 0; i < mesh->mNumFaces; i++)
{
    aiFace face = mesh->mFaces[i];
    for(GLuint j = 0; j < face.mNumIndices; j++)
        indices.push_back(face.mIndices[j]);
}
```

所有外部循環結束後，我們現在有了一個完整點的頂點和索引數據來繪製網格，這要調用`glDrawElements`函數。可是，為了結束這個討論，並向網格提供一些細節，我們同樣希望處理網格的材質。

 

## 材質

如同節點，一個網格只有一個指向材質對象的索引，獲取網格實際的材質，我們需要索引場景的`mMaterials`數組。網格的材質索引被設置在`mMaterialIndex`屬性中，通過這個屬性我們同樣能夠檢驗一個網格是否包含一個材質：

```c++
if(mesh->mMaterialIndex >= 0)
{
    aiMaterial* material = scene->mMaterials[mesh->mMaterialIndex];
    vector<Texture> diffuseMaps = this->loadMaterialTextures(material, 
                                        aiTextureType_DIFFUSE, "texture_diffuse");
    textures.insert(textures.end(), diffuseMaps.begin(), diffuseMaps.end());
    vector<Texture> specularMaps = this->loadMaterialTextures(material, 
                                        aiTextureType_SPECULAR, "texture_specular");
    textures.insert(textures.end(), specularMaps.begin(), specularMaps.end());
}
```

我麼先從場景的`mMaterials`數組獲取`aimaterial`對象，然後，我們希望加載網格的diffuse或/和specular紋理。一個材質儲存了一個數組，這個數組為每個紋理類型提供紋理位置。不同的紋理類型都以`aiTextureType_`為前綴。我們使用一個幫助函數：`loadMaterialTextures`來從材質獲取紋理。這個函數返回一個`Texture`結構體的向量，我們在之後儲存在模型的`textures`座標的後面。

`loadMaterialTextures`函數遍歷所有給定紋理類型的紋理位置，獲取紋理的文件位置，然後加載生成紋理，把信息儲存到`Vertex`結構體。看起來像這樣：


```c++
vector<Texture> loadMaterialTextures(aiMaterial* mat, aiTextureType type, string typeName)
{
    vector<Texture> textures;
    for(GLuint i = 0; i < mat->GetTextureCount(type); i++)
    {
        aiString str;
        mat->GetTexture(type, i, &str);
        Texture texture;
        texture.id = TextureFromFile(str.C_Str(), this->directory);
        texture.type = typeName;
        texture.path = str;
        textures.push_back(texture);
    }
    return textures;
}
```

我們先通過`GetTextureCount`函數檢驗材質中儲存的紋理，以期得到我們希望得到的紋理類型。然後我們通過`GetTexture`函數獲取每個紋理的文件位置，這個位置以`aiString`類型儲存。然後我們使用另一個幫助函數，它被命名為：`TextureFromFile`加載一個紋理（使用SOIL），返回紋理的ID。你可以查看列在最後的完整的代碼，如果你不知道這個函數應該怎樣寫出來的話。

!!! Important

    注意，我們假設紋理文件與模型是在相同的目錄裡。我們可以簡單的鏈接紋理位置字符串和之前獲取的目錄字符串（在`loadModel`函數中得到的）來獲得完整的紋理路徑（這就是為什麼`GetTexture`函數同樣需要目錄字符串）。
    
    有些在互聯網上找到的模型使用絕對路徑，它們的紋理位置就不會在每天機器上都有效了。例子裡，你可能希望手工編輯這個文件來使用本地路徑為紋理所使用（如果可能的話）。

這就是使用Assimp來導入一個模型的全部了。你可以在這裡找到[Model類的代碼](http://learnopengl.com/code_viewer.php?code=model_loading/model_unoptimized)。

## 重大優化

我們現在還沒做完。因為我們還想做一個重大的優化（但是不是必須的）。大多數場景重用若干紋理，把它們應用到網格；還是思考那個別墅，它有個花崗岩的紋理作為牆面。這個紋理也可能應用到地板、天花板，樓梯，或者一張桌子、一個附近的小物件。加載紋理需要不少操作，當前的實現中一個新的紋理被加載和生成，來為每個網格使用，即使同樣的紋理之前已經被加載了好幾次。這會很快轉變為你的模型加載實現的瓶頸。

所以我們打算添加一個小小的微調，把模型的代碼改成，儲存所有的已加載紋理到全局。無論在哪兒我們都要先檢查這個紋理是否已經被加載過了。如果加載過了，我們就直接使用這個紋理並跳過整個加載流程來節省處理能力。為了對比紋理我們同樣需要儲存它們的路徑：


```c++
struct Texture {
    GLuint id;
    string type;
    aiString path;  // We store the path of the texture to compare with other textures
};
```

然後我們把所有家在過的紋理儲存到另一個向量中，它是作為一個私有變量聲明在模型類的頂部：


```c++
vector<Texture> textures_loaded;
```

然後，在`loadMaterialTextures`函數中，我們希望把紋理路徑和所有`texture_loaded`向量對比，看看是否當前紋理路徑和其中任何一個是否相同，如果是，我們跳過紋理加載/生成部分，簡單的使用已加載紋理結構體作為網格紋理。這個函數如下所示：


```c++
vector<Texture> loadMaterialTextures(aiMaterial* mat, aiTextureType type, string typeName)
{
    vector<Texture> textures;
    for(GLuint i = 0; i < mat->GetTextureCount(type); i++)
    {
        aiString str;
        mat->GetTexture(type, i, &str);
        GLboolean skip = false;
        for(GLuint j = 0; j < textures_loaded.size(); j++)
        {
            if(textures_loaded[j].path == str)
            {
                textures.push_back(textures_loaded[j]);
                skip = true; 
                break;
            }
        }
        if(!skip)
        {   // 如果紋理沒有被加載過，加載之
            Texture texture;
            texture.id = TextureFromFile(str.C_Str(), this->directory);
            texture.type = typeName;
            texture.path = str;
            textures.push_back(texture);
            this->textures_loaded.push_back(texture);  // 添加到紋理列表 textures
        }
    }
    return textures;
}
```

所以現在我們不僅有了一個通用模型加載系統，同時我們也得到了一個能使加載對象更快的優化版本。

!!! Attention

    有些版本的Assimp當使用調試版或/和使用你的IDE的調試模式時，模型加載模型實在慢，所以確保在當你加載得很慢的時候用發佈版再測試。

你可以從這裡獲得優化的[Model類的完整源代碼](http://learnopengl.com/code_viewer.php?code=model&type=header)。

## 和箱子模型告別!

現在給我們導入一個天才藝術家創建的模型看看效果，不是我這個天才做的（你不得不承認，這個箱子也許是你見過的最漂亮的立體圖形）。因為我不想過於自誇，所以我會時不時的給其他藝術家進入這個行列的機會，這次我們會加載Crytek原版的孤島危機遊戲中的納米鎧甲。這個模型被輸出為obj和mtl文件，mtl包含模型的diffuse和specular以及法線貼圖（後面會講）。你可以下載這個模型，注意，所有的紋理和模型文件都應該放在同一個目錄，以便載入紋理。

!!! Important
    
    你從這個站點下載的版本是修改過的版本，每個紋理文件路徑已經修改改為本地相對目錄，原來的資源是絕對目錄。

現在在代碼中，聲明一個Model對象，把它模型的文件位置傳遞給它。模型應該自動加載（如果沒有錯誤的話）在遊戲循環中使用它的Draw函數繪製這個對象。沒有更多的緩衝配置，屬性指針和渲染命令，僅僅簡單的一行。如果你創建幾個簡單的著色器，像素著色器只輸出對象的diffuse紋理顏色，結果看上去會有點像這樣：

![](http://www.learnopengl.com/img/model_loading/model_diffuse.png)

你可以從這裡找到帶有[頂點](http://learnopengl.com/code_viewer.php?code=model_loading/model&type=vertex)和[片段](http://learnopengl.com/code_viewer.php?code=model_loading/model&type=fragment)著色器的[完整的源碼](http://learnopengl.com/code_viewer.php?code=model_loading/model_diffuse)。

我們也可以變得更加有創造力，引入兩個點光源到我們之前從光照教程學過的渲染等式，結合高光貼圖獲得驚豔效果：

![](http://www.learnopengl.com/img/model_loading/model_lighting.png)

雖然我不得不承認這個相比之前用過的容器也太炫了。使用Assimp，你可以載入無數在互聯網上找到的模型。只有很少的資源網站提供多種格式的免費3D模型給你下載。一定注意，有些模型仍然不能很好的載入，紋理路徑無效或者這種格式Assimp不能讀。

## 練習

你可以使用兩個點光源重建上個場景嗎？[方案](http://learnopengl.com/code_viewer.php?code=model_loading/model-exercise1)，[著色器](http://learnopengl.com/code_viewer.php?code=model_loading/model-exercise1-shaders)。