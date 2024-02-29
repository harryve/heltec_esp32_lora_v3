# Heltec ESP32 LoRa v3

## The unofficial library

![](images/ESP32_LoRa_v3.png)

### Introduction

There's this Chinese company named Heltec, and they make a cool little development board that has an Espressif ESP32S3 (which has WiFi and Bluetooth) and an SX1262 863-928 MHz radio on it. It sells under different names on the internet, but internally they call it **[HTIT-WB32LA](images/heltec_esp32_lora_v3_documentation.pdf)**. (They have a 470-510 MHz version also, called HTIT-WB32LAF.) The hardware is cool, the software that comes with it is not so much my taste. There's multiple GitHub repositories, it's initailly unclear what is what, they use some radio stack of unknown origin, code-quality and documentation varies, some examples need tinkering and what could be a cool toy could easily become a very long weekend of frustration before things sort of work.

This library allows you to use that time to instead play with this cool board. The examples are tested, and this library assumes that for all things sub-GHz, you want to use the popular RadioLib.

&nbsp;

##### (Click blue headings for more detailed documentation)

### Setting up

Make sure your Arduino IDE knows about ESP-32 boards by putting the following URL in the following URL under "Additional board manager URLs" in the Arduino IDE settings:

```
https://espressif.github.io/arduino-esp32/package_esp32_index.json
```

Then under "*Settings / Board*" select "*Heltec WiFi LoRa 32(V3) / Wireless shell (V3) / Wireless stick lite (V3)*".

### [RadioLib](https://github.com/jgromes/RadioLib)

As said, this library depends on RadioLib, meaning it should automatically be installed if you use the Arduino Library Manager to install this library. All RadioLib examples that should work with an SX1262 should work here, simply `#include <heltec.h>` and remove any code that creates a `radio` instance, it already exists when you include this library.

> * _It might otherwise confuse you at some point: while Heltec wired the DIO1 line from the SX1262 to the ESP32 (as they should, it is the interrupt line), they labeled it in their `pins_arduino.h` and much of their own software as DIO0. The SX1262 IO pins start at DIO1._
> * _If you place `#define HELTEC_NO_RADIOLIB` before `#include <heltec.h>`, RadioLib will not be included and this library won't create a radio object. Handy if you are not using the radio, need the spacein flash for something else or want full control to play with another radio library or so._

### [Display](https://github.com/ThingPulse/esp8266-oled-ssd1306)

The tiny OLED display uses the same library that the original library from Heltec uses, except now the examples work so you don't have to figure out how to make things work. It is included inside this library because the Heltec board needs a hardware reset and to make the Arduino `print` functionality work better. (The latter change [submitted](https://github.com/ThingPulse/esp8266-oled-ssd1306/pull/389#issuecomment-1962005989) to the original library also.)

