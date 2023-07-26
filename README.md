# Smart-speaker
This repository is a quick how-to with the objective of making any smart speaker smarter and any dumb speaker smart.
Concretely, we are going to make any speaker compatible with Airplay 2, allowing multi-room audio (i.e., you can choose one or more speakers to play music from).

This involves the following steps:
- Hardware:
	- Install any HiFi hat on a Raspberry PI Zero W
	- Design and 3D print a component to physically install the Raspberry PI Zero W inside the (smart)speaker
	- Make any necessary modifications to the PCBs to connect all the three components in a professional way (the Raspberry, the HiFi hat and the (smart)speaker)
- Software
	- Install the awesome shairport-sync with all dependencies etc.
	- Make the Rapsberry PI Zero W read-only, in order to prevent damages to the microSD card

# License


# Hardware
Installing the HiFi hat is pretty straight forward, you just 



# Software
Premise: I assume that we are running a Raspberry Pi with a headless (we don't need the GUI) Raspberry OS based on debian-buster. I guess it would work also on other versions, but I cannot guarantee it.
You should run everything as root.

## Install shairport-sync
The source of the following is the repository of shairport-sync.

1. Install dependencies
```
apt update
apt upgrade
apt install --no-install-recommends build-essential git autoconf automake libtool libpopt-dev libconfig-dev libasound2-dev avahi-daemon libavahi-client-dev libssl-dev libsoxr-dev libplist-dev libsodium-dev libavutil-dev libavcodec-dev libavformat-dev uuid-dev libgcrypt-dev xxd
```

2. Build and install NQPTP
Install the software using git
```
git clone https://github.com/mikebrady/nqptp.git
cd nqptp
autoreconf -fi
./configure --with-systemd-startup
make
make install
```

Enable the service unit
```
systemctl enable nqptp
systemctl start nqptp
```

Test the installation
```
systemctl status nqptp
```
Just make sure you get the green light

3. Build and install shairport-sync
Install the software using git
```
git clone https://github.com/mikebrady/shairport-sync.git
cd shairport-sync
autoreconf -fi
./configure --sysconfdir=/etc --with-alsa --with-soxr --with-avahi --with-ssl=openssl --with-systemd --with-airplay-2
make
make install
```
Enable the service unit
```
systemctl enable shairport-sync
systemctl start shairport-sync
```
Test the service
```
systemctl status shairport-sync
```
Make sure you get the green light

4. Make the file-system read-only
We are doing this to make sure no data is written to the microSD card in order to give it long life.
The source of the following instructions is [this](https://medium.com/@andreas.schallwig/how-to-make-your-raspberry-pi-file-system-read-only-raspbian-stretch-80c0f7be7353).

- Prepare the system
```
apt remove --purge triggerhappy anacron logrotate dphys-swapfile xserver-common lightdm
apt autoremove --purge
```
- Remove some startup scripts
```
systemctl disable bootlogs
systemctl disable console-setup
```
- Replace your log manager
```
apt install busybox-syslogd
dpkg --purge rsyslog
```
- Disable swap and filesystem check and set it to read-only
```
nano /boot/cmdline.txt
```
Add the following three words at the end of the line, when you're done use ctrl+x, y, enter
```
fastboot noswap ro
```
For example, you might obtain something like this:
```
dwc_otg.lpm_enable=0 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait fastboot noswap ro
```
- Move some system files to temp filesystem
```
rm -rf /var/lib/dhcp /var/lib/dhcpcd5 /var/run /var/spool /var/lock /etc/resolv.conf
ln -s /tmp /var/lib/dhcp
ln -s /tmp /var/lib/dhcpcd5
ln -s /tmp /var/run
ln -s /tmp /var/spool
ln -s /tmp /var/lock
touch /tmp/dhcpcd.resolv.conf
ln -s /tmp/dhcpcd.resolv.conf /etc/resolv.conf
```
- Update the systemd random seed
```
rm /var/lib/systemd/random-seed
ln -s /tmp/random-seed /var/lib/systemd/random-seed
nano /lib/systemd/system/systemd-random-seed.service
```
Add the following line under the `[Service]` section
```
ExecStartPre=/bin/echo "" >/tmp/random-seed
```
For example, you might obtain something like this:
```
[Service]Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/echo "" >/tmp/random-seed
ExecStart=/lib/systemd/systemd-random-seed load
ExecStop=/lib/systemd/systemd-random-seed save
```
- Reload the systemd service:
```
systemctl daemon-reload
```
- Make the file-systems read-only
```
nano /etc/fstab
```
Add the string `,ro` to all block devices, like so:
```
proc            /proc           proc    defaults             0       0
/dev/mmcblk0p1  /boot           vfat    defaults,ro          0       2
/dev/mmcblk0p2  /               ext4    defaults,noatime,ro  0       1
```
Add the following lines at the end of the file
```
tmpfs           /tmp            tmpfs   nosuid,nodev         0       0
tmpfs           /var/log        tmpfs   nosuid,nodev         0       0
tmpfs           /var/tmp        tmpfs   nosuid,nodev         0       0
```
Adding some fancy commands to switch between read-only and read-write modes
```
nano /etc/bash.bashrc
```
Add the following lines at the end of the file
```
set_bash_prompt() {
fs_mode=$(mount | sed -n -e "s/^\/dev\/.* on \/ .*(\(r[w|o]\).*/\1/p")
PS1='\[\033[01;32m\]\u@\h${fs_mode:+($fs_mode)}\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
}
alias ro='sudo mount -o remount,ro / ; sudo mount -o remount,ro /boot'
alias rw='sudo mount -o remount,rw / ; sudo mount -o remount,rw /boot'
PROMPT_COMMAND=set_bash_prompt
```
Now let's make sure that the system goes back to read-only once you log out
```
nano /etc/bash.bash_logout
```
Add the following lines at the end of the file
```
mount -o remount,ro /
mount -o remount,ro /boot
```

5. All done! Just reboot
```
shutdown -r now
```

6. Optional extra
You might want to change the name of the device that appears when you play music from your device. You can do this as follows:
```
nano /etc/shairport-sync.conf
```
Add the following lines at the end of the file
```
general = {
  name = "Your desired stereo name";
};
```
Restart the shairport-sync service
```
service shairport-sync restart
```
	- Install any HiFi hat on a Raspberry PI Zero W
	- Design and 3D print a component to physically install the Raspberry PI Zero W inside the (smart)speaker
	- Make any necessary modifications to the PCBs to connect all the three components in a professional way (the Raspberry, the HiFi hat and the (smart)speaker)
- Software
	- Install the awesome shairport-sync with all dependencies etc.
	- Make the Rapsberry PI Zero W read-only, in order to prevent damages to the microSD card

# License


# Hardware
Installing the HiFi hat is pretty straight forward, you just 



# Software
Premise: I assume that we are running a Raspberry Pi with a headless (we don't need the GUI) Raspberry OS based on debian-buster. I guess it would work also on other versions, but I cannot guarantee it.
You should run everything as root.

## Install shairport-sync
The source of the following is the repository of shairport-sync.

1. Install dependencies
```
apt update
apt upgrade
apt install --no-install-recommends build-essential git autoconf automake libtool libpopt-dev libconfig-dev libasound2-dev avahi-daemon libavahi-client-dev libssl-dev libsoxr-dev libplist-dev libsodium-dev libavutil-dev libavcodec-dev libavformat-dev uuid-dev libgcrypt-dev xxd
```

2. Build and install NQPTP
Install the software using git
```
git clone https://github.com/mikebrady/nqptp.git
cd nqptp
autoreconf -fi
./configure --with-systemd-startup
make
make install
```

Enable the service unit
```
systemctl enable nqptp
systemctl start nqptp
```

Test the installation
```
systemctl status nqptp
```
Just make sure you get the green light

3. Build and install shairport-sync
Install the software using git
```
git clone https://github.com/mikebrady/shairport-sync.git
cd shairport-sync
autoreconf -fi
./configure --sysconfdir=/etc --with-alsa --with-soxr --with-avahi --with-ssl=openssl --with-systemd --with-airplay-2
make
make install
```
Enable the service unit
```
systemctl enable shairport-sync
systemctl start shairport-sync
```
Test the service
```
systemctl status shairport-sync
```
Make sure you get the green light

4. Make the file-system read-only
We are doing this to make sure no data is written to the microSD card in order to give it long life.
The source of the following instructions is [this](https://medium.com/@andreas.schallwig/how-to-make-your-raspberry-pi-file-system-read-only-raspbian-stretch-80c0f7be7353).

- Prepare the system
```
apt remove --purge triggerhappy anacron logrotate dphys-swapfile xserver-common lightdm
apt autoremove --purge
```
- Remove some startup scripts
```
systemctl disable bootlogs
systemctl disable console-setup
```
- Replace your log manager
```
apt install busybox-syslogd
dpkg --purge rsyslog
```
- Disable swap and filesystem check and set it to read-only
```
nano /boot/cmdline.txt
```
Add the following three words at the end of the line, when you're done use ctrl+x, y, enter
```
fastboot noswap ro
```
For example, you might obtain something like this:
```
dwc_otg.lpm_enable=0 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait fastboot noswap ro
```
- Move some system files to temp filesystem
```
rm -rf /var/lib/dhcp /var/lib/dhcpcd5 /var/run /var/spool /var/lock /etc/resolv.conf
ln -s /tmp /var/lib/dhcp
ln -s /tmp /var/lib/dhcpcd5
ln -s /tmp /var/run
ln -s /tmp /var/spool
ln -s /tmp /var/lock
touch /tmp/dhcpcd.resolv.conf
ln -s /tmp/dhcpcd.resolv.conf /etc/resolv.conf
```
- Update the systemd random seed
```
rm /var/lib/systemd/random-seed
ln -s /tmp/random-seed /var/lib/systemd/random-seed
nano /lib/systemd/system/systemd-random-seed.service
```
Add the following line under the `[Service]` section
```
ExecStartPre=/bin/echo "" >/tmp/random-seed
```
For example, you might obtain something like this:
```
[Service]Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/echo "" >/tmp/random-seed
ExecStart=/lib/systemd/systemd-random-seed load
ExecStop=/lib/systemd/systemd-random-seed save
```
- Reload the systemd service:
```
systemctl daemon-reload
```
- Make the file-systems read-only
```
nano /etc/fstab
```
Add the string `,ro` to all block devices, like so:
```
proc            /proc           proc    defaults             0       0
/dev/mmcblk0p1  /boot           vfat    defaults,ro          0       2
/dev/mmcblk0p2  /               ext4    defaults,noatime,ro  0       1
```
Add the following lines at the end of the file
```
tmpfs           /tmp            tmpfs   nosuid,nodev         0       0
tmpfs           /var/log        tmpfs   nosuid,nodev         0       0
tmpfs           /var/tmp        tmpfs   nosuid,nodev         0       0
```
Adding some fancy commands to switch between read-only and read-write modes
```
nano /etc/bash.bashrc
```
Add the following lines at the end of the file
```
set_bash_prompt() {
fs_mode=$(mount | sed -n -e "s/^\/dev\/.* on \/ .*(\(r[w|o]\).*/\1/p")
PS1='\[\033[01;32m\]\u@\h${fs_mode:+($fs_mode)}\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
}
alias ro='sudo mount -o remount,ro / ; sudo mount -o remount,ro /boot'
alias rw='sudo mount -o remount,rw / ; sudo mount -o remount,rw /boot'
PROMPT_COMMAND=set_bash_prompt
```
Now let's make sure that the system goes back to read-only once you log out
```
nano /etc/bash.bash_logout
```
Add the following lines at the end of the file
```
mount -o remount,ro /
mount -o remount,ro /boot
```

5. All done! Just reboot
```
shutdown -r now
```

6. Optional extra

You might want to change the name of the device that appears when you play music from your device. You can do this as follows:
```
nano /etc/shairport-sync.conf
```
Add the following lines at the end of the file
```
general = {
  name = "Your desired stereo name";
};
```
Restart the shairport-sync service
```
service shairport-sync restart
```
