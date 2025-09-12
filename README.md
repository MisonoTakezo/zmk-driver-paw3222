# ZMK PAW3222 Driver

This driver enables the use of the PIXART PAW3222 optical sensor with the ZMK framework, including scroll functionality.

## Overview

The PAW3222 is a low-power optical mouse sensor suitable for tracking applications such as mice and trackballs. This driver communicates with the PAW3222 sensor via SPI interface and supports multiple input modes including mouse movement, scrolling, and snipe mode.

## Features

- Mouse movement tracking
- Scroll wheel functionality (horizontal and vertical)
- Snipe mode for precise movement
- Layer-based input mode switching
- Automouse layer activation
- Configurable CPI and sensitivity

## Installation

1. Add as a ZMK module in your west.yml:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: sekigon-gonnoc
      url-base: https://github.com/sekigon-gonnoc
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    - name: zmk-driver-paw3222
      remote: sekigon-gonnoc
      revision: main
```

## Device Tree Configuration

Configure in your shield or board config file (.overlay or .dtsi):

```dts
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 12)>,
                <NRF_PSEL(SPIM_MOSI, 1, 9)>,
                <NRF_PSEL(SPIM_MISO, 1, 9)>;
        };
    };

    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 12)>,
                <NRF_PSEL(SPIM_MOSI, 1, 9)>,
                <NRF_PSEL(SPIM_MISO, 1, 9)>;
            low-power-enable;
        };
    };
};

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 13 GPIO_ACTIVE_LOW>;

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,paw3222";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 15 GPIO_ACTIVE_LOW>;
        
        /* Basic sensor configuration */
        res-cpi = <800>;
        force-awake;
        
        /* Scroll mode configuration */
        scroll-layers = <1 2>;  /* Layers where scroll mode is active */
        scroll-tick = <20>;     /* Movement threshold for scroll events */
        
        /* Snipe mode configuration */
        snipe-layers = <3>;     /* Layers where snipe mode is active */
        snipe-cpi = <400>;      /* Lower CPI for precise movement */
        snipe-cpi-dividor = <2>; /* Additional sensitivity reduction */
        
        /* Automouse layer */
        automouse-layer = <4>;     /* Layer to activate on movement */
        automouse-timeout-ms = <600>; /* Timeout before deactivation */
        
        /* Movement sensitivity */
        cpi-dividor = <1>;      /* Normal mode sensitivity divider */
    };
};
```

## Enable the module in your keyboard's Kconfig file

Add the following to your keyboard's `Kconfig.defconfig`:

```kconfig
if ZMK_KEYBOARD_YOUR_KEYBOARD

config ZMK_POINTING
    default y

config ZMK_MOUSE
    default y

config PAW3222
    default y

