本文作者JoeyDeVries，由Meow J翻譯自[http://learnopengl.com](http://learnopengl.com/#!Getting-started/Shaders)

##變換(Transformations)

儘管我們現在已經知道了如何創建物體，著色，與加入紋理，他們仍然是靜態的，還不是那麼有趣. 我們可以嘗試著在每一幀改變物體的頂點並且重設緩衝區從而使他們移動，但這太繁瑣了，而且會消耗很多的處理時間. 然而，我們現在有一個更好的解決方案，那就是使用(多個)矩陣來對這個物體進行**變換**(Transform).

**矩陣**(Matrix)是一種非常有用的數學工具，儘管聽起來可能有些嚇人. 不過一旦你適應了使用它們，你會發現它們非常的有用. 在討論矩陣的過程中，我們必須使用到一些數學知識，如果你對這塊的數學非常感興趣，我會提供給你提供額外的拓展資源.

為了深入瞭解變換，我們需要在討論矩陣之前先來深入探討**矢量**(Vector). 這一節的目標是讓你擁有將來需要的最基礎的數學背景知識. 如果你發現這節十分困難，儘量去理解吧. 你以後遇見問題也可以回來複習這節的概念.

##矢量(Vectors)

最簡單的來定義，矢量表示一個方向. 更正式的說，矢量同時擁有**方向** (Direction)和**大小** (Magnitude). 你可以想象一下，把矢量當做藏寶圖的指示：左走10步，之後向北走3步，再向右走5步. 在這個例子當中，“左”就是方向，“10步”就是這個矢量的大小. 你可以發現，這個藏寶圖的指示一共有三個矢量. 矢量可以在任意**維度**(Dimension)上，但我們最常使用的是從二維到四維. 如果一個矢量有兩個維度，它就表示在平面上的一個方向；如果一個矢量有三個維度，他就能表示在3D世界中的一個方向.

下圖中你可以看到三個矢量，每一個矢量都由一個箭頭表示，並且記為(x,y). 我們在2D圖片中展示這些矢量，因為這樣子會更直觀. 你仍然可以把這些2D矢量當做z座標為0的3D矢量. 因為矢量代表著一個方向，矢量原點的不同**並不會**改變它的值. 在圖像中，矢量![](http://latex2png.com/output//latex_e91010b29e958e4fbc824584145939c6.png)和![](http://latex2png.com/output//latex_d9ed1f291de6a7f8f8b98910b32d1b1f.png)是**相等**的，儘管他們的原點不一樣.

![](http://learnopengl.com/img/getting-started/vectors.png)

當表示一個矢量的時候，通常我們使用在字符上面加一道槓的方法來表示，比如![](http://latex2png.com/output//latex_bd890aa3934604aac5038acd23d62d50.png). 在公式中是這樣表示的:

![](http://latex2png.com/output//latex_f6b0b4b613d888bc3239069623169884.png)

因為矢量是方向，所以有些時候會很難形象地用位置(Position)表示出來. 我們通常設定這個方向的原點為(0,0,0)，然後指向對應座標的點，使其變為位置矢量(Position Vector)來表示(我們也可以定義一個不同的原點並認為其從該原點的空間指向那個點). 位置矢量(3,5)將在圖像中從原點(0,0)指向點(3,5). 因此，我們可以使用矢量在2D或3D空間中表示方向**與**位置.


就像普通的數字一樣，我們也可以給矢量定義一些運算(你可能已經知道一些了).

### 矢量與標量運算(Scalar Vector Operations)

**標量**(Scalar)僅僅是一個數字(或者說是僅有一個分量的矢量). 當使用標量對矢量進行加/減/乘/除運算時，我們可以簡單地對該矢量的每一個元素進行該運算. 對於加法來說會像這樣:

![](http://latex2png.com/output//latex_ced260b3ce642ed56f177244c3b4189e.png)

其中的＋可以是＋/－/×/÷，注意－和÷運算時不能顛倒，顛倒的運算是沒有定義的(標量-/÷矢量)

### 矢量的取反(Vector Negation)

對一個矢量取反會將其方向逆轉，比如說一個指向東北方向的矢量取反後將會指向西南方向. 我們在一個矢量的每個分量前加負號從而實現取反(或者說用-1數乘該矢量):

![](http://latex2png.com/output//latex_8d5dcc978ccf9d559d64af04b1ddfa7c.png)

### 矢量的加與減(Addition and Subtraction)

矢量的加法可以被定義為是分量的相加，即將一個矢量中的每一個分量加上另一個矢量的對應分量：

![](http://latex2png.com/output//latex_3d5a8fd9db45ecfed81acdad2691caa8.png)

在圖像上v=(4,2)與k=(1,2)相加是這樣的：

![](http://learnopengl.com/img/getting-started/vectors_addition.png)

就像數字的加減一樣，矢量的減法等同於一個矢量加上取反的另一個矢量.

![](http://latex2png.com/output//latex_aa68be1c1c3294bf4d931c39d9fe8ea1.png)

兩個矢量的相減會得到這兩個矢量指向位置的差. 這在我們想要獲取兩點的差會非常有用.

### 矢量的長度(Length)

我們使用勾股定理(Pythagoras theorem)來獲取矢量的長度(大小). 如果你把矢量的x與y分量畫出來，該矢量會形成一個以x與y分量為邊的三角形:

![](http://learnopengl.com/img/getting-started/vectors_triangle.png)

因為x與y已知，我們可以用勾股定理求出斜邊![](http://latex2png.com/output//latex_e91010b29e958e4fbc824584145939c6.png)：

![](http://latex2png.com/output//latex_25a4b63018e587812dd6625a075ec9dd.png)

其中![](http://latex2png.com/output//latex_8f7bf6541904f09a5318814c6b03fe17.png)代表矢量![](http://latex2png.com/output//latex_e91010b29e958e4fbc824584145939c6.png)的大小. 我們也可以很容易加上![](http://latex2png.com/output//latex_a1049fd26252d4e0795dd75bd0bb8e12.png)把這個公式拓展到三維空間

#WIP
