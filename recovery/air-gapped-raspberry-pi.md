# Air-gapped raspberry pi

Requirements:
- SD card with at least 16G capacity
- Raspberry pi: Suggest model 4B or 5.  (These instructions were confirmed to work on a Raspberry pi model 4B device.)


## 1. Install debootstrap and qemu-system-arm to host machine
user@localPC
```
sudo apt install debootstrap qemu-system-arm qemu-user-static binfmt-support
```


## 2. Insert SD card and set $sddev to correct value
Carefully check the device for your SD card and don't get confused with any host machine filesystem devices.  
Eg: Check the output of ```sudo dmesg``` and ```df``` commands.
```
sddev=/dev/sdX
```

### 2.1 Unmount all partitions on $sddev
```
for part in ${sddev}*; do sudo umount -q $part; done;
```


## 3. Download Raspberry Pi OS image
Steps:

- Pick a Raspberry Pi OS (raspios) image to download from: https://www.raspberrypi.com/software/operating-systems/
- These instructions were confirmed to work with this particular arm64 image: https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-07-04/2024-07-04-raspios-bookworm-arm64-lite.img.xz

Set the following variable to the chosen download link:
```
raspios_img_link=https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-07-04/2024-07-04-raspios-bookworm-arm64-lite.img.xz
```

The following commands will download the image file to /tmp directory and output its sha256sum.
```
raspios_img_compressed="${raspios_img_link##*/}"; \

cd /tmp; \
wget $raspios_img_link; \
sha256sum $raspios_img_compressed;
```
>43d150e7901583919e4eb1f0fa83fe0363af2d1e9777a5bb707d696d535e2599  2024-07-04-raspios-bookworm-arm64-lite.img.xz

Check the sha256sum is the same value as that displayed on the raspberrypi.com download site.


## 4. Uncompress and write raspios image to SD card
```
unxz $raspios_img_compressed; \
raspios_img="${raspios_img_compressed%.*}"; \

sudo dd bs=4M if=$raspios_img of=$sddev status=progress conv=fsync;
```
Ensure there are no write errors:

>676+0 records in
>676+0 records out
>2835349504 bytes (2.8 GB, 2.6 GiB) copied, 51.8909 s, 54.6 MB/s

SD cards have a limited number of write cycles.  If you see an error like:

>dd: fsync failed for '/dev/sda': Input/output error

Try repeating the step of writing the raspios image to SD card, or find another SD card and try again: ```sudo dd bs=4M if=$raspios_img of=$sddev status=progress conv=fsync;```

### 4.1 Check host PC didn't automount SD card partitions and unmount any that were mounted
Check with:
```
df
```
If necessary unmount them first with:
```
for part in ${sddev}*; do sudo umount -q $part; done;
```


## 5. Disable running raspberrypi-sys-mode/firstboot init script at first boot
The raspberrypi.com image is set to expand the raspios root partition and ext4 filesystem on the first boot which we now don't need, or want, to happen.

```
sd_p1_dev=$(lsblk -n -o NAME --sort NAME --paths $sddev | sed -n '2{p;q}'); \

raspios_boot=$(mktemp -d -t raspios_boot-XXXXXXXX); \
sudo mount $sd_p1_dev $raspios_boot; \

sudo sed -i 's| init=/usr/lib/raspberrypi-sys-mods/firstboot||' "${raspios_boot}/cmdline.txt";

sudo umount $raspios_boot; \
rmdir $raspios_boot;
```
Notes:

- lsblk -n -o NAME --sort NAME --paths $sddev | sed -n '2{p;q}' # second sorted output line from ```lsblk --noheadings --output NAME --paths $sddev``` which will be first partition of $sddev
- Remove init=/usr/lib/raspberrypi-sys-mods/firstboot parameter from cmdline.txt file.  Post removal cmdline.txt looks like 'console=serial0,115200 console=tty1 root=PARTUUID=a3f161f3-02 rootfstype=ext4 fsck.repair=yes rootwait quiet'.  When the init= parameter is omitted the kernel will run /sbin/init.


## 6. Expand raspios root partition and debootstrap a new Debian system
```
sd_p2_dev=$(lsblk -n -o NAME --sort NAME --paths $sddev | sed -n '3{p;q}');

sudo parted --script $sddev \
  resizepart 2 100%; \

sudo mkfs.ext4 -F $sd_p2_dev; \

raspios_root=$(mktemp -d -t raspios_root-XXXXXXXX); \
sudo mount $sd_p2_dev $raspios_root; \

debdist=bookworm; \

sudo debootstrap --arch=arm64 $debdist $raspios_root http://deb.debian.org/debian/;
```
Notes:

- lsblk -n -o NAME --sort NAME --paths $sddev | sed -n '3{p;q}' # third sorted output line from ```lsblk --noheadings --output NAME --paths $sddev``` which will be second partition of $sddev


## 7. Customise the new Debian system

### 7.1 Enable arm64 emulation for chroot from host PC
```
sudo cp /usr/bin/qemu-aarch64-static $raspios_root/usr/bin/;
```

### 7.2 Chroot into Debian arm64 base system
```
sudo mount -t proc /proc $raspios_root/proc; \
sudo mount -t sysfs /sys $raspios_root/sys; \
sudo mount --make-slave --bind /dev $raspios_root/dev; \

sudo chroot $raspios_root /bin/bash;
```

The following is done in the chroot environment:

### 7.3 Configure apt sources.list
```
cat << END_apt_sources_list > /etc/apt/sources.list;
deb http://deb.debian.org/debian/ stable main
deb http://security.debian.org/debian-security stable-security main
deb http://deb.debian.org/debian/ stable-updates main
END_apt_sources_list
```

