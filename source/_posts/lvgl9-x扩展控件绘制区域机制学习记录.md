---
title: lvgl9.x 扩展控件绘制区域机制学习记录
date: 2026-03-04 21:41:15
tags:
    - LVGL
---
虽然标题说是 lvgl9.x 版本，但其实 8.x 版本上也有类似的扩展控件绘制区域的机制，只不过这个需求是在进行 9.x 版本的开发时遇到的，所以先基于 9.x 版本进行研究。

## lvgl9.x 扩展控件绘制区域的示例代码

这部分的代码是我通过阅读部分源码和一点点试错后总结出来的，并不能代表正规的官方思路，以后可能需要继续修正。

``` c
/**
 * 这里的事件回调是将控件的绘制区域扩展到控件 border 外的 6 像素范围。
 * 并在扩展范围的底部绘制一个宽 24 像素，高 2 像素的矩形。
 * 扩展的像素是向上、下、左、右全方向扩展的，不能单独向一个方向上扩展。
 */
static void control_bar_btn_draw_line_cb(lv_event_t *e)
{
    lv_event_code_t code = lv_event_get_code(e);

    if(code == LV_EVENT_REFR_EXT_DRAW_SIZE)
    {
        lv_event_set_ext_draw_size(e, 6);   //扩展控件绘制区域的 api。
    }
    else if(code == LV_EVENT_DRAW_MAIN)
    {
        lv_obj_t *obj = lv_event_get_current_target_obj(e);

        lv_area_t base_area;
        lv_obj_get_coords(obj, &base_area); //获取控件的实际区域，实际区域不会因为修改了绘制区域或点击区域而改变。

        lv_area_t line_area;
        line_area.x1 = 0;
        line_area.x2 = 23;
        line_area.y1 = 0;
        line_area.y2 = 1;   //设置要绘制的矩形的实际区域，这里还是局部坐标。
        lv_area_align(&base_area, &line_area, LV_ALIGN_OUT_BOTTOM_MID, 0, 4);   
        //将要绘制的矩形的实际区域对齐到控件的实际区域的底部向下4像素的位置并居中，此时变为全局坐标。

        lv_draw_rect_dsc_t line_dsc;
        lv_draw_rect_dsc_init(&line_dsc);   //创建并初始化矩形绘制描述符，用来描述矩形的样式信息。

        lv_layer_t *draw_layer = lv_event_get_layer(e);     //获取控件的绘制层级，要将矩形绘制到同一层级上。
        lv_draw_rect(draw_layer, &line_dsc, &line_area);    //绘制矩形。
    }
}
...
{
    ...
    /**
     * 在测试控件的 LV_EVENT_DRAW_MAIN 和 LV_EVENT_REFR_EXT_DRAW_SIZE 事件挂载上面的事件回调就可以了。
     */
    lv_obj_add_event_cb(btn, control_bar_btn_draw_line_cb, LV_EVENT_DRAW_MAIN, NULL);
    lv_obj_add_event_cb(btn, control_bar_btn_draw_line_cb, LV_EVENT_REFR_EXT_DRAW_SIZE, NULL);
    ...
}
```

可以发现实际扩展控件绘制区域的代码只有一行：`lv_event_set_ext_draw_size`，下面我们通过源码来追溯一下流程和机制。

## lvgl9.x 扩展控件绘制区域的相关源码

首先就是 `lv_event_set_ext_draw_size` 函数，源码如下：

``` c
/* from lvgl/src/core/lv_obj_event.c */

/**
 * 设置新的扩展绘制尺寸. 可用于 LV_EVENT_REFR_EXT_DRAW_SIZE 事件中。
 * @param e     指向一个事件的指针。
 * @param size  新的扩展绘制尺寸。
 */
void lv_event_set_ext_draw_size(lv_event_t * e, int32_t size)
{
    if(e->code == LV_EVENT_REFR_EXT_DRAW_SIZE) {
        int32_t * cur_size = lv_event_get_param(e);
        *cur_size = LV_MAX(*cur_size, size);
    }
    else {
        LV_LOG_WARN("Not interpreted with this event code");
    }
}
```

通过源码可以发现，虽然官方的函数说明中表示 “可用于 `LV_EVENT_REFR_EXT_DRAW_SIZE` 事件中”，但应该是只能用于 `LV_EVENT_REFR_EXT_DRAW_SIZE` 事件中，如果当前事件类型不是 `LV_EVENT_REFR_EXT_DRAW_SIZE` 是会抛出异常警告的。这个函数做的事情很简单，只是通过指针改变了事件参数的值，改为了用户新设置的扩展绘制尺寸和当前事件参数中的最大值，可以猜测这个事件参数也代表了某种情况下扩展绘制尺寸的值。
想要知道这个事件参数究竟有什么作用，只能去查看 `LV_EVENT_REFR_EXT_DRAW_SIZE` 事件的来源了。全局搜索后发现，能产生这个事件的只有 `lv_obj_refresh_ext_draw_size` 函数，源码如下：

