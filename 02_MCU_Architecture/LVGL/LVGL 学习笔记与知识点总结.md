# LVGL 学习笔记与知识点总结

本文档基于 `ui.c` 源码整理，涵盖了从对象创建、布局、样式到事件处理及观察者模式的高级用法。

## 1. 基础对象管理 (Objects & Hierarchy)

LVGL 中万物皆对象 (`lv_obj_t`)，所有组件都继承自基础对象。

### 1.1 创建与层级

- **获取当前屏幕**：

  C

  ```c
  lv_obj_t* scr = lv_scr_act();
  ```

- **创建对象**：

  C

  ```c
  // 在屏幕上创建
  lv_obj_t* div = lv_obj_create(scr); 
  // 在另一个对象(父对象)上创建，形成父子关系
  lv_obj_t* subDiv = lv_obj_create(div); 
  ```

- **父子关系操作**：

  - `lv_obj_get_parent(obj)`: 获取父节点。
  - `lv_obj_get_child(parent, index)`: 根据索引获取子节点（0是第一个创建的）。
  - *(注：代码中出现的 `lv_obj_set_name` 和 `lv_obj_get_child_by_name` 通常需要开启特定的配置宏 `LV_USE_OBJ_NAME` 才能使用，非默认开启)*。

- **删除对象**：

  - `lv_obj_delete_delayed(obj, time_ms)`: 延时删除组件。

### 1.2 图层顺序 (Z-Order)

LVGL 的渲染顺序默认是“后创建的在上面”。

- **调整层级**：

  C

  ```c
  // 获取当前层级索引
  int32_t index = lv_obj_get_index(div3);
  // 移动到指定层级 (index越小越在底层)
  lv_obj_move_to_index(div3, index - 1); 
  ```

------

## 2. 布局与定位 (Position & Layout)

### 2.1 大小设置

- **固定像素**：

  C

  ```
  lv_obj_set_size(div, 100, 50); // 宽100, 高50
  ```

- **百分比 (响应式)**：

  C

  ```
  lv_obj_set_size(div, lv_pct(100), lv_pct(30)); // 占父容器宽度的100%，高度的30%
  ```

### 2.2 位置与对齐

- **绝对坐标**：

  C

  ```
  lv_obj_set_pos(div, 50, 50); // 相对于父容器左上角的 x, y
  ```

- **自身内部对齐**：

  C

  ```c
  // 让 div 在父容器内部居中，偏移 (0, -50)
  lv_obj_align(div, LV_ALIGN_CENTER, 0, -50);
  // 常用对齐宏：LV_ALIGN_TOP_MID, LV_ALIGN_BOTTOM_MID, LV_ALIGN_LEFT_MID 等
  ```

- **相对外部对齐 (锚点对齐)**：

  C

  ```c
  // 让 div2 位于 div 的“正下方”
  lv_obj_align_to(div2, div, LV_ALIGN_OUT_BOTTOM_MID, 0, 0);
  ```

- **刷新布局**：

  - `lv_obj_update_layout(obj)`: 如果需要立即获取更新后的坐标（`lv_obj_get_x`），需手动调用此函数刷新，否则默认在下一帧渲染时才更新。

------

## 3. 样式系统 (Styles)

LVGL 的样式非常类似 CSS，分为“本地样式”和“共享样式”。

### 3.1 核心概念：选择器 (Selector)

样式应用时由 `Part` (部件) 和 `State` (状态) 组合而成。

- **Part (部件)**：
  - `LV_PART_MAIN`: 主体背景。
  - `LV_PART_SCROLLBAR`: 滚动条。
  - `LV_PART_INDICATOR`: 指示器（如开关的填充色、进度条的当前值）。
  - `LV_PART_KNOB`: 旋钮（滑块或开关的圆头）。
- **State (状态)**：
  - `LV_STATE_DEFAULT`: 默认状态。
  - `LV_STATE_CHECKED`: 选中/开启状态。
  - `LV_STATE_PRESSED`: 按下状态。
- **组合写法**：`LV_PART_MAIN | LV_STATE_CHECKED`

### 3.2 方式一：本地样式 (直接设置)

针对单个对象，最简单快捷。

C

```c
// 设置背景色 (需要 lv_palette_main 或 lv_color_hex)
lv_obj_set_style_bg_color(obj, lv_palette_main(LV_PALETTE_RED), LV_PART_MAIN);
// 设置边框
lv_obj_set_style_border_width(obj, 10, LV_PART_MAIN);
// 设置内边距 (Padding)
lv_obj_set_style_pad_all(obj, 0, LV_PART_MAIN);
// 设置透明度
lv_obj_set_style_bg_opa(obj, LV_OPA_COVER, LV_PART_MAIN);
```

### 3.3 方式二：共享样式 (`lv_style_t`)

创建一种风格，应用给多个对象。

C

```c
// 1. 必须是 static 或全局变量
static lv_style_t style;
lv_style_init(&style);

// 2. 配置样式属性
lv_style_set_bg_color(&style, lv_color_white());
lv_style_set_text_color(&style, lv_color_black());

// 3. 添加到对象
lv_obj_add_style(obj, &style, LV_PART_MAIN);
```

------

## 4. 常用组件 (Widgets)

### 4.1 Label (标签)

- `lv_label_create(parent)`
- `lv_label_set_text(label, "Text")`: 设置静态文本。
- `lv_label_set_text_fmt(label, "%d", val)`: 格式化设置文本（类似 printf）。

### 4.2 Button (按钮)

- `lv_button_create(parent)`
- `lv_obj_add_flag(bt, LV_OBJ_FLAG_CHECKABLE)`: **关键**，开启后按钮具有“开关”特性（Toggle 模式），可以保持 `CHECKED` 状态。

### 4.3 Slider (滑动条)

- `lv_slider_create(parent)`
- `lv_slider_set_range(slider, 0, 100)`: 设置范围。
- `lv_slider_set_value(slider, 60, LV_ANIM_OFF)`: 设置当前值（关闭动画）。

### 4.4 Dropdown (下拉列表)

- `lv_dropdown_create(parent)`

- `lv_dropdown_set_options(dp, "A\nB\nC")`: 设置选项，用 `\n` 分隔。

- **特殊技巧**：获取弹出的列表部分来修改样式。

  C

  ```c
  lv_obj_t * list = lv_dropdown_get_list(dp);
  lv_obj_set_style_bg_color(list, ..., LV_PART_MAIN);
  ```

------

## 5. 事件系统 (Events)

传统的事件处理方式。

### 5.1 添加回调

C

```c
// 参数：对象，回调函数，触发事件类型，用户数据(User Data)
lv_obj_add_event_cb(slider, slider_callback, LV_EVENT_VALUE_CHANGED, lb);
```

### 5.2 回调函数写法

C

```c
void slider_callback(lv_event_t *e)
{
    // 1. 获取用户数据 (即传入的 lb)
    lv_obj_t *userData = lv_event_get_user_data(e);
    // 2. 获取触发事件的对象 (即 slider)
    lv_obj_t *target = lv_event_get_target_obj(e);
    
    // 3. 执行逻辑
    int32_t value = lv_slider_get_value(target);
    lv_label_set_text_fmt(userData, "%d", value);
}
```

------

## 6. 观察者模式 (Observer / Data Binding)

这是代码中较高级的用法，实现了数据与 UI 的自动同步（类似 MVVM）。

### 6.1 初始化 Subject (数据源)

C

```c
static lv_subject_t subject; // 全局或静态定义
lv_subject_init_int(&subject, 0); // 初始化为整数 0
```

### 6.2 读写数据

- `lv_subject_set_int(&subject, 1)`: 修改数据，所有绑定的 UI 会自动更新。
- `lv_subject_get_int(&subject)`: 读取当前数据。

### 6.3 绑定 UI (Binding)

- **绑定数值**：(代码中注释部分)

  - `lv_slider_bind_value(slider, &subject)`: 滑块动 -> 数据变；数据变 -> 滑块动。
  - `lv_label_bind_text(lb, &subject, "%d")`: 数据变 -> 标签文字自动变。

- **绑定状态**：

  - `lv_obj_bind_checked(bt, &subject)`: 按钮按下/弹起 <-> 数据 1/0 自动同步。

- **绑定样式 (实现换肤)**：

  C

  ```c
  // 当 subject == 0 时，应用 lightStyle
  lv_obj_bind_style(dp, &lightStyle, LV_PART_MAIN, &subject, 0);
  // 当 subject == 1 时，应用 greyStyle
  lv_obj_bind_style(dp, &greyStyle, 0, &subject, 1);
  ```

------

## 7. 调试小技巧

- **Printf 调试**：代码中使用了 standard `printf` 来打印逻辑判断（如 `div == parent`）。
- **Visual Studio Code 提示**：鼠标悬停在函数名上可以看到参数类型（如 `lv_event_t *e`），这对理解回调函数参数非常重要。



# 2026-02-06 补充笔记 (进阶布局与美化)

本章节涵盖了网格布局的细节、滚动机制的控制、下拉框的深度美化以及 UI 架构优化的实战经验。

## 8. 高级布局系统 (Layouts)

LVGL 提供了两种强大的布局引擎，代替手动的 `set_pos`，适配不同尺寸屏幕。

### 8.1 弹性布局 (Flex Layout)

适用于**一维**排列（单行或单列），类似于穿珠子，内容多了自动往下排。

- **开启布局**：`lv_obj_set_layout(div, LV_LAYOUT_FLEX)`
- **流向控制**：`lv_obj_set_flex_flow(div, LV_FLEX_FLOW_COLUMN)` (垂直) 或 `ROW` (水平)。
- **对齐方式**：`lv_obj_set_flex_align(...)` 控制主轴（Main Axis）和交叉轴（Cross Axis）的对齐。

