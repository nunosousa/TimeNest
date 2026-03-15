# Firmware

This directory contains the firmware for the TimeNest device.

The firmware runs on the RP2040 microcontroller and is responsible for:

- generating the DCF77 signal
- controlling the antenna driver
- managing LED solar charging
- providing USB console access
- handling Ethernet networking and SNTP synchronization
- driving the display

## Structure

rp2040/ main firmware project

drivers/ hardware drivers

protocols/ time signal generation

ui/ display and console interface

## Build System

Firmware is built using the RP2040 SDK.

Further build instructions will be added as development progresses.
