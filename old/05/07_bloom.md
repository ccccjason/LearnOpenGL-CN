本文作者JoeyDeVries，由Meow J翻譯自[http://learnopengl.com](http://learnopengl.com/#!Advanced-Lighting/Bloom)

## 泛光(Bloom)

亮光源與被光照區域常常很難傳達給觀眾，因為顯示器的光強度範圍是受限制的. 其中一個解決方案是讓這些光源泛光(Glow)從而區分亮光源: 讓光漏(bleed)出光源. 這有效讓觀眾感覺到光源和光照區域非常的亮.

光的是通過一個後期處理效果叫做泛光(Bloom)來完成的. 泛光讓所有光照地區一種在發光的效果.下面兩個場景就是有泛光(右)與無泛光(左)的區別(圖像來自於虛幻引擎(Unreal)):

![](http://learnopengl.com/img/advanced-lighting/bloom_example.png)

泛光提供了對於物體亮度顯眼的視覺暗示，因為泛光能給我們物體真的是很亮的視覺效果. 如果我們能夠很好的完成它(有一些遊戲實現的很糟糕)，泛光將能很大程度的加強我們場景的光照效果，並且也能給我們很多特效.