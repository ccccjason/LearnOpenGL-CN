# 創建窗口

原文     | [Creating a window](http://learnopengl.com/#!Getting-started/Creating-a-window)
      ---|---
作者     | JoeyDeVries
翻譯     | gjy_1992
校對     | Geequlim

在我們畫出出色的效果之前，首先要做的就是創建一個OpenGL上下文(Context)和一個用於顯示的窗口。然而，這些操作在每個系統上都是不一樣的，OpenGL有目的的抽象(Abstract)這些操作。這意味著我們不得不自己處理創建窗口，定義OpenGL上下文以及處理用戶輸入。

幸運的是，有一些庫已經提供了我們所需的功能，其中一部分是特別針對OpenGL的。這些庫節省了我們書寫平臺相關代碼的時間，提供給我們一個窗口和上下文用來渲染。最流行的幾個庫有GLUT，SDL，SFML和GLFW。在教程裡我們將使用**GLFW**。

## GLFW

GLFW是一個專門針對OpenGL的C語言庫，它提供了一些渲染物件所需的最低限度的接口。它允許用戶創建OpenGL上下文，定義窗口參數以及處理用戶輸入。

這一節和下一節的內容是建立GLFW環境，並保證它恰當地創建窗口和OpenGL上下文。本教程會一步步從獲取，編譯，鏈接GLFW庫講起。我們使用Microsoft Visual Studio 2012 IDE，如果你用的不是它(或者只是Visual Studio的舊版本)請不要擔心，大多數IDE上的操作都是類似的。Visual Studio 2012(或其他版本)可以從微軟網站上免費下載(選擇Express版本或Community版本)。

## 構建GLFW

GLFW可以從它們網站的[下載頁](http://www.glfw.org/download.html)上獲取。GLFW已經有針對Visual Studio 2012/2013的預編譯的二進制版本和相應的頭文件，但是為了完整性我們將從編譯源代碼開始，所以需要下載**源代碼包**。


!!! Attention

	當你下載二進制版本時，請下載32位的版本而不是64位的除非你清楚你在做什麼。大部分讀者報告64位版本會出現很多奇怪的問題。


一旦下載完了源碼包，解壓到某處。我們只關心裡面的這些內容：

- 編譯生成的庫
- **include**文件夾

從源代碼編譯庫可以保證生成的目標代碼是針對你的操作系統和CPU的，而一個預編譯的二進制代碼並不保證總是適合。提供源代碼的一個問題是不是每個人都用相同的IDE來編譯，因而提供的工程文件可能和一些人的IDE不兼容。所以人們只能從.cpp和.h文件來自己建立工程，這是一項笨重的工作。因此誕生了一個叫做CMake的工具。

### CMake

CMake是一個工程文件生成工具，可以使用預定義好的CMake腳本，根據用戶的選擇生成不同IDE的工程文件。這允許我們從GLFW源碼裡創建一個Visual Studio 2012工程文件。首先，我們需要從[這裡](http://www.cmake.org/cmake/resources/software.html)下載安裝CMake。我們選擇Win32安裝程序。

一旦CMake安裝成功，你可以選擇從命令行或者GUI啟動CMake，為了簡易我們選擇後者。CMake需要一個源代碼目錄和一個存放編譯結果的目標文件目錄。源代碼目錄我們選擇GLFW的源代碼的根目錄，然後我們新建一個_build_文件夾來作為目標目錄。

![](http://learnopengl.com/img/getting-started/cmake.png)

之後，點擊**Configure(設置)**按鈕，我們選擇生成的目標平臺為**Visual Studio 11**(因為Visual Studio 2012的內部版本號是11.0)。CMake會顯示可選的編譯選項，這裡我們使用默認設置，再次點擊**Configure(設置)**按鈕，保存這些設置。保存之後，我們可以點擊**Generate(生成)**按鈕，生成的工程文件就會出現在你的*build*文件夾中。

### 編譯

在**build**文件夾裡可以找到**GLFW.sln**文件，用Visual Studio 2012打開。因為CMake已經配置好了項目所以我們直接點擊**Build Solution(構建解決方案)**然後編譯的結果**glfw3.lib**就會出現在**src/Debug**文件夾內。(注意我們現在使用的glfw的版本號為3.1)

生成庫之後，我們需要讓IDE知道庫和頭文件的位置。有兩種方法：

1. 找到IDE或者編譯器的**/lib**和**/include**文件夾，之後添加GLFW的**include**目錄到**/include**裡去，相似的將**glfw3.lib**添加到**/lib**裡去。這不是推薦的方式，因為很難去追蹤library/include文件夾，而且重新安裝IDE/Compiler可能會導致這些文件丟失。
2. 推薦的方式是建立一個新的目錄包含所有的第三方庫文件和頭文件，並且在你的IDE/Compiler中指定這些文件夾。我個人使用一個單獨的文件夾包含**Libs**和**Include**文件夾，在這裡存放OpenGL工程用到的所有第三方庫和頭文件。這樣我的所有第三方庫都在同一個路徑(並且應該在你的多臺電腦間共享)，然而要求是每次新建一個工程我們都需要告訴IDE/編譯器在哪能找到這些文件

完成上面步驟後，我們就可以使用GLFW創建我們的第一個OpenGL工程了！

## 我們的第一個工程

現在，讓我們打開Visual Studio，創建一個新的工程。如果提供了多個選項，選擇Visual C++，然後選擇**空工程(Empty Project)**，別忘了給你的工程起一個合適的名字。現在我們有了一個空的工程去創建我們的OpenGL程序。

## 鏈接(Linking)

為了使我們的程序使用GLFW，我們需要把GLFW庫**鏈接(Link)**進工程。於是我們需要在鏈接器的設置裡寫上**glfw3.lib**。但是我們的工程還不知道在哪尋找這個文件，於是我們首先需要將我們放第三方庫的目錄添加進設置。

為了添加這些目錄，我們首先進入Project Properties(工程屬性)(在解決方案窗口裡右鍵項目)，然後選擇**VC++ Directories**選項卡(如下圖)。在下面的兩欄添加目錄：

![](http://learnopengl.com/img/getting-started/vc_directories.png)

從這裡你可以把自己的目錄加進去從而工程知道從哪去尋找庫文件和頭文件。可以手動把目錄加在後面，也可以點**<Edit..>**選項，下面的圖是Include Directories的設置：

![](http://learnopengl.com/img/getting-started/include_directories.png)

這裡可以添加任意多個目錄，IDE會從這些目錄裡尋找頭文件。所以只要你將GLFW的**Include**文件夾加進路徑中，你就可以使用**<GLFW/..>**來引用頭文件。庫文件也是一樣的。

現在VS可以找到我們鏈接GLFW需要的所有文件了。最後需要在**Linker(鏈接器)**選項卡里的**Input**選項卡里添加**glfw3.lib**這個文件：

![](http://learnopengl.com/img/getting-started/linker_input.png)

要鏈接一個庫我們必須告訴鏈接器它的文件名。因為我們的庫名字是**glfw3.lib**，我們把它加到**Additional Dependencies**域裡面(手動或者使用**<Edit..>**選項)。這樣GLFW就會被鏈接進我們的工程。除了GLFW，你也需要鏈接OpenGL的庫，但是這個庫可能因為系統的不同而有一些差別。

### Windows上的OpenGL庫

如果你是Windows平臺，**opengl32.lib**已經隨著Microsoft SDK裝進了Visual Studio的默認目錄，所以Windows上我們只需將**opengl32.lib**添加進Additional Dependencies。

### Linux上的OpenGL庫

在Linux下你需要鏈接**libGl.so**，所以要添加**-lGL**到你的鏈接器設置裡。如果找不到這個庫你可能需要安裝Mesa，NVidia或AMD的開發包，這部分因平臺而異就不仔細講解了。

現在，如果你添加好了GLFW和OpenGL庫，你可以用如下方式添加GLFW頭文件：

```c++
	#include <GLFW\glfw3.h>
```

這個頭文件包含了GLFW的設置。

## GLEW

到這裡，我們仍然有一件事要做。因為OpenGL只是一個規範，具體的實現是由驅動開發商針對特定顯卡實現的。由於顯卡驅動版本眾多，大多數函數都無法在編譯時確定下來，需要在運行時獲取。開發者需要運行時獲取函數地址並保存下來供以後使用。取得地址的方法因平臺而異，Windows下看起來類似這樣：

```c++
// 定義函數類型
typedef void (*GL_GENBUFFERS) (GLsizei, GLuint*);
// 找到正確的函數並賦值給函數指針
GL_GENBUFFERS glGenBuffers  = (GL_GENBUFFERS)wglGetProcAddress("glGenBuffers");
// 現在函數可以被正常調用了
GLuint buffer;
glGenBuffers(1, &buffer);
```

你可以看到代碼複雜而笨重，因為我們對於每個函數都必須這樣。幸運的是，有一個針對此目的的庫，GLEW，是目前最流行的做這件事的方式。

### 編譯和鏈接GLEW

GLEW是OpenGL Extension Wrangler Library的縮寫，它管理我們上面提到的一系列繁瑣的任務。因為GLEW也是一個庫，我們同樣需要鏈接進工程。GLEW可以從[這裡](http://glew.sourceforge.net/index.html)下載，你可以選擇下載二進制版本或者下載源碼編譯。記住，優先選擇32位的二進制版本。

我們使用GLEW的靜態版本glew32s.lib(注意這裡的's')，用如上的方式添加其庫文件和頭文件，最後在鏈接器的選項里加上glew32s.lib。注意GLFW3也是編譯成了一個靜態庫。


!!! Important

	**靜態(Static)**鏈接是指編譯時就將庫代碼裡的內容合併進二進制文件。優點就是你不需要再放額外的文件，只需要發佈你最終的二進制代碼文件。缺點就是你的程序會變得更大，另外當庫有升級版本時，你必須重新進行編譯。
	**動態(Dynamic)**鏈接是指一個庫通過.dll或.so的方式存在，它的代碼與你的二進制文件的代碼是分離的。優點是使你的程序大小變小並且更容易升級，缺點是你發佈時必須帶上這些dll。


如果你希望靜態鏈接GLEW，必須在包含GLEW頭文件之前定義預編譯宏`GLEW_STATIC`：

```c++
#define GLEW_STATIC
#include <GL/glew.h>
```

如果你希望動態鏈接，那麼就不要定義這個宏。但是使用動態鏈接的話你需要拷貝一份dll文件到你的應用程序目錄。

!!! Important

	對於Linux用戶建議使用這個命令行`-lGLEW -lglfw3 -lGL -lX11 -lpthread -lXrandr -lXi`。沒有正確鏈接相應的庫會產生*undefined reference*(未定義的引用)這個錯誤。

我們現在成功編譯了GLFW和GLEW庫，我們將進入[下一節](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/03%20Hello%20Window/)去使用GLFW和GLEW來設置OpenGL上下文並創建窗口。記住確保你的頭文件和庫文件的目錄設置正確，以及鏈接器裡引用的庫文件名正確。如果仍然遇到錯誤，請參考額外資源中的例子。

##額外的資源

- [Building applications](http://www.opengl-tutorial.org/miscellaneous/building-your-own-c-application/): 提供了很多編譯鏈接相關的信息以及一大批錯誤的解決方法。
- [GLFW with Code::Blocks](http://wiki.codeblocks.org/index.php?title=Using_GLFW_with_Code::Blocks):使用Code::Blocks IDE編譯GLFW。
- [Running CMake](http://www.cmake.org/runningcmake/): 簡要的介紹如何在Windows和Linux上使用CMake。
- [Writing a build system under Linux](http://learnopengl.com/demo/autotools_tutorial.txt): Wouter Verholst寫的一個自動工具的教程，關於如何在Linux上建立編譯環境，尤其是針對這些教程。
- [Polytonic/Glitter](https://github.com/Polytonic/Glitter): 一個簡單的樣板項目，它已經提前配置了所有相關的庫；如果你想要很方便地搞到一個LearnOpenGL教程的範例工程，這是一個很好的東西。