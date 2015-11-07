本文作者JoeyDeVries，由Django翻譯自[http://learnopengl.com](http://learnopengl.com)

## 法線貼圖 （Normal Mapping）

我們的場景中已經充滿了多邊形物體，其中每個都可能由成百上千平坦的三角形組成。我們以向三角形上附加紋理的方式來增加額外細節，提升真實感，隱藏多邊形幾何體是由無數三角形組成的事實。紋理確有助益，然而當你近看它們時，這個事實便隱藏不住了。現實中的物體表面並非是平坦的，而是表現出無數（凹凸不平的）細節。

例如，磚塊的表面。磚塊的表面非常粗糙，顯然不是完全平坦的：它包含著接縫處水泥凹痕，以及非常多的細小的空洞。如果我們在一個有光的場景中看這樣一個磚塊的表面，問題就出來了。下圖中我們可以看到磚塊紋理應用到了平坦的表面，並被一個點光源照亮。

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_flat.png)

光照並沒有呈現出任何裂痕和孔洞，完全忽略了磚塊之間凹進去的線條；表面看起來完全就是平的。我們可以使用specular貼圖根據深度或其他細節阻止部分表面被照的更亮，以此部分地解決問題，但這並不是一個好方案。我們需要的是某種可以告知光照系統給所有有關物體表面類似深度這樣的細節的方式。

如果我們一光的視角來看這個問題：是什麼使表面被視為完全平坦的表面來照亮？答案會是表面的法線向量。以光照算法的視角考慮的話，只有一件事決定物體的形狀，這就是垂直於它的法線向量。磚塊表面只有一個法線向量，表面完全根據這個法線向量被以一致的方式照亮。如果每個fragment都是用自己的不同的法線會怎樣？這樣我們就可以根據表面細微的細節對法線向量進行改變；這樣就會獲得一種表面看起來要複雜得多的幻覺：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_surfaces.png)

每個fragment使用了自己的法線，我們就可以讓光照相信一個表面由很多微小的（垂直於法線向量的）平面所組成，物體表面的細節將會得到極大提升。這種每個fragment使用各自的法線，替代一個面上所有fragment使用同一個法線的技術叫做法線貼圖（normal mapping）或凹凸貼圖（bump mapping）。應用到磚牆上，效果像這樣：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_compare.png)

你可以看到細節獲得了極大提升，開銷卻不大。因為我們只需要改變每個fragment的法線向量，並不需要改變所有光照公式。現在我們是為每個fragment傳遞一個法線，不再使用插值表面法線。這樣光照使表面擁有了自己的細節。

 

### 法線貼圖

為使法線貼圖工作，我們需要為每個fragment提供一個法線。像diffuse貼圖和specular貼圖一樣，我們可以使用一個2D紋理來儲存法線數據。2D紋理不僅可以儲存顏色和光照數據，還可以儲存法線向量。這樣我們可以從2D紋理中採樣得到特定紋理的法線向量。

由於法線向量是個幾何工具，而紋理通常只用於儲存顏色信息，用紋理儲存法線向量不是非常直接。如果你想一想，就會知道紋理中的顏色向量用r、g、b元素代表一個3D向量。類似的我們也可以將法線向量的x、y、z元素儲存到紋理中，代替顏色的r、g、b元素。法線向量的範圍在-1到1之間，所以我們先要將其映射到0到1的範圍：


1
vec3 rgb_normal = normal * 0.5 - 0.5; // transforms from [-1,1] to [0,1]
將法線向量變換為像這樣的RGB顏色元素，我們就能把根據表面的形狀的fragment的法線保存在2D紋理中。教程開頭展示的那個磚塊的例子的法線貼圖如下所示：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_normal_map.png)

這會是一種偏藍色調的紋理（你在網上找到的幾乎所有法線貼圖都是這樣的）。這是因為所有法線的指向都偏向z軸（0, 0, 1）這是一種偏藍的顏色。法線向量從z軸方向也向其他方向輕微偏移，顏色也就發生了輕微變化，這樣看起來便有了一種深度。例如，你可以看到在每個磚塊的頂部，顏色傾向於偏綠，這是因為磚塊的頂部的法線偏向於指向正y軸方向（0, 1, 0），這樣它就是綠色的了。

