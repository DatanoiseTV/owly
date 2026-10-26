# Raspberry Pi OS installation

This guide covers Raspberry Pi OS Bookworm and Trixie, formerly called Raspbian. Owly needs Rust 1.88 or newer, NetworkManager, and a Linux Wi-Fi driver that can capture and inject 802.11 frames. Keep Ethernet connected while changing the Wi-Fi setup.

## Install Owly

From a checkout of this repository on the Pi:

```sh
sudo apt update
sudo apt install git curl build-essential network-manager iproute2 iputils-ping polkitd bluez iw
rustc --version
```

If `rustc` is missing or older than 1.88, install the current stable Rust toolchain with [rustup](https://rustup.rs/):

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"
rustc --version
```

Build and install the CLI and radio service:

```sh
cargo build --release --locked -p owlyd -p owly-cli
sudo packaging/install.sh owly
sudo systemctl daemon-reload
sudo busctl call org.freedesktop.DBus /org/freedesktop/DBus org.freedesktop.DBus ReloadConfig
sudo nmcli general reload conf
owly doctor
```

Raspberry Pi OS Bookworm and Trixie use NetworkManager. If the Wi-Fi interface is managed by a different network service, switch it to NetworkManager before starting a radio session; Owly uses `nmcli` to release and restore it.

## Choose a radio

### Stock Raspberry Pi OS with a USB Wi-Fi adapter

The onboard Wi-Fi's stock firmware cannot inject the AWDL frames Owly needs. A compatible USB adapter is the direct route on an otherwise stock image. The [supported hardware table](../README.md#supported-hardware) includes the MT7612U USB adapter; use `owly doctor` to check the exact adapter and regulatory channels available on your Pi.

Find its interface with `iw dev`, then name it in `/etc/owly/radio.toml` so Owly does not select the onboard `wlan0` first:

```toml
station = "wlan1"
```

Replace `wlan1` with the adapter's actual name. Run `owly doctor`, then `owly radio up`. In another terminal, run `owly receive --dir ~/Downloads` or `owly send` as described in the [main guide](../README.md#usage).

### Onboard Wi-Fi with Nexmon

The `firmware-monitor` mode requires both a Nexmon-patched `brcmfmac` driver and matching firmware for the board, Wi-Fi chip, and running kernel. Upstream [Nexmon's Raspberry Pi OS instructions](https://github.com/seemoo-lab/nexmon/blob/master/README.md) cover specific older kernel and firmware combinations; they are not a universal installation procedure for current Raspberry Pi OS. Check their supported-device and kernel list before applying a patch. Kali's Nexmon packages target Kali images and should not be installed by adding Kali repositories to Raspberry Pi OS.

After installing a matching patched stack, check that `iw phy` lists monitor mode and verify capture and injection using Nexmon's instructions. Then set the onboard interface and opt in:

```toml
station = "wlan0"
allow_untested = true
```

Run `owly doctor`. It must report `firmware-monitor` and a usable social channel before `owly radio up`. If it reports no monitor interface, the patched stack is not active. A successful monitor or injection test does not establish that AirDrop works; run `owly test-card` with an iPhone and report both transfer directions. Wi-Fi connectivity on the onboard interface is interrupted while this mode runs, so keep Ethernet available.
