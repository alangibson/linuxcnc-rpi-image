LinuxCNC >=2.9.3 on Rapsberry Pi OS with Kernel =6.12
=====================================================

# Prepare Raspberry Pi

First install Raspberry Pi OS on SD card and boot it.

```bash
sudo apt update
sudo apt upgrade
sudo rpi-update next
```

# Realtime kernel

```bash
sudo apt install git bc bison flex libssl-dev make libncurses5-dev raspberrypi-kernel-headers
git clone --depth 1 --branch rpi-6.12.y https://github.com/raspberrypi/linux
cd linux
```

## Configure

### RPi 4

```bash
KERNEL=kernel8
make bcm2711_defconfig
```

### RPi 5

```bash
KERNEL=kernel_2712
make bcm2712_defconfig
```

### Configure kernel

See https://ubuntu.com/blog/real-time-kernel-tuning

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
~/linux/scripts/diffconfig
```

### Patch to create /sys/kernel/realtime
https://forum.linuxcnc.org/38-general-linuxcnc-questions/54542-real-time-kerel-not-detected-on-patched-6-12?start=10#315437

```bash
echo 'CONFIG_CRASH_DUMP=y' >> .config
echo 'CONFIG_ARCH_SUPPORTS_CRASH_DUMP=y' >> .config
echo 'CONFIG_KEXEC_CORE=y' >> .config
echo 'CONFIG_PROC_KCORE=y' >> .config
echo 'CONFIG_PROC_FS=y' >> .config
echo 'CONFIG_MMU=y' >> .config
git apply sys-kernel-realtime.patch
```

### Build RT kernel
https://www.raspberrypi.com/documentation/computers/linux_kernel.html#building

```bash
make prepare
make -j6 Image.gz modules dtbs 
sudo make -j6 modules_install
```

# Install kernel

```bash
sudo cp /boot/firmware/$KERNEL.img /boot/firmware/$KERNEL-backup.img
sudo cp arch/arm64/boot/Image.gz /boot/firmware/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/overlays/
sudo cp arch/arm64/boot/dts/overlays/README /boot/firmware/overlays/
```

# Performance tuning

https://raspberrytips.com/disable-wifi-raspberry-pi/
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
echo " processor.max_cstate=1 isolcpus=2,3 workqueue.power_efficient=0" | sudo tee -a /boot/firmware/cmdline.txt
sudo reboot
```

# Install LinuxCNC

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

# Hotshot Driver

## For bcm2835 driver

Enable access to /dev/mem by anyone in kmem group

```bash
sudo apt-get install libcap2 libcap-dev
echo 'SUBSYSTEM=="mem", KERNEL=="mem", GROUP="kmem", MODE="0660"' | sudo tee /etc/udev/rules.d/98-mem.rules
sudo usermod -a -G kmem $USER
```

Enable /dev/mem access

```bash
sudo setcap cap_sys_rawio+ep /usr/bin/linuxcnc
```

## Copy driver code

```bash
git clone https://github.com/alangibson/hotshot-driver-linuxcnc
scp -r hotshot-driver-linuxcnc/ operator@plasma.local:/home/operator/
```

## Compile

```bash
ssh operator@plasma.local
cd hotshot-driver-linuxcnc
./hotshot.install.sh
```

# Performance Test

## Test performance

```bash
sudo apt install rt-tests
sudo cyclictest --mlockall --smp --priority=80 --interval=30 --distance=0
```