### 8.2 网格布局 (Grid Layout)

适用于**二维**排列（严格的行列），类似于 Excel 表格或九宫格。

- **1. 定义骨架 (描述符)**： 使用 `LV_GRID_FR(x)` 定义比例 (Fraction)，自适应剩余空间。

  C

  ```c
  // 两列：各占 1 份 (即 50% / 50%)
  static int32_t col_dsc[] = {LV_GRID_FR(1), LV_GRID_FR(1), LV_GRID_TEMPLATE_LAST}; 
  // 三行：高度比例为 3:2:2
  static int32_t row_dsc[] = {LV_GRID_FR(3), LV_GRID_FR(2), LV_GRID_FR(2), LV_GRID_TEMPLATE_LAST}; 
  
  // 应用描述符
  lv_obj_set_grid_dsc_array(div, col_dsc, row_dsc);
  ```

- **2. 放置子控件 (Cell 定位)**： 需要指定对象放在第几列、第几行，以及是否跨列/跨行。

  C

  ```c
  // 参数：对象, 水平对齐, 第几列(从0起), 占几列, 垂直对齐, 第几行(从0起), 占几行
  lv_obj_set_grid_cell(obj, LV_GRID_ALIGN_STRETCH, 0, 1, LV_GRID_ALIGN_STRETCH, 0, 1);
  ```

- **3. 去除间隙 (Gap)**： 默认网格之间有缝隙，若要实现背景无缝拼接：

  C

  ```
  lv_obj_set_style_pad_row(div, 0, LV_PART_MAIN);    // 行间距设为0
  lv_obj_set_style_pad_column(div, 0, LV_PART_MAIN); // 列间距设为0
  ```

## 9. 滚动控制 (Scrolling)

- **滚动条独立样式**： 滚动条属于特定的部件 `LV_PART_SCROLLBAR`，可以单独上色，不影响容器背景。

  C

  ```c
  // 只把滚动条染成红色
  lv_obj_set_style_bg_color(div, lv_palette_main(LV_PALETTE_RED), LV_PART_SCROLLBAR);
  ```

- **滚动链 (Scroll Chaining)**：

  - **概念**：当子容器滑到尽头时，滑动事件是否“冒泡”传给父容器。
  - **控制**：`LV_OBJ_FLAG_SCROLL_CHAIN`。
  - **应用**：如果不想让内部列表滚动时误触外部页面滚动，可清除此标志。

## 10. 控件深度美化 (Styling Deep Dive)

### 10.1 调色板进阶 (Palette Darken)

LVGL 内置颜色不仅有标准色 (Main)，还有深浅变体，是制作高级暗色模式的神器。

- **函数**：`lv_palette_darken(LV_PALETTE_GREY, level)` (Level 范围 1~4)
- **应用技巧**：
  - **Level 1-2**: 适合做深色模式下的边框。
  - **Level 3**: **完美暗黑背景**（深灰色）。相比纯黑 (`#000000`)，Level 3 的深灰更柔和，能减少 OLED 拖影和视觉疲劳，提升高级感。

### 10.2 下拉框 (Dropdown) 的特殊机制

下拉框由两部分组成：**按钮本体** 和 **弹出列表**。 直接给本体设置样式，**不会**自动应用到弹出列表。

- **解决方案**：手动获取列表对象并单独绑定。

  C

  ```c
  // 1. 获取弹出列表对象
  lv_obj_t* list = lv_dropdown_get_list(dd);
  // 2. 给列表也绑定同样的暗色样式
  lv_obj_bind_style(list, &style_night, LV_PART_MAIN, &subject, 1);
  ```

### 10.3 消除 UI“钝感”

- **圆角 (Radius)**：`lv_style_set_radius(&style, 6)`。适度的圆角是现代 UI 的特征。
- **边框 (Border)**：在暗色模式下，避免使用纯白边框。使用深灰色 (`darken 1` 或 `2`) 代替白色，能让界面更融合。

## 11. 架构优化 (Refactoring)

从“能跑就行”进阶到“易于维护”。

- **痛点**：`ui_common_bindStyle` 这种通用函数虽然方便，但如果内部写死了全局样式（如 `&darkStyle`），会导致所有控件在暗色模式下长得一模一样，无法单独定制（例如：无法单独把输入框做成深灰，而背景做成纯黑）。

- **最佳实践：专用样式组**。 不要复用为容器设计的 `lightStyle`。为交互控件单独定义静态样式（如 `style_input_day` / `style_input_night`），并在其中封装好圆角、边框、内边距等细节。

  C

  ```c
  // 伪代码示例：更解耦的写法
  static lv_style_t style_input_night;
  lv_style_init(&style_input_night);
  // 在样式内部封装细节，而不是在创建对象时手写 set_style
  lv_style_set_radius(&style_input_night, 6); 
  
  // 绑定时直接使用专用样式
  lv_obj_bind_style(obj, &style_input_night, LV_PART_MAIN, &subject, 1);
  ```