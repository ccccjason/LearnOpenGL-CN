本文作者JoeyDeVries，由[Django](http://bullteacher.com/4-hello-window.html)翻譯自[http://learnopengl.com](http://www.learnopengl.com/#!Getting-started/Hello-Window)

# Hello Window

我們看看GLFW是否能夠運行。首先，創建一個.cpp文件，在新創建的文件的頂部包含下面的頭文件。注意，我們定義了`GLEW_STATIC`，這是因為我們將使用靜態GLEW庫。

```c++
// GLEW
#define GLEW_STATIC
#include <GL/glew.h>
// GLFW
#include <GLFW/glfw3.h>
```

<div style="border:solid #E1B3B3;border-radius:10px;background-color:#FFD2D2;margin:10px 10px 10px 0px;padding:10px">
必須在GLFW之前引入GLEW。GLEW的頭文件已經包含了OpenGL的頭文件（`GL/gl.h`），所以要在其他頭文件之前引入GLEW，因為它們需要有OpenGL才能起作用。
</div>

下面，我們創建main函數，在main函數中我們會實例化一個GLFW窗口：

```c++
int main()
{
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    glfwWindowHint(GLFW_RESIZABLE, GL_FALSE);

    return 0;
}
```

在main函數中我們首先使用`glfwInit`來初始化GLFW，然後我們可以使用`glfwWindowHint`來配置GLFW。`glfwWindowHint`的第一個參數告訴我們，我們打算配置哪個選項，這裡我們可以從一個枚舉中選擇可用的選項，這些選項帶有`GLFW_`前綴。第二個參數是一個整數，它代表我們為選項所設置的值。可用的選項和對應的值可以在[GLFW窗口管理文檔](http://www.glfw.org/docs/latest/window.html#window_hints "GLFW窗口管理文檔")中列出。現在，如果你去運行應用報出了很多未定義引用錯誤，這意味著你還沒有成功的鏈接GLFW庫。

由於這個教程關注的是OpenGL 3.3版本，我們會告訴GLFW我們使用的OpenGL版本是3.3。這樣GLFW在創建**OpenGL環境**的時候可以做出合理的安排。這會保證如果一個用戶沒有特定的OpenGL版本，GLFW就會運行失敗。我們把主和次版本都設置為3。我們同樣告訴GLFW，我們希望明確地使用**核心模式(Core Profile)**，同時用戶不可以調整窗口大小。顯式地告訴GLFW我們希望是用核心模式會導致當我們調用一個OpenGL的遺留函數會產生**非法操作(Invalid Operation)**錯誤，當我們意外地使用了不該使用的舊函數時它是一個很好的提醒。注意在Mac OS X上在初始化代碼裡你還需要添加`glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);`才能工作。

<div style="border:solid #AFDFAF;border-radius:10px;background-color:#D8F5D8;margin:10px 10px 10px 0px;padding:10px">
你需要確定你的系統/硬件上已經安裝了OpenGL3.3或更高的版本，否則應用會崩潰或產生不可預測的結果。在Linux上找到你的機器上OpenGL的版本可以調用`glxinfo`，Windows機器上要使用[OpenGL Extension Viewer](http://download.cnet.com/OpenGL-Extensions-Viewer/3000-18487_4-34442.html "OpenGL Extension Viewer")這樣的工具。如果支持的版本太低了，檢查一下的你的顯卡是否支持OpenGL3.3+（如果不支持那就太老了），先去更新你的驅動。
</div>

下面我們需要創建一個窗口對象。這個窗口對象帶有所有的窗口數據，它們通常是GLFW的其他函數經常使用的。

```c++
GLFWwindow* window = glfwCreateWindow(800, 600, "LearnOpenGL", nullptr, nullptr);
if (window == nullptr)
{
    std::cout << "Failed to create GLFW window" << std::endl;
    glfwTerminate();
    return -1;
}
glfwMakeContextCurrent(window);
```
函數`glfwCreateWindow`需要窗口的**寬度**和**高度**作為它前兩個參數。第三個參數允許我們給窗口創建一個名字；現在我們把它命名為`"LearnOpenGL"`但是你也可以取個自己喜歡的名字。我們**可以忽略最後兩個參數**。這個函數返回一個`GLFWwindow`對象，在後面的其他GLFW操作會需要它。之後，我們告訴GLFW去創建我們窗口的環境（`glfwMakeContextCurrent`），這個環境是當前線程的主環境。

## GLEW

前面的教程裡，我們提到GLEW管理著OpenGL的函數指針，所以我們希望在調用任何OpenGL函數前初始化GLEW。

```c++
glewExperimental = GL_TRUE;
if (glewInit() != GLEW_OK)
{
    std::cout << "Failed to initialize GLEW" << std::endl;
    return -1;
}
```

注意，在初始化GLEW前我們把`glewExperimental`變量設置為`GL_TRUE`。設置`glewExperimental`為true可以保證GLEW使用更多的現代技術來管理OpenGL功能。如果不這麼設置，它就會使用默認的`GL_FALSE`，這樣當使用核心模式時有可能發生問題。

## 視口(Viewport)

在我們開始渲染前，我們必須做這最後一件事。我們必須告訴OpenGL渲染窗口的大小，這樣OpenGL才能知道我們希望如何設置窗口的大小和位置。我們可以通過`glViewport`函數設置這些尺寸：

```c++
glViewport(0, 0, 800, 600);
```

前兩個參數設置了窗口**左下角的位置**。第三個和第四個參數是這個渲染窗口的**寬度**和**高度**，它和GLFW窗口是一樣大的。我們可以把這個值設置得比GLFW窗口尺寸小；這樣OpenGL的渲染都會在一個更小的窗口（區域）進行顯示，我們可以在OpenGL的視區之外顯示其他的元素。

<div style="border:solid #AFDFAF;border-radius:10px;background-color:#D8F5D8;margin:10px 10px 10px 0px;padding:10px">
在幕後OpenGL是使用通過`glViewport`指定的數據將2D座標加工為屏幕上的座標。比如，一個被加工的點的位置是(-0.5, 0.5)會（作為它最後的變換）被映射到屏幕座標(200, 450)上。注意，OpenGL中處理的座標是在-1和1之間，所以我們事實上是把（-1到1）的範圍映射到(0, 800)和(0, 600)上了。
</div>

## 準備好你的引擎

我們不希望應用繪製了一個圖像之後立即退出，然後關閉窗口。我們想讓應用持續地繪製圖像，監聽用戶輸入直到軟件被明確告知停止。為了達到這個目的，我們必須創建一個while循環，我們稱其為遊戲循環（Game Loop），這樣，在我們告訴GLFW停止之前應用就會一直保持運行狀態。下面的代碼展示了一個非常簡單的遊戲循環。

```c++
while(!glfwWindowShouldClose(window))
{
    glfwPollEvents();
    glfwSwapBuffers(window);
}
```

`glfwWindowShouldClose`函數從開始便檢驗每一次循環迭代中gLFW是否已經得到關閉指示，如果得到這樣的指示，函數就會返回true，並且遊戲循環停止運行，之後我們就可以關閉應用了。

`glfwPollEvents`函數檢驗是否有任何事件被處觸發（比如鍵盤輸入或是鼠標移動的事件），接著調用相應函數（我們可以通過回調方法設置它們）。我們經常在循環迭代前調用事件處理函數。

`glfwSwapBuffers`函數會交換**顏色緩衝**（顏色緩衝是一個GLFW窗口為每一個像素儲存顏色數值的大緩衝），它是在這次迭代中繪製的，也作為輸出顯示在屏幕上。

<div style="border:solid #AFDFAF;border-radius:10px;background-color:#D8F5D8;margin:10px 10px 10px 0px;padding:10px">
**雙緩衝（Double buffer）**

當一個應用以單緩衝方式繪製的時候，圖像會產生閃縮的問題。這是因為最後的圖像輸出不是被立即繪製出來的，而是一個像素一個像素繪製出來的，通常是以從左到右從上到下這樣的方式。由於這些圖像不是立即呈現在用戶面前，而是一步一步地生成結果，這就產生很多不真實感。為了規避這些問題，窗口應用使用雙緩衝的方式進行渲染。**前緩衝**包含最終的輸出圖像，它被顯示在屏幕上，與此同時，所有的渲染命令繪製**後緩衝**。所有的渲染命令執行結束，我們就把後緩衝**交換**到前緩衝，這樣圖像就會立即顯示到用戶面前了，前面提到的不真實感就這樣被解決了。
</div>

## 最後一件事

退出遊戲循環後，我們就可以合理地清理/釋放之前分配的所有資源了。我們可以在`main`函數結尾使用`glfwTerminate`函數來做這件事。

```c++
glfwTerminate();
return 0;
```

這樣會清理所有資源，並正確地退出應用。現在嘗試編譯你的應用，如果所有事情都工作的很好你就會看到下面的結果：

![](http://www.learnopengl.com/img/getting-started/hellowindow.png)

如果出現一個枯燥的黑色圖像，你就做對了！如果你沒有得到這個圖像，或者不知道如何把所有東西組合起來，可以看我們的[完整源碼](http://www.learnopengl.com/code_viewer.php?code=getting-started/hellowindow "完整源碼")。

如果你在編譯應用的時候出現狀況，首先要確保鏈接器的所有配置都正確的做好了，在你的IDE中正確地包含了路徑（前面的教程裡講過）。同時你要確保代碼也是對的；你可以通過看源代碼簡單地去驗證一下。如果仍然報錯，在下面提交一個評論，寫上你的問題，我或者其他人會嘗試幫助你。

## 輸入(Input)

我們同樣希望在GLFW中有些控制輸入的方式，我們可以使用GLFW的回調函數(Callback functions)做到這點。**回調函數**簡單來說就是一個你可以設置的，從而使得GLFW能夠在合適的時刻調用的函數指針。其中有一個我們可以設置的回調函數是**KeyCallback**函數，它在用戶使用鍵盤交互的時候被調用。函數的原型是：

```c++
void key_callback(GLFWwindow* window, int key, int scancode, int action, int mode);
```

按鍵輸入函數的接收一個`GLFWwindow`參數，一個代表按下按鍵的整型數字，一個特定動作，按鈕是被按下、還是釋放，一個代表某個標識的整數告訴你`shift`、`control`、`alt`或`super`是否被同時按下。每當一個用戶按下一個按鈕，GLFW都會調用這個函數，為你的這個函數填充合適的參數。

```c++
void key_callback(GLFWwindow * window, int key, int scancode, int action, int mode)
{
    // 當用戶按下ESC, 我們就把WindowShouldClose設置為true, 關閉應用
    if(key == GLFW_KEY_ESCAPE && action == GLFW_PRESS)
        glfwSetWindowShouldClose(window, GL_TRUE);
}
```

在我們（新創建）的`key_callback`函數中，我們檢查被按下的按鍵是否等於`ESC`健，如果是這個按鍵被按下（不是釋放）的話，我們就用`glfwSetWindowShouldClose`設置它的`WindowShouldClose`屬性為`true`來關閉GLFW。下一個主while循環條件檢驗會失敗，應用就關閉了。

最後一件事就是通過GLFW使用合適的回調註冊函數。可以這麼做：

```c++
glfwSetKeyCallback(window, key_callback);
```

有許多回調函數可供用於註冊我們自己的函數。比如，我們可以做一個回調函數處理窗口尺寸的改變，處理錯誤信息等等。我們要在創建窗口之後，在遊戲循環初始化之前註冊回調函數。  

## 渲染(Rendering)

我們想把所有渲染命令都放在遊戲循環裡，因為我們打算在每個循環迭代裡都執行所有的渲染命令。這看起來有點像這樣：
```c++
// 程序循環
while(!glfwWindowShouldClose(window))
{
    // 檢查及調用事件
    glfwPollEvents();

    // 渲染指令放在這裡
    ...

    // 交換緩衝
    glfwSwapBuffers(window);
}
```

我們用一種自己選擇的顏色來清空屏幕，測試一下是否能夠正常工作。每個渲染迭代的開始，我們都要清理屏幕，否則只能一直看到前一個迭代的結果（這可能就是你想要的效果，但通常你不會這麼想）。我們可以使用`glClear`函數清理屏幕的顏色緩衝，在這個函數中我們以緩衝位（`BUFFER_BIT`）指定我們希望清理哪個緩衝。可用的位可以是`GL_COLOR_BUFFER_BIT`、`GL_DEPTH_BUFFER_BIT`和`GL_STENCIL_BUFFER_BIT`(譯註：緩衝是顯存上的一段空間，用來儲存多種數據。)。現在，我們關心的只是顏色值，所以我們只清空顏色緩衝。

```c++
glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT);
```

要注意，我們使用了`glClearColor`來設置了清空屏幕用的顏色。當調用`glClear`去清空顏色緩衝時，全部顏色緩衝都將被`glClearColor`所配置的顏色填充。本例會填充為暗藍綠(Dark Green-blueish)色。

<div style="border:solid #AFDFAF;border-radius:10px;background-color:#D8F5D8;margin:10px 10px 10px 0px;padding:10px">
你可能會回想起OpenGL教程，`glClearColor`函數是一個狀態設置函數，`glClear`是一個狀態使用函數，在這個函數裡，它從當前狀態獲取清空所用的顏色。
</div>

![](http://www.learnopengl.com/img/getting-started/hellowindow2.png)

這個應用的完整代碼可以在[這裡](http://www.learnopengl.com/code_viewer.php?code=getting-started/hellowindow2 "這裡")找到。

現在我們已經準備好把遊戲循環用大量渲染函數填滿了，但是這是下節要做的事。我認為我們已經在這裡講地夠久的了。