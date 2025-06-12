LinuxCNC >=2.9.3 on Rapsberry Pi OS with Kernel =6.12
=====================================================

# Install OS

First install Raspberry Pi OS on SD card and boot it.

# Prepare Raspberry Pi

Then run on the Raspberry Pi

```bash
git clone https://github.com/alangibson/linuxcnc-rpi-image
cd https://github.com/alangibson/linuxcnc-rpi-image
```

```bash
sudo apt update
sudo apt -y upgrade
sudo WANT_64BIT_RT=1 rpi-update rpi-6.12.y
```

# Compile Kernel

## Download kernel source

```bash
git clone --depth 1 --branch rpi-6.12.y https://github.com/raspberrypi/linux
cd linux
```

## Prepare compiler

### Cross-compile on Debian/Ubuntu

```bash
sudo apt install bc bison flex libssl-dev make libc6-dev libncurses5-dev
sudo apt install crossbuild-essential-arm64

MAKE="make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-"
```

### Compile on Raspberry Pi

```bash
sudo apt install git bc bison flex libssl-dev make libncurses5-dev raspberrypi-kernel-headers

MAKE="make -j6"
```

### Generate base kernel config

#### RPi 4

```bash
KERNEL=kernel8
$MAKE bcm2711_defconfig
```

#### RPi 5

```bash
KERNEL=kernel_2712
$MAKE bcm2712_defconfig
```

### Custom kernel config

```bash
cp .config .config.old
echo 'CONFIG_PREEMPT=n' >> .config
echo 'CONFIG_PREEMPT_RT=y' >> .config
echo 'CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y' >> .config
echo 'CONFIG_CPU_FREQ_GOV_CONSERVATIVE=n' >> .config
echo 'CONFIG_CPU_FREQ_GOV_POWERSAVE=n' >> .config
echo 'CONFIG_CPU_FREQ_GOV_SCHEDUTIL=n' >> .config
echo 'CONFIG_CPU_FREQ_GOV_USERSPACE=n' >> .config
echo 'CONFIG_NO_HZ_FULL=y' >> .config
echo 'CONFIG_HZ_1000=y' >> .config
echo 'CONFIG_HZ_PERIODIC=y' >> .config
echo 'CONFIG_LEDS_TRIGGER_CPU=n' >> .config
echo 'CONFIG_EFI_DISABLE_RUNTIME=y' >> .config
echo 'CONFIG_NTP_PPS=y' >> .config
echo 'CONFIG_RTC_INTF_DEV_UIE_EMUL=y' >> .config
echo 'CONFIG_VIRT_CPU_ACCOUNTING_GEN=y' >> .config
echo 'CONFIG_LOCALVERSION=-v8-16k-NTP' >> .config
echo 'CONFIG_PPS_CLIENT_GPIO=y' >> .config
./scripts/diffconfig
```

See https://ubuntu.com/blog/real-time-kernel-tuning

### Patch to create /sys/kernel/realtime

```bash
echo 'CONFIG_CRASH_DUMP=y' >> .config
echo 'CONFIG_ARCH_SUPPORTS_CRASH_DUMP=y' >> .config
echo 'CONFIG_KEXEC_CORE=y' >> .config
echo 'CONFIG_PROC_KCORE=y' >> .config
echo 'CONFIG_PROC_FS=y' >> .config
echo 'CONFIG_MMU=y' >> .config
git apply ../sys-kernel-realtime.patch
```

https://forum.linuxcnc.org/38-general-linuxcnc-questions/54542-real-time-kerel-not-detected-on-patched-6-12?start=10#315437

### Compile Realtime kernel

```bash
# make prepare
$MAKE Image modules dtbs
```

### Install kernel

#### To SD card

```bash
ROOT="/media/alangibson/rootfs"
BOOT="/media/alangibson/bootfs"

sudo env PATH=$PATH $MAKE INSTALL_MOD_PATH="$ROOT" modules_install

sudo cp -v $BOOT/$KERNEL.img $BOOT/$KERNEL-backup.img
sudo cp -v arch/arm64/boot/Image $BOOT/$KERNEL.img
sudo cp -v arch/arm64/boot/dts/broadcom/*.dtb $BOOT/
sudo cp -v arch/arm64/boot/dts/overlays/*.dtb* $BOOT/overlays/
sudo cp -v arch/arm64/boot/dts/overlays/README $BOOT/overlays/
```

