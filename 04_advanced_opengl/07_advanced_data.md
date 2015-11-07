# 高級數據

原文     | [Advanced Data](http://learnopengl.com/#!Advanced-OpenGL/Advanced-Data)
      ---|---
作者     | JoeyDeVries
翻譯     | [Django](http://bullteacher.com/)
校對     | [Geequlim](http://geequlim.com)

## 緩衝數據寫入

我們在OpenGL中大量使用緩衝來儲存數據已經有一會兒了。有一些有趣的方式來操縱緩衝，也有一些有趣的方式通過紋理來向著色器傳遞大量數據。本教程中，我們會討論一些更加有意思的緩衝函數，以及如何使用紋理對象來儲存大量數據（教程中紋理部分還沒寫）。

OpenGL中緩衝只是一塊兒內存區域的對象，除此沒有更多點的了。當把緩衝綁定到一個特定緩衝對象的時候，我們就給緩衝賦予了一個特殊的意義。當我們綁定到`GL_ARRAY_BUFFER`的時候，這個緩衝就是一個頂點數組緩衝，我們也可以簡單地綁定到`GL_ELEMENT_ARRAY_BUFFER`。OpenGL內部為每個目標（target）儲存一個緩衝，並基於目標來處理不同的緩衝。

到目前為止，我們使用`glBufferData`函數填充緩衝對象管理的內存，這個函數分配了一塊內存空間，然後把數據存入其中。如果我們向它的`data`這個參數傳遞的是NULL，那麼OpenGL只會幫我們分配內存，而不會填充它。如果我們先打算開闢一些內存，稍後回到這個緩衝一點一點的填充數據，有些時候會很有用。

我們還可以調用`glBufferSubData`函數填充特定區域的緩衝，而不是一次填充整個緩衝。這個函數需要一個緩衝目標（target），一個偏移量（offset），數據的大小以及數據本身作為參數。這個函數新的功能是我們可以給它一個偏移量（offset）來指定我們打算填充緩衝的位置與起始位置之間的偏移量。這樣我們就可以插入/更新指定區域的緩衝內存空間了。一定要確保修改的緩衝要有足夠的內存分配，所以在調用`glBufferSubData`之前，調用`glBufferData`是必須的。

```c++
glBufferSubData(GL_ARRAY_BUFFER, 24, sizeof(data), &data); // 範圍： [24, 24 + sizeof(data)]
```

把數據傳進緩衝另一個方式是向緩衝內存請求一個指針，你自己直接把數據複製到緩衝中。調用`glMapBuffer`函數OpenGL會返回一個當前綁定緩衝的內存的地址，供我們操作：

```c++
float data[] = {
  0.5f, 1.0f, -0.35f
  ...
};

glBindBuffer(GL_ARRAY_BUFFER, buffer);
// 獲取當前綁定緩存buffer的內存地址
void* ptr = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
// 向緩衝中寫入數據
memcpy(ptr, data, sizeof(data));
// 完成夠別忘了告訴OpenGL我們不再需要它了
glUnmapBuffer(GL_ARRAY_BUFFER);
```

調用`glUnmapBuffer`函數可以告訴OpenGL我們已經用完指針了，OpenGL會知道你已經做完了。通過解映射（unmapping），指針會不再可用，如果OpenGL可以把你的數據映射到緩衝上，就會返回`GL_TRUE`。

把數據直接映射到緩衝上使用`glMapBuffer`很有用，因為不用把它儲存在臨時內存裡。你可以從文件讀取數據然後直接複製到緩衝的內存裡。

## 分批處理頂點屬性

使用`glVertexAttribPointer`函數可以指定緩衝內容的頂點數組的屬性的佈局(layout)。我們已經知道，通過使用頂點屬性指針我們可以交叉屬性，也就是說我們可以把每個頂點的位置、法線、紋理座標放在彼此挨著的地方。現在我們瞭解了更多的緩衝的內容，可以採取另一種方式了。

我們可以做的是把每種類型的屬性的所有向量數據批量保存在一個佈局，而不是交叉佈局。與交叉佈局123123123123不同，我們採取批量方式111122223333。

當從文件加載頂點數據時你通常獲取一個位置數組，一個法線數組和一個紋理座標數組。需要花點力氣才能把它們結合為交叉數據。使用`glBufferSubData`可以簡單的實現分批處理方式：

```c++
GLfloat positions[] = { ... };
GLfloat normals[] = { ... };
GLfloat tex[] = { ... };
// 填充緩衝
glBufferSubData(GL_ARRAY_BUFFER, 0, sizeof(positions), &positions);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions), sizeof(normals), &normals);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions) + sizeof(normals), sizeof(tex), &tex);
```

這樣我們可以把屬性數組當作一個整體直接傳輸給緩衝，不需要再處理它們了。我們還可以把它們結合為一個更大的數組然後使用`glBufferData`立即直接填充它，不過對於這項任務使用`glBufferSubData`是更好的選擇。

我們還要更新頂點屬性指針來反應這些改變：

```c++
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), 0);  
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid*)(sizeof(positions)));  
glVertexAttribPointer(
  2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(GLfloat), (GLvoid*)(sizeof(positions) + sizeof(normals)));
```

注意，`stride`參數等於頂點屬性的大小，由於同類型的屬性是連續儲存的，所以下一個頂點屬性向量可以在它的後面3（或2）的元素那兒找到。

這是我們有了另一種設置和指定頂點屬性的方式。使用哪個方式對OpenGL來說也不會有立竿見影的效果，這只是一種採用更加組織化的方式去設置頂點屬性。選用哪種方式取決於你的偏好和應用類型。

 ## 複製緩衝

當你的緩衝被數據填充以後，你可能打算讓其他緩衝能分享這些數據或者打算把緩衝的內容複製到另一個緩衝裡。`glCopyBufferSubData`函數讓我們能夠相對容易地把一個緩衝的數據複製到另一個緩衝裡。函數的原型是：

```c++
void glCopyBufferSubData(GLenum readtarget, GLenum writetarget, GLintptr readoffset, GLintptr writeoffset, GLsizeiptr size);
```

`readtarget`和`writetarget`參數是複製的來源和目的的緩衝目標。例如我們可以從一個`VERTEX_ARRAY_BUFFER`複製到一個`VERTEX_ELEMENT_ARRAY_BUFFER`，各自指定源和目的的緩衝目標。當前綁定到這些緩衝目標上的緩衝會被影響到。

但如果我們打算讀寫的兩個緩衝都是頂點數組緩衝(`GL_VERTEX_ARRAY_BUFFER`)怎麼辦？我們不能用通一個緩衝作為操作的讀取和寫入目標次。出於這個理由，OpenGL給了我們另外兩個緩衝目標叫做：`GL_COPY_READ_BUFFER`和`GL_COPY_WRITE_BUFFER`。這樣我們就可以把我們選擇的緩衝，用上面二者作為`readtarget`和`writetarget`的參數綁定到新的緩衝目標上了。

接著`glCopyBufferSubData`函數會從readoffset處讀取的size大小的數據，寫入到writetarget緩衝的writeoffset位置。下面是一個複製兩個頂點數組緩衝的例子：

```c++
GLfloat vertexData[] = { ... };
glBindBuffer(GL_COPY_READ_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sizeof(vertexData));
```

我們也可以把`writetarget`緩衝綁定為新緩衝目標類型其中之一：

```c++
GLfloat vertexData[] = { ... };
glBindBuffer(GL_ARRAY_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_ARRAY_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sizeof(vertexData));
```

有了這些額外的關於如何操縱緩衝的知識，我們已經可以以更有趣的方式來使用它們了。當你對OpenGL更熟悉，這些新緩衝方法就變得更有用。下個教程中我們會討論unform緩衝對象，彼時我們會充分利用`glBufferSubData`。
