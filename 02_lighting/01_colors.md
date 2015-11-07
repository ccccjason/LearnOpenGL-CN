# 顏色

原文     | [Colors](http://learnopengl.com/#!Lighting/Colors)
      ---|---
作者     | JoeyDeVries
翻譯     | [Geequlim](http://geequlim.com/)
校對     | [Geequlim](http://geequlim.com/)

在前面的教程中我們已經簡要提到過該如何在OpenGL中使用顏色(Color)，但是我們至今所接觸到的都是很淺層的知識。本節我們將會更廣泛地討論顏色，並且還會為接下來的光照(Lighting)教程創建一個場景。


現實世界中有無數種顏色，每一個物體都有它們自己的顏色。我們要做的工作是使用(有限的)數字來模擬真實世界中(無限)的顏色，因此並不是所有的現實世界中的顏色都可以用數字來表示。然而我們依然可以用數字來代表許多種顏色，並且你甚至可能根本感覺不到他們與真實顏色之間的差異。顏色可以數字化的由紅色(Red)、綠色(Green)和藍色(Blue)三個分量組成，它們通常被縮寫為RGB。這三個不同的分量組合在一起幾乎可以表示存在的任何一種顏色。例如,要獲取一個**珊瑚紅(Coral)**顏色我們可以這樣定義一個顏色向量:

```c++
glm::vec3 coral(1.0f, 0.5f, 0.31f);
```

我們在現實生活中看到某一物體的顏色並不是這個物體的真實顏色，而是它所反射(Reflected)的顏色。換句話說，那些不能被物體吸收(Absorb)的顏色(被反射的顏色)就是我們能夠感知到的物體的顏色。例如,太陽光被認為是由許多不同的顏色組合成的白色光(如下圖所示)。如果我們將白光照在一個藍色的玩具上，這個藍色的玩具會吸收白光中除了藍色以外的所有顏色，不被吸收的藍色光被反射到我們的眼中，使我們看到了一個藍色的玩具。下圖顯示的是一個珊瑚紅的玩具，它以不同強度的方式反射了幾種不同的顏色。

![](http://learnopengl.com/img/lighting/light_reflection.png)

正如你所見，白色的陽光是一種所有可見顏色的集合，上面的物體吸收了其中的大部分顏色，它僅反射了那些代表這個物體顏色的部分，這些被反射顏色的組合就是我們感知到的顏色(此例中為珊瑚紅)。

這些顏色反射的規律被直接地運用在圖形領域。我們在OpenGL中創建一個光源時都會為它定義一個顏色。在前面的段落中所提到光源的顏色都是白色的，那我們就繼續來創建一個白色的光源吧。當我們把光源的顏色與物體的顏色相乘，所得到的就是這個物體所反射該光源的顏色(也就是我們感知到的顏色)。讓我們再次審視我們的玩具(這一次它還是珊瑚紅)並看看如何計算出他的反射顏色。我們通過檢索結果顏色的每一個分量來看一下光源色和物體顏色的反射運算：

```c++
glm::vec3 lightColor(1.0f, 1.0f, 1.0f);
glm::vec3 toyColor(1.0f, 0.5f, 0.31f);
glm::vec3 result = lightColor * toyColor; // = (1.0f, 0.5f, 0.31f);
```

我們可以看到玩具在進行反射時**吸收**了白色光源顏色中的大部分顏色，但它對紅、綠、藍三個分量都有一定的反射，反射量是由物體本身的顏色所決定的。這也代表著現實中的光線原理。由此，我們可以定義物體的顏色為**這個物體從一個光源反射各個顏色分量的多少**。現在，如果我們使用一束綠色的光又會發生什麼呢？

```c++
glm::vec3 lightColor(0.0f, 1.0f, 0.0f);
glm::vec3 toyColor(1.0f, 0.5f, 0.31f);
glm::vec3 result = lightColor * toyColor; // = (0.0f, 0.5f, 0.0f);
```

可以看到，我們的玩具沒有紅色和藍色的光讓它來吸收或反射，這個玩具也吸收了光線中一半的綠色，當然它仍然反射了光的一半綠色。它現在看上去是深綠色(Dark-greenish)的。我們可以看到，如果我們用一束綠色的光線照來照射玩具，那麼只有綠色能被反射和感知到，沒有紅色和藍色能被反射和感知。這樣做的結果是，一個珊瑚紅的玩具突然變成了深綠色物體。現在我們來看另一個例子，使用深橄欖綠色(Dark olive-green)的光線：

```c++
glm::vec3 lightColor(0.33f, 0.42f, 0.18f);
glm::vec3 toyColor(1.0f, 0.5f, 0.31f);
glm::vec3 result = lightColor * toyColor; // = (0.33f, 0.21f, 0.06f);
```

如你所見，我們可以通過物體對不同顏色光的反射來的得到意想不到的不到的顏色，從此創作顏色已經變得非常簡單。

目前有了這些顏色相關的理論已經足夠了，接下來我們將創建一個場景用來做更多的實驗。

## 創建一個光照場景

在接下來的教程中，我們將通過模擬真實世界中廣泛存在的光照和顏色現象來創建有趣的視覺效果。現在我們將在場景中創建一個看得到的物體來代表光源，並且在場景中至少添加一個物體來模擬光照。

首先我們需要一個物體來投光(Cast the light)，我們將無恥地使用前面教程中的立方體箱子。我們還需要一個物體來代表光源，它代表光源在這個3D空間中的確切位置。簡單起見，我們依然使用一個立方體來代表光源(我們已擁有立方體的[頂點數據](http://www.learnopengl.com/code_viewer.php?code=getting-started/cube_vertices)是吧？)。

當然，填一個頂點緩衝對象(VBO)，設定一下頂點屬性指針和其他一些亂七八糟的東西現在對你來說應該很容易了，所以我們就不再贅述那些步驟了。如果你仍然覺得這很困難，我建議你複習[之前的教程](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/04%20Hello%20Triangle/)，並且在繼續學習之前先把練習過一遍。

所以，我們首先需要一個頂點著色器來繪製箱子。與上一個教程的頂點著色器相比，容器的頂點位置保持不變(雖然這一次我們不需要紋理座標)，因此頂點著色器中沒有新的代碼。我們將會使用上一篇教程頂點著色器的精簡版：

```c++
#version 330 core
layout (location = 0) in vec3 position;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
    gl_Position = projection * view * model * vec4(position, 1.0f);
}
```

請確認更新你的頂點數據和屬性對應的指針與新的頂點著色器一致(當然你可以繼續保留紋理數據並保持屬性對應的指針有效。在這一節中我們不使用紋理，但如果你想要一個全新的開始那也不是什麼壞主意)。

因為我們還要創建一個表示燈(光源)的立方體，所以我們還要為這個燈創建一個特殊的VAO。當然我們也可以讓這個燈和其他物體使用同一個VAO然後對他的`model`(模型)矩陣做一些變換，然而接下來的教程中我們會頻繁地對頂點數據做一些改變並且需要改變屬性對應指針設置，我們並不想因此影響到燈(我們只在乎燈的位置)，因此我們有必要為燈創建一個新的VAO。

```c++
GLuint lightVAO;
glGenVertexArrays(1, &lightVAO);
glBindVertexArray(lightVAO);
// 只需要綁定VBO不用再次設置VBO的數據，因為容器(物體)的VBO數據中已經包含了正確的立方體頂點數據
glBindBuffer(GL_ARRAY_BUFFER, VBO);
// 設置燈立方體的頂點屬性指針(僅設置燈的頂點數據)
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid*)0);
glEnableVertexAttribArray(0);
glBindVertexArray(0);
```

這段代碼對你來說應該非常直觀。既然我們已經創建了表示燈和被照物體的立方體，我們只需要再定義一個東西就行了了，那就是片段著色器

```c++
#version 330 core
out vec4 color;

uniform vec3 objectColor;
uniform vec3 lightColor;

void main()
{
    color = vec4(lightColor * objectColor, 1.0f);
}
```

這個片段著色器接受兩個分別表示物體顏色和光源顏色的uniform變量。正如本篇教程一開始所討論的一樣，我們將光源的顏色與物體(能反射)的顏色相乘。這個著色器應該很容易理解。接下來讓我們把物體的顏色設置為上一節中所提到的珊瑚紅並把光源設置為白色：

```c++
// 在此之前不要忘記首先'使用'對應的著色器程序(來設定uniform)
GLint objectColorLoc = glGetUniformLocation(lightingShader.Program, "objectColor");
GLint lightColorLoc  = glGetUniformLocation(lightingShader.Program, "lightColor");
glUniform3f(objectColorLoc, 1.0f, 0.5f, 0.31f);// 我們所熟悉的珊瑚紅
glUniform3f(lightColorLoc,  1.0f, 1.0f, 1.0f); // 依舊把光源設置為白色
```

要注意的是，當我們修改頂點或者片段著色器後，燈的位置或顏色也會隨之改變，這並不是我們想要的效果。我們不希望燈對象的顏色在接下來的教程中因光照計算的結果而受到影響，而希望它能夠獨立。希望表示燈不受其他光照的影響而一直保持明亮(這樣它才更像是一個真實的光源)。

為了實現這個目的，我們需要為燈創建另外的一套著色器程序，從而能保證它能夠在其他光照著色器變化的時候保持不變。頂點著色器和我們當前的頂點著色器是一樣的，所以你可以直接把燈的頂點著色器複製過來。片段著色器保證了燈的顏色一直是亮的，我們通過給燈定義一個常量的白色來實現：

```c++
#version 330 core
out vec4 color;

void main()
{
    color = vec4(1.0f); //設置四維向量的所有元素為 1.0f
}
```

當我們想要繪製我們的物體的時候，我們需要使用剛剛定義的光照著色器繪製箱子(或者可能是其它的一些物體)，讓我們想要繪製燈的時候，我們會使用燈的著色器。在之後的教程裡我們會逐步升級這個光照著色器從而能夠緩慢的實現更真實的效果。

使用這個燈立方體的主要目的是為了讓我們知道光源在場景中的具體位置。我們通常在場景中定義一個光源的位置，但這只是一個位置，它並沒有視覺意義。為了顯示真正的燈，我們將表示光源的燈立方體繪製在與光源同樣的位置。我們將使用我們為它新建的片段著色器讓它保持它一直處於白色狀態，不受場景中的光照影響。

我們聲明一個全局`vec3`變量來表示光源在場景的世界空間座標中的位置：

```c++
glm::vec3 lightPos(1.2f, 1.0f, 2.0f);
```

然後我們把燈平移到這兒，當然我們需要對它進行縮放，讓它不那麼明顯：

```c++
model = glm::mat4();
model = glm::translate(model, lightPos);
model = glm::scale(model, glm::vec3(0.2f));
```

繪製燈立方體的代碼應該與下面的類似：

```c++
lampShader.Use();
// 設置模型、視圖和投影矩陣uniform
...
// 繪製燈立方體對象
glBindVertexArray(lightVAO);
glDrawArrays(GL_TRIANGLES, 0, 36);
glBindVertexArray(0);
```

請把上述的所有代碼片段放在你程序中合適的位置，這樣我們就能有一個乾淨的光照實驗場地了。如果一切順利，運行效果將會如下圖所示：

![](http://learnopengl.com/img/lighting/colors_scene.png)

沒什麼好看的是嗎？但我保證在接下來的教程中它會給你有趣的視覺效果。

如果你在把上述代碼片段放到一起編譯遇到困難，可以去認真地看看我的[源代碼](http://learnopengl.com/code_viewer.php?code=lighting/colors_scene)。你好最自己實現一遍這些操作。

現在我們有了一些關於顏色的知識，並且創建了一個基本的場景能夠繪製一些漂亮的光線。你現在可以閱讀[下一個教程](http://learnopengl-cn.readthedocs.org/zh/latest/02%20Lighting/02%20Basic%20Lighting/)，真正的魔法即將開始！