endif
```

## Configuration Properties

### Basic Properties
- `irq-gpios`: GPIO connected to the motion pin (required)
- `power-gpios`: GPIO connected to the power control pin (optional)
- `res-cpi`: Default CPI resolution for the sensor (default: 800)
- `force-awake`: Initialize the sensor in "force awake" mode (optional, boolean)

### Scroll Mode Properties
- `scroll-layers`: Array of layer numbers where scroll mode is active
- `scroll-tick`: Movement threshold for generating scroll events (default: 20)

### Snipe Mode Properties  
- `snipe-layers`: Array of layer numbers where snipe mode is active
- `snipe-cpi`: CPI resolution for snipe mode (default: same as res-cpi)
- `snipe-cpi-dividor`: Movement divider for snipe mode (default: 1)

### Automouse Properties
- `automouse-layer`: Layer number to automatically activate on movement (0 = disabled)
- `automouse-timeout-ms`: Timeout for deactivating automouse layer (default: 600ms)

### Sensitivity Properties
- `cpi-dividor`: Movement divider for normal mode (default: 1)

## Usage Examples

### Basic Mouse Setup
```dts
trackball: trackball@0 {
    compatible = "pixart,paw3222";
    /* ... basic SPI config ... */
    res-cpi = <1200>;
    force-awake;
};
```

### Mouse with Scroll Layer
```dts
trackball: trackball@0 {
    compatible = "pixart,paw3222";
    /* ... basic SPI config ... */
    res-cpi = <800>;
    scroll-layers = <1>;  /* Layer 1 = scroll mode */
    scroll-tick = <15>;   /* More sensitive scrolling */
};
```

### Full Featured Setup
```dts
trackball: trackball@0 {
    compatible = "pixart,paw3222";
    /* ... basic SPI config ... */
    res-cpi = <1000>;
    
    /* Scroll on layer 1 */
    scroll-layers = <1>;
    scroll-tick = <20>;
    
    /* Snipe mode on layer 2 */
    snipe-layers = <2>;
    snipe-cpi = <400>;
    snipe-cpi-dividor = <2>;
    
    /* Automouse activates layer 3 */
    automouse-layer = <3>;
    automouse-timeout-ms = <800>;
    
    /* Reduce normal sensitivity slightly */
    cpi-dividor = <2>;
};
```

## Layer Configuration in ZMK

Configure your keymap to include layers for different modes:

```c
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer {
            bindings = <
                &kp Q &kp W &kp E &mo 1    // mo 1 = momentary layer 1 (scroll)
                &kp A &kp S &kp D &mo 2    // mo 2 = momentary layer 2 (snipe)
            >;
        };

        scroll_layer {  // Layer 1 - trackball becomes scroll wheel
            bindings = <
                &kp ESC &kp TAB &kp ENTER &trans
                &kp LSHIFT &kp LCTRL &kp LALT &trans
            >;
        };

        snipe_layer {   // Layer 2 - trackball becomes precise cursor
            bindings = <
                &kp F1 &kp F2 &kp F3 &trans
                &kp LGUI &kp LALT &kp LCTRL &trans
            >;
        };

        mouse_layer {   // Layer 3 - automouse layer
            bindings = <
                &mkp LCLK &mkp RCLK &mkp MCLK &trans
                &kp LSHIFT &kp LCTRL &kp LALT &trans
            >;
        };
    };
};
```

## Troubleshooting

1. **Sensor not detected**: Check SPI wiring and IRQ GPIO configuration
2. **No scroll events**: Verify scroll-layers configuration and ensure you're on the correct layer
3. **Sensitivity issues**: Adjust cpi-dividor, snipe-cpi-dividor, or scroll-tick values
4. **Automouse not working**: Ensure CONFIG_ZMK_MOUSE is enabled and automouse-layer is set

---

# ZMK PAW3222 ドライバ（スクロール機能付き）

このドライバは、PIXART PAW3222光学センサーをZMKフレームワークで使用でき、スクロール機能も含んでいます。

PAW3222は、マウスやトラックボールなどのトラッキングアプリケーションに適した低消費電力の光学マウスセンサーです。このドライバはSPIインターフェースを介してPAW3222センサーと通信します。

## インストール

1. ZMKモジュールとして追加：

```
# west.yml に追加
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: sekigon-gonnoc
      url-base: https://github.com/sekigon-gonnoc
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    - name: zmk-driver-paw3222
      remote: sekigon-gonnoc
      revision: main
```

## デバイスツリー設定

シールドまたはボード設定ファイル（.overlayまたは.dtsi）で設定：

```dts
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 12)>,
                <NRF_PSEL(SPIM_MOSI, 1, 9)>,
                <NRF_PSEL(SPIM_MISO, 1, 9)>;
        };
    };

    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 12)>,
                <NRF_PSEL(SPIM_MOSI, 1, 9)>,
                <NRF_PSEL(SPIM_MISO, 1, 9)>;
            low-power-enable;
        };
    };
};

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 13 GPIO_ACTIVE_LOW>;

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,paw3222";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 15 GPIO_ACTIVE_LOW>;
    };
};
```

## キーボードのKconfigファイルでモジュールを有効化

キーボードの `Kconfig.defconfig` に以下を追加：

```kconfig
if ZMK_KEYBOARD_YOUR_KEYBOARD

config ZMK_POINTING
    default y

config PAW3222
    default y

endif
```

## プロパティ

- `irq-gpios`: モーションピンに接続されたGPIO（必須）
- `res-cpi`: センサーのCPI解像度（任意）
- `force-awake`: センサーを「強制起動」モードで初期化（任意、ブール値）