There's the primary display library and there's an additinal UI library that allows for multiple frames. The display examples will show you how things work. The library, courtesy of ThingPulse, is well-written and well-documented. [Check them out](https://thingpulse.com/) and buy their stuff.

##### Printing to both Serial and display: `both.print()`

Instead of using `print`, `println` or `printf` on either `Serial` or `display`, you can also print to `both`. As the name implies, this prints the same thing on both devices.

### [Button](https://github.com/poelstra/arduino-multi-button)

The user button marked 'PRG' on the board is handled by another library this one depends on, called MultiButton. Since we have only one button, it makes sense to have `button.isSingleClick()`, `button.isDoubleClick()` and so forth. Just remember to put `heltec.loop()` in the`loop()` of your sketch if you use it.

##### Using it as the power button

If you hook up this board to power, and especially if you hook up a LiPo battery (see below), you'll notice there's no on/off switch. Luckily the ESP32 comes with a very low-power "deep sleep" mode where it draws so little current it can essentially be considered off. Since signals on GPIO pins can wake it back up, we can use the button on the board as a power switch. In your sketch, simply put **`#define HELTEC_POWER_BUTTON`** before `#include <heltec.h`, make sure `heltec_loop()` is in your own `loop()` and then a button press will wake it up and a long press will turn it off. You can still use `button.isSingleClick()` and `button.isDoubleClick()` in your `loop()` function when you use it as a power button.

> * _When you use this feature, **the system will start in the off state**. So every time you flash a new sketch or press reset, things will be off until you press the 'PRG' button._
> * _If you use `delay()` in your code, the power off function will not work during that delay. To fix that, simply use **`heltec_delay()`** instead._

### [Deep Sleep](https://randomnerdtutorials.com/esp32-deep-sleep-arduino-ide-wake-up-sources/)

You can use `heltec_deep_sleep(<seconds>)` to put the board into this 'off' deep sleep state yourself. This will put the board in deep sleep for the specified number of seconds. After it wakes up, it will run your sketch from the start again. You can use `heltec_wakeup_was_button()` and `heltec_wakeup_was_timer()` to find out whether the wakeup was caused by the power button or because your specified time has elapsed. You can even keep data in variables that survive deep sleep by tagging them `RTC_DATA_ATTR`. More is in [this tutorial](https://randomnerdtutorials.com/esp32-deep-sleep-arduino-ide-wake-up-sources/).

> * _If you call `heltec_deep_sleep()` without a number in seconds when not using the power button feature, you will need to reset it to turn it back on. This does zap the `RTC_DATA_ATTR` variables though._

### LED

The board has a bright white LED, next to the orange power/charge LED. This library provides a function `heltec_led` that takes the LED brightness in percent.

### Battery

The board is capable of charging a LiPo battery connected to the little 2-pin connector at the bottom. `heltec_vbat()` gives you a float with the battery voltage, `heltec_battery_percent()` provides the estimated percentage full. 

> * _According to the [schematic](images/heltec_esp32_lora_v3_schematic.pdf), the charge current is set to 500 mA. There's a voltage measuring setup where if GPIO37 is pulled low, the battery voltage appears on GPIO1. (Resistor-divided: VBAT - 390kΩ - GPIO1 - 100kΩ - GND)_
> * _You can optionally provide the float that `heltec_vbat()` returns to `heltec_battery_percent()` to make sure both are based on the same measurement._
> * _The [charging IC](images/tp4054.pdf) used will charge the battery to ~4.2V, then hold the voltage there until charge current is 1/10 the set current (i.e. 50 mA) and then stop and let it discharge to 4.05V (about 90%) and then charge it again, so this is expected._
> * _The orange charging LED, on but very dim is no battery is plugged in, is awfully bright when charging, and the IC on the reverse side of the reset switch gets quite hot when the battery is charging but still fairly empty. It's limited to 100 ℃, so nothing too bad can happen, just so you know._

The battery percentage estimate in this library is based on a real LiPo discharge curve.

![](/images/battery_curve.png)

> * _The library contains all the tools to measure your own curve and use it instead, see [`heltec.h`](src/heltec.h) for details._

### Ve - external power

There's two pins marked 'Ve', connected to a GPIO-controlled FET that can source 350 mA at 3.3V to power sensors etc. Turn on by calling `heltec_ve(true)`, `heltec_ve(false)` turns it off.

&nbsp;

&nbsp;

## Minimal example to show everything:

```cpp
// Turns the 'PRG' button into the power button, long press is off 
#define HELTEC_POWER_BUTTON   // must be before "#include <heltec.h>"

// creates 'radio', 'display' and 'button' instances 
#include <heltec.h>

void setup() {
  heltec_setup();
  Serial.println("Serial works");
  // Display
  display.println("Display works");
  // Radio
  display.print("Radio ");
  int state = radio.begin();
  if (state == RADIOLIB_ERR_NONE) {
    display.println("works");
  } else {
    display.printf("fail, code: %i\n", state);
  }
  // Battery
  float vbat = heltec_vbat();
  display.printf("Vbat: %.2fV (%d%%)\n", vbat, heltec_battery_percent(vbat));
}

void loop() {
  heltec_loop();
  // Button
  if (button.isSingleClick()) {
    display.println("Button works");
    // LED
    for (int n = 0; n <= 100; n++) { heltec_led(n); delay(5); }
    for (int n = 100; n >= 0; n--) { heltec_led(n); delay(5); }
    display.println("LED works");
  }
}
```

&nbsp;

For a more meaningful demo, especially if you have two of these boards, check out `LoRa_rx_tx` in the examples.

![](images/pins.png)
