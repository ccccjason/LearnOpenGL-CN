# 光照貼圖

原文     | [Lighting maps](http://learnopengl.com/#!Lighting/Lighting-maps)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

前面的教程，我們討論了每個物體都擁有各自不同的材質，因而會對光做出不同的反應。在一個光照場景中，給每個物體和其他物體不同的外觀很棒，但是這仍然不會對一個物體的圖像輸出提供很多的伸縮性。

前面的教程我們為一個物體的作為一個整體定義了一個材質，但是現實世界的物體通常不會只有這麼一種材質，而是由多種材質組成。想象一輛車：它的外表質地光亮，車窗會部分反射環境，它的輪胎沒有specular高光，輪彀卻非常閃亮（在洗過之後）。汽車同樣有diffuse和ambient顏色，它們在整個車上都不相同；一輛車顯示了多種不同的ambient/diffuse顏色。總之，這樣一個物體每個部分都有多種材質屬性。

所以，前面的材質系統對於除了最簡單的模型以外都是不夠的，所以我們需要擴展前面的系統，我們要介紹diffuse和specular貼圖。它們允許你對一個物體的diffuse（而對於簡潔的ambient成分來說，它們幾乎總是是一樣的）和specular成分能夠有更精確的影響。

## 漫反射貼圖

我們希望通過某種方式對每個原始像素獨立設置diffuse顏色。有可以讓我們基於物體原始像素的位置來獲取顏色值的系統嗎？

這可能聽起來極其相似，坦白來講我們我們使用這樣的系統已經有一會兒了。聽起來很像在一個較早的教程中談論的紋理，它基本就是一個紋理。我們其實是使用同一個潛在原則下的不同名稱：使用一張圖片包裹住物體，我們為每個原始像素索引獨立顏色值。在有光的場景裡，通常叫做漫反射貼圖(Diffuse texture)，因為這個紋理圖像表現了所有物體的diffuse顏色。

為了強調漫反射貼圖，我們將會使用下面的圖片，它是一個有一圈鋼邊的木箱：

![](http://www.learnopengl.com/img/textures/container2.png)

在著色器中使用漫反射貼圖和紋理教程介紹的一樣。這次我們把紋理儲存為sampler2D，並在Material結構體中。我們使用diffuse貼圖替代早期定義的vec3類型的diffuse顏色。

!!! Attention
    
    要記住的是sampler2D也叫做模糊類型，這意味著我們不能以某種類型對它實例化，只能用uniform定義它們。如果我們用結構體而不是uniform實例化（就像函數的參數那樣），GLSL會拋出奇怪的錯誤；這同樣也適用於其他模糊類型。
我們也要移除amibient材質顏色向量，因為ambient顏色絕大多數情況等於diffuse顏色，所以不需要分別去儲存它：

```c++
struct Material
{
    sampler2D diffuse;
    vec3 specular;
    float shininess;
};
...
in vec2 TexCoords;
```
!!! Important

    如果你非把ambient顏色設置為不同的值不可（不同於diffuse值），你可以繼續保持ambient的vec3，但是整個物體的ambient顏色會繼續保持不變。為了使每個原始像素得到不同ambient值，你需要對ambient值單獨使用另一個紋理。

注意，在片段著色器中我們將會再次需要紋理座標，所以我們聲明一個額外輸入變量。然後我們簡單地從紋理採樣，來獲得原始像素的diffuse顏色值：
    
```c++
vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
```

同樣，不要忘記把ambient材質的顏色設置為diffuse材質的顏色：

```c++
vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
```

這就是diffuse貼圖的全部內容了。就像你看到的，這不是什麼新的東西，但是它卻極大提升了視覺品質。為了讓它工作，我們需要用到紋理座標更新頂點數據，把它們作為頂點屬性傳遞到片段著色器，把紋理加載並綁定到合適的紋理單元。

[更新的頂點數據可以從這裡找到](http://learnopengl.com/code_viewer.php?code=lighting/vertex_data_textures)。頂點數據現在包括了頂點位置，法線向量和紋理座標，每個立方體的頂點都有這些屬性。讓我們更新頂點著色器來接受紋理座標作為頂點屬性，然後發送到片段著色器：

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 texCoords;
...
out vec2 TexCoords;

void main()
{
    ...
    TexCoords = texCoords;
}
```

要保證更新的頂點屬性指針，不僅是VAO匹配新的頂點數據，也要把箱子圖片加載為紋理。在繪製箱子之前，我們希望首選紋理單元被賦為material.diffuse這個uniform採樣器，並綁定箱子的紋理到這個紋理單元：

```c++
glUniform1i(glGetUniformLocation(lightingShader.Program, "material.diffuse"), 0);
...
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, diffuseMap);
```

現在，使用一個diffuse貼圖，我們在細節上再次獲得驚人的提升，這次添加到箱子上的光照開始閃光了（名符其實）。你的箱子現在可能看起來像這樣：

![](http://www.learnopengl.com/img/lighting/materials_diffuse_map.png)

你可以在這裡得到應用的[全部代碼](http://learnopengl.com/code_viewer.php?code=lighting/lighting_maps_diffuse)。


## 鏡面貼圖

你可能注意到，specular高光看起來不怎麼樣，由於我們的物體是個箱子，大部分是木頭，我們知道木頭是不應該有鏡面高光的。我們通過把物體設置specular材質設置為vec3(0.0f)來修正它。但是這樣意味著鐵邊會不再顯示鏡面高光，我們知道鋼鐵是會顯示一些鏡面高光的。我們會想要控制物體部分地顯示鏡面高光，它帶有修改了的亮度。這個問題看起來和diffuse貼圖的討論一樣。這是巧合嗎？我想不是。

我們同樣用一個紋理貼圖，來獲得鏡面高光。這意味著我們需要生成一個黑白（或者你喜歡的顏色）紋理來定義specular亮度，把它應用到物體的每個部分。下面是一個specular貼圖的例子：

![](http://www.learnopengl.com/img/textures/container2_specular.png)

一個specular高光的亮度可以通過圖片中每個紋理的亮度來獲得。specular貼圖的每個像素可以顯示為一個顏色向量，比如：在那裡黑色代表顏色向量vec3(0.0f)，灰色是vec3(0.5f)。在片段著色器中，我們採樣相應的顏色值，把它乘以光的specular亮度。像素越“白”，乘積的結果越大，物體的specualr部分越亮。

由於箱子幾乎是由木頭組成，木頭作為一個材質不會有鏡面高光，整個不透部分的diffuse紋理被用黑色覆蓋：黑色部分不會包含任何specular高光。箱子的鐵邊有一個修改的specular亮度，它自身更容易受到鏡面高光影響，木紋部分則不會。

從技術上來講，木頭也有鏡面高光，儘管這個閃亮值很小（更多的光被散射），影響很小，但是為了學習目的，我們可以假裝木頭不會有任何specular光反射。

使用Photoshop或Gimp之類的工具，剪切一些部分，非常容易變換一個diffuse紋理，為specular圖片，以增加亮度/對比度的方式，可以把這個部分變換為黑色或白色。


### 鏡面貼圖採樣

一個specular貼圖和其他紋理一樣，所以代碼和diffuse貼圖的代碼也相似。確保合理的加載了圖片，生成一個紋理對象。由於我們在同樣的片段著色器中使用另一個紋理採樣器，我們必須為specular貼圖使用一個不同的紋理單元（查看紋理），所以在渲染前讓我們把它綁定到合適的紋理單元

```c++
glUniform1i(glGetUniformLocation(lightingShader.Program, "material.specular"), 1);
...
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, specularMap);
```

然後更新片段著色器材質屬性，接受一個sampler2D作為這個specular部分的類型，而不是vec3：

```c++
struct Material
{
    sampler2D diffuse;
    sampler2D specular;
    float shininess;
};
```

最後我們希望採樣這個specular貼圖，來獲取原始像素相應的specular亮度：

```c++
vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
color = vec4(ambient + diffuse + specular, 1.0f);
```

通過使用一個specular貼圖我們可以定義極為精細的細節，物體的這個部分會獲得閃亮的屬性，我們可以設置它們相應的亮度。specular貼圖給我們一個附加的高於diffuse貼圖的控制權限。

如果你不想成為主流，你可以在specular貼圖裡使用顏色，不單單為每個原始像素設置specular亮度，同時也設置specular高光的顏色。從真實角度來說，specular的顏色基本是由光源自身決定的，所以它不會生成真實的圖像（這就是為什麼圖片通常是黑色和白色的：我們只關心亮度）。

如果你現在運行應用，你可以清晰地看到箱子的材質現在非常類似真實的鐵邊的木頭箱子了：

![](http://www.learnopengl.com/img/lighting/materials_specular_map.png)

你可以在這裡找到[全部源碼](http://learnopengl.com/code_viewer.php?code=lighting/lighting_maps_specular)。也對比一下你的[頂點著色器](http://learnopengl.com/code_viewer.php?code=lighting/lighting_maps&type=vertex)和[片段著色器](http://learnopengl.com/code_viewer.php?code=lighting/lighting_maps&type=fragment)。

使用diffuse和specular貼圖，我們可以給相關但簡單物體添加一個極為明顯的細節。我們可以使用其他紋理貼圖，比如法線/bump貼圖或者反射貼圖，給物體添加更多的細節。但是這些在後面教程才會涉及。把你的箱子給你所有的朋友和家人看，有一天你會很滿足，我們的箱子會比現在更漂亮！
