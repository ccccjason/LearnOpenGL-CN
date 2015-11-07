# 校對指南(Styleguide)

- **對照原文**
- 使用中文的標點符號: 逗號，雙引號
- 使用英文的標點符號: 括號(去討論下剩下的，像句號之類的)
- 文本中常量/代碼用``加註為代碼
- 代碼塊不使用Tab標註，改為用```式標註
- 每一節標題使用`#` (h1)標題(到後面會有幾節有很多小節的，太小的標題不明顯)
	- 根據原文標題大小逐漸遞增標題
	- 注意"#"後標題前要空格
- 專有名詞在對照表裡找翻譯，需要用括號標註原文
	- 原文單詞按照標題大寫規則大寫首字母
	- 儘量大部分都翻譯掉
- 在Markdown文件中如需插入圖片或者代碼，請與正文空一行

例:

```markdown
[text]

[img]

[text]
```

- 翻譯註釋
- 原文中的斜體一律用加粗表示(中文並不存在斜體)
- 每一節前面加上

```
原文     | [英文標題](原文地址)
      ---|---
作者     | 作者
翻譯     | 翻譯
校對     | 校對
```

- 公式用[http://latex2png.com/](http://latex2png.com/)轉換，文本中的公式Resolution為150，單獨的公式為200
- 視頻用Video標籤添加

```html
<video src="url" controls="controls">
</video>
```

- 如果有比較長的譯註:

```
xxxx(([譯註x])

[譯註x]: http://learnopengl-cn.readthedocs.org "text"
```
效果會像這樣([譯註1])


**如果有不全的繼續加**

[譯註1]: http://learnopengl-cn.readthedocs.org "bakabaka我是譯註"