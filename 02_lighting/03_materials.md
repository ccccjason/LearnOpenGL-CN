# 材質(Material)

原文     | [Materials](http://learnopengl.com/#!Lighting/Materials)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

在真實世界裡，每個物體會對光產生不同的反應。鋼看起來比陶瓷花瓶更閃閃發光，一個木頭箱子不會像鋼箱子一樣對光產生很強的反射。每個物體對鏡面高光也有不同的反應。有些物體不會散射(Scatter)很多光卻會反射(Reflect)很多光，結果看起來就有一個較小的高光點(Highlight)，有些物體散射了很多，它們就會產生一個半徑更大的高光。如果我們想要在OpenGL中模擬多種類型的物體，我們必須為每個物體分別定義材質(Material)屬性。

在前面的教程中，我們指定一個物體和一個光的顏色來定義物體的圖像輸出，並使之結合環境(Ambient)和鏡面強度(Specular Intensity)元素。當描述物體的時候，我們可以使用3種光照元素：環境光照(Ambient Lighting)、漫反射光照(Diffuse Lighting)、鏡面光照(Specular Lighting)定義一個材質顏色。通過為每個元素指定一個顏色，我們已經對物體的顏色輸出有了精密的控制。現在把一個鏡面高光元素添加到這三個顏色裡，這是我們需要的所有材質屬性：

```c++
#version 330 core
struct Material
{
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
    float shininess;
};
uniform Material material;
```

在片段著色器中，我們創建一個結構體(Struct)，來儲存物體的材質屬性。我們也可以把它們儲存為獨立的uniform值，但是作為一個結構體來儲存可以更有條理。我們首先定義結構體的佈局，然後簡單聲明一個uniform變量，以新創建的結構體作為它的類型。

就像你所看到的，我們為每個馮氏光照模型的元素都定義一個顏色向量。`ambient`材質向量定義了在環境光照下這個物體反射的是什麼顏色；通常這是和物體顏色相同的顏色。`diffuse`材質向量定義了在漫反射光照照下物體的顏色。漫反射顏色被設置為(和環境光照一樣)我們需要的物體顏色。`specular`材質向量設置的是物體受到的鏡面光照的影響的顏色(或者可能是反射一個物體特定的鏡面高光顏色)。最後，`shininess`影響鏡面高光的散射/半徑。

這四個元素定義了一個物體的材質，通過它們我們能夠模擬很多真實世界的材質。這裡有一個列表[devernay.free.fr](http://devernay.free.fr/cours/opengl/materials.html)展示了幾種材質屬性，這些材質屬性模擬外部世界的真實材質。下面的圖片展示了幾種真實世界材質對我們的立方體的影響：

![](http://www.learnopengl.com/img/lighting/materials_real_world.png)

如你所見，正確地指定一個物體的材質屬性，似乎就是改變我們物體的相關屬性的比例。效果顯然很引人注目，但是對於大多數真實效果，我們最終需要更加複雜的形狀，而不單單是一個立方體。在[下面的教程](http://learnopengl-cn.readthedocs.org/zh/latest/03%20Model%20Loading/01%20Assimp/)中，我們會討論更復雜的形狀。

為一個物體賦予一款正確的材質是非常困難的，這需要大量實驗和豐富的經驗，所以由於錯誤的設置材質而毀了物體的畫面質量是件經常發生的事。

讓我們試試在著色器中實現這樣的一個材質系統。


## 設置材質

我們在片段著色器中創建了一個uniform材質結構體，所以下面我們希望改變光照計算來順應新的材質屬性。由於所有材質元素都儲存在結構體中，我們可以從uniform變量`material`取得它們：

```c++
void main()
{
    // 環境光
    vec3 ambient = lightColor * material.ambient;

    // 漫反射光
    vec3 norm = normalize(Normal);
    vec3 lightDir = normalize(lightPos - FragPos);
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = lightColor * (diff * material.diffuse);

    // 鏡面高光
    vec3 viewDir = normalize(viewPos - FragPos);
    vec3 reflectDir = reflect(-lightDir, norm);  
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    vec3 specular = lightColor * (spec * material.specular);  

    vec3 result = ambient + diffuse + specular;
    color = vec4(result, 1.0f);
}
```

你可以看到，我們現在獲得所有材質結構體的屬性，無論在哪兒我們都需要它們，這次通過材質顏色的幫助，計算結果輸出的顏色。物體的每個材質屬性都乘以它們對應的光照元素。

通過設置適當的uniform，我們可以在應用中設置物體的材質。當設置uniform時，GLSL中的一個結構體並不會被認為有什麼特別之處。一個結構體值扮演uniform變量的封裝體，所以如果我們希望填充這個結構體，我們就仍然必須設置結構體中的各個元素的uniform值，但是這次帶有結構體名字作為前綴：

```c++
GLint matAmbientLoc = glGetUniformLocation(lightingShader.Program, "material.ambient");
GLint matDiffuseLoc = glGetUniformLocation(lightingShader.Program, "material.diffuse");
GLint matSpecularLoc = glGetUniformLocation(lightingShader.Program, "material.specular");
GLint matShineLoc = glGetUniformLocation(lightingShader.Program, "material.shininess");

glUniform3f(matAmbientLoc, 1.0f, 0.5f, 0.31f);
glUniform3f(matDiffuseLoc, 1.0f, 0.5f, 0.31f);
glUniform3f(matSpecularLoc, 0.5f, 0.5f, 0.5f);
glUniform1f(matShineLoc, 32.0f);
```

我們將`ambient`和`diffuse`元素設置成我們想要讓物體所呈現的顏色，設置物體的`specular`元素為中等亮度顏色；我們不希望`specular`元素對這個指定物體產生過於強烈的影響。我們同樣設置`shininess`為32。我們現在可以簡單的在應用中影響物體的材質。

運行程序，會得到下面這樣的結果：

![](http://www.learnopengl.com/img/lighting/materials_with_material.png)

看起來很奇怪不是嗎？


### 光的屬性

這個物體太亮了。物體過亮的原因是環境、漫反射和鏡面三個顏色任何一個光源都會去全力反射。光源對環境、漫反射和鏡面元素同時具有不同的強度。前面的教程，我們通過使用一個強度值改變環境和鏡面強度的方式解決了這個問題。我們想做一個相同的系統，但是這次為每個光照元素指定了強度向量。如果我們想象`lightColor`是`vec3(1.0)`，代碼看起來像是這樣：

```c++
vec3 ambient = vec3(1.0f) * material.ambient;
vec3 diffuse = vec3(1.0f) * (diff * material.diffuse);
vec3 specular = vec3(1.0f) * (spec * material.specular);
```

所以物體的每個材質屬性返回了每個光照元素的全強度。這些vec3(1.0)值可以各自獨立的影響各個光源，這通常就是我們想要的。現在物體的`ambient`元素完全地展示了立方體的顏色，可是環境元素不應該對最終顏色有這麼大的影響，所以我們要設置光的`ambient`亮度為一個小一點的值，從而限制環境色：

```c++
vec3 result = vec3(0.1f) * material.ambient;
```

我們可以用同樣的方式影響光源`diffuse`和`specular`的強度。這和我們[前面教程](http://learnopengl-cn.readthedocs.org/zh/latest/02%20Lighting/02%20Basic%20Lighting/)所做的極為相似；你可以說我們已經創建了一些光的屬性來各自獨立地影響每個光照元素。我們希望為光的屬性創建一些與材質結構體相似的東西：

```c++
struct Light
{
    vec3 position;
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};
uniform Light light;
```

一個光源的`ambient`、`diffuse`和`specular`光都有不同的亮度。環境光通常設置為一個比較低的亮度，因為我們不希望環境色太過顯眼。光源的`diffuse`元素通常設置為我們希望光所具有的顏色；經常是一個明亮的白色。`specular`元素通常被設置為`vec3(1.0f)`類型的全強度發光。要記住的是我們同樣把光的位置添加到結構體中。

就像材質uniform一樣，需要更新片段著色器：

```c++
vec3 ambient = light.ambient * material.ambient;
vec3 diffuse = light.diffuse * (diff * material.diffuse);
vec3 specular = light.specular * (spec * material.specular);
```

然後我們要在應用裡設置光的亮度：

```c++
GLint lightAmbientLoc = glGetUniformLocation(lightingShader.Program, "light.ambient");
GLint lightDiffuseLoc = glGetUniformLocation(lightingShader.Program, "light.diffuse");
GLint lightSpecularLoc = glGetUniformLocation(lightingShader.Program, "light.specular");

glUniform3f(lightAmbientLoc, 0.2f, 0.2f, 0.2f);
glUniform3f(lightDiffuseLoc, 0.5f, 0.5f, 0.5f);// 讓我們把這個光調暗一點，這樣會看起來更自然
glUniform3f(lightSpecularLoc, 1.0f, 1.0f, 1.0f);
```

現在，我們調整了光是如何影響物體所有的材質的，我們得到一個更像前面教程的視覺輸出。這次我們完全控制了物體光照和材質：

![](http://www.learnopengl.com/img/lighting/materials_light.png)

現在改變物體的外觀相對簡單了些。我們做點更有趣的事！


### 不同的光源顏色

目前為止，我們使用光源的顏色僅僅去改變物體各個元素的強度(通過選用從白到灰到黑範圍內的顏色)，並沒有影響物體的真實顏色(只是強度)。由於現在能夠非常容易地訪問光的屬性了，我們可以隨著時間改變它們的顏色來獲得一些有很意思的效果。由於所有東西都已經在片段著色器做好了，改變光的顏色很簡單，我們可以立即創建出一些有趣的效果：

<video src="http://www.learnopengl.com/video/lighting/materials.mp4" controls="controls">
</video>

如你所見，不同光的顏色極大地影響了物體的顏色輸出。由於光的顏色直接影響物體反射的顏色(你可能想起在顏色教程中有討論過)，它對視覺輸出有顯著的影響。

利用`sin`和`glfwGetTime`改變光的環境和漫反射顏色，我們可以隨著時間流逝簡單的改變光源顏色：

```c++
glm::vec3 lightColor; lightColor.x = sin(glfwGetTime() * 2.0f);
lightColor.y = sin(glfwGetTime() * 0.7f);
lightColor.z = sin(glfwGetTime() * 1.3f);

glm::vec3 diffuseColor = lightColor * glm::vec3(0.5f);
glm::vec3 ambientColor = diffuseColor * glm::vec3(0.2f);

glUniform3f(lightAmbientLoc, ambientColor.x, ambientColor.y, ambientColor.z);
glUniform3f(lightDiffuseLoc, diffuseColor.x, diffuseColor.y, diffuseColor.z);
```

嘗試和實驗使用這些光照和材質值，看看它們怎樣影響圖像輸出的。你可以從這裡找到[程序的源碼](http://learnopengl.com/code_viewer.php?code=lighting/materials)，[片段著色器](http://learnopengl.com/code_viewer.php?code=lighting/materials&type=fragment)的源碼。

## 練習

- 你能像我們教程一開始那樣根據一些材質的屬性來模擬一個真實世界的物體嗎？
注意[材質表](http://devernay.free.fr/cours/opengl/materials.html)中的環境光顏色與漫反射光的顏色可能不一樣，因為他們並沒有把光照強度考慮進去來模擬，你需要將光照顏色的強度改為`vec(1.0f)`來輸出正確的結果：[參考解答](http://learnopengl.com/code_viewer.php?code=lighting/materials-exercise1)，我做了一個青色(Cyan)的塑料箱子
