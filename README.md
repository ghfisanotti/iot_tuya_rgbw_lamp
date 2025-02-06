# Flashing ESPHome to a Beken/Tuya CBLC5-Powered RGBW lamp

This WiFi lamp is powered by a Tuya/Beken CBLC5 module with a BK7231N MCU. It is sold in my country under the brand **Alic**, model **SMW0510**, it has a GU10 socket, and is used with the **Tuya app** on Android. It has a group of five RGB LEDs, and another group of twelve white LEDs (six cold and six warm). The LEDs are controlled by a SM2235 chip which is connected to the BK7231N via the I2C bus. The original firmware version was 1.3.21

Before any hardware hacking, it is recommended to configure the device using the Tuya app to ensure everything is functioning correctly. Take note of the original firmware version, it could be useful later when running tuya-convert.

## Hacking It for Local Use with Home Assistant

While it is possible to integrate Tuya devices with Home Assistant via the cloud, I prefer a local integration. To achieve this, I replaced the CBLC5 firmware with **ESPHome**. In this case I was able to flash the new firmware by exploiting a vulnerability present in many (not all) of this BK72xxx devices, this can be done with a popular tool called [tuya-cloudcutter](https://github.com/tuya-cloudcutter/tuya-cloudcutter). This is a very clever tool that not only allows you to flash a new firmware via OTA (over-the-air protocol) but it also lets you extract the pin configuration of the device and generates a YAML compatible with ESPhome.

Tuya-cloudcutter is run on a Raspberry Pi to flash an initial (kickstart) version of ESPhome, then you can use **ltchiptool** with its **upk2esphome** plugin to obtain the device configuration and generate the YAML for ESPhome, finally the ESPhome dashboard is used to generate the firmware and install it wirelessly (OTA) as usual.

## Installing tuya-cloudcutter & flashing the kickstart ESPhome

Follow the instructions in the [official site](https://github.com/tuya-cloudcutter/tuya-cloudcutter/blob/main/HOST_SPECIFIC_INSTRUCTIONS.md). I even used a more recent version of RapiOS than the one suggested there and I had no problems.

Once tuya-cloudcutter has installed the ESPhome kickstart in the device, the Raspberry Pi is no longer needed, the next steps should be done from any computer connected to the local network.

## Connecting the newly flashed device to the local network

* The newly flashed device starts a WiFi AP (Access Point), use your phone to connect to it.

* Using your phone browser, connect to 192.168.4.1 and configure the SSID and password for the local WiFi network. Restart the device and verify that it has successfully obtained and IP in the local network and it responds to ping.

## Obtaining the device configuration

This step requires the ltchiptool:

~~~
sudo pip install ltchiptool
sudo pip3 install upk2esphome
~~~

Use ltchiptool to get the device configuration in YAML format:

~~~
ltchiptool plugin upk2esphome kickstart -o tuyargbw.yaml
~~~

## Generating ESPHome Firmware

This can be done using the **ESPHome Dashboard**. One way to run it is via Docker:

~~~
docker run --rm --detach --name esphome \
  -v /home/gfisanot/esphome:/config \
  --net=host esphome/esphome
~~~

You can then access the dashboard using Google Chrome at [http://localhost:6052](http://localhost:6052).

1. Create a new **BK72XX** device and select **CBLC5** as the board.
2. Replace the default template with the YAML code obtained previously, for example:

```yaml
esphome:
  name: tuya-rgbw
  friendly_name: tuya-rgbw

bk72xx:
  board: generic-bk7231n-qfn32-tuya

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "fGoi+r9........................CMNhoGP5oz7I="

ota:
  - platform: esphome
    password: "946315d8................70bf6cae"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  manual_ip:
    static_ip: 192.168.0.148
    gateway: 192.168.0.1
    subnet: 255.255.255.0
    dns1: 192.168.0.1

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Tuya-Rgbw Fallback Hotspot"
    password: "711W.....M8a"

captive_portal:

text_sensor:
  - platform: libretiny
    version:
      name: LibreTiny Version

sm2235:
  clock_pin: P26
  data_pin: P24
  max_power_color_channels: 4
  max_power_white_channels: 2

output:
  - platform: sm2235
    id: output_red
    channel: 1
  - platform: sm2235
    id: output_green
    channel: 0
  - platform: sm2235
    id: output_blue
    channel: 2
  - platform: sm2235
    id: output_cold
    channel: 4
  - platform: sm2235
    id: output_warm
    channel: 3

light:
  - platform: rgbww
    id: light_rgbww
    name: Light
    color_interlock: true
    cold_white_color_temperature: 6500 K
    warm_white_color_temperature: 2700 K
    red: output_red
    green: output_green
    blue: output_blue
    cold_white: output_cold
    warm_white: output_warm
```

3. Choose **Install**, then **Wirelessly**. In this step it is important to include the "manual_ip" in the Wifi section of the YAML, otherwise the dashboard will not find the device.

### REFERENCES

- [CBLC5 Module Datasheet](https://developer.tuya.com/en/docs/iot/cblc5-module-datasheet?id=Ka07iqyusq1wm)
- [LibreTiny Documentation](https://docs.libretiny.eu/)
- [Helpful YouTube Tutorial](https://www.youtube.com/watch?v=yndcrR_5ASs&list=WL&index=5&pp=gAQBiAQB)

