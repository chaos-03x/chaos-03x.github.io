---
title: 被嫌弃的按钮的一生 - ESP32实战
date: 2025-10-19 21:53:10
tags:
  - ESP32
  - DIY
categories:
  - 嵌入式
mathjax: true
---

# 被嫌弃的按钮的一生[^1] - ESP32实战

*将蘑菇按钮改造成蓝牙快捷键*



### 引言

世界上散落着无数按钮：当你按下白宫总统办公桌上的按钮，就会有一瓶健怡可乐失去生命[^2]；按下某个手提箱里的按钮，就将点燃远方的导弹发射井。按钮赋予了决定物理的重量，每一次按压，都意味着一个承诺的诞生。换言之，被按下就是它与生俱来的宿命——用哲学的话来说，这正是一种“被抛状态”（Geworfenheit）[^3]：按钮无从选择。

因此，我知道我必须做点什么，来拯救我桌上那个绿色蘑菇头按钮。

<!--more-->

材料：

- 工业按钮（自复位） - 约10元
- ESP32 - 约8元



### 准备

打开按钮的外壳，可以发现其同时提供两种模式：

- 常开：平时绝缘，按下时导通。
- 常闭：平时导通，按下时绝缘。

我选择让按钮常开，将两条导线的绝缘层剪开，将露出的铜丝分别接到常开侧的螺柱上，再将工业开关侧面预留圆孔处敲开，将导线分别接到 ESP32 的 GND 引脚和GPIO4引脚。电路连通会使 ESP32 的电平发生变化，这样即可知道按钮被按下。

示例图片：

{% collapsecard 展开 收起 %}

<div class="img-row">
    <figure class="img-col-3">
        <img src="/images/Button/Closed.jpg" alt="绝缘">
        <figcaption>图1: 平时状态：绝缘</figcaption>
    </figure>
    <figure class="img-col-3">
        <img src="/images/Button/Open.jpg" alt="导通">
        <figcaption>图2: 按钮按下：导通</figcaption>
    </figure>
    <figure class="img-col-3">
        <img src="/images/Button/ESP32.jpg" alt="ESP32引脚">
        <figcaption>图3: ESP32引脚</figcaption>
    </figure>
</div>
{% endcollapsecard %}

### 部署

这个项目仅需要一个第三方库，我使用的平台是 PlatformIO ，在项目根目录下的 `platformio.ini` 文件中，新起一行输入:

```ini
lib_deps = 
	T-vk/ESP32 BLE Keyboard
```

如果使用 Arduino 环境，可以通过库管理工具直接安装。

为了便于更改，将快捷键定义在代码顶部，参考以下表格自定义快捷键：

**常用快捷键：**

|   **按键**   |               **常量**                |
| :----------: | :-----------------------------------: |
|     Ctrl     |            `KEY_LEFT_CTRL`            |
|    Shift     |           `KEY_LEFT_SHIFT`            |
|     Alt      |            `KEY_LEFT_ALT`             |
|  Win / Cmd   |            `KEY_LEFT_GUI`             |
|   右 Ctrl    |           `KEY_RIGHT_CTRL`            |
|   右 Shift   |           `KEY_RIGHT_SHIFT`           |
|    右 Alt    |            `KEY_RIGHT_ALT`            |
| 右 Win / Cmd |            `KEY_RIGHT_GUI`            |
|   F1 - F12   |   `KEY_F1`, `KEY_F2` ... `KEY_F12`    |
|    A - Z     | `KEY_A`, `KEY_B`, `KEY_C` ... `KEY_Z` |
|    0 - 9     | `KEY_0`, `KEY_1`, `KEY_2` ... `KEY_9` |

未被列出的其余快捷键常量名（例如播放、暂停、音量），可以在[这里进行查询](https://github.com/T-vK/ESP32-BLE-Keyboard/blob/master/BleKeyboard.h)。

<br>

代码主体如下：

{% collapsecard 展开 收起 %}

```cpp
#include <Arduino.h>
#include <BleKeyboard.h>

// ---  配置快捷键  ---

// --- Ctrl + PrtSc ---
#define MODIFIER_KEY_1 KEY_LEFT_CTRL
#define MODIFIER_KEY_2 0	// 不启用修饰键2
#define MAIN_KEY       KEY_PRTSC


// --- 硬件和设备配置 ---
#define BUTTON_PIN 4 // GPIO4
const char* DEVICE_NAME = "ESP32 快捷键"; 
const char* DEVICE_MANUFACTURER = "MyESP32"; 

// --- 全局变量 ---
BleKeyboard bleKeyboard(DEVICE_NAME, DEVICE_MANUFACTURER, 100);

// --- 防抖逻辑 ---
int buttonState = HIGH;             
int lastButtonState = HIGH;         
unsigned long lastDebounceTime = 0; 
unsigned long debounceDelay = 50;   

// --- 状态 ---
bool wasConnected = false; 

void setup() {
  Serial.begin(115200); 
  Serial.println("ESP32 快捷键启动...");
  pinMode(BUTTON_PIN, INPUT_PULLUP);	// 启用上拉电阻
  bleKeyboard.begin();
}

void loop() {
  if (bleKeyboard.isConnected()) {
    
    if (!wasConnected) {
      Serial.println("蓝牙已连接，更新按钮状态...");
      wasConnected = true;
      lastButtonState = digitalRead(BUTTON_PIN);
      buttonState = lastButtonState;
      lastDebounceTime = millis();
    }

    // --- 防抖逻辑 ---
    int reading = digitalRead(BUTTON_PIN); 

    if (reading != lastButtonState) {
      lastDebounceTime = millis();
    }

    if ((millis() - lastDebounceTime) > debounceDelay) {
      
      if (reading != buttonState) {
        buttonState = reading; 

        if (buttonState == LOW) {
          Serial.println("按钮按下，发送快捷键...");
          
          // --- 快捷键发送逻辑 ---
          
          // 1. 按下修饰键1和修饰键2
          if (MODIFIER_KEY_1 != 0) {
            bleKeyboard.press((uint8_t)MODIFIER_KEY_1);
          }
          if (MODIFIER_KEY_2 != 0) {
            bleKeyboard.press((uint8_t)MODIFIER_KEY_2);
          }
          
          // 2. 按下主键
          if (MAIN_KEY != 0) { 
            bleKeyboard.press((uint8_t)MAIN_KEY);
          }
          
          // 3. 按住 100 毫秒，确保系统正确识别组合键
          delay(100); 
          
          // 4. 释放所有按键
          bleKeyboard.releaseAll(); 
          
          Serial.println("发送完毕。");
        }
      }
    }
    
    lastButtonState = reading; 
  }
  else {
    if (wasConnected) {
      Serial.println("蓝牙已断开...");
      wasConnected = false; 
    }
    Serial.println("等待蓝牙连接...");
    delay(2000);
  }
}
```

{% endcollapsecard %}

<br>

编译代码并上传后，在蓝牙连接页面找到 “ESP32 快捷键” ，连接成功后，锤一下蘑菇头，出现了截屏界面，确认按钮正常工作。

<video width="50%" controls>
  <source src="/images/Button/Example.mp4" type="video/mp4">
  您的浏览器不支持 HTML5 视频，请点击<a href="/images/my-video.mp4">此链接</a>下载。
</video>
<br>

我找了个 5V1A 的小插头为 ESP32 供电，让它 7*24 小时工作。根据电流表测算，平均功率约为 0.5W，一年 8760 小时，即：
$$
年耗电量 =\frac{0.5(W) \times 8760(h)}{1000} \approx 4.4(kWh)
$$
一度电（1kWh）算0.55元，即：
$$
电费 =4.4(kWh) \times 0.55(元) =2.42(元)
$$

电脑 USB 端口供电也是个不错的主意，这样可以让 ESP32 仅在开机时运转，但是要注意是否在 BIOS 设置中开启了 **USB 关机供电**。

### 展望

一些额外的改动可以让这个项目大为不同：

1. **复数快捷键**：`ESP32-BLE-Keyboard`支持识别键盘的三个状态灯，即 **Num Lock** (数字键盘锁定)、**Caps Lock** (大写锁定)、**Scroll Lock** (滚动锁定)。即可以考虑通过切换大小写，来定义两种不同的快捷键。
2. **识别场景**：定义一个固定快捷键，在PC端通过 `AutoHotkey` 判断不同的场景，触发不同功能。例如在视频播放器里播放/暂停，在浏览器里打开新标签页。
3. **无线化**：为其设计休眠模式，通过小型锂电池供电，将其无线化部署。
4. **复杂手势**：为单击、长按、双击定义不同逻辑。
5. **更多硬件**：利用 ESP32 的剩余引脚，可以连接好几个小型按钮，甚至编码器、 LED 灯、麦克风，实现更复杂的功能。

### 结尾

我就把按钮丢在桌子上。他那蘑菇，我们总能再见到。不过，按钮教给人升华的承诺，既否定键盘又触发按键。他也一样，断定一切皆善。这片天地从此没有了救世主，在他看来既没有更贫瘠，也不是微不足道。这按钮外壳的每一颗粒、这张棕色的书桌上每道木纹的反光，都单独为他形成一个世界。被按下去这场搏斗本身，就足以充实一颗人心。

**必须想象，这个按钮是幸福的[^4]**。

<div class="img-row">
    <figure>
        <img src="/images/Button/paragraph.jpg" class="img-small" alt="西西弗神话">
		<figcaption>西西弗神话</figcaption>
    </figure>
</div>

[^1]:被嫌弃的按钮的一生:[被嫌弃的松子的一生 (豆瓣)](https://movie.douban.com/subject/1787291/)
[^2]:健怡可乐:[据福克斯新闻报道](https://www.foxnews.com/politics/trump-brings-back-diet-coke-button-white-house-oval-office)，随着特朗普重新入主白宫，总统办公桌上的按钮的用途又变回了召唤健怡可乐。
[^3]:[被抛状态]([https://en.wikipedia.org/wiki/Thrownness):出自德国哲学家海德格尔的《存在与时间》，将人类的个体存在描述为“被投掷”到世界中。
[^4]:这段话改编自加缪的《西西弗神话》，李玉明译本，西西弗象征在荒谬世界中通过反抗与觉醒实现自由与幸福。
