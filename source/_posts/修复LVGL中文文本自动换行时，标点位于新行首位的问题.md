---
title: 修复LVGL中文文本自动换行时，标点位于新行首位的问题
date: 2026-05-03 00:31:15
tags:
    - LVGL
---

## 步骤

以下步骤仅在 lvgl 8.x 版本中经过测试验证。

修改 lvgl/src/misc/lv_txt.h 文件：

``` c
static inline bool _lv_txt_is_break_char(uint32_t letter)
{
    uint8_t i;
    bool ret = false;

    /* each chinese character can be break */
    if(letter >= 0x4E00 && letter <= 0x9FA5) {
        return true;
    }

    //新增
    switch (letter)
    {
    case 0x201C:    //“
    case 0x201D:    //”
    case 0x3001:    //、
    case 0x3002:    //。
    case 0xFF01:    //！
    case 0xFF08:    //（
    case 0xFF09:    //）
    case 0xFF0C:    //，
    case 0xFF1A:    //:
    case 0xFF1B:    //；
    case 0xFF1F:    //？
        return true;
    }

    /*Compare the letter to TXT_BREAK_CHARS*/
    for(i = 0; LV_TXT_BREAK_CHARS[i] != '\0'; i++) {
        if(letter == (uint32_t)LV_TXT_BREAK_CHARS[i]) {
            ret = true; /*If match then it is break char*/
            break;
        }
    }

    return ret;
}
```

## 原因

`_lv_txt_is_break_char` 函数用于判断文本中哪个字符是可以在换行时被打断的（就是在换行时留在上一行的末尾）。举一个例子，有一个文本是 `hello world!`，它包含标点和空格一共有 12 个字符，若这个文本所在的 label 的宽度只能容纳 7 个字符，就会触发换行。如果没有前面那个函数，换行结果就是：

```
hello w
orld!
```

这显然不符合常规的排版设计，`world` 作为一个单词应该在单行中被完整显示，所以当文本换行时不能在英文字母处被打断，只能在空格处被打断，正确的换行结果应该是：

```
hello 
world!
```

对于中文来说，每一个中文字符都是独立的，都可以被打断。lvgl 考虑到了这一点，所以 `_lv_txt_is_break_char` 函数中对于每一个中文字符的 unicode 值都会返回 `true`。但是 lvgl 忽略了中文标点，所以在换行时每一个中文标点都被处理为不可打断的字符，它们会与其后的中文字符一起转到下一行去显示。举一个例子，有一个文本是 `你好，世界！`，它包含标点和空格一共有 6 个字符，若这个文本所在的 label 的宽度只能容纳 3 个字符，就会触发换行。如果没有修改前面那个函数，换行结果就是：

```
你好
，世界
！
```

所以我们应该在函数中加入中文标点的判断，让中文标点像中文字符一样都可以被打断，正确的换行结果应该是：

```
你好，
世界！
```

但是中文标点的 unicode 码不像中文字符那么连续，询问 AI 得知中文标点的 unicode 码分散在至少 5 个区域中，所以这里我只是先加入了最常见的中文标点，待后续完善。
