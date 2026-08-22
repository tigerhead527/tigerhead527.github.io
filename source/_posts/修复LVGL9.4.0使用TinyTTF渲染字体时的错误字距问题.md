---
title: 修复LVGL_9.4.0使用TinyTTF渲染字体时的错误字距问题
date: 2026-08-13 10:31:15
tags:
    - LVGL
---
以下步骤仅在 lvgl 9.4.0 版本中经过测试验证，更高版本的 lvgl 似乎已经修复了此问题。
在 lv_conf.h 中开启 LV_USE_TINY_TTF 选项并设置 LV_TINY_TTF_CACHE_GLYPH_CNT 大于 0 时会发生此问题。

## 问题现象

团队在进行 AR 眼镜项目的开发，使用了 lvgl 内置的 TinyTTF 库（其实是 stb_truetype 库）渲染字体。其中有一个 AI 翻译页面需要同时渲染中文和英文的字型，在测试时发现了下面的问题：当先渲染英文文本后渲染中文文本时，英文文本正常显示，而中文文本的某些字符发生了横向重叠现象，后一个字从前一个字的中间处开始渲染，两个字左右各一半的区域被重叠；当先渲染中文文本后渲染英文文本时，中文文本正常显示，而英文文本的某些字符发生了横向字距过大的现象，后一个字与前一个字中间多出一个空格的距离。
由于这两种现象都是只在横向方向上发生错位，而竖向方向没有偏差，所以初步认定是 TinyTTF 库的字距（这里是指 Kerning）计算有误。

## 问题排查

经过学习 TTF 字体格式和阅读 TinyTTF 的部分源码，并没有找到 TinyTTF 的问题。最终发现，是 lvgl 在管理 TinyTTF 的字型信息缓存时犯了低级错误。
如果开启了 TinyTTF 的缓存（LV_TINY_TTF_CACHE_GLYPH_CNT 大于 0），TinyTTF 在计算前后两个字型的字距后，lvgl 会将这两个字型的 unicode 码和计算得到的字距缓存起来。下次再需要计算字距时，lvgl 会用新字型的一对 unicode 码在缓存中查找，如果找到就直接复用之前的计算结果了。但在查找缓存的对比函数中，lvgl 犯了一个低级错误。对比函数让查找值和缓存值的两对 unicode 码相减，若相减结果为 0 则代表找到缓存，而 unicode 码是 32 位数据，lvgl 却使用了一个 8 位数据类型（lv_cache_compare_res_t）去存储两个 unicode 码相减的结果。由于结果值的数据截断，导致部分字型的对比会查找到错误的缓存。

## 解决步骤

修改 lvgl\src\libs\tiny_ttf\lv_tiny_ttf.c 文件：

``` c
static lv_cache_compare_res_t tiny_ttf_kerning_cache_compare_cb(const tiny_ttf_kerning_cache_data_t * lhs,
                                                                const tiny_ttf_kerning_cache_data_t * rhs)
{
    // 修改前：
    // lv_cache_compare_res_t ret = lhs->glyph1_idx - rhs->glyph1_idx;
    // if(ret == 0) {
    //     return lhs->glyph2_idx - rhs->glyph2_idx;
    // }
    // return ret;

    // 修改后：
    if (lhs->glyph1_idx != rhs->glyph1_idx) {
        return lhs->glyph1_idx > rhs->glyph1_idx ? 1 : -1;
    }
    if (lhs->glyph2_idx != rhs->glyph2_idx) {
        return lhs->glyph2_idx > rhs->glyph2_idx ? 1 : -1;
    }
    return 0;
}
```