在一個簡單的朝向正z軸的平面上，我們可以用這個diffuse紋理和這個法線貼圖來渲染前面部分的圖片。要注意的是這個鏈接裡的法線貼圖和上面展示的那個不一樣。原因是OpenGL讀取的紋理的y（或V）座標和紋理通常被創建的方式相反。鏈接裡的法線貼圖的y（或綠色）元素是相反的（你可以看到綠色現在在下邊）；如果你沒考慮這個，光照就不正確了（譯註：如果你現在不再使用SOIL了，那就不要用鏈接裡的那個法線貼圖，這個問題是SOIL載入紋理上下顛倒所致，它也會把法線在y方向上顛倒）。加載紋理，把它們綁定到合適的紋理單元，然後使用下面的改變了的像素著色器來渲染一個平面：

```c++
uniform sampler2D normalMap;  
 
void main()
{           
    // 從法線貼圖範圍[0,1]獲取法線
    normal = texture(normalMap, fs_in.TexCoords).rgb;
    // 將法線向量轉換為範圍[-1,1]
    normal = normalize(normal * 2.0 - 1.0);   
  
    [...]
    // 像往常那樣處理光照
}
```

這裡我們將被採樣的法線顏色從0到1重新映射回-1到1，便能將RGB顏色重新處理成法線，然後使用採樣出的法線向量應用於光照的計算。在例子中我們使用的是Blinn-Phong著色器。

通過慢慢隨著時間慢慢移動光源，你就能明白法線貼圖是什麼意思了。運行這個例子你就能得到本教程開始的那個效果：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_correct.png)

你可以在這裡找到這個簡單demo的源代碼及其頂點和像素著色器。

然而有個問題限制了剛才講的那種法線貼圖的使用。我們使用的那個法線貼圖裡面的所有法線向量都是指向正z方向的。上面的例子能用，是因為那個平面的表面法線也是指向正z方向的。可是，如果我們在表面法線指向正y方向的平面上使用同一個法線貼圖會發生什麼？

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_ground.png)

光照看起來完全不對！發生這種情況是平面的表面法線現在指向了y，而採樣得到的法線仍然指向的是z。結果就是光照仍然認為表面法線和之前朝向正z方向時一樣；這樣光照就不對了。下面的圖片展示了這個表面上採樣的法線的近似情況：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_ground_normals.png)

你可以看到所有法線都指向z方向，它們本該朝著表面法線指向y方向的。一個可行方案是為每個表面製作一個單獨的法線貼圖。如果是一個立方體的話我們就需要6個法線貼圖，但是如果模型上有無數的朝向不同方向的表面，這就不可行了（譯註：實際上對於複雜模型可以把朝向各個方向的法線儲存在同一張貼圖上，你可能看到過不只是藍色的法線貼圖，不過用那樣的法線貼圖有個問題是你必須記住模型的起始朝向，如果模型運動了還要記錄模型的變換，這是非常不方便的；此外就像作者所說的，如果把一個diffuse紋理應用在同一個物體的不同表面上，就像立方體那樣的，就需要做6個法線貼圖，這也不可取）。

另一個稍微有點難的解決方案是，在一個不同的座標空間中進行光照，這個座標空間裡，法線貼圖向量總是指向這個座標空間的正z方向；所有的光照向量都相對與這個正z方向進行變換。這樣我們就能始終使用同樣的法線貼圖，不管朝向問題。這個座標空間叫做切線空間（tangent space）。

 

### 切線空間

法線貼圖中的法線向量在切線空間中，法線永遠指著正z方向。切線空間是位於三角形表面之上的空間：法線相對於單個三角形的本地參考框架。它就像法線貼圖向量的本地空間；它們都被定義為指向正z方向，無論最終變換到什麼方向。使用一個特定的矩陣我們就能將本地/切線空寂中的法線向量轉成世界或視圖座標，使它們轉向到最終的貼圖表面的方向。

