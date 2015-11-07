# 高級光照

原文     | [Advanced Lighting](http://learnopengl.com/#!Advanced-Lighting/Advanced-Lighting)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | gjy_1992

在光照教程中，我們簡單的介紹了Phong光照模型，它給我們的場景帶來的基本的現實感。Phong模型看起來還不錯，但本章我們把重點放在一些細微差別上。



## Blinn-Phong

Phong光照很棒，而且性能較高，但是它的鏡面反射在某些條件下會失效，特別是當發光值屬性低的時候，對應一個非常大的粗糙的鏡面區域。下面的圖片展示了，當我們使用鏡面的發光值為1.0時，一個帶紋理地板的效果：

![](http://learnopengl.com/img/advanced-lighting/advanced_lighting_phong_limit.png)

你可以看到，鏡面區域邊緣迅速減弱並截止。出現這個問題的原因是在視線向量和反射向量的角度不允許大於90度。如果大於90度的話，點乘的結果就會是負數，鏡面的貢獻成分就會變成0。你可能會想，這不是一個問題，因為大於90度時我們不應看到任何光，對吧？

錯了，這隻適用於漫散射部分，當法線和光源之間的角度大於90度時意味著光源在被照亮表面的下方，這樣光的散射成分就會是0.0。然而，對於鏡面光照，我們不會測量光源和法線之間的角度，而是測量視線和反射方向向量之間的。看看下面的兩幅圖：

![](http://learnopengl.com/img/advanced-lighting/advanced_lighting_over_90.png)

現在看來問題就很明顯了。左側圖片顯示Phong反射的θ小於90度的情況。我們可以看到右側圖片視線和反射之間的角θ大於90度，這樣鏡面反射成分將會被消除。通常這也不是問題，因為視線方向距離反射方向很遠，但如果我們使用一個數值較低的發光值參數的話，鏡面半徑就會足夠大，以至於能夠貢獻一些鏡面反射的成份了。在例子中，我們在角度大於90度時消除了這個貢獻（如第一個圖片所示）。

1977年James F. Blinn引入了Blinn-Phong著色，它擴展了我們目前所使用的Phong著色。Blinn-Phong模型很大程度上和Phong是相似的，不過它稍微改進了Phong模型，使之能夠克服我們所討論到的問題。它放棄使用反射向量，而是基於我們現在所說的一個叫做半程向量（halfway vector）的向量，這是個單位向量，它在視線方向和光線方向的中間。半程向量和表面法線向量越接近，鏡面反射成份就越大。

![](http://learnopengl.com/img/advanced-lighting/advanced_lighting_halfway_vector.png)

當視線方向恰好與反射向量對稱時，半程向量就與法線向量重合。這樣觀察者距離原來的反射方向越近，鏡面反射的高光就會越強。

這裡，你可以看到無論觀察者往哪裡看，半程向量和表面法線之間的夾角永遠都不會超過90度（當然除了光源遠遠低於表面的情況）。這樣會產生和Phong反射稍稍不同的結果，但這時看起來會更加可信，特別是發光值參數比較低的時候。Blinn-Phong著色模型也正是早期OpenGL固定函數輸送管道（fixed function pipeline）所使用的著色模型。

得到半程向量很容易，我們將光的方向向量和視線向量相加，然後將結果歸一化（normalize）；

![](http://learnopengl-cn.readthedocs.org/zh/latest/img/05_01_01.png)

翻譯成GLSL代碼如下：

```c++
vec3 lightDir = normalize(lightPos - FragPos);
vec3 viewDir = normalize(viewPos - FragPos);
vec3 halfwayDir = normalize(lightDir + viewDir);
```

實際的鏡面反射的計算，就成為計算表面法線和半程向量的點乘，並對其結果進行約束（大於或等於0），然後獲取它們之間角度的餘弦，再添加上發光值參數：

```c++
float spec = pow(max(dot(normal, halfwayDir), 0.0), shininess);
vec3 specular = lightColor * spec;
```

除了我們剛剛討論的，Blinn-Phong沒有更多的內容了。Blinn-Phong和Phong的鏡面反射唯一不同之處在於，現在我們要測量法線和半程向量之間的角度，而半程向量是視線方向和反射向量之間的夾角。

!!! Important
    
    Blinn-Phong著色的一個附加好處是，它比Phong著色性能更高，因為我們不必計算更加複雜的反射向量了。

引入了半程向量來計算鏡面反射後，我們再也不會遇到Phong著色的驟然截止問題了。下圖展示了兩種不同方式下發光值指數為0.5時鏡面區域的不同效果：

![](http://learnopengl.com/img/advanced-lighting/advanced_lighting_comparrison.png)

Phong和Blinn-Phong著色之間另一個細微差別是，半程向量和表面法線之間的角度經常會比視線和反射向量之間的夾角更小。結果就是，為了獲得和Phong著色相似的效果，必須把發光值參數設置的大一點。通常的經驗是將其設置為Phong著色的發光值參數的2至4倍。

下圖是Phong指數為8.0和Blinn-Phong指數為32的時候，兩種specular反射模型的對比：

![](http://learnopengl.com/img/advanced-lighting/advanced_lighting_comparrison2.png)

你可以看到Blinn-Phong的鏡面反射成分要比Phong銳利一些。這通常需要使用一點小技巧才能獲得之前你所看到的Phong著色的效果，但Blinn-Phong著色的效果比默認的Phong著色通常更加真實一些。

這裡我們用到了一個簡單像素著色器，它可以在普通Phong反射和Blinn-Phong反射之間進行切換：

```c++
void main()
{
    [...]
    float spec = 0.0;
    if(blinn)
    {
        vec3 halfwayDir = normalize(lightDir + viewDir);  
        spec = pow(max(dot(normal, halfwayDir), 0.0), 16.0);
    }
    else
    {
        vec3 reflectDir = reflect(-lightDir, normal);
        spec = pow(max(dot(viewDir, reflectDir), 0.0), 8.0);
    }
``` 

你可以在這裡找到這個簡單的[demo的源碼](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/blinn_phong)以及[頂點](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/blinn_phong&type=vertex)和[片段](http://www.learnopengl.com/code_viewer.php?code=advanced-lighting/blinn_phong&type=fragment)著色器。按下b鍵，這個demo就會從Phong切換到Blinn-Phong光照，反之亦然。

