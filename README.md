# EarPods 线控控制墨墨背单词（macOS）

用 EarPods 耳机线上的音量键和中间键来背单词，不用碰键盘鼠标。

在 Mac 上把耳机线控映射成背单词 App 的复习快捷键，**只在 App 处于最前台时生效**，切走就还给系统。

> 手机上做不到。iOS 把音量键收在系统层，第三方 App 既不能劫持也不能转交（这条规则是苹果为安全定的）。Mac 上通过 Karabiner-Elements 可以做到。

## 效果

| 耳机动作 | 第一次按 | 第二次按 |
|---|---|---|
| **音量 +** | 显示释义 + 发音 | 标记「认识」，跳下一词 |
| **音量 −** | 显示释义 + 发音 | 标记「忘记」，跳下一词 |
| **中键 单击** | 显示释义 + 发音 | 标记「模糊」，跳下一词 |
| **中键 长按**（约 0.4 秒） | 返回上一个单词 | — |

同一个键按两下完成一轮：先看答案，再给判断。整套动作不需要看屏幕上的按钮在哪。

## 前置条件

| 项 | 要求 |
|---|---|
| 系统 | macOS |
| 耳机 | **USB-C 版 EarPods**（3.5mm 版走耳机口的内部通道，本规则未验证） |
| 软件 | [Karabiner-Elements](https://karabiner-elements.pqrs.org/) **15.3.18 或更新**（`is_consumer` 设备匹配从这版开始支持） |
| App | 墨墨背单词 Mac 版（App Store 搜「墨墨背单词」，开发者「成都墨墨科技有限公司」） |

## 安装

### 1. 装 Karabiner-Elements 并授权

装完第一次打开时，它会依次索要三项权限，**缺一项就用不了**：

| 权限 | 位置 | 用途 |
|---|---|---|
| 输入监控 | 系统设置 → 隐私与安全性 → 输入监控 | 读取你的按键 |
| 辅助功能 | 系统设置 → 隐私与安全性 → 辅助功能 | 模拟按键发出去 |
| 驱动扩展 | 系统设置 → 通用 → 登录项与扩展 → 驱动扩展 | 虚拟键盘驱动 |

授权后建议退出 Karabiner 再打开一次，权限才会真正生效。

### 2. 导入规则

**方式 A：一键导入**（需要规则文件有一个 HTTPS 地址）

在浏览器里打开：

```
karabiner://karabiner/assets/complex_modifications/import?url=<规则文件的 HTTPS 地址>
```

**方式 B：手动放置**

把 `maimemo-earpods-mac.json` 拷到：

```
~/.config/karabiner/assets/complex_modifications/
```

### 3. 启用规则

Karabiner-Elements → **Complex Modifications** → **Add rule** → 找到「EarPods 线控控制墨墨背单词 (Mac)」，把三条规则都点 **Enable**。

### 4. 确认耳机已被接管

打开墨墨，进入复习页面，**让墨墨窗口保持在最前面**，按一下耳机音量 +。

没反应的话按下面顺序排查：

1. **Karabiner → Devices 页**，确认耳机在列表里、`Modify events` 是勾上的（没勾 = 配置里的 `ignore: true`，等于完全不管这个设备）
2. **输入监控权限**是不是被系统关掉了（系统更新后偶尔会重置）
3. **墨墨是不是最前面的窗口** —— 规则带前台限制，切走了就不生效
4. **耳机 ID 是不是和规则里写的一致** —— 见下一节

## 耳机 ID 可能不一致

规则按 USB-C EarPods 的硬件 ID 匹配：

```json
{ "vendor_id": 1452, "product_id": 4363 }
```

> 耳机的按键在系统里属于 **Consumer Control** 设备（`is_consumer`）。`is_consumer` 是 Karabiner 15.3.18 才支持的类型，这也是上面要求版本的原因。

**你的耳机 ID 如果不是这一组，规则一次都不会触发。**

查自己的 ID：打开 **Karabiner-EventViewer** → **Devices** 标签 → 找到你的耳机，看 `vendor_id` 和 `product_id`，把规则里所有 `device_if` 的两行数字改掉即可。

> `vendor_id` 是厂商 ID，1452 对应苹果。`product_id` 才是具体型号。

## 改成别的 App

规则里有两处跟「墨墨」绑定，改掉就能用在别的软件上：

**1. 前台判断的包标识符**

```json
{ "type": "frontmost_application_if", "bundle_identifiers": ["^com\\.maimemo\\.ios\\.momo$"] }
```

查目标 App 的标识符：

```bash
mdls -name kMDItemCFBundleIdentifier /Applications/你的App.app
```

**2. 发出的按键**

`key_code` 换成目标 App 实际认的快捷键。这些沿用墨墨的默认映射：

| 功能 | 键 |
|---|---|
| 显示答案 | `spacebar` |
| 认识 / 模糊 / 忘记 | `1` / `2` / `3` |
| 单词发音 | `CMD + P` |
| 返回上一个单词 | `CMD + X` |

发组合键的写法是多加一个对象：

```json
{ "key_code": "x", "modifiers": ["left_command"] }
```

## 实现要点

- **状态机**：变量 `memo_shown`（0/1）记录「这一词有没有显示释义」。没显示时按任意键 → 先翻面并发音并置 1；已显示时按 → 发出对应反馈键并置 0，跳下一词后状态自然复位。
- **中键长按**：用 `to_if_alone` + `to_if_held_down` 配合，两个超时都设 400ms。单击的响应发生在**松手瞬间**，这是长按功能的必然代价。
- **前台限制**：所有规则都带 `frontmost_application_if`。没有它，你会在别的 App 里乱敲出数字键。

## 已知限制

- **中键长按的特殊情况**：Karabiner 的 `to_if_alone` 默认 1000ms 后取消，本规则把它压到 400ms 与长按阈值对齐。按住耳机音量键不放时，部分耳机固件会连发事件，会导致状态机连续翻转 —— 正常点按不受影响。
- **返回上一个单词后**，规则会把状态重置为「未显示释义」。如果你回退到的那个词其实已经显示着答案，再按音量 + 会直接标记为「认识」。
- 仅验证过 USB-C 版 EarPods。3.5mm 版在 Mac 上是否上报 HID 事件因机型和系统版本而异，自测方法同上。

## 许可

MIT。随意取用和修改。