我們可以說，上個部分那個朝向正y的法線貼圖錯誤的貼到了表面上。法線貼圖被定義在切線空間中，所以一種解決問題的方式是計算出一種矩陣，把法線從切線空間變換到一個不同的空間，這樣它們就能和表面法線方向對齊了：法線向量都會指向正y方向。切線空間的一大好處是我們可以為任何類型的表面計算出一個這樣的矩陣，由此我們可以把切線空間的z方向和表面的法線方向對齊。

這種矩陣叫做TBN矩陣這三個字母分別代表tangent、bitangent和normal向量。這是建構這個矩陣所需的向量。要建構這樣一個把切線空間轉變為不同空間的變異矩陣，我們需要三個相互垂直的向量，它們沿一個表面的法線貼圖對齊於：上、右、前；這和我們在[攝像機教程](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/09%20Camera/)中做的類似。

已知上向量是表面的法線向量。右和前向量是切線和副切線向量。下面的圖片展示了一個表面的三個向量：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_tbn_vectors.png)

計算出切線和副切線並不像法線向量那麼容易。從圖中可以看到法線貼圖的切線和副切線與紋理座標的兩個方向對齊。我們就是用到這個特性計算每個表面的切線和副切線的。需要用到一些數學才能得到它們；請看下圖：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_surface_edges.png)

上圖中我們可以看到邊<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msub>
    <mi>E</mi>
    <mn>2</mn>
  </msub>
</math>紋理座標的不同，<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msub>
    <mi>E</mi>
    <mn>2</mn>
  </msub>
</math>是一個三角形的邊，這個三角形的另外兩條邊是<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
  <msub>
    <mi>U</mi>
    <mn>2</mn>
  </msub>
</math>和<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
  <msub>
    <mi>V</mi>
    <mn>2</mn>
  </msub>
</math>，它們與切線向量*T*和副切線向量*B*方向相同。這樣我們可以把邊</math>和<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msub>
    <mi>E</mi>
    <mn>1</mn>
  </msub>
</math>和<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msub>
    <mi>E</mi>
    <mn>2</mn>
  </msub>
</math>用切線向量 *T* 和副切線向量 *B* 的線性組合表示出來（譯註：注意*T*和*B*都是單位長度，在*TB*平面中所有點的*T*、*B*座標都在0到1之間，因此可以進行這樣的組合）：

```math
E_1 = \Delta U_1T + \Delta V_1B

E_2 = \Delta U_2T + \Delta V_2B
```
我們也可以寫成這樣：

```math
(E_{1x}, E_{1y}, E_{1z}) = \Delta U_1(T_x, T_y, T_z) + \Delta V_1(B_x, B_y, B_z)
```

*E*是兩個向量位置的差，*U*和*V*是紋理座標的差。然後我們得到兩個未知數（切線*T*和副切線*B*）和兩個等式。你可能想起你的代數課了，這是讓我們去接*T*和*B*。

上面的方程允許我們把它們寫成另一種格式：矩陣乘法

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mrow>
    <mo>[</mo>
    <mtable rowspacing="4pt" columnspacing="1em">
      <mtr>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>x</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>y</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>z</mi>
            </mrow>
          </msub>
        </mtd>
      </mtr>
      <mtr>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>x</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>y</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>z</mi>
            </mrow>
          </msub>
        </mtd>
      </mtr>
    </mtable>
    <mo>]</mo>
  </mrow>
  <mo>=</mo>
  <mrow>
    <mo>[</mo>
    <mtable rowspacing="4pt" columnspacing="1em">
      <mtr>
        <mtd>
          <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
          <msub>
            <mi>U</mi>
            <mn>1</mn>
          </msub>
        </mtd>
        <mtd>
          <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
          <msub>
            <mi>V</mi>
            <mn>1</mn>
          </msub>
        </mtd>
      </mtr>
      <mtr>
        <mtd>
          <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
          <msub>
            <mi>U</mi>
            <mn>2</mn>
          </msub>
        </mtd>
        <mtd>
          <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
          <msub>
            <mi>V</mi>
            <mn>2</mn>
          </msub>
        </mtd>
      </mtr>
    </mtable>
    <mo>]</mo>
  </mrow>
  <mrow>
    <mo>[</mo>
    <mtable rowspacing="4pt" columnspacing="1em">
      <mtr>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>x</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>y</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>z</mi>
          </msub>
        </mtd>
      </mtr>
      <mtr>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>x</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>y</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>z</mi>
          </msub>
        </mtd>
      </mtr>
    </mtable>
    <mo>]</mo>
  </mrow>
</math>

嘗試會以一下矩陣乘法，它們確實是同一種等式。把等式寫成矩陣形式的好處是，解*T*和*B*會因此變得很容易。兩邊都乘以<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
  <mi>U</mi>
  <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
  <mi>V</mi>
</math>的反數等於：

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msup>
    <mrow>
      <mo>[</mo>
      <mtable rowspacing="4pt" columnspacing="1em">
        <mtr>
          <mtd>
            <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
            <msub>
              <mi>U</mi>
              <mn>1</mn>
            </msub>
          </mtd>
          <mtd>
            <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
            <msub>
              <mi>V</mi>
              <mn>1</mn>
            </msub>
          </mtd>
        </mtr>
        <mtr>
          <mtd>
            <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
            <msub>
              <mi>U</mi>
              <mn>2</mn>
            </msub>
          </mtd>
          <mtd>
            <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
            <msub>
              <mi>V</mi>
              <mn>2</mn>
            </msub>
          </mtd>
        </mtr>
      </mtable>
      <mo>]</mo>
    </mrow>
    <mrow class="MJX-TeXAtom-ORD">
      <mo>&#x2212;<!-- − --></mo>
      <mn>1</mn>
    </mrow>
  </msup>
  <mrow>
    <mo>[</mo>
    <mtable rowspacing="4pt" columnspacing="1em">
      <mtr>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>x</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>y</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>z</mi>
            </mrow>
          </msub>
        </mtd>
      </mtr>
      <mtr>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>x</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>y</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>z</mi>
            </mrow>
          </msub>
        </mtd>
      </mtr>
    </mtable>
    <mo>]</mo>
  </mrow>
  <mo>=</mo>
  <mrow>
    <mo>[</mo>
    <mtable rowspacing="4pt" columnspacing="1em">
      <mtr>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>x</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>y</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>z</mi>
          </msub>
        </mtd>
      </mtr>
      <mtr>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>x</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>y</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>z</mi>
          </msub>
        </mtd>
      </mtr>
    </mtable>
    <mo>]</mo>
  </mrow>
</math>

這樣我們就可以解出*T*和*B*了。這需要我們計算出delta紋理座標矩陣的擬陣。我不打算講解計算逆矩陣的細節，但大致是把它變化為，1除以矩陣的行列式，再乘以它的共軛矩陣。

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mrow>
    <mo>[</mo>
    <mtable rowspacing="4pt" columnspacing="1em">
      <mtr>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>x</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>y</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>T</mi>
            <mi>z</mi>
          </msub>
        </mtd>
      </mtr>
      <mtr>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>x</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>y</mi>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>B</mi>
            <mi>z</mi>
          </msub>
        </mtd>
      </mtr>
    </mtable>
    <mo>]</mo>
  </mrow>
  <mo>=</mo>
  <mfrac>
    <mn>1</mn>
    <mrow>
      <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
      <msub>
        <mi>U</mi>
        <mn>1</mn>
      </msub>
      <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
      <msub>
        <mi>V</mi>
        <mn>2</mn>
      </msub>
      <mo>&#x2013;</mo>
      <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
      <msub>
        <mi>U</mi>
        <mn>2</mn>
      </msub>
      <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
      <msub>
        <mi>V</mi>
        <mn>1</mn>
      </msub>
    </mrow>
  </mfrac>
  <mrow>
    <mo>[</mo>
    <mtable rowspacing="4pt" columnspacing="1em">
      <mtr>
        <mtd>
          <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
          <msub>
            <mi>V</mi>
            <mn>2</mn>
          </msub>
        </mtd>
        <mtd>
          <mo>&#x2212;<!-- − --></mo>
          <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
          <msub>
            <mi>V</mi>
            <mn>1</mn>
          </msub>
        </mtd>
      </mtr>
      <mtr>
        <mtd>
          <mo>&#x2212;<!-- − --></mo>
          <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
          <msub>
            <mi>U</mi>
            <mn>2</mn>
          </msub>
        </mtd>
        <mtd>
          <mi mathvariant="normal">&#x0394;<!-- Δ --></mi>
          <msub>
            <mi>U</mi>
            <mn>1</mn>
          </msub>
        </mtd>
      </mtr>
    </mtable>
    <mo>]</mo>
  </mrow>
  <mrow>
    <mo>[</mo>
    <mtable rowspacing="4pt" columnspacing="1em">
      <mtr>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>x</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>y</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>1</mn>
              <mi>z</mi>
            </mrow>
          </msub>
        </mtd>
      </mtr>
      <mtr>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>x</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>y</mi>
            </mrow>
          </msub>
        </mtd>
        <mtd>
          <msub>
            <mi>E</mi>
            <mrow class="MJX-TeXAtom-ORD">
              <mn>2</mn>
              <mi>z</mi>
            </mrow>
          </msub>
        </mtd>
      </mtr>
    </mtable>
    <mo>]</mo>
  </mrow>
</math>

有了最後這個等式，我們就可以用公式、三角形的兩條邊以及紋理座標計算出切線向量*T*和副切線*B*。

如果你對這些數學內容不理解也不用擔心。當你知道我們可以用一個三角形的頂點和紋理座標（因為紋理座標和切線向量在同一空間中）計算出切線和副切線你就已經部分地達到目的了（譯註：上面的推導已經很清楚了，如果你不明白可以參考任意線性代數教材，就像作者所說的記住求得切線空間的公式也行，不過不管怎樣都得理解切線空間的含義）。

### 手工計算切線和副切線

這個教程的demo場景中有一個簡單的2D平面，它朝向正z方向。這次我們會使用切線空間來實現法線貼圖，所以我們可以使平面朝向任意方向，法線貼圖仍然能夠工作。使用前面討論的數學方法，我們來手工計算出表面的切線和副切線向量。

假設平面使用下面的向量建立起來（1、2、3和1、3、4，它們是兩個三角形）：
// positions
glm::vec3 pos1(-1.0,  1.0, 0.0);
glm::vec3 pos2(-1.0, -1.0, 0.0);
glm::vec3 pos3(1.0, -1.0, 0.0);
glm::vec3 pos4(1.0, 1.0, 0.0);
// texture coordinates
glm::vec2 uv1(0.0, 1.0);
glm::vec2 uv2(0.0, 0.0);
glm::vec2 uv3(1.0, 0.0);
glm::vec2 uv4(1.0, 1.0);
// normal vector
glm::vec3 nm(0.0, 0.0, 1.0);
 

我們先計算第一個三角形的邊和deltaUV座標：


1
2
3
4
glm::vec3 edge1 = pos2 - pos1;
glm::vec3 edge2 = pos3 - pos1;
glm::vec2 deltaUV1 = uv2 - uv1;
glm::vec2 deltaUV2 = uv3 - uv1;
 

有了計算切線和副切線的必備數據，我們就可以開始寫出來自於前面部分中的下列等式：

```c++
GLfloat f = 1.0f / (deltaUV1.x * deltaUV2.y - deltaUV2.x * deltaUV1.y);
 
tangent1.x = f * (deltaUV2.y * edge1.x - deltaUV1.y * edge2.x);
tangent1.y = f * (deltaUV2.y * edge1.y - deltaUV1.y * edge2.y);
tangent1.z = f * (deltaUV2.y * edge1.z - deltaUV1.y * edge2.z);
tangent1 = glm::normalize(tangent1);
 
bitangent1.x = f * (-deltaUV2.x * edge1.x + deltaUV1.x * edge2.x);
bitangent1.y = f * (-deltaUV2.x * edge1.y + deltaUV1.x * edge2.y);
bitangent1.z = f * (-deltaUV2.x * edge1.z + deltaUV1.x * edge2.z);
bitangent1 = glm::normalize(bitangent1);  
  
[...] // similar procedure for calculating tangent/bitangent for plane's second triangle
```

我們預先計算出等式的分數部分f，然後把它和每個向量的元素進行相應矩陣乘法。如果你把代碼和最終的等式對比你會發現，這就是直接套用。最後我們還要進行標準化，來確保切線/副切線向量最後是單位向量。

因為一個三角形永遠是平坦的形狀，我們只需為每個三角形計算一個切線/副切線，它們對於每個三角形上的頂點都是一樣的。要注意的是大多數實現通常三角形和三角形之間都會共享頂點。這種情況下開發者通常將每個頂點的法線和切線/副切線等頂點屬性平均化，以獲得更加柔和的效果。我們的平面的三角形之間分享了一些頂點，但是因為兩個三角形相互並行，因此並不需要將結果平均化，但無論何時只要你遇到這種情況記住它就是件好事。

最後的切線和副切線向量的值應該是(1, 0, 0)和(0, 1, 0)，它們和法線(0, 0, 1)組成相互垂直的TBN矩陣。在平面上顯示出來TBN應該是這樣的：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_tbn_shown.png)

每個頂點定義了切線和副切線向量，我們就可以開始實現正確的法線貼圖了。


### 切線空間法線貼圖

為讓法線貼圖工作，我們先得在著色器中創建一個TBN矩陣。我們先將前面計算出來的切線和副切線向量傳給頂點著色器，作為它的屬性：

```c++
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 texCoords;
layout (location = 3) in vec3 tangent;
layout (location = 4) in vec3 bitangent;
```

在頂點著色器的main函數中我們創建TBN矩陣：

```c++
void main()
{
   [...]
   vec3 T = normalize(vec3(model * vec4(tangent,   0.0)));
   vec3 B = normalize(vec3(model * vec4(bitangent, 0.0)));
   vec3 N = normalize(vec3(model * vec4(normal,    0.0)));
   mat3 TBN = mat3(T, B, N)
}
```

我們先將所有TBN向量變換到我們所操作的座標系中，現在是世界空間，我們可以乘以model矩陣。然後我們創建實際的TBN矩陣，直接把相應的向量應用到mat3構造器就行。注意，如果我們希望更精確的話就不要講TBN向量乘以model矩陣，而是使用法線矩陣，但我們只關心向量的方向，不會平移也和縮放這個變換。

從技術上講，頂點著色器中無需副切線。所有的這三個TBN向量都是相互垂直的所以我們可以在頂點著色器中庸T和N向量的叉乘，自己計算出副切線：vec3 B = cross(T, N);
現在我們有了TBN矩陣，如果來使用它呢？基本有兩種方式可以使用，我們會把這兩種方式都說明一下：

我們可以用TBN矩陣把所有向量從切線空間轉到世界空間，傳給像素著色器，然後把採樣得到的法線用TBN矩陣從切線空間變換到世界空間；法線就處於和其他光照變量一樣的空間中了。
我們用TBN的逆矩陣把所有世界空間的向量轉換到切線空間，使用這個矩陣將除法線以外的所有相關光照變量轉換到切線空間中；這樣法線也能和其他光照變量處於同一空間之中。
我們來看看第一種情況。我們從法線貼圖重採樣得來的法線向量，是以切線空間表達的，儘管其他光照向量是以世界空間表達的。把TBN傳給像素著色器，我們就能將採樣得來的切線空間的法線乘以這個TBN矩陣，將法線向量變換到和其他光照向量一樣的參考空間中。這種方式隨後所有光照計算都可以簡單的理解。

把TBN矩陣發給像素著色器很簡單：


```c++
out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    mat3 TBN;
} vs_out;  
  
void main()
{
    [...]
    vs_out.TBN = mat3(T, B, N);
}
```

在像素著色器中我們用mat3作為輸入變量：

```c++
in VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    mat3 TBN;
} fs_in;
```

有了TBN矩陣我們現在就可以更新法線貼圖代碼，引入切線到世界空間變換：

```c++
normal = texture(normalMap, fs_in.TexCoords).rgb;
normal = normalize(normal * 2.0 - 1.0);   
normal = normalize(fs_in.TBN * normal);
```

因為最後的normal現在在世界空間中了，就不用改變其他像素著色器的代碼了，因為光照代碼就是假設法線向量在世界空間中。

我們同樣看看第二種情況，我們用TBN矩陣的逆矩陣將所有相關的世界空間向量轉變到採樣所得法線向量的空間：切線空間。TBN的建構還是一樣，但我們在將其發送給像素著色器之前先要求逆矩陣：

```c++
vs_out.TBN = transpose(mat3(T, B, N));
```

注意，這裡我們使用transpose函數，而不是inverse函數。正交矩陣（每個軸既是單位向量同時相互垂直）的一大屬性是一個正交矩陣的置換矩陣與它的逆矩陣相等。這個屬性和重要因為逆矩陣的求得比求置換開銷大；結果卻是一樣的。

在像素著色器中我們不用對法線向量變換，但我們要把其他相關向量轉換到切線空間，它們是lightDir和viewDir。這樣每個向量還是在同一個空間（切線空間）中了。


```c++
void main()
{           
    vec3 normal = texture(normalMap, fs_in.TexCoords).rgb;
    normal = normalize(normal * 2.0 - 1.0);   
   
    vec3 lightDir = fs_in.TBN * normalize(lightPos - fs_in.FragPos);
    vec3 viewDir  = fs_in.TBN * normalize(viewPos - fs_in.FragPos);    
    [...]
}
```

第二種方法看似要做的更多，它還需要在像素著色器中進行更多的乘法操作，所以為何還用第二種方法呢？

將向量從世界空間轉換到切線空間有個額外好處，我們可以把所有相關向量在頂點著色器中轉換到切線空間，不用在像素著色器中做這件事。這是可行的，因為lightPos和viewPos不是每個fragment運行都要改變，對於fs_in.FragPos，我們也可以在頂點著色器計算它的切線空間位置。基本上，不需要把任何向量在像素著色器中進行變換，而第一種方法中就是必須的，因為採樣出來的法線向量對於每個像素著色器都不一樣。

所以現在不是把TBN矩陣的逆矩陣發送給像素著色器，而是將切線空間的光源位置，觀察位置以及頂點位置發送給像素著色器。這樣我們就不用在像素著色器裡進行矩陣乘法了。這是一個極佳的優化，因為頂點著色器通常比像素著色器運行的少。這也是為什麼這種方法是一種更好的實現方式的原因。

```c++
out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} vs_out;
 
uniform vec3 lightPos;
uniform vec3 viewPos;
 
[...]
  
void main()
{    
    [...]
    mat3 TBN = transpose(mat3(T, B, N));
    vs_out.TangentLightPos = TBN * lightPos;
    vs_out.TangentViewPos  = TBN * viewPos;
    vs_out.TangentFragPos  = TBN * vec3(model * vec4(position, 0.0));
}
```

在像素著色器中我們使用這些新的輸入變量來計算切線空間的光照。因為法線向量已經在切線空間中了，光照就有意義了。

將法線貼圖應用到切線空間上，我們會得到混合教程一開始那個例子相似的結果，但這次我們可以將平面朝向各個方向，光照一直都會是正確的：

```c++
glm::mat4 model;
model = glm::rotate(model, (GLfloat)glfwGetTime() * -10, glm::normalize(glm::vec3(1.0, 0.0, 1.0)));
glUniformMatrix4fv(modelLoc 1, GL_FALSE, glm::value_ptr(model));
RenderQuad();
```

看起來是正確的法線貼圖：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_correct_tangent.png)

你可以在這裡找到[源代碼](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/normal_mapping)、[頂點](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/normal_mapping&type=vertex)和[像素](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/normal_mapping&type=fragment)著色器。

### 複雜的物體

我們已經說明了如何通過手工計算切線和副切線向量，來使用切線空間和法線貼圖。幸運的是，計算這些切線和副切線向量對於你來說不是經常能遇到的事；大多數時候，在模型加載器中實現了一次就行了，我們是在使用了Assimp的那個加載器中實現的。

Assimp有個很有用的配置，在我們加載模型的時候調用aiProcess_CalcTangentSpace。當aiProcess_CalcTangentSpace應用到Assimp的ReadFile函數時，Assimp會為每個加載的頂點計算出柔和的切線和副切線向量，它所使用的方法和我們本教程使用的類似。

