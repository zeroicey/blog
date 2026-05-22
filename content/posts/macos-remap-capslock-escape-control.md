+++
title = 'macOS 改键实战：让中英键短按 ESC 长按 Control'
date = '2026-05-22T10:50:00+08:00'
draft = false
tags = ['macOS', 'Karabiner-Elements', '改键', '键盘优化', '效率', 'Vim', '输入法']
+++

还在纠结 macOS 上的中英切换键占用了宝贵的键盘位置？本文教你用 Karabiner-Elements 将它改造为短按 ESC、长按 Control 的高效双功能键，告别键位浪费！

# 修改系统设置

我们需要取消掉默认的 `中英切换键` 的功能

打开系统设置 -> 键盘 -> 文字输入 -> 输入法 -> 编辑

![](https://s3.blog.zeroicey.me/20260522164833466.png)

然后将 所有输入法 -> 使用大写锁定键切换"ABC"输入法 关闭

![](https://s3.blog.zeroicey.me/20260522165103082.png)

# 下载安装 Karabiner-Elements

首先去官网 [Karabiner-Elements](https://karabiner-elements.pqrs.org/) 下载

![](https://s3.blog.zeroicey.me/20260522163841720.png)

下载安装完成之后打开

![](https://s3.blog.zeroicey.me/20260522164330004.png)

然后按要求将 `setup` 栏目的所有权限打开，不然按键无法生效

![](https://s3.blog.zeroicey.me/20260522164509790.png)


# 配置 [Karabiner-Elements](https://karabiner-elements.pqrs.org/)

侧边栏 -> Complex Modifications -> Add predefined rule 然后会打开一个窗口

![](https://s3.blog.zeroicey.me/20260522165316971.png)

选择窗口正上方的 `Import more rules from the Internet (Open a web browser)` 然后会打开游览器

然后搜索 `Map Caps -> Esc (when alone), L_Ctrl (when chorded)`

![](https://s3.blog.zeroicey.me/20260522165548188.png)

选择这个作者，点击导入，然后询问是否打开 `Karabiner-Elements` 选择是，然后点击 `+ Enable`，再开启刚刚导入的配置即可

如果不存在就复制我这里的
```json
// https://ke-complex-modifications.pqrs.org/json/torma_Caps_Lock_to_Escape_or_L_Ctrl_when_Chorded.json

{
  "title": "Map Caps -> Esc (when alone), L_Ctrl (when chorded)",
  "maintainers": [
    "alextorma"
  ],
  "rules": [
    {
      "description": "Map Caps -> Esc (when alone), L_Ctrl (when chorded)",
      "manipulators": [
        {
          "description": "Map Caps -> Esc (when alone), L_Ctrl (when chorded)",
          "from": {
            "key_code": "caps_lock",
            "modifiers": {
              "optional": [
                "any"
              ]
            }
          },
          "to": [
            {
              "key_code": "left_control",
              "lazy": true
            }
          ],
          "to_if_alone": [
            {
              "key_code": "escape"
            }
          ],
          "type": "basic"
        }
      ]
    }
  ]
}
```
