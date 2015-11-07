# OpenGL (Open Graphics Library)

原文     | [OpenGL](http://learnopengl.com/#!Getting-started/OpenGL)
      ---|---
作者     | JoeyDeVries
翻譯     | gjy_1992
校對     | Geequlim


在開始這段旅程之前我們先了解一下OpenGL到底是什麼。一般它被認為是一個**應用程序編程接口**(Application Programming Interface, API)，它包含了一系列可以操作圖形、圖像的方法。然而，OpenGL本身並不是一個API，僅僅是一個規範，由[Khronos組織](http://www.khronos.org/)制定並維護。

OpenGL嚴格規定了每個函數該如何執行，以及它們該如何返回。至於內部具體每個函數是如何實現的，將由OpenGL庫的開發者自行決定(注：這裡開發者是指編寫OpenGL庫的人)。因為OpenGL規範並沒有規定實現的細節，具體的OpenGL庫允許使用不同的實現，只要其功能和結果與規範相匹配(亦即，作為用戶不會感受到功能上的差異)。

實際的OpenGL庫的開發者通常是顯卡的生產商。每個你購買的顯卡都會支持特定版本的OpenGL，通常是為一個系列的顯卡專門開發的。當你使用蘋果系統的時候，OpenGL庫是由蘋果自身維護的。在Linux下，有顯卡生產商提供的OpenGL庫，也有一些愛好者改編的版本。這也意味著任何時候OpenGL庫表現的行為與規範規定的不一致時，基本都是庫的開發者留下的bug。(快甩鍋)


!!! Important

	由於OpenGL的大多數實現都是由顯卡廠商編寫的，當產生一個bug時通常可以通過升級顯卡驅動來解決。這些驅動會包括你的顯卡能支持的最新版本的OpenGL，這也是為什麼總是建議你偶爾更新一下顯卡驅動。


所有版本的OpenGL規範書都被寄存在Khronos那裡，並且都是公開的。有興趣的讀者可以找到OpenGL3.3(我們將要使用的版本)的[規範書](https://www.opengl.org/registry/doc/glspec33.core.20100311.withchanges.pdf)。如果你想深入到OpenGL的細節(只關心函數的功能描述而不是函數的實現)，這是個很好的選擇。該規範還提供一個強大的可以尋找到每個函數具體功能的參考。

## 核心模式(Core-profile)與立即渲染模式(Immediate mode)

早期的OpenGL使用**立即渲染模式**(也就是**固定渲染管線**)，這個模式下繪製圖形很方便。OpenGL的大多數功能都被庫隱藏起來，開發者很少能控制OpenGL如何進行計算。而開發者希望更多的靈活性。隨著時間推移，規範越來越靈活，開發者也能更多的控制繪圖細節。立即渲染模式確實容易使用和理解，但是效率太低。因此從OpenGL3.2開始，規範書開始廢棄立即渲染模式，推出核心模式，這個模式完全移除了舊的特性。

當使用核心模式時，OpenGL迫使我們使用現代的做法。當我們試圖使用一個廢棄的函數時，OpenGL會拋出一個錯誤並終止繪圖。現代做法的優勢是更高的靈活性和效率，然而也更難於學習。立即渲染模式從OpenGL實際操作中抽象掉了很多細節，因而它易於學習的同時，也很難去把握OpenGL具體是如何操作的。現代做法要求使用者真正理解OpenGL和圖形編程，它有一些難度，然而提供了更多的靈活性，更高的效率，更重要的是可以更深入的理解圖形編程。

這也是為什麼我們的教程面向OpenGL3.3的核心模式。雖然上手更困難，但是值得去努力。

現今更高版本的OpenGL已經發布(目前最新是4.5)，你可能會問：為什麼我們還要學習3.3？答案很簡單，所有OpenGL的更高的版本都是在3.3的基礎上，添加了額外的功能，並不更改進核心架構。新版本只是引入了一些更有效率或更有用的方式去完成同樣的功能。因此所有的概念和技術在現代OpenGL版本里都保持一致。當你的經驗足夠，你可以輕鬆使用來自更高版本OpenGL的新特性。


!!! Attention

	當使用新版本的OpenGL特性時，只有新一代的顯卡能夠支持你的應用程序。這也是為什麼大多數開發者基於較低版本的OpenGL編寫程序，並有選擇的啟用新特性。

在有些教程裡你會發現像如下方式註明的新特性。

## 擴展(Extension)

OpenGL的一大特性就是對擴展的支持，當一個顯卡公司提出一個新特性或者渲染上的大優化，通常會以擴展的方式在驅動中實現。如果一個程序在支持這個擴展的顯卡上運行，開發者可以使用這個擴展提供的一些更先進更有效的圖形功能。通過這種方式，開發者不必等待一個新的OpenGL規範面世，就可以方便的檢查顯卡是否支持此擴展。通常，當一個擴展非常流行或有用的時候，它將最終成為未來的OpenGL規範的一部分。

使用擴展的代碼大多看上去如下：

```c++
if(GL_ARB_extension_name)
{
    // 使用一些新的特性
}
else
{
    // 不支持此擴展: 用舊的方式去做
}
```

使用OpenGL3.3時，我們很少需要使用擴展來完成大多數功能，但是掌握這種方式是必須的。

## 狀態機(State Machine)

OpenGL自身是一個巨大的狀態機：一個描述OpenGL該如何操作的所有變量的大集合。OpenGL的狀態通常被稱為OpenGL**上下文(Context)**。我們通常使用如下途徑去更改OpenGL狀態：設置一些選項，操作一些緩衝。最後，我們使用當前OpenGL上下文來渲染。

假設當我們想告訴OpenGL去畫線而不是三角形的時候，我們通過改變一些上下文變量來改變OpenGL狀態，從而告訴OpenGL如何去繪圖。一旦我們改變了OpenGL的狀態為繪製線段，下一個繪製命令就會畫出線段而不是三角形。

用OpenGL工作時，我們會遇到一些**狀態設置函數**(State-changing Function)，以及一些在這些狀態的基礎上**狀態應用的函數**(State-using Function)。只要你記住OpenGL本質上是個大狀態機，就能更容易理解它的大部分特性。

## 對象(Object)

OpenGL庫是用C語言寫的，同時也支持多種語言的派生，但是核心是一個C庫。一些C語言的結構不易被翻譯到其他高層語言，因此OpenGL設計的時候引入了一些抽象概念。“對象”就是其中一個。

在OpenGL中一個對象是指一些選項的集合，代表OpenGL狀態的一個子集。比如，我們可以用一個對象來代表繪圖窗口的設置，可以設置它的大小、支持的顏色位數等等。可以把對象看做一個C風格的結構體：

```c++
struct object_name {
    GLfloat  option1;
    GLuint   option2;
    GLchar[] name;
};
```

!!! Important
	
	**原始類型(Primitive Type)**
	使用OpenGL時，建議使用OpenGL定義的原始類型。比如使用`float`時我們加上前綴GL(因此寫作`GLfloat`)。`int`，`uint`，`char`，`bool`等等類似。OpenGL定義的這些GL原始類型是平臺無關的內存排列方式。而int等原始類型在不同平臺上可能有不同的內存排列方式。使用GL原始類型可以保證你的程序在不同的平臺上工作一致。

當我們使用一個對象時，通常看起來像如下一樣(把OpenGL上下文比作一個大的結構體)：

```c++
// OpenGL的狀態
struct OpenGL_Context 
{
    ...
    object* object_Window_Target;
    ...  	
};
```

```c++
// 創建對象
GLuint objectId = 0;
glGenObject(1, &objectId);
// 綁定對象至上下文
glBindObject(GL_WINDOW_TARGET, objectId);
// 設置GL_WINDOW_TARGET對象的一些選項
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_WIDTH, 800);
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_HEIGHT, 600);
// 將上下文的GL_WINDOW_TARGET對象設回默認
glBindObject(GL_WINDOW_TARGET, 0);
```

這一小片代碼將會是以後使用OpenGL時常見的工作流。我們首先創建一個對象，然後用一個id保存它的引用(實際數據被儲存在後臺)。然後我們將對象綁定至上下文的目標位置(例子裡窗口對象的目標位置被定義成`GL_WINDOW_TARGET`)。接下來我們設置窗口的選項。最後我們通過將目標位置的對象id設回0的方式解綁這個對象。設置的選項被保存在`objectId`代表的對象中，一旦我們重新綁定這個對象到`GL_WINDOW_TARGET`位置，這些選項就會重新生效。


!!! Attention

	目前提供的示例代碼只是OpenGL如何操作的一個大致描述，通過閱讀以後的教程你會遇到很多實際的例子。

使用對象的一個好處是我們在程序中不止可以定義一個對象並且設置他們的狀態，在我們需要進行一個操作的時候，只需要綁定預設了需要設置的對象即可。比如，有一個作為3D模型的數據(一棟房子或一個人物，由多個子模型構成)容器對象，在我們想繪製其中任何一個3D模型的時候，只需綁定相應的子模型數據的對象(我們預先創建並設置好了它們的選項)就可以了。擁有數個這樣的對象允許我們指定多個模型，在想畫其中任何一個的時候，簡單的將相應的對象綁定上去，便不需要再進行重複的設置選項的操作了。

## 讓我們開始吧

你現在已經知道一些OpenGL的相關知識了，包括OpenGL作為規範和庫，OpenGL大致的操作流程，以及一些使用擴展的小技巧。不要擔心沒有你還完全消化它們，通過後面的教程我們會仔細地講解每一步，你會通過足夠的例子來把握OpenGL。如果你已經做好了開始下一步的準備，我們可以開始建立OpenGL上下文以及我們的第一個窗口了。[點擊這裡](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/02%20Creating%20a%20window/)(第一章第二節)

## 額外的資源

- [opengl.org](https://www.opengl.org/): OpenGL官方網站。
- [OpenGL registry](https://www.opengl.org/registry/): OpenGL各版本的規範和擴展的主站。