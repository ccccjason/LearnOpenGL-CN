# 你好，窗口

//TODO 注意仔細檢查翻譯，錯誤有點多，可能還要繼續校對

原文     | [Hello Window](http://learnopengl.com/#!Getting-started/Hello-Window)
      ---|---
作者     | JoeyDeVries
翻譯     | Geequlim
校對     | Geequlim


上一節中我們獲取並編譯了GLFW和GLEW這兩個開源庫，現在我們就可以使用它們來創建一個OpenGL繪圖窗口了。首先，新建一個`.cpp`文件，然後把下面的代碼粘貼到該文件的最前面。注意，之所以定義`GLEW_STATIC`宏，是因為我們使用GLEW的靜態鏈接庫。

```c++
// GLEW
#define GLEW_STATIC
#include <GL/glew.h>
// GLFW
#include <GLFW/glfw3.h>
```

!!! Attention

	請確認在包含GLFW的頭文件之前包含了GLEW的頭文件。在包含glew.h頭文件時會引入許多OpenGL必要的頭文件(例如GL/gl.h)，所以#include <GL/glew.h>應放在引入其他頭文件的代碼之前。

接下來我們創建`main`函數，並做一些初始化GLFW的操作：

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

首先我們在`main`函數中調用`glfwInit`函數來初始化GLFW，然後我們可以使用`glfwWindowHint`函數來配置GLFW。`glfwWindowHint`函數的第一個參數表示我們要進行什麼樣的配置，我們可以選擇大量以`GLFW_`開頭的枚舉值；第二個參數接受一個整形，用來設置這個配置的值。該函數的所有的選項以及對應的值都可以在 [GLFW's window handling](http://www.glfw.org/docs/latest/window.html#window_hints) 這篇文檔中找到。如果你現在編譯你的cpp文件會得到大量的連接錯誤，這是因為你還需要進一步設置GLFW。

由於本站的教程都是基於OpenGL3.3以後的版本展開討論的，所以我們需要告訴GLFW我們要使用的OpenGL版本是3.3，這樣GLFW會在創建OpenGL上下文時做出適當的調整。這也可以確保用戶在沒有適當的OpenGL版本支持的情況下無法運行。在這裡我們告訴GLFW想要的OpenGL版本號是3.3，並且不允許用戶調整窗口的大小。我們明確地告訴GLFW我們想要使用核心模式(Core-profile)，這將導致我們無法使用那些已經廢棄的API，而這不正是一個很好的提醒嗎？當我們不小心用了舊功能時報錯，就能避免使用一些被廢棄的用法了。如果你使用的是Mac OSX系統你還需要加下面這行代碼這些配置才能起作用：

```c++
glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
```

!!! Important

	請確認您的系統支持OpenGL3.3或更高版本，否則此應用有可能會崩潰或者出現不可預知的錯誤。可以通過運行glew附帶的glxinfo程序或者其他的工具(例如[OpenGL Extension Viewer](http://download.cnet.com/OpenGL-Extensions-Viewer/3000-18487_4-34442.html)來查看你的OpenGL版本。如果你的OpenGL版本低於3.3請更新你的驅動程序或者有必要的話更新設備。

接下來我們創建一個窗口對象，這個窗口對象中具有和窗口相關的許多數據，而且會被GLFW的其他函數頻繁地用到。

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

`glfwCreateWindow`函數需要窗口的寬和高作為它的前兩個參數；第三個參數表示只這個窗口的名稱(標題)，這裡我們使用**"LearnOpenGL"**，當然你也可以使用你喜歡的名稱；最後兩個參數我們暫時忽略，先置為空指針就行。它的返回值`GLFWwindow`對象的指針會在其他的GLFW操作中使用到。創建完窗口我們就可以通知GLFW給我們的窗口在當前的線程中創建我們等待已久的OpenGL上下文了。

### GLEW

在之前的教程中已經提到過，GLEW是用來管理OpenGL的函數指針的，所以在調用任何OpenGL的函數之前我們需要初始化GLEW。

```c++
glewExperimental = GL_TRUE;
if (glewInit() != GLEW_OK)
{
    std::cout << "Failed to initialize GLEW" << std::endl;
    return -1;
}
```

請注意，我們在初始化GLEW之前設置`glewExperimental`變量的值為`GL_TRUE`，這樣做能讓GLEW在管理OpenGL的函數指針時更多地使用現代化的技術，如果把它設置為`GL_FALSE`的話可能會在使用OpenGL的核心模式(Core-profile)時出現一些問題。

### 視口(Viewport)

在我們繪製之前還有一件重要的事情要做，我們必須告訴OpenGL渲染窗口的尺寸大小，這樣OpenGL才只能知道要顯示數據的窗口座標。我們可以通過調用`glViewport`函數來設置這些維度：

```c++
glViewport(0, 0, 800, 600);  
```

前兩個參數設置窗口左下角的位置。第三個和第四個參數設置渲染窗口的寬度和高度,我們設置成與GLFW的窗口的寬高大小，我們也可以將這個值設置成比窗口小的數值，然後所有的OpenGL渲染將會顯示在一個較小的區域。

!!!Important

	OpenGL使用`glViewport`定義的位置和寬高進行位置座標的轉換，將OpenGL中的位置座標轉換為你的屏幕座標。例如，OpenGL中的座標(0.5,0.5)有可能被轉換為屏幕中的座標(200,450)。注意，OpenGL只會把-1到1之間的座標轉換為屏幕座標，因此在此例中(-1，1)轉換為屏幕座標是(0,600)。

## 準備好你的引擎

我們可不希望只繪製一個圖像之後我們的應用程序就關閉窗口並立即退出。我們希望程序在我們明確地關閉它之前一直保持運行狀態並能夠接受用戶輸入。因此，我們需要在程序中添加一個while循環，我們可以把它稱之為遊戲循環(Game Loop)，這樣我們的程序就能在我們讓GLFW退出前保持運行了。下面幾行的代碼就實現了一個簡單的遊戲循環：

```c++
while(!glfwWindowShouldClose(window))
{
    glfwPollEvents();
    glfwSwapBuffers(window);
}
```

- `glfwWindowShouldClose`函數在我們每次循環的開始前檢查一次GLFW是否準備好要退出，如果是這樣的話該函數返回true然後遊戲循環便結束了，之後為我們就可以關閉應用程序了。
- `glfwPollEvents`函數檢查有沒有觸發什麼事件(比如鍵盤有按鈕按下、鼠標移動等)然後調用對應的回調函數(我們可以手動設置這些回調函數)。我們一般在遊戲循環的一開始就檢查事件。
- 調用`glfwSwapBuffers`會交換緩衝區(儲存著GLFW窗口每一個像素顏色的緩衝區)


!!! Important

	**雙緩衝區(Double buffer)**

	應用程序使用單緩衝區繪圖可能會存在圖像閃爍的問題。 這是因為生成的圖像不是一下子被繪製出來的，而是按照從左到右，由上而下逐像素地繪製而成的。最終圖像不是在瞬間顯示給用戶,而是通過一步一步地計算結果繪製的，這可能會花費一些時間。為了規避這些問題，我們應用雙緩衝區渲染窗口應用程序。前面的緩衝區保存著計算後可顯示給用戶的圖像，被顯示到屏幕上；所有的的渲染命令被傳遞到後臺的緩衝區進行計算。當所有的渲染命令執行結束後，我們交換前臺緩衝和後臺緩衝，這樣圖像就立即呈顯出來，之後清空緩衝區。

### 最後一件事

當遊戲循環結束後我們需要釋放之前的操作分配的資源，我們可以在main函數的最後調用`glfwTerminate`函數來釋放GLFW分配的內存。

```c++
glfwTerminate();
return 0;
```

這樣便能清空GLFW分配的內存然後正確地退出應用程序。現在你可以嘗試編譯並運行你的應用程序了，你將會看到如下的一個黑色窗口：

![](http://learnopengl.com/img/getting-started/hellowindow.png)

如果你沒有編譯通過或者有什麼問題的話，首先請檢查你程序的的鏈接選項是否正確
。然後對比本教程的代碼，檢查你的代碼是不是哪裡寫錯了，你也可以[點擊這裡](http://learnopengl.com/code_viewer.php?code=getting-started/hellowindow)獲取我的完整代碼。

### 輸入

我們同樣也希望能夠在GLFW中實現一些鍵盤控制，這是通過設置GLFW的**回調函數(Callback Function)**來實現的。回調函數事實上是一個函數指針，當我們為GLFW設置回調函數後，GLWF會在恰當的時候調用它。**按鍵回調(KeyCallback)**是眾多回調函數中的一種，當我們為GLFW設置按鍵回調之後，GLFW會在用戶有鍵盤交互時調用它。該回調函數的原型如下所示：

```c++
void key_callback(GLFWwindow* window, int key, int scancode, int action, int mode);
```

按鍵回調函數接受一個`GLFWwindow`指針作為它的第一個參數；第二個整形參數用來表示事件的按鍵；第三個整形參數描述用戶是否有第二個鍵按下或釋放；第四個整形參數表示事件類型，如按下或釋放；最後一個參數是表示是否有Ctrl、Shift、Alt、Super等按鈕的操作。GLFW會在恰當的時候調用它，併為各個參數傳入適當的值。


```c++
void key_callback(GLFWwindow* window, int key, int scancode, int action, int mode)
{
    // 當用戶按下ESC鍵,我們設置window窗口的WindowShouldClose屬性為true
    // 關閉應用程序
    if(key == GLFW_KEY_ESCAPE && action == GLFW_PRESS)
    	glfwSetWindowShouldClose(window, GL_TRUE);
}    
```

在這個`key_callback`函數中，它檢測鍵盤是否按下了Escape鍵。如果鍵的確按下了(不釋放)，我們使用`glfwSetwindowShouldClose`函數設定`WindowShouldClose`屬性為true從而關閉GLFW。main函數的while循環下一次的檢測將失敗並且程序關閉。

最後一件事就是通過GLFW使用適合的回調來註冊我們的函數，代碼是這樣的:

```c++
glfwSetKeyCallback(window, key_callback);  
```

除了按鍵回調函數之外，我們還能為GLFW註冊其他的回調函數。例如，我們可以註冊一個函數來處理窗口尺寸變化、處理一些錯誤信息等。我們可以在創建窗口之後到開始遊戲循環之前註冊各種回調函數。


### 渲染(Rendering)

我們要把所有的渲染操作放到遊戲循環中，因為我們想讓這些渲染操作在每次遊戲循環迭代的時候都能被執行。我們將做如下的操作：

```c++
// 程序循環
while(!glfwWindowShouldClose(window))
{
    // 檢查事件
    glfwPollEvents();

    // 在這裡執行各種渲染操作
    ...

    //交換緩衝區
    glfwSwapBuffers(window);
}
```

為了測試一切都正常，我們想讓屏幕清空為一種我們選擇的顏色。在每次執行新的渲染之前我們都希望清除上一次循環的渲染結果，除非我們想要看到上一次的結果。我們可以通過調用`glClear`函數來清空屏幕緩衝區的顏色，他接受一個整形常量參數來指定要清空的緩衝區，這個常量可以是`GL_COLOR_BUFFER_BIT`，`GL_DEPTH_BUFFER_BIT`和`GL_STENCIL_BUFFER_BIT`。由於現在我們只關心顏色值，所以我們只清空顏色緩衝區。

```c++
glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT);
```
注意，除了`glClear`之外，我們還要調用`glClearColor`來設置要清空緩衝的顏色。當調用`glClear`函數之後，整個指定清空的緩衝區都被填充為`glClearColor`所設置的顏色。在這裡，我們將屏幕設置為了類似黑板的深藍綠色。

!!! Important

	你應該能夠想起來我們在[OpenGL](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/01%20OpenGL/)教程的內容，`glClearColor`函數是一個狀態設置函數，而`glClear`函數則是一個狀態應用的函數。

![](http://learnopengl.com/img/getting-started/hellowindow2.png)

此程序的完整源代碼可以在[這裡](http://learnopengl.com/code_viewer.php?code=getting-started/hellowindow2)找到。

好了，到目前為止我們已經做好開始在遊戲循環中添加許多渲染操作的準備了，我認為我們的閒扯已經夠多了，從下一篇教程開始我們將真正的征程。