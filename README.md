# roza-scavenger

voice-over-FSK walkie talkie on ESP32-S3 + SX1262 (LoRa modem) + MSM261 mic + MAX98357A amp. FSK @ 19.2 kbps, Codec2 mode 3200 (8 bytes per 20ms frame), 5-frame packets (100ms = ~22ms air time at 19.2k).

## architecture

**audio pipeline:**
- IIS microphone capture at 8 kHz, 16-bit PCM
- Codec2 3200 encodes to 8 bytes per frame (160 samples)
- Ring buffer (thread-safe, O(1) push/pop) feeds packet transmit

**radio stack:**
- FSK modulation on 914.6 MHz (US ISM), ±10 kHz frequency deviation, 39 kHz RX bandwidth
- 12-byte header (MAC, sequence, flags) + 40-byte audio payload = 52 bytes total
- 16-bit preamble, self-MAC filtering to mute local echo
- Non-blocking push-to-talk via 200ms debounce

**receive path:**
- bounded RX buffer prevents OOM on garbage packets
- drop duplicates via sequence number
- int32_t gain with saturation (avoids silent wrap)
- MAX98357A playback synchronized to packet clock

## hardware mapping

- ESP32-S3 GPIO → RadioLib SX1262 (chip select, IRQ, reset)
- I2S port → MSM261 microphone ADC
- DAC → MAX98357A speaker amplifier

## notes

fixes vs. original design:
- thread-safe ring buffers replace std::vector (race-condition-free)
- Codec2 3200 frame size corrected: 8 bytes, not 40
- reduced payload to 40 bytes (5 frames) for low-latency push-to-talk
- O(1) ring buffer instead of O(n) vector::erase from front
