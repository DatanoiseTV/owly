# Raspberry Pi OS: onboard Wi-Fi

This is an experimental source build for a Raspberry Pi 5 or wireless CM5 with
the CYW43455 radio, Raspberry Pi OS Trixie **64-bit**, a **6.18** kernel, and
firmware **7.45.265 (28bca26 CY)**. The same chip is present on Pi 4, but its
Owly radio path still needs a board test. No USB Wi-Fi adapter is involved.
The [October 2026 Raspberry Pi OS image](https://www.raspberrypi.com/software/operating-systems/)
uses kernel 6.18. An existing Bookworm or older Trixie image may use another
kernel; do not apply these commands to it without a matching Nexmon driver.

Nexmon's [Pi 5 monitor/injection port](https://github.com/seemoo-lab/nexmon/pull/692)
is an open upstream PR. Its driver and firmware build from source here, but
neither that port nor Owly's two-way AirDrop operation has been verified on
our hardware yet. Keep an Ethernet connection for the build and recovery.

## Check the installed combination

```sh
uname -m
uname -r
cat /etc/os-release
sudo dmesg | grep 'Firmware: BCM4345/6' | tail -1
iw dev
```

Continue only when the architecture is `aarch64`, the kernel begins `6.18.`,
the firmware line contains `7.45.265 (28bca26 CY)`, and `iw dev` shows the
onboard interface. These checks prevent loading a patch made for another
firmware image or kernel ABI. A wireless CM5 and Pi 5 should expose `wlan0`;
use the actual interface name if yours differs.

## Build the Nexmon firmware and driver

Install headers for the running kernel and the Nexmon build dependencies:

```sh
sudo apt update
sudo apt full-upgrade
sudo reboot
```

After reconnecting, repeat the checks above, then run:

```sh
sudo apt install git build-essential raspberrypi-kernel-headers libgmp3-dev \
  gawk qpdf bison flex make autoconf libtool texinfo zlib1g-dev \
  libssl-dev iw tcpdump
test -d "/lib/modules/$(uname -r)/build"
sudo dpkg --add-architecture armhf
sudo apt update
sudo apt install libc6:armhf libisl23:armhf libmpfr6:armhf \
  libmpc3:armhf libstdc++6:armhf
git clone https://github.com/kimocoder/nexmon.git
cd nexmon
git checkout 7adfdb8df8f36afaa5e04ebcd33850ebf48b6b36
```

Nexmon's bundled firmware compiler is an armhf program. It looks for older
library sonames than Trixie supplies. Keep the compatibility links local to
the checkout, where this patch branch's `setup_env.sh` adds them to its
library path:

```sh
mkdir -p buildtools/armhf-compat-libs
ln -s "$(readlink -f /usr/lib/arm-linux-gnueabihf/libisl.so.23)" \
  buildtools/armhf-compat-libs/libisl.so.10
ln -s "$(readlink -f /usr/lib/arm-linux-gnueabihf/libmpfr.so.6)" \
  buildtools/armhf-compat-libs/libmpfr.so.4
source setup_env.sh
"${CC}gcc" --version
make
cd patches/bcm43455c0/7_45_265-28bca26-CY/nexmon
make
test -s ../../../driver/brcmfmac_6.18.y-nexmon/brcmfmac.ko
```

If the compiler or firmware build fails, stop and keep the stock driver and
firmware. The soname links follow the [Nexmon Raspberry Pi build instructions](https://github.com/seemoo-lab/nexmon#build-patches-for-bcm43430a1-on-the-rpi3zero-w-or-bcm434355c0-on-the-rpi3rpi4-or-bcm43436b0-on-the-rpi-zero-2w-using-raspbianraspberry-pi-os-recommended);
they are a compatibility workaround, not a general fix for all versions.

## Install and load

Save the stock firmware before the firmware target installs the patched image
as an alternative and loads the freshly built module for this boot. Installing
the module under `updates` makes it the preferred `brcmfmac` on later boots.
Do this from an Ethernet or local session because it interrupts onboard Wi-Fi.

```sh
sudo cp -L /lib/firmware/cypress/cyfmac43455-sdio.bin \
  "$HOME/cyfmac43455-sdio.bin.stock"
sudo -E make install-firmware
sudo install -D -m 0644 \
  ../../../driver/brcmfmac_6.18.y-nexmon/brcmfmac.ko \
  "/lib/modules/$(uname -r)/updates/brcmfmac.ko"
sudo depmod -a
modinfo -n brcmfmac
sudo reboot
```

After reboot, `modinfo -n brcmfmac` should show the `updates` module. Check
that the patched stack actually exposes monitor mode:

```sh
modinfo -n brcmfmac
sudo dmesg | grep 'Firmware: BCM4345/6' | tail -1
iw phy | grep -A12 'Supported interface modes'
```

Do not infer injection from a monitor listing alone. A second radio capturing
the same channel is needed to confirm transmitted frames. Nexmon's monitor
interface recipe is in its [README](https://github.com/seemoo-lab/nexmon#using-the-monitor-mode-patch).

## Build and run Owly

From a checkout of this repository on the Pi, install the runtime services
and Rust 1.88 or newer:

```sh
sudo apt install curl network-manager iproute2 iputils-ping polkitd bluez
rustc --version
```

If Rust is absent or older than 1.88, install the stable toolchain using
[rustup](https://rustup.rs/), then build:

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"
cargo build --release --locked -p owlyd -p owly-cli
sudo packaging/install.sh owly
sudo systemctl daemon-reload
sudo busctl call org.freedesktop.DBus /org/freedesktop/DBus org.freedesktop.DBus ReloadConfig
sudo nmcli general reload conf
```

Set the onboard interface in `/etc/owly/radio.toml`:

```toml
station = "wlan0"
allow_untested = true
```

Then run `owly doctor`; it must report `firmware-monitor` and a permitted
social channel. Run `owly radio up` before `owly receive --dir ~/Downloads`
or `owly send`. Record both transfer directions with `owly test-card`. Owly
releases the onboard Wi-Fi connection while its radio session runs.

## Restore stock Wi-Fi

From an Ethernet or local session, remove this kernel's patched module and
select the stock firmware alternative, then reboot:

```sh
sudo rm "/lib/modules/$(uname -r)/updates/brcmfmac.ko"
sudo depmod -a
sudo update-alternatives --remove cyfmac43455-sdio.bin \
  /lib/firmware/nexmon/cyfmac43455-sdio-standard.bin
sudo install -m 0644 "$HOME/cyfmac43455-sdio.bin.stock" \
  /lib/firmware/cypress/cyfmac43455-sdio.bin
sudo reboot
```

Kernel updates require rebuilding the driver against the new running kernel.
If the kernel is no longer 6.18, first check for a matching Nexmon driver
port rather than reusing the existing module.
