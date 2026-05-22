+++
title = 'macOS Key Remap: Make CapsLock Short Press ESC, Long Press Control'
date = '2026-05-22T10:50:00+08:00'
draft = false
tags = ['macOS', 'Karabiner-Elements', 'Key Remapping', 'Keyboard Optimization', 'Productivity', 'Vim', 'Input Method']
+++

Still bothered by the Chinese/English toggle key taking up valuable keyboard space on macOS? This tutorial shows you how to use Karabiner-Elements to transform it into a dual-function key: short press for ESC, long press for Control. Say goodbye to wasted key positions!

# Modify System Settings

First, we need to disable the default `Chinese/English toggle key` function.

Open System Settings -> Keyboard -> Text Input -> Input Sources -> Edit

![](https://s3.blog.zeroicey.me/20260522164833466.png)

Then turn off "Use CapsLock to switch to ABC input source" under All Input Sources

![](https://s3.blog.zeroicey.me/20260522165103082.png)

# Download and Install Karabiner-Elements

First, go to the official website [Karabiner-Elements](https://karabiner-elements.pqrs.org/) to download

![](https://s3.blog.zeroicey.me/20260522163841720.png)

After downloading and installing, open the application

![](https://s3.blog.zeroicey.me/20260522164330004.png)

Then enable all permissions in the `Setup` tab as required, otherwise the key remapping won't take effect

![](https://s3.blog.zeroicey.me/20260522164509790.png)


# Configure [Karabiner-Elements](https://karabiner-elements.pqrs.org/)

Sidebar -> Complex Modifications -> Add predefined rule, then a window will open

![](https://s3.blog.zeroicey.me/20260522165316971.png)

Select `Import more rules from the Internet (Open a web browser)` at the top of the window, then a browser will open

Search for `Map Caps -> Esc (when alone), L_Ctrl (when chorded)`

![](https://s3.blog.zeroicey.me/20260522165548188.png)

Select this author, click Import, then when asked if you want to open `Karabiner-Elements` select Yes, then click `+ Enable`, and enable the newly imported configuration

If it doesn't exist, copy this:

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