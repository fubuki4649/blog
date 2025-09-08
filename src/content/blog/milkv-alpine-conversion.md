---
title: 'Porting Alpine Linux to a RISC-V Microcontroller'
description: 'Running Alpine on the MilkV Duo S'
#heroImage: 'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSgVfHORQFLyUf_rNove-xUmxIskDeMJ63REz_YIMQ6S0vCyQdkBvJos4igKspvCgpqnpy8h0xM--1uckzZIxDgyoHy37-MowkF-YzvVx8'
pubDate: 'Sept 8, 2025'
---

A few months back I had gotten my hands on this[^1] RISC-V microcontroller which advertised itself to be capable of running
Linux. The prospect of programming a microcontroller the same way I would any computer sounded totally awesome, but 
could it really be that simple?

Turns out, the answer is no. Instead of mainline kernel support, the board came with a custom buildroot instead, no package manager
among other annoyances. Naturally, I decided to remedy this by replacing the rootfs with something more sensible for an 
embedded SBC, like Alpine Linux.

In this blog post, I will explain how I accomplished this.

#### Creating an Alpine `chroot`

First, let's create an alpine root at `/alpine`. We can grab a copy of the latest Alpine rootfs [here](https://dl-cdn.alpinelinux.org/latest-stable/releases/riscv64/).

```shell
mkdir /alpine && cd /alpine
wget -O- https://dl-cdn.alpinelinux.org/latest-stable/releases/riscv64/alpine-minirootfs-{LATEST_VER}-riscv64.tar.gz | tar xz
```

We'll need to access the real system devices inside the chroot:
```shell
for dir in /proc /sys /dev /dev/pts; do
    mount --bind "$dir" "/alpine/$dir"
done
```

We also need to move some configs into the new root as well:
```shell
cp /etc/resolv.conf /etc/fstab /alpine/etc
cp /etc/network/interfaces /alpine/etc/network
```

Let's also make sure we can still access the old root from inside the `chroot`:
```shell
mkdir /alpine/old_root
mount --bind / /alpine/old_root
```

Now, we can enter the chroot and move the old root out of the way:
```shell
chroot /alpine /bin/ash
mkdir /old_root/old
mv old_root/* old_root/old
```

Then we can move the Alpine root into position:
```shell
apk add rsync
cd /
rsync -ax bin etc home lib media mnt opt root sbin srv tmp usr var
```

Now we can exit the chroot and delete `/alpine`.


#### Configuring Alpine

At this point, we have successfully replaced our root with Alpine. However, as the `minirootfs` is designed for docker 
containers, it doesn't have things like an init system or an SSH server among other things. We will need them to boot 
and access our new Alpine installation.
```shell
apk add openrc dropbear vim chrony
rc-update add chrony default
rc-update add dropbear default
```

Since the MilkV Duo S doesn't have normal `/dev/ttyX` devices, let's comment out everything in `/etc/inittab`. Optionally, 
uncomment the `ttyS0` line. This will allow us to use connect to the board via UART console if anything ever goes wrong.

SSH need a network connection to work, so add the following to `/etc/network/interfaces`
```
auto eth0
iface eth0 inet dhcp
    pre-up ip link set dev eth0 address <YOUR_MAC_ADDR>
```

Since the MilkV Duo S doesn't have a fixed MAC address, we'll need to fix one manually. Replace `<YOUR_MAC_ADDR>` with 
any MAC address unique within the network. I suggest using `ip a` and taking the preexisting one.

At this point, we should be ready to reboot. However, there are a few more steps left for a fully functional system.


#### Copying Board-Specific Drivers

If we want things like the TPU and Wi-Fi to work, we need to copy some drivers from the old root. Luckily for us, more are 
located in one place under `/mnt`. While this is very bad practice, I found it much more convenient to leave it be.
```shell
cp -ar /old/mnt/* /mnt/
cp -a /old/etc/uhubon.sh /old/etc/run_usb.sh /etc/
cp -a /old/usr/bin/duo-pinmux /old/usr/bin/cvi-pinmux /usr/bin/
cp -a /old/usr/sbin/wiringx-* /usr/sbin/
cp -a /old/usr/lib/libwiringx.so /usr/lib/
```

The old `/etc/profile` contains some exports we will need. Since it's better than the one that Alpine ships with, 
let's replace Alpine's `/etc/profile` with it.
```shell
cp -a /old/etc/profile /etc/profile
```

#### Loading Firmware on Startup

To do this, create `/etc/init.d/firmware` with the following content:
```shell
#!/sbin/openrc-run

name="Firmware loader"
description="Load kernel modules and start user apps"

# Dependencies: make sure root fs and basic system services are ready
depend() {
    need localmount
    after sysinit
}

start() {
    ebegin "Initializing MPP system"

    export USERDATAPATH=/mnt/data/
    export SYSTEMPATH=/mnt/system/

    # Load kernel modules
    if [ -d "$SYSTEMPATH/ko" ]; then
        sh "$SYSTEMPATH/ko/loadsystemko.sh"
    fi

    # Start system apps
    for f in duo-init.sh blink.sh usb.sh; do
        if [ -f "$SYSTEMPATH/$f" ]; then
            "$SYSTEMPATH/$f" &
        fi
    done

    # Start user auto scripts
    for f in auto.sh; do
        if [ -f "$USERDATAPATH/$f" ]; then
            usleep 30000
            "$USERDATAPATH/$f" &
        elif [ -f "$SYSTEMPATH/$f" ]; then
            usleep 30000
            "$SYSTEMPATH/$f" &
        fi
    done

    eend 0
}

stop() {
    # Optionally implement stopping/killing background scripts
    ebegin "Stopping firmware scripts"
    # pkill -f duo-init.sh (or other scripts) if needed
    eend 0
}
```

And make sure that it runs on startup:
```shell
rc-update add firmware default
```

Reboot the board to see the changes take effect.


#### Conclusion

At this point, you should now have Alpine Linux working on the MilkV Duo S, without sacrificing any of the board's 
peripheral functionalities. I hope you found this interesting or helpful!


##### Notes

[^1]: [MilkV Duo S](https://milkv.io/docs/duo/getting-started/duos)