``` c
/* from lvgl/src/core/lv_obj_draw.c */

/**
 * 发送一个 LV_EVENT_REFR_EXT_DRAW_SIZE 事件来调用父级对象的事件处理程序以刷新扩展绘制尺寸的值。
 * 修改结果将存储在控件的数据中。
 * @param obj   指向控件的指针。
 */
void lv_obj_refresh_ext_draw_size(lv_obj_t * obj)
{
    LV_ASSERT_OBJ(obj, MY_CLASS);

    int32_t s_old = lv_obj_get_ext_draw_size(obj);
    int32_t s_new = 0;
    lv_obj_send_event(obj, LV_EVENT_REFR_EXT_DRAW_SIZE, &s_new);

    /* 如果已为特殊属性分配了内存，则将结果存储起来。*/
    if(obj->spec_attr) {
        obj->spec_attr->ext_draw_size = s_new;
    }
    /* 只有在结果不为零的情况下，才分配特殊属性的内存。
     * 如果未定义 spec.attr，则扩展绘制尺寸默认为 0。*/
    else if(s_new != 0) {
        lv_obj_allocate_spec_attr(obj);
        obj->spec_attr->ext_draw_size = s_new;
    }

    if(s_new != s_old) lv_obj_invalidate(obj);
}
```

可以发现，事件参数确实代表了扩展绘制尺寸，是可能会存储的最新的扩展绘制尺寸。在这个函数中，会先缓存一份当前的扩展绘制尺寸，然后触发 `LV_EVENT_REFR_EXT_DRAW_SIZE` 事件来让用户通过指针设置新的扩展绘制尺寸，如果新的扩展绘制尺寸不等于 0（如果是通过 `lv_event_set_ext_draw_size` 函数来设定新的扩展绘制尺寸，新的值不会小于 0），就会将新的结果存储起来。如果新的扩展绘制尺寸与当前的尺寸不一致，就会触发重绘。扩展绘制尺寸的值存储在控件的 `spec_attr` 变量中，该变量的数据类型的声明如下：

``` c
/* from lvgl/src/core/lv_obj_private.h */

/**
 * 特殊且不常用的属性。
 * 如果设置了其中的任何元素的值，它们就会自动分配内存。
 */
struct lv_obj_spec_attr_t {
    lv_obj_t ** children;           /**< 将子控件的指针存储在一个数组中。*/
    lv_group_t * group_p;
    lv_event_list_t event_list;

    lv_point_t scroll;              /**< 当前滚动偏移量的 X/Y 值。*/

    int32_t ext_click_pad;          /**< 向所有方向扩展的额外点击区域尺寸。*/
    int32_t ext_draw_size;          /**< 向所有方向扩展的额外绘制区域尺寸。*/

    uint16_t child_cnt;             /**< 子控件的数量。*/
    uint16_t scrollbar_mode : 2;    /**< 如何显示滚动条, 详见 lv_scrollbar_mode_t */
    uint16_t scroll_snap_x : 2;     /**< 子控件在 X 轴方向上的滚动对齐模式, 详见 lv_scroll_snap_t */
    uint16_t scroll_snap_y : 2;     /**< 子控件在 Y 轴方向上的滚动对齐模式, 详见 lv_scroll_snap_t */
    uint16_t scroll_dir : 4;        /**< 允许的滚动方向, 详见 lv_dir_t */
    uint16_t layer_type : 2;        /**< 层类型。详见 lv_intermediate_layer_type_t */
};
```

涉及修改扩展绘制尺寸的核心源码应该只有上面的两个函数了，但是我们依然对这部分的处理机制有几个疑问：

1. 为什么修改扩展绘制尺寸需要通过注册 `LV_EVENT_REFR_EXT_DRAW_SIZE` 事件回调来修改。

2. `LV_EVENT_REFR_EXT_DRAW_SIZE` 事件会在什么情况下被触发。

这些问题都需要进一步的追踪研究。

## lvgl9.x 扩展控件绘制区域的机制研究

经过进一步查阅源码，发现 lvgl 已经把扩展绘制尺寸的机制融入到绘制流程的方方面面了，不彻底了解 lvgl 完整的绘制流程很难说清楚绘制区域的更新机制，这里只能先尝试简单说明一下上面的问题。
对于同一个控件，它在处于不同的状态，不同的绘制步骤时，它所需要的扩展绘制尺寸是不一样的，无法在绘制流程之外就把扩展绘制尺寸定死。所以 lvgl 的绘制管线会在若干个关键的绘制步骤中调用 `lv_obj_refresh_ext_draw_size` 函数触发 `LV_EVENT_REFR_EXT_DRAW_SIZE` 事件，通过回调函数来动态获取当前绘制步骤下适当的扩展绘制尺寸。

(To be continued ...)
