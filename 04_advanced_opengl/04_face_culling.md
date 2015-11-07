# 面剔除（Face culling）

原文     | [Face culling](http://learnopengl.com/#!Advanced-OpenGL/Face-culling)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

嘗試在頭腦中想象一下有一個3D立方體，你從任何一個方向去看它，最多可以同時看到多少個面。如果你的想象力不是過於豐富，你最終最多能數出來的面是3個。你可以從一個立方體的任意位置和方向上去看它，但是你永遠不能看到多於3個面。所以我們為何還要去繪製那三個不會顯示出來的3個面呢。如果我們可以以某種方式丟棄它們，我們會提高片段著色器超過50%的性能！

!!! Important

        我們所說的是超過50%而不是50%，因為從一個角度只有2個或1個面能夠被看到。這種情況下我們就能夠提高50%以上性能了。


這的確是個好主意，但是有個問題需要解決：我們如何知道某個面在觀察者的視野中不會出現呢？如果我們去想象任何封閉的幾何平面，它們都有兩面，一面面向用戶，另一面背對用戶。假如我們只渲染面向觀察者的面會怎樣？

這正是**面剔除**(Face culling)所要做的。OpenGL允許檢查所有正面朝向（Front facing）觀察者的面，並渲染它們，而丟棄所有背面朝向（Back facing）的面，這樣就節約了我們很多片段著色器的命令（它們很昂貴！）。我們必須告訴OpenGL我們使用的哪個面是正面，哪個面是反面。OpenGL使用一種聰明的手段解決這個問題——分析頂點數據的連接順序（Winding order）。


## 頂點連接順序（Winding order）

當我們定義一系列的三角頂點時，我們會把它們定義為一個特定的連接順序，它們可能是順時針的或逆時針的。每個三角形由3個頂點組成，我們從三角形的中間去看，從而把這三個頂點指定一個連接順序。

![](http://learnopengl.com/img/advanced/faceculling_windingorder.png)

正如你所看到的那樣，我們先定義了頂點1，接著我們定義頂點2或3，這個不同的選擇決定了這個三角形的連接順序。下面的代碼展示出這點：

```c++
GLfloat vertices[] = {
    //順時針
    vertices[0], // vertex 1
    vertices[1], // vertex 2
    vertices[2], // vertex 3
    // 逆時針
    vertices[0], // vertex 1
    vertices[2], // vertex 3
    vertices[1] // vertex 2
};
```

每三個頂點都形成了一個包含著連接順序的基本三角形。OpenGL使用這個信息在渲染你的基本圖形的時候決定這個三角形是三角形的正面還是三角形的背面。默認情況下，**逆時針**的頂點連接順序被定義為三角形的**正面**。

當定義你的頂點順序時，你如果定義能夠看到的一個三角形，那它一定是正面朝向的，所以你定義的三角形應該是逆時針的，就像你直接面向這個三角形。把所有的頂點指定成這樣是件炫酷的事，實際的頂點連接順序是在**光柵化**階段（Rasterization stage）計算的，所以當頂點著色器已經運行後。頂點就能夠在觀察者的觀察點被看到。

我們指定了它們以後，觀察者面對的所有的三角形的頂點的連接順序都是正確的，但是現在渲染的立方體另一面的三角形的頂點的連接順序被反轉。最終，我們所面對的三角形被視為正面朝向的三角形，後部的三角形被視為背面朝向的三角形。下圖展示了這個效果：

![](http://learnopengl.com/img/advanced/faceculling_frontback.png)

在頂點數據中，我們定義的是兩個逆時針順序的三角形。然而，從觀察者的方面看，後面的三角形是順時針的，如果我們仍以1、2、3的順序以觀察者當面的視野看的話。即使我們以逆時針順序定義後面的三角形，它現在還是變為順時針。它正是我們打算剔除（丟棄）的不可見的面！



## 面剔除

在教程的開頭，我們說過OpenGL可以丟棄背面朝向的三角形。現在我們知道了如何設置頂點的連接順序，我們可以開始使用OpenGL默認關閉的面剔除選項了。

記住我們上一節所使用的立方體的定點數據不是以逆時針順序定義的。所以我更新了頂點數據，好去反應為一個逆時針鏈接順序，你可以[從這裡複製它](http://learnopengl.com/code_viewer.php?code=advanced/faceculling_vertexdata)。把所有三角的頂點都定義為逆時針是一個很好的習慣。

開啟OpenGL的`GL_CULL_FACE`選項就能開啟面剔除功能：

```c++
glEnable(GL_CULL_FACE);
```

從這兒以後，所有的不是正面朝向的面都會被丟棄（嘗試飛入立方體看看，裡面什麼面都看不見了）。目前，在渲染片段上我們節約了超過50%的性能，但記住這隻對像立方體這樣的封閉形狀有效。當我們繪製上個教程中那個草的時候，我們必須關閉面剔除，這是因為它的前、後面都必須是可見的。

OpenGL允許我們改變剔除面的類型。要是我們剔除正面而不是背面會怎樣？我們可以調用`glCullFace`來做這件事：

```c++
glCullFace(GL_BACK);
```

`glCullFace`函數有三個可用的選項：

* GL_BACK：只剔除背面。
* GL_FRONT：只剔除正面。
* GL_FRONT_AND_BACK：剔除背面和正面。

`glCullFace`的初始值是`GL_BACK`。另外，我們還可以告訴OpenGL使用順時針而不是逆時針來表示正面，這通過glFrontFace來設置：

```c++
glFrontFace(GL_CCW);
```

默認值是`GL_CCW`，它代表逆時針，`GL_CW`代表順時針順序。

我們可以做個小實驗，告訴OpenGL現在順時針代表正面：

```c++
glEnable(GL_CULL_FACE);
glCullFace(GL_BACK);
glFrontFace(GL_CW);
```

最後的結果只有背面被渲染了：

![](http://learnopengl.com/img/advanced/faceculling_reverse.png)

要注意，你可以使用默認逆時針順序剔除正面，來創建相同的效果：

```c
glEnable(GL_CULL_FACE);
glCullFace(GL_FRONT);
```

正如你所看到的那樣，面剔除是OpenGL提高效率的一個強大工具，它使應用節省運算。你必須跟蹤下來哪個物體可以使用面剔除，哪些不能。

## 練習

你可以自己重新定義一個順時針的頂點順序，然後用順時針作為正面把它渲染出來嗎：[解決方案](http://learnopengl.com/code_viewer.php?code=advanced/faceculling-exercise1)。
