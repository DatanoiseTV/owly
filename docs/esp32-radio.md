# ESP32 radio feasibility

ESP32 chips with Wi-Fi can receive raw 802.11 frames through ESP-IDF's
promiscuous callback and transmit selected raw frames through
`esp_wifi_80211_tx()`. This is useful for an Owly radio coprocessor, but it
does not make an ESP32 a Linux monitor-mode card or establish AirDrop support.
The public transmit API explicitly accepts action frames and **non-QoS** data;
it rejects QoS or encrypted data. Owly currently emits non-QoS frames for
ordinary IPv6 packets and QoS A-MSDU frames when aggregating. An ESP32
backend would need to disable the latter and prove the former on the air.
There is already a separate [ESP32Drop implementation](https://github.com/s-iwaki-d/ESP32Drop)
reporting real AirDrop transfers on one ESP32-S3 board, which makes channel 6
a demonstrated path rather than a purely theoretical one. Its results do not
validate an Owly backend or another ESP32 board.

## Chip choice

| Chip | Wi-Fi bands | BLE | Raw receive/transmit assessment |
| --- | --- | --- | --- |
| ESP32-C5 | 2.4 and 5 GHz | Yes | Best candidate. The public API can tune AWDL social channels 6, 44, and 149 where local rules permit. |
| ESP32-S3 with PSRAM | 2.4 GHz | Yes | ESP32Drop reports sending and receiving on channel 6 on an M5Stack StopWatch. Owly integration remains untested. |
| ESP32, C2, C3, C6, C61, S31 | 2.4 GHz | Yes | Can investigate channel 6 only. They cannot follow peers that remain on 5 GHz. |
| ESP32-S2 | 2.4 GHz | No | Can serve as a Wi-Fi coprocessor if the Linux host supplies BLE. |
| ESP32-H2 | None | Yes | Cannot carry AWDL Wi-Fi frames. |
| ESP32-P4 | None onboard | Depends on an added radio | Requires a separate Wi-Fi coprocessor; its own silicon is not an AWDL radio. |

The bands and BLE capabilities come from [Espressif's chip table](https://docs.espressif.com/projects/esp-techpedia/en/latest/esp-friends/advanced-development/wifi-protocol-support.html)
and [ESP-IDF's Bluetooth support list](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/bt-architecture/overview.html).
The C5 is a single-radio device: selecting both bands permits switching, not
simultaneous 2.4 and 5 GHz use. Its [channel API](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c5/api-reference/network/esp_wifi.html)
names 44 and 149, subject to the configured country policy.

## Relevant GitHub implementations

- [ESP32Drop](https://github.com/s-iwaki-d/ESP32Drop) implements AWDL and
  AirDrop directly on an ESP32-S3 with PSRAM. Its author reports receiving a
  3.9 MB photo and sending small PNG/JPEG files to Macs and iPhones; sending
  larger files to an iPhone is untested. It fixes channel 6, uses distinct
  send and receive builds due to memory, and alternates BLE and Wi-Fi during
  sending. These are project-reported hardware results, not our measurements.
- [ESP-Hosted-Linux](https://github.com/espressif/esp-hosted-linux) already
  runs an ESP32-C5 as a Linux Wi-Fi/Bluetooth coprocessor over SDIO. Its
  documented Linux path provides station and AP modes. Owly still needs a
  raw AWDL receive/transmit path, so ordinary hosted `wlanX` support is not
  sufficient on its own.
- [ESP-Hosted-MCU](https://github.com/espressif/esp-hosted-mcu) defines raw
  transmit and promiscuous requests in its RPC schema. A
  [P4/C6 experiment](https://github.com/r4d10n/esp32p4-c6-wifi-test)
  reports that these calls needed a custom RPC extension in practice. That
  work is a transport reference, not a ready Owly adapter.

## What the public ESP-IDF API provides

- `esp_wifi_set_promiscuous()` and its receive callback expose management,
  data, and selected control frames on the tuned channel. The callback must
  copy accepted frames promptly; it runs in the Wi-Fi task. See the
  [C5 sniffer guide](https://docs.espressif.com/projects/esp-idf/en/v5.5.4/esp32c5/api-guides/wifi.html).
- `esp_wifi_80211_tx()` sends beacon, probe, action, and non-QoS data frames,
  with a 24 to 1500 byte raw buffer limit. It cannot send QoS or encrypted
  frames. Its return value means the driver accepted a transmit request, not
  that an Apple peer received the frame. See the [C5 transmit guide](https://docs.espressif.com/projects/esp-idf/en/v5.5.4/esp32c5/api-guides/wifi.html)
  and [API reference](https://docs.espressif.com/projects/esp-idf/en/v5.5/esp32c5/api-reference/network/esp_wifi.html).
- The Wi-Fi MAC generates FCS. It is not included in the transmit buffer,
  according to [Espressif's Wi-Fi FAQ](https://github.com/espressif/esp-faq/blob/master/docs/en/software-framework/wifi.rst).
- Wi-Fi and BLE share RF time on the C5. BLE activity can preempt Wi-Fi;
  discovery and transfer timing must be measured with both enabled. See the
  [C5 coexistence guide](https://docs.espressif.com/projects/esp-idf/en/v5.5/esp32c5/api-guides/coexist.html).

The [ESP-Hosted MCU protocol](https://github.com/espressif/esp-hosted-mcu/blob/main/common/proto/esp_hosted_rpc.proto)
already defines promiscuous mode and raw transmit requests. That is a useful
reference for a host link. It does not by itself provide Owly's required
Linux raw-frame interface, timestamping, channel control, or recovery.

## Owly integration boundary

An ESP32 is a microcontroller, so the existing Linux `brcmfmac` and nl80211
backend cannot drive it directly. Keep AirDrop, BLE discovery, IPv6, and TLS
on the Linux host. A C5 firmware would own radio tuning, raw receive, raw
transmit, and RF statistics. A host transport over USB, SPI, or UART would
carry length-delimited frames, commands, and acknowledgments. It must expose
an actual `owly_radio::radio::Backend` implementation rather than pretending
to be an nl80211 monitor interface.

The host must also disable QoS A-MSDU output for this backend and keep
transmitted 802.11 buffers within 1500 bytes. `owly-awdl` already has a
non-QoS data-frame encoder. A smaller TAP MTU may be needed; that needs an
end-to-end transfer test, including IPv6 fragmentation and TCP behavior.
Fixed-channel operation is the first target. Channel hopping and BLE
coexistence make AWDL timing harder, especially with a one-radio bridge.

## Evidence required before implementation is called usable

1. On a C5, capture an Apple AWDL action frame on channel 44 or 149 with its
   complete body, BSSID, channel, and timestamp.
2. Transmit an Owly action frame and a non-QoS data frame; verify both with
   an independent 5 GHz sniffer and inspect return codes and frame bytes.
3. Confirm whether the C5 delivers peer unicast AWDL data without joining a
   BSS, and whether the radio handles ACKs in a way that permits transfers.
4. Measure missed AWDL availability windows while BLE is advertising and
   scanning; test channel 6 separately from 44 and 149.
5. Run discovery and two-way AirDrop transfers through a real host transport,
   including a file larger than one IPv6 packet.

Until these pass for Owly, the C5 is a plausible dual-band target. The
ESP32-S3 has project-reported AirDrop evidence on channel 6. No ESP32 family
member is listed as supported Owly hardware on the basis of another project's
results or API availability alone.