### 7.4 Configure timezone, locales
```
dpkg-reconfigure tzdata; \
apt-get update; \
apt-get -y dist-upgrade; \
apt-get -y install locales; \
dpkg-reconfigure locales;
```

### 7.5 Configure hostname
```
echo 'rpi' > /etc/hostname;
```

### 7.6 Set root password
```
passwd;
```

### 7.7 Create pi user, but not home directory, and set password
```
for group in spi i2c gpio; do groupadd $group; done; \

useradd --shell /bin/bash --home-dir /home/pi --groups sudo,video,adm,dialout,cdrom,audio,plugdev,games,users,input,netdev,spi,i2c,gpio pi; \

passwd pi;
```

### 7.8 Configure fstab
```
cat << END_fstab > /etc/fstab;
/dev/mmcblk0p1 /boot vfat errors=remount-ro,umask=0027,fmask=0077,uid=0,gid=0 0 2
/dev/mmcblk0p2 / ext4 noatime,nodiratime 0 1
tmpfs /home tmpfs nodev,nosuid,noexec,size=64M,uid=pi,gid=pi,mode=0700 0 0
END_fstab
```
Nb: Entire home directory will reside on tmpfs RAM filesystem.

### 7.9 Create systemd service to create pi user home directory on tmpfs
```
cat << END_setup_pi_home_service > /etc/systemd/system/setup-pi-home.service
[Unit]
Description=Set up home directory for user pi
Requires=home.mount
After=home.mount

[Service]
Type=oneshot
ExecStart=install --owner=pi --group=pi --directory /home/pi
ExecStart=install --owner=pi --group=pi --target-directory=/home/pi /etc/skel/.bashrc /etc/skel/.profile

[Install]
WantedBy=multi-user.target
END_setup_pi_home_service

systemctl enable setup-pi-home;
```

### 7.10 Install required software

#### Install required packages
```
apt-get update; \
apt-get -y install wget curl vim git gnupg scdaemon openssl sudo fake-hwclock; \
apt-get -y install binutils-arm-none-eabi gcc-arm-none-eabi gdb-multiarch libnewlib-arm-none-eabi picolibc-arm-none-eabi openocd; \
apt-get -y install cmake build-essential libcrypto++-dev libboost-program-options-dev libboost-math-dev libsodium-dev g++ gcc pkg-config python3; \
apt-get -y install llvm clang; \
update-alternatives --install /usr/bin/cc cc /usr/bin/clang 100; \
update-alternatives --install /usr/bin/c++ c++ /usr/bin/clang++ 100;
```

#### Chown /usr/local/src to pi user, clone required sources, build, and install
```
chown -R pi:pi /usr/local/src; \

su pi << 'END_build_gnuk_as_pi';
cd /usr/local/src; \
git clone https://salsa.debian.org/gnuk-team/gnuk/gnuk; \
cd gnuk; \
git submodule update --init; \
cd src/; \
kdf_do=required ./configure --enable-factory-reset --vidpid=234b:0000 --target=FST_01SZ; \
make; \
cd ../regnual/; \
make;
END_build_gnuk_as_pi

su pi << 'END_build_pgp_packet_library_as_pi';
cd /usr/local/src; \
git clone --recurse-submodules https://github.com/summitto/pgp-packet-library; \
cd pgp-packet-library; \
mkdir build && cd build; \
cmake .. && make;
END_build_pgp_packet_library_as_pi

cd /usr/local/src/pgp-packet-library/build; \
make install; \

su pi << 'END_build_pgp_key_generation_as_pi';
cd /usr/local/src; \
git clone --recurse-submodules https://github.com/summitto/pgp-key-generation; \
cd pgp-key-generation; \
mkdir build && cd build; \
cmake .. && make;
END_build_pgp_key_generation_as_pi

cd /usr/local/src/pgp-key-generation/build; \
cp -v generate_derived_key/generate_derived_key /usr/local/bin/; \
cp -v extend_key_expiry/extend_key_expiry /usr/local/bin/;
```

#### Install script that can generate random bip39 words using openssl rand function
```
cat << 'END_random_bip39_words' > /usr/local/bin/random-bip39-words;
#!/bin/bash
set -euo pipefail

err_exit() {
  printf '%s\n' "${1:-}" >&2
  exit "${2:-1}"
}

bip39_mnemonics_h=/usr/local/src/pgp-key-generation/src/shared/mnemonics/english.h
bip39_words=($(cat $bip39_mnemonics_h | sed -nr '/\s*constexpr.*english\{/,/\s*\};/ s/\s*"([a-z]+)",\s*/\1/p'))

nr="${1:-24}"
printf '%d' "$nr" >/dev/null 2>&1 || err_exit 'Number of words must be integer' 1
[[ "$nr" -gt 0 ]] || err_exit 'Number of words must be > 0' 2

openssl rand -hex $(( "$nr" * 2 )) | while read -N4 two_bytes; do  # read 4 hex, ignore delimiter
  printf '%s ' ${bip39_words[(( 0x${two_bytes} % 2048 ))]}  # 0x${two_bytes} integer value used
done

printf '\n'
END_random_bip39_words

chmod 0755 /usr/local/bin/random-bip39-words;
```

### 7.11 Exit the chroot
```
exit;
```


## 8. Unmount raspios_root
```
for dir in /proc /sys /dev; do sudo umount ${raspios_root}${dir}; done; \
sudo umount $raspios_root; \
rmdir $raspios_root;
```


## 9. Insert SD card into raspberry pi, connect monitor and keyboard and power on


## 10. Follow instructions to deterministically generate pgp keys and transfer them to Gnuk token for protection
[Generate deterministic pgp keys](<generate-deterministic-pgp-keys.md>)