#### On Raspberry Pi

```bash
BOOT="/boot"

sudo $MAKE modules_install

sudo cp $BOOT/firmware/$KERNEL.img $BOOT/firmware/$KERNEL-backup.img
sudo cp arch/arm64/boot/Image.gz $BOOT/firmware/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb $BOOT/firmware/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* $BOOT/firmware/overlays/
sudo cp arch/arm64/boot/dts/overlays/README $BOOT/firmware/overlays/
sudo reboot
```







# Install LinuxCNC

## Install using apt

```bash
curl -O https://www.linuxcnc.org/linuxcnc-install.sh
sudo sh ./linuxcnc-install.sh
rm linuxcnc-install.sh
sudo apt install -y linuxcnc-uspace-dev \
    python3-gst-1.0 dunst imagemagick \
    python3-lxml source-highlight w3c-linkchecker xsltproc \
    texlive-extra-utils texlive-font-utils texlive-fonts-recommended \
    texlive-lang-cyrillic texlive-lang-french texlive-lang-german \
    texlive-lang-polish texlive-lang-spanish texlive-xetex \
    texlive-latex-recommended dblatex asciidoc-dblatex \
    libusb-1.0-0-dev graphviz inkscape texlive-lang-european \
    tclreadline libxml2 libxml2-dev python3-configobj python3-numpy \
    libgtksourceview-3.0-dev python3-cairo python3-gi-cairo python3-opengl \
    gir1.2-gtksource-3.0 libglew2.2 mesa-utils
```

## Create shortcut to config

```bash
sudo nano /usr/share/applications/cnc.desktop
```
Paste this:
```
[Desktop Entry]
Comment=
Terminal=false
Name=Darkstar CNC
Exec=sh -c "taskset -c 2,3 linuxcnc $HOME/hotshot-driver-linuxcnc/my-plasma/my-plasma.ini"
Type=Application
Icon=/usr/share/linuxcnc/linuxcncicon.png
Categories=Utility;
```

```bash
sudo chmod +x /usr/share/applications/cnc.desktop
```

Add .desktop file to user desktop
```bash
cp /usr/share/applications/cnc.desktop "$HOME/Desktop/"
```

TODO Add .desktop file to launcher. 
API is ridiculous so just do it manually.


# Hotshot Driver

### Enable /dev/mem access

Enable access to /dev/mem by anyone in kmem group

```bash
sudo apt -y install libcap2 libcap-dev
echo 'SUBSYSTEM=="mem", KERNEL=="mem", GROUP="kmem", MODE="0660"' | sudo tee /etc/udev/rules.d/98-mem.rules
sudo usermod -a -G kmem $USER
sudo setcap cap_sys_rawio+ep /usr/bin/linuxcnc
```

### Compile

```bash
git clone https://github.com/alangibson/hotshot-driver-linuxcnc
cd hotshot-driver-linuxcnc
./hotshot.install.sh
```






# Performance tuning

```bash
uname -a
```

Add to bottom of /boot/firmware/config.txt

```bash
dtoverlay=disable-wifi
dtoverlay=disable-bt
```

```bash
sudo systemctl disable cups-browsed.service cups.path cups.service cups.socket
sudo systemctl disable wpa_supplicant
sudo systemctl disable bluetooth
sudo systemctl disable hciuart
sudo systemctl stop apache2 systemd-timesyncd.service wayvnc.service \
    systemd-udevd.service systemd-udevd-control.socket systemd-udevd-kernel.socket \
    systemd-journald.service systemd-journald.socket systemd-journald-audit.socket systemd-journald-dev-log.socket
```

```bash
echo " processor.max_cstate=1 isolcpus=2,3 workqueue.power_efficient=0" | sudo tee -a /boot/firmware/cmdline.txt
sudo reboot
```

https://raspberrytips.com/disable-wifi-raspberry-pi/

# Performance Test

### Test performance

```bash
sudo apt install rt-tests
sudo cyclictest --mlockall --smp --priority=80 --interval=30 --distance=0
```

### See if performance is throttled

```bash
vcgencmd get_throttled
```

