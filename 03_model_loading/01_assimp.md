# Assimp開源模型導入庫

原文     | [Assimp](http://learnopengl.com/#!Model-Loading/Assimp)
      ---|---
作者     | JoeyDeVries
翻譯     | Cocoonshu
校對     | [Geequlim](http://geequlim.com)


到目前為止，我們已經在所有的場景中大面積濫用了我們的容器盒小盆友，但就是容器盒是我們的好朋友，時間久了我們也會喜新厭舊。一些圖形應用裡經常會使用很多複雜且好玩兒的模型，它們看起來比靜態的容器盒可愛多了。但是，我們無法像定義容器盒一樣手動地去指定房子、貨車或人形角色這些複雜模型的頂點、法線和紋理座標。我們需要做的也是應該要做的，是把這些模型導入到應用程序中，而設計製作這些3D模型的工作應該交給像[Blender](http://www.blender.org/)、[3DS Max](http://www.autodesk.nl/products/3ds-max/overview)或者[Maya](http://www.autodesk.com/products/autodesk-maya/overview)這樣的工具軟件。

那些3D建模工具，可以讓美工們構建一些複雜的形狀，並將貼圖應用到形狀上去，即紋理映射。然後，在導出模型文件時，建模工具會自己生成所有的頂點座標、頂點法線和紋理座標。這樣，美工們可以不用瞭解大量的圖像技術細節，就能有大量的工具集去隨心地構建高品質的模型。所有的技術細節內容都隱藏在裡導出的模型文件裡。而我們，這些圖形開發者，就必須得去關注這些技術細節了。

因此，我們的工作就是去解析這些導出的模型文件，並將其中的模型數據存儲為OpenGL能夠使用的數據。一個常見的問題是，導出的模型文件通常有幾十種格式，不同的工具會根據不同的文件協議把模型數據導出到不同格式的模型文件中。有的模型文件格式只包含模型的靜態形狀數據和顏色、漫反射貼圖、高光貼圖這些基本的材質信息，比如Wavefront的.obj文件。而有的模型文件則採用XML來記錄數據，且包含了豐富的模型、光照、各種材質、動畫、攝像機信息和完整的場景信息等，比如Collada文件格式。Wavefront的obj格式是為了考慮到通用性而設計的一種便於解析的模型格式。建議去Wavefront的Wiki上看看obj文件格式是如何封裝的。這會給你形成一個對模型文件格式的一個基本概念和印象。

## 模型加載庫

現在市面上有一個很流行的模型加載庫，叫做Assimp，全稱為Open Asset Import Library。Assimp可以導入幾十種不同格式的模型文件（同樣也可以導出部分模型格式）。只要Assimp加載完了模型文件，我們就可以從Assimp上獲取所有我們需要的模型數據。Assimp把不同的模型文件都轉換為一個統一的數據結構，所有無論我們導入何種格式的模型文件，都可以用同一個方式去訪問我們需要的模型數據。

當導入一個模型文件時，即Assimp加載一整個包含所有模型和場景數據的模型文件到一個scene對象時，Assimp會為這個模型文件中的所有場景節點、模型節點都生成一個具有對應關係的數據結構，且將這些場景中的各種元素與模型數據對應起來。下圖展示了一個簡化的Assimp生成的模型文件數據結構：

<div class="centerHV">
<img src="http://learnopengl.com/img/model_loading/assimp_structure.png"/>
</div>

 - 所有的模型、場景數據都包含在scene對象中，如所有的材質和Mesh。同樣，場景的根節點引用也包含在這個scene對象中
 - 場景的根節點可能也會包含很多子節點和一個指向保存模型點雲數據mMeshes[]的索引集合。根節點上的mMeshes[]裡保存了實際了Mesh對象，而每個子節點上的mMesshes[]都只是指向根節點中的mMeshes[]的一個引用(譯者注：C/C++稱為指針，Java/C#稱為引用)
 - 一個Mesh對象本身包含渲染所需的所有相關數據，比如頂點位置、法線向量、紋理座標、面片及物體的材質
 - 一個Mesh會包含多個面片。一個Face（面片）表示渲染中的一個最基本的形狀單位，即圖元（基本圖元有點、線、三角面片、矩形面片）。一個面片記錄了一個圖元的頂點索引，通過這個索引，可以在mMeshes[]中尋找到對應的頂點位置數據。頂點數據和索引分開存放，可以便於我們使用緩存（VBO、NBO、TBO、IBO）來高速渲染物體。（詳見[Hello Triangle](http://www.learnopengl.com/#!Getting-started/Hello-Triangle)）
 - 一個Mesh還會包含一個Material（材質）對象用於指定物體的一些材質屬性。如顏色、紋理貼圖（漫反射貼圖、高光貼圖等）

所以我們要做的第一件事，就是加載一個模型文件為scene對象，然後獲取每個節點對應的Mesh對象（我們需要遞歸搜索每個節點的子節點來獲取所有的節點），並處理每個Mesh對象對應的頂點數據、索引以及它的材質屬性。最終我們得到一個只包含我們需要的數據的Mesh集合。

!!! Important

    **Mesh(網格,或被譯為“模型點雲”)**
    
    用建模工具構建物體時，美工通常不會直接使用單個形狀來構建一個完整的模型。一般來說，一個模型會由幾個子模型/形狀組合拼接而成。而模型中的那些子模型/形狀就是我們所說的一個Mesh。例如一個人形模型，美工通常會把頭、四肢、衣服、武器這些組件都分別構建出來，然後在把所有的組件拼合在一起，形成最終的完整模型。一個Mesh（包含頂點、索引和材質屬性）是我們在OpenGL中繪製物體的最小單位。一個模型通常有多個Mesh組成。

下一節教程中，我們將用上述描述的數據結構來創建我們自己的Model類和Mesh類，用於加載和保存那些導入的模型。如果我們想要繪製一個模型，我們不會去渲染整個模型，而是去渲染這個模型所包含的所有獨立的Mesh。不管怎樣，我們開始導入模型之前，我們需要先把Assimp導入到我們的工程中。

## 構建Assimp

你可以在[Assimp的下載頁面](http://assimp.sourceforge.net/main_downloads.html)選擇一個想要的版本去下載Assimp庫。到目前為止，Assimp可用的最新版本是3.1.1。我們建議你自己編譯Assimp庫，因為Assimp官方的已編譯庫不能很好地覆蓋在所有平臺上運行。如果你忘記怎樣使用CMake編譯一個庫，請詳見[Creating a window(創建一個窗口)](http://www.learnopengl.com/#!Getting-started/Creating-a-window)教程。

這裡我們列出一些編譯Assimp時可能遇到的問題，以便大家參考和排除:

 - CMake在讀取配置列表時，報出與DirectX庫丟失相關的一些錯誤。報錯如下：
 
```
Could not locate DirecX
CMake Error at cmake-modules/FindPkgMacros.cmake:110 (message):
Required library DirectX not found! Install the library (including dev packages) and try again. If the library is already installed, set the missing variables manually in cmake.
```

這個問題的解決方案：如果你之前沒有安裝過DirectX SDK，那麼請安裝。下載地址：[DirectX SDK](http://www.microsoft.com/en-us/download/details.aspx?id=6812)
 - 安裝DirectX SDK時，可以遇到一個錯誤碼為<b>S1023</b>的錯誤。遇到這個問題，請在安裝DirectX SDK前，先安裝C++ Redistributable package(s)。
  問題解釋：[已知問題：DirectX SDK (June 2010) 安裝及S1023錯誤](Known Issue: DirectX SDK (June 2010) Setup and the S1023 error)
 - 一旦配置完成，你就可以生成解決方案文件了，打開解決方案文件並編譯Assimp庫（編譯為Debug版本還是Release版本，根據你的需要和心情來定吧）
 - 使用默認配置構建的Assimp是一個動態庫，所以我們需要把編譯出來的assimp.dll文件拷貝到我們自己程序的可執行文件的同一目錄裡
 - 編譯出來的Assimp的LIB文件和DLL文件可以在code/Debug或者code/Release裡找到
 - 把編譯好的LIB文件和DLL文件拷貝到工程的相應目錄下，並鏈接到你的解決方案中。同時還好記得把Assimp的頭文件也拷貝到工程裡去（Assimp的頭文件可以在include目錄裡找到）

如果你還遇到了其他問題，可以在下面給出的鏈接裡獲取幫助。

!!! Important

    如果你想要讓Assimp使用多線程支持來提高性能，你可以使用<b>Boost</b>庫來編譯 Assimp。在[Boost安裝頁面](http://assimp.sourceforge.net/lib_html/install.html)，你能找到關於Boost的完整安裝介紹。

現在，你應該已經能夠編譯Assimp庫，並鏈接Assimp到你的工程裡去了。下一節內容：[導入完美的3D物件！](http://learnopengl-cn.readthedocs.org/zh/latest/03%20Model%20Loading/02%20Mesh/)
