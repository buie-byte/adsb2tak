# ADS-B to TAK Server Guide

This guide documents the working configuration of tak-sns01, a Raspberry Pi 3B sensor node that receives 1090 MHz ADS-B transmissions with an RTL-SDR, decodes aircraft data with dump1090-fa, converts fresh positioned aircraft into Cursor-on-Target (CoT) XML, and publishes those tracks to TAK Server over mutually authenticated TLS.

---

## Installation Process

Please follow the steps in order. Each section is organized as a separate "tab" in this guide.
