# 第 4 章 View 的工作原理
## 4.1 初识 ViewRoot 和 DecorView
### 4.1.1
## 4.2 理解 MeasureSpec
### 4.2.1 MeasureSpec
### 4.2.2 MeasureSpec 和 LayoutParams 的对应关系

| 不同条件下 view 的实际 size 规则：依 TextView 设置超出屏幕大小的文字和 2 个字符做对比分别获取 TextView 实际大小 | UNSPECIFIED | EXACTLY | AT_MOST |
|---------------------------------------------------------------------------|--|--|--|
| view size 设置成固定值                                                          | view 的 size 能多大就多大且可超出父布局大小和超出自身的 size；如果小于父布局大小和自身设置的 size，则能多小也就多小 | 设置的 size 大小 | view 的 size 能多大就多大且不可超出设置的 size 大小，但可超出父布局的大小；如果小于父布局大小，则能多小也就多小 |
| view size 设置成 wrap_content 或 match_parent                                 | view 的 size 能多大就多大且可超出父布局大小；如果小于父布局大小，则能多小也就多小 | view 的 size 与父布局等大 | view 的 size 能多大就多大且不可超出父布局大小；如果小于父布局大小，则能多小也就多小 |

![](./images/4_001.png)