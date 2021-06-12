# LoRaWAN, Raspberry Pi Pico, HOPERF Module RFM95W, The Things Network

## Overview

This guideline is based on excellent article: 
[Connect Your Adafruit Feather RP2040 to The Things Network](https://www.hackster.io/sandeep-mistry/connect-your-adafruit-feather-rp2040-to-the-things-network-5c0c84)

The difference from this article is to provide guideline for:

- European region
- HOPERF Module based on RFM95W - I'm happy owner of [RFM95W-868-S2_BOB](https://www.soselectronic.com/products/various/rfm95w-868-s2-bob-342652)

Goal is to provide guideline how to make all those components working Together in European region with The Things Network.

## Source code and Build

I'm assuming you have Raspberry Pi Pico C/C++ SDK installed. In my case I have it in my home directory: ~/pico/pico-sdk.

```
git clone --recurse-submodules https://github.com/sandeepmistry/pico-lorawan.git
cd pico-lorawan
mkdir build && cd build && cmake -DPICO_SDK_PATH=~/pico/pico-sdk -DCMAKE_BUILD_TYPE=Debug -DPICO_BOARD=pico .. && make -j4
```
