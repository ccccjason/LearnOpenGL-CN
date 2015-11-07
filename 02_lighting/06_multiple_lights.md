# 多光源(Multiple lights)

原文     | [Multiple lights](http://learnopengl.com/#!Lighting/Multiple-lights)
      ---|---
作者     | JoeyDeVries
翻譯     | [Geequlim](http://geequlim.com)
校對     | [Geequlim](http://geequlim.com)

我們在前面的教程中已經學習了許多關於OpenGL 光照的知識，其中包括馮氏照明模型（Phong shading）、光照材質（Materials）、光照圖（Lighting maps）以及各種投光物（Light casters）。本教程將結合上述所學的知識，創建一個包含六個光源的場景。我們將模擬一個類似陽光的平行光（Directional light）和4個定點光（Point lights）以及一個手電筒(Flashlight).

要在場景中使用多光源我們需要封裝一些GLSL函數用來計算光照。如果我們對每個光源都去寫一遍光照計算的代碼，這將是一件令人噁心的事情，並且這些放在main函數中的代碼將難以理解，所以我們將一些操作封裝為函數。

GLSL中的函數與C語言的非常相似，它需要一個函數名、一個返回值類型。並且在調用前必須提前聲明。接下來我們將為下面的每一種光照來寫一個函數。

當我們在場景中使用多個光源時一般使用以下途徑：創建一個代表輸出顏色的向量。每一個光源都對輸出顏色貢獻一些顏色。因此，場景中的每個光源將進行獨立運算，並且運算結果都對最終的輸出顏色有一定影響。下面是使用這種方式進行多光源運算的一般結構：

```c++
out vec4 color;

void main()
{
  // 定義輸出顏色
  vec3 output;
  // 將平行光的運算結果顏色添加到輸出顏色
  output += someFunctionToCalculateDirectionalLight();
  // 同樣，將定點光的運算結果顏色添加到輸出顏色
  for(int i = 0; i < nr_of_point_lights; i++)
  	output += someFunctionToCalculatePointLight();
  // 添加其他光源的計算結果顏色（如投射光）
  output += someFunctionToCalculateSpotLight();

  color = vec4(output, 1.0);
}  
```

即使對每一種光源的運算實現不同，但此算法的結構一般是與上述出入不大的。我們將定義幾個用於計算各個光源的函數，並將這些函數的結算結果（返回顏色）添加到輸出顏色向量中。例如，靠近被照射物體的光源計算結果將返回比遠離背照射物體的光源更明亮的顏色。

## 平行光（Directional light）

我們要在片段著色器中定義一個函數用來計算平行光在對應的照射點上的光照顏色，這個函數需要幾個參數並返回一個計算平行光照結果的顏色。

首先我們需要設置一系列用於表示平行光的變量，正如上一節中所講過的，我們可以將這些變量定義在一個叫做**DirLight**的結構體中，並定義一個這個結構體類型的uniform變量。

```c++
struct DirLight {
    vec3 direction;

    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};  
uniform DirLight dirLight;
```

之後我們可以將`dirLight`這個uniform變量作為下面這個函數原型的參數。

```c++
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir);  
```

!!! Important

        和C/C++一樣，我們調用一個函數的前提是這個函數在調用前已經被聲明過（此例中我們是在main函數中調用）。通常情況下我們都將函數定義在main函數之後，為了能在main函數中調用這些函數，我們就必須在main函數之前聲明這些函數的原型，這就和我們寫C語言是一樣的。
        
你已經知道，這個函數需要一個`DirLight`和兩個其他的向量作為參數來計算光照。如果你看過之前的教程的話，你會覺得下面的函數定義得一點也不意外：

```c++
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir)
{
    vec3 lightDir = normalize(-light.direction);
    // 計算漫反射強度
    float diff = max(dot(normal, lightDir), 0.0);
    // 計算鏡面反射強度
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    // 合併各個光照分量
    vec3 ambient  = light.ambient  * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse  = light.diffuse  * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    return (ambient + diffuse + specular);
}  
```

我們從之前的教程中複製了代碼，並用兩個向量來作為函數參數來計算出平行光的光照顏色向量，該結果是一個由該平行光的環境反射、漫反射和鏡面反射的各個分量組成的一個向量。

## 定點光（Point light）

和計算平行光一樣，我們同樣需要定義一個函數用於計算定點光照。同樣的，我們定義一個包含定點光源所需屬性的結構體：

```c++
struct PointLight {
    vec3 position;

    float constant;
    float linear;
    float quadratic;  

    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};  
#define NR_POINT_LIGHTS 4  
uniform PointLight pointLights[NR_POINT_LIGHTS];
```

如你所見，我們在GLSL中使用預處理器指令來定義定點光源的數目。之後我們使用這個`NR_POINT_LIGHTS`常量來創建一個`PointLight`結構體的數組。和C語言一樣，GLSL也是用一對中括號來創建數組的。現在我們有了4個`PointLight`結構體對象了。

!!! Important

    我們同樣可以簡單粗暴地定義一個大號的結構體（而不是為每一種類型的光源定義一個結構體），它包含所有類型光源所需要屬性變量。並且將這個結構體應用與所有的光照計算函數，在各個光照結算時忽略不需要的屬性變量。然而，就我個人來說更喜歡分開定義，這樣可以省下一些內存，因為定義一個大號的光源結構體在計算過程中會有用不到的變量。

```c++
vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir);  
```

這個函數將所有用得到的數據作為它的參數並使用一個`vec3`作為它的返回值類表示一個頂點光的結算結果。我們再一次聰明地從之前的教程中複製代碼來把它定義成下面的樣子：

```c++
// 計算定點光在確定位置的光照顏色
vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir)
{
    vec3 lightDir = normalize(light.position - fragPos);
    // 計算漫反射強度
    float diff = max(dot(normal, lightDir), 0.0);
    // 計算鏡面反射
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    // 計算衰減
    float distance    = length(light.position - fragPos);
    float attenuation = 1.0f / (light.constant + light.linear * distance +
  			     light.quadratic * (distance * distance));
    // 將各個分量合併
    vec3 ambient  = light.ambient  * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse  = light.diffuse  * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    ambient  *= attenuation;
    diffuse  *= attenuation;
    specular *= attenuation;
    return (ambient + diffuse + specular);
}
```

有了這個函數我們就可以在main函數中調用它來代替寫很多個計算點光源的代碼了。通過循環調用此函數就能實現同樣的效果，當然代碼更簡潔。

## 把它們放到一起

我們現在定義了用於計算平行光和定點光的函數，現在我們把這些代碼放到一起，寫入文開始的一般結構中：

```c++
void main()
{
    // 一些屬性
    vec3 norm = normalize(Normal);
    vec3 viewDir = normalize(viewPos - FragPos);

    // 第一步，計算平行光照
    vec3 result = CalcDirLight(dirLight, norm, viewDir);
    // 第二步，計算頂點光照
    for(int i = 0; i < NR_POINT_LIGHTS; i++)
        result += CalcPointLight(pointLights[i], norm, FragPos, viewDir);
    // 第三部，計算 Spot light
    //result += CalcSpotLight(spotLight, norm, FragPos, viewDir);

    color = vec4(result, 1.0);
}
```

每一個光源的運算結果都添加到了輸出顏色上，輸出顏色包含了此場景中的所有光源的影響。如果你想實現手電筒的光照效果，同樣的把計算結果添加到輸出顏色上。我在這裡就把`CalcSpotLight`的實現留作個讀者們的練習吧。

設置平行光結構體的uniform值和之前所講過的方式沒什麼兩樣，但是你可能想知道如何設置場景中`PointLight`結構體的uniforms變量數組。我們之前並未討論過如何做這件事。

慶幸的是，這並不是什麼難題。設置uniform變量數組和設置單個uniform變量值是相似的，只需要用一個合適的下標就能夠檢索到數組中我們想要的uniform變量了。

```c++
glUniform1f(glGetUniformLocation(lightingShader.Program, "pointLights[0].constant"), 1.0f);
```

這樣我們檢索到`pointLights`數組中的第一個`PointLight`結構體元素，同時也可以獲取到該結構體中的各個屬性變量。不幸的是這一位置我們還需要手動對這個四個光源的每一個屬性都進行設置，這樣手動設置這28個uniform變量是相當乏味的工作。你可以嘗試去定義個光源類來為你設置這些uniform屬性來減少你的工作，但這依舊不能改變去設置每個uniform屬性的事實。

別忘了，我們還需要為每個光源設置它們的位置。這裡，我們定義一個`glm::vec3`類的數組來包含這些點光源的座標：

```c++
glm::vec3 pointLightPositions[] = {
	glm::vec3( 0.7f,  0.2f,  2.0f),
	glm::vec3( 2.3f, -3.3f, -4.0f),
	glm::vec3(-4.0f,  2.0f, -12.0f),
	glm::vec3( 0.0f,  0.0f, -3.0f)
};  
```

同時我們還需要根據這些光源的位置在場景中繪製4個表示光源的立方體，這樣的工作我們在之前的教程中已經做過了。

如果你在還是用了手電筒的話，將所有的光源結合起來看上去應該和下圖差不多：

![](http://learnopengl.com/img/lighting/multiple_lights_combined.png)

你可以在此處獲取本教程的[源代碼](http://learnopengl.com/code_viewer.php?code=lighting/multiple_lights)，同時可以查看[頂點著色器](http://learnopengl.com/code_viewer.php?code=lighting/lighting_maps&type=vertex)和[片段著色器](http://learnopengl.com/code_viewer.php?code=lighting/multiple_lights&type=fragment)的代碼。

上面的圖片的光源都是使用默認的屬性的效果，如果你嘗試對光源屬性做出各種修改嘗試的話，會出現很多有意思的畫面。很多藝術家和場景編輯器都提供大量的按鈕或方式來修改光照以使用各種環境。使用最簡單的光照屬性的改變我們就足已創建有趣的視覺效果：

![](http://learnopengl.com/img/lighting/multiple_lights_atmospheres.png)

相信你現在已經對OpenGL的光照有很好的理解了。有了這些知識我們便可以創建豐富有趣的環境和氛圍了。快試試改變所有的屬性的值來創建你的光照環境吧！

## 練習

* 創建一個表示手電筒光的結構體Spotlight並實現CalcSpotLight(...)函數：[解決方案](http://learnopengl.com/code_viewer.php?code=lighting/multiple_lights-exercise1)
* 你能通過調節不同的光照屬性來重新創建一個不同的氛圍嗎？[解決方案](http://learnopengl.com/code_viewer.php?code=lighting/multiple_lights-exercise2)