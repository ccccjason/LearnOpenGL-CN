# 模板測試(Stencil testing)

原文     | [Stencil testing](http://learnopengl.com/#!Advanced-OpenGL/Stencil-testing)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

當片段著色器處理完片段之後，**模板測試(stencil test)** 就開始執行了，和深度測試一樣，它能丟棄一些片段。仍然保留下來的片段進入深度測試階段，深度測試可能丟棄更多。模板測試基於另一個緩衝，這個緩衝叫做**模板緩衝(stencil buffer)**，我們被允許在渲染時更新它來獲取有意思的效果。

模板緩衝中的模板值（stencil value）通常是8位的，因此每個片段（像素）共有256種不同的模板值（譯註：8位就是1字節大小，因此和char的容量一樣是256個不同值）。這樣我們就能將這些模板值設置為我們鏈接的，然後在模板測試時根據這個模板值，我們就可以決定丟棄或保留它了。

!!! Important

    每個窗口庫都需要為你設置模板緩衝。GLFW自動做了這件事，所以你不必告訴GLFW去創建它，但是其他庫可能沒默認創建模板庫，所以一定要查看你使用的庫的文檔。

下面是一個模板緩衝的簡單例子：

![image description](http://learnopengl.com/img/advanced/stencil_buffer.png)

模板緩衝先清空模板緩衝設置所有片段的模板值為0，然後開啟矩形片段用1填充。場景中的模板值為1的那些片段才會被渲染（其他的都被丟棄）。

無論我們在渲染哪裡的片段，模板緩衝操作都允許我們把模板緩衝設置為一個特定值。改變模板緩衝的內容實際上就是對模板緩衝進行寫入。在同一次（或接下來的）渲染迭代我們可以讀取這些值來決定丟棄還是保留這些片段。當使用模板緩衝的時候，你可以隨心所欲，但是需要遵守下面的原則：

* 開啟模板緩衝寫入。
* 渲染物體，更新模板緩衝。
* 關閉模板緩衝寫入。
* 渲染（其他）物體，這次基於模板緩衝內容丟棄特定片段。

使用模板緩衝我們可以基於場景中已經繪製的片段，來決定是否丟棄特定的片段。

你可以開啟`GL_STENCIL_TEST`來開啟模板測試。接著所有渲染函數調用都會以這樣或那樣的方式影響到模板緩衝。

```c
glEnable(GL_STENCIL_TEST);
```
要注意的是，像顏色和深度緩衝一樣，在每次循環，你也得清空模板緩衝。

```c
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
```

同時，和深度測試的`glDepthMask`函數一樣，模板緩衝也有一個相似函數。`glStencilMask`允許我們給模板值設置一個**位遮罩（bitmask）**，它與模板值進行按位與（and）運算決定緩衝是否可寫。默認設置的位遮罩都是1，這樣就不會影響輸出，但是如果我們設置為0x00，所有寫入深度緩衝最後都是0。這和深度緩衝的`glDepthMask(GL_FALSE)`很類似：

```c++

// 0xFF == 0b11111111
//此時，模板值與它進行按位與運算結果是模板值，模板緩衝可寫
glStencilMask(0xFF); 

// 0x00 == 0b00000000 == 0
//此時，模板值與它進行按位與運算結果是0，模板緩衝不可寫
glStencilMask(0x00); 
```

大多數情況你的模板遮罩（stencil mask）寫為0x00或0xFF就行，但是最好知道有一個選項可以自定義位遮罩。

## 模板函數（stencil functions）

和深度測試一樣，我們也有幾個不同控制權，決定何時模板測試通過或失敗以及它怎樣影響模板緩衝。一共有兩種函數可供我們使用去配置模板測試：`glStencilFunc`和`glStencilOp`。

`void glStencilFunc(GLenum func, GLint ref, GLuint mask)`函數有三個參數：

* **func**：設置模板測試操作。這個測試操作應用到已經儲存的模板值和`glStencilFunc`的`ref`值上，可用的選項是：`GL_NEVER`、`GL_LEQUAL`、`GL_GREATER`、`GL_GEQUAL`、`GL_EQUAL`、`GL_NOTEQUAL`、`GL_ALWAYS`。它們的語義和深度緩衝的相似。
* **ref**：指定模板測試的引用值。模板緩衝的內容會與這個值對比。
* **mask**：指定一個遮罩，在模板測試對比引用值和儲存的模板值前，對它們進行按位與（and）操作，初始設置為1。

在上面簡單模板的例子裡，方程應該設置為：

```c
glStencilFunc(GL_EQUAL, 1, 0xFF)
```

它會告訴OpenGL，無論何時，一個片段模板值等於(`GL_EQUAL`)引用值`1`，片段就能通過測試被繪製了，否則就會被丟棄。

但是`glStencilFunc`只描述了OpenGL對模板緩衝做什麼，而不是描述我們如何更新緩衝。這就需要`glStencilOp`登場了。

`void glStencilOp(GLenum sfail, GLenum dpfail, GLenum dppass)`函數包含三個選項，我們可以指定每個選項的動作：

* **sfail**：  如果模板測試失敗將採取的動作。
* **dpfail**： 如果模板測試通過，但是深度測試失敗時採取的動作。
* **dppass**： 如果深度測試和模板測試都通過，將採取的動作。

每個選項都可以使用下列任何一個動作。

操作 | 描述
  ---|---
GL_KEEP     | 保持現有的模板值
GL_ZERO	    | 將模板值置為0
GL_REPLACE  | 將模板值設置為用`glStencilFunc`函數設置的**ref**值
GL_INCR	    | 如果模板值不是最大值就將模板值+1
GL_INCR_WRAP| 與`GL_INCR`一樣將模板值+1，如果模板值已經是最大值則設為0
GL_DECR	    | 如果模板值不是最小值就將模板值-1
GL_DECR_WRAP| 與`GL_DECR`一樣將模板值-1，如果模板值已經是最小值則設為最大值
GL_INVERT   | Bitwise inverts the current stencil buffer value.

`glStencilOp`函數默認設置為 (GL_KEEP, GL_KEEP, GL_KEEP) ，所以任何測試的任何結果，模板緩衝都會保留它的值。默認行為不會更新模板緩衝，所以如果你想寫入模板緩衝的話，你必須像任意選項指定至少一個不同的動作。

使用`glStencilFunc`和`glStencilOp`，我們就可以指定在什麼時候以及我們打算怎麼樣去更新模板緩衝了，我們也可以指定何時讓測試通過或不通過。什麼時候片段會被拋棄。

## 物體輪廓

看了前面的部分你未必能理解模板測試是如何工作的，所以我們會展示一個用模板測試實現的一個特別的和有用的功能，叫做物體輪廓（object outlining）。

![](http://learnopengl.com/img/advanced/stencil_object_outlining.png)

物體輪廓就像它的名字所描述的那樣，它能夠給每個（或一個）物體創建一個有顏色的邊。在策略遊戲中當你打算選擇一個單位的時候它特別有用。給物體加上輪廓的步驟如下：

1. 在繪製物體前，把模板方程設置為`GL_ALWAYS`，用1更新物體將被渲染的片段。
2. 渲染物體，寫入模板緩衝。
3. 關閉模板寫入和深度測試。
4. 每個物體放大一點點。
5. 使用一個不同的片段著色器用來輸出一個純顏色。
6. 再次繪製物體，但只是當它們的片段的模板值不為1時才進行。
7. 開啟模板寫入和深度測試。

這個過程將每個物體的片段模板緩衝設置為1，當我們繪製邊框的時候，我們基本上繪製的是放大版本的物體的通過測試的地方，放大的版本繪製後物體就會有一個邊。我們基本會使用模板緩衝丟棄所有的不是原來物體的片段的放大的版本內容。

我們先來創建一個非常基本的片段著色器，它輸出一個邊框顏色。我們簡單地設置一個固定的顏色值，把這個著色器命名為shaderSingleColor：

```c++
void main()
{
    outColor = vec4(0.04, 0.28, 0.26, 1.0);
}
```

我們只打算給兩個箱子加上邊框，所以我們不會對地面做什麼。這樣我們要先繪製地面，然後再繪製兩個箱子（同時寫入模板緩衝），接著我們繪製放大的箱子（同時丟棄前面已經繪製的箱子的那部分片段）。

我們先開啟模板測試，設置模板、深度測試通過或失敗時才採取動作：

```c++
glEnable(GL_DEPTH_TEST);
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);
```

如果任何測試失敗我們都什麼也不做，我們簡單地保持深度緩衝中當前所儲存著的值。如果模板測試和深度測試都成功了，我們就將儲存著的模板值替換為`1`，我們要用`glStencilFunc`來做這件事。

我們清空模板緩衝為0，為箱子的所有繪製的片段的模板緩衝更新為1：

```c++
glStencilFunc(GL_ALWAYS, 1, 0xFF); //所有片段都要寫入模板緩衝
glStencilMask(0xFF); // 設置模板緩衝為可寫狀態
normalShader.Use();
DrawTwoContainers();
```

使用`GL_ALWAYS`模板測試函數，我們確保箱子的每個片段用模板值1更新模板緩衝。因為片段總會通過模板測試，在我們繪製它們的地方，模板緩衝用引用值更新。

現在箱子繪製之處，模板緩衝更新為1了，我們將要繪製放大的箱子，但是這次關閉模板緩衝的寫入：

```c++
glStencilFunc(GL_NOTEQUAL, 1, 0xFF);
glStencilMask(0x00); // 禁止修改模板緩衝
glDisable(GL_DEPTH_TEST);
shaderSingleColor.Use();
DrawTwoScaledUpContainers();
```

我們把模板方程設置為`GL_NOTEQUAL`，它保證我們只箱子上不等於1的部分，這樣只繪製前面繪製的箱子外圍的那部分。注意，我們也要關閉深度測試，這樣放大的的箱子也就是邊框才不會被地面覆蓋。

做完之後還要保證再次開啟深度緩衝。

場景中的物體邊框的繪製方法最後看起來像這樣：

```c++
glEnable(GL_DEPTH_TEST);
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);  

glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);

glStencilMask(0x00); // 繪製地板時確保關閉模板緩衝的寫入
normalShader.Use();
DrawFloor()  

glStencilFunc(GL_ALWAYS, 1, 0xFF);
glStencilMask(0xFF);
DrawTwoContainers();

glStencilFunc(GL_NOTEQUAL, 1, 0xFF);
glStencilMask(0x00);
glDisable(GL_DEPTH_TEST);
shaderSingleColor.Use();
DrawTwoScaledUpContainers();
glStencilMask(0xFF);
glEnable(GL_DEPTH_TEST);
```

理解這段代碼後面的模板測試的思路並不難以理解。如果還不明白嘗試再仔細閱讀上面的部分，嘗試理解每個函數的作用，現在你已經看到了它的使用方法的例子。

這個邊框的算法的結果在深度測試教程的那個場景中，看起來像這樣：

![](http://bullteacher.com/wp-content/uploads/2015/06/stencil_scene_outlined.png)

在這裡[查看源碼](http://learnopengl.com/code_viewer.php?code=advanced/stencil_testing)和[著色器](http://learnopengl.com/code_viewer.php?code=advanced/depth_testing_func_shaders)，看看完整的物體邊框算法是怎樣的。

!!! Important

        你可以看到兩個箱子邊框重合通常正是我們希望得到的（想想策略遊戲中，我們打算選擇10個單位；我們通常會希望把邊界合併）。如果你想要讓每個物體都有自己的邊界那麼你需要為每個物體清空模板緩衝，創造性地使用深度緩衝。

你目前看到的物體邊框算法在一些遊戲中顯示備選物體（想象策略遊戲）非常常用，這樣的算法可以在一個模型類中輕易實現。你可以簡單地在模型類設置一個布爾類型的標識來決定是否繪製邊框。如果你想要更多的創造性，你可以使用後處理（post-processing）過濾比如高斯模糊來使邊框看起來更自然。

除了物體邊框以外，模板測試還有很多其他的應用目的，比如在後視鏡中繪製紋理，這樣它會很好的適合鏡子的形狀，比如使用一種叫做shadow volumes的模板緩衝技術渲染實時陰影。模板緩衝在我們的已擴展的OpenGL工具箱中給我們提供了另一種好用工具。