```c++
const aiScene* scene = importer.ReadFile(
    path, aiProcess_Triangulate | aiProcess_FlipUVs | aiProcess_CalcTangentSpace
);
```
 

我們可以通過下面的代碼用Assimp獲取計算出來的切線空間：

```c++
vector.x = mesh->mTangents[i].x;
vector.y = mesh->mTangents[i].y;
vector.z = mesh->mTangents[i].z;
vertex.Tangent = vector;
```

然後，你還必須更新模型加載器，用以從帶紋理模型中加載法線貼圖。wavefront的模型格式（.obj）導出的法線貼圖有點不一樣，Assimp的aiTextureType_NORMAL並不會加載它的法線貼圖，而aiTextureType_HEIGHT卻能，所以我們經常這樣加載它們：

```c++
vector<Texture> specularMaps = this->loadMaterialTextures(
    material, aiTextureType_HEIGHT, "texture_normal"
);
```
 

當然，對於每個模型的類型和文件格式來說都是不同的。同樣瞭解aiProcess_CalcTangentSpace並不能總是很好的工作也很重要。計算切線是需要根據紋理座標的，有些模型製作者使用一些紋理小技巧比如鏡像一個模型上的紋理表面時也鏡像了另一半的紋理座標；這樣當不考慮這個鏡像的特別操作的時候（Assimp就不考慮）結果就不對了。

運行程序，用新的模型加載器，加載一個有specular和法線貼圖的模型，看起來會像這樣：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_complex_compare.png)

你可以看到在沒有太多點的額外開銷的情況下法線貼圖難以置信地提升了物體的細節。

使用法線貼圖也是一種提升你的場景的表現的重要方式。在使用法線貼圖之前你不得不使用相當多的頂點才能表現出一個更精細的網格，但使用了法線貼圖我們可以使用更少的頂點表現出同樣豐富的細節。下圖來自Paolo Cignoni，圖中對比了兩種方式：

![](http://learnopengl.com/img/advanced-lighting/normal_mapping_comparison.png)

高精度網格和使用法線貼圖的低精度網格幾乎區分不出來。所以法線貼圖不僅看起來漂亮，它也是一個將高精度多邊形轉換為低精度多邊形而不失細節的重要工具。

 

### 最後一件事

關於法線貼圖還有最後一個技巧要討論，它可以在不必花費太多性能開銷的情況下稍稍提升畫質表現。

當在更大的網格上計算切線向量的時候，它們往往有很大數量的共享頂點，當發下貼圖應用到這些表面時將切線向量平均化通常能獲得更好更平滑的結果。這樣做有個問題，就是TBN向量可能會不能互相垂直，這意味著TBN矩陣不再是正交矩陣了。法線貼圖可能會稍稍偏移，但這仍然可以改進。

使用叫做*格拉姆-施密特*正交化過程（Gram-Schmidt process）的數學技巧，我們可以對TBN向量進行重正交化，這樣每個向量就又會重新垂直了。在頂點著色器中我們這樣做：

```c++
vec3 T = normalize(vec3(model * vec4(tangent, 0.0)));
vec3 N = normalize(vec3(model * vec4(tangent, 0.0)));
// re-orthogonalize T with respect to N
T = normalize(T - dot(T, N) * N);
// then retrieve perpendicular vector B with the cross product of T and N
vec3 B = cross(T, N);
 
mat3 TBN = mat3(T, B, N)
```

這樣稍微花費一些性能開銷就能對法線貼圖進行一點提升。看看最後的那個附加資源： Normal Mapping Mathematics視頻，裡面有對這個過程的解釋。

### 附加資源

* [Tutorial 26: Normal Mapping](http://ogldev.atspace.co.uk/www/tutorial26/tutorial26.html)：ogldev的法線貼圖教程。
* [How Normal Mapping Works](https://www.youtube.com/watch?v=LIOPYmknj5Q)：TheBennyBox的講述法線貼圖如何工作的視頻。
* [Normal Mapping Mathematics](https://www.youtube.com/watch?v=4FaWLgsctqY)：TheBennyBox關於法線貼圖的數學原理的教程。
* [Tutorial 13: Normal Mapping](http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-13-normal-mapping/)：opengl-tutorial.org提供的法線貼圖教程。