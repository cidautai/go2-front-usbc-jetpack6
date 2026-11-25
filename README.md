# Unitree Go2 EDU: front USB-C port dead after reflashing the Jetson Orin NX to JetPack 6

**Symptom.** After reflashing the Go2's onboard Jetson (Orin NX on the Unitree expansion board) from the factory image (Ubuntu 20.04 / JetPack 5) to JetPack 6, the **front USB-C port stops working in host mode**: nothing plugged into it enumerates, not even a USB 2 mouse. Flashing through that same port still works. The rear ports keep working, but the one we tested only links at USB 2 (480 Mb/s), so a RealSense D435i runs degraded there.

**Cause.** The factory image has an `/etc/rc.local` that drives a GPIO high on every boot. A reflash wipes it. Without that GPIO the front USB-C port stays disabled. It is not a device-tree problem and not a USB role (OTG/host) problem.

**Fix.** Recreate what the factory `rc.local` did, as a systemd service. No device tree, bootloader or firmware changes are needed.

## Tested on

| | |
|---|---|
| Robot | Unitree Go2 EDU |
| Module | Jetson Orin NX 16 GB (p3767-0000) on the Unitree expansion board |
| Factory system | Ubuntu 20.04, L4T R35.3.1 (JetPack 5.1.1), kernel 5.10 |
| After reflash | Ubuntu 22.04, JetPack 6.2.1, L4T R36.4.x, kernel 5.15.148-tegra |
| Flashed with | stock NVIDIA BSP, `jetson-orin-nano-devkit` configuration |

One unit only. If your hardware revision differs, check your own factory image first (see "Verify on your unit").

## What the factory image does

From the factory `/etc/rc.local`:

```sh
busybox devmem 0x02430030 w 0x004
echo 446 > /sys/class/gpio/export
echo out > /sys/class/gpio/PP.06/direction
echo 1 > /sys/class/gpio/PP.06/value
```

- `0x02430030` is the pinmux register of pad `soc_gpio25_pp6`. Writing `0x004` makes it a plain GPIO output. With the stock NVIDIA pinmux the pad reads `0x70` (input, tristated), so nothing drives it.
- `PP.06` is that pad's GPIO: `gpio-446` on both kernels, line 98 of `gpiochip0` on JetPack 6.

We did not trace the board to see what PP.06 switches (port power, a mux, or a redriver). What we measured is the effect: with PP.06 low the front port is dead; with PP.06 high it works at USB 3 in both plug orientations.

## Quick test (not persistent, a reboot undoes it)

```bash
# Current state, for reference
sudo busybox devmem 0x02430030            # stock value: 0x00000070
sudo gpioinfo | grep "PP.06"              # stock: unused, input

# Enable the port
sudo busybox devmem 0x02430030 w 0x004
sudo sh -c 'gpioset --mode=signal $(gpiofind PP.06)=1'    # stays in foreground; Ctrl-C releases it
```

Plug a device into the front USB-C port from another terminal and check `lsusb -t`. A USB 3 device should appear under bus 2 at `5000M`.

`gpioset --mode=signal` is libgpiod v1 syntax (the version shipped with Ubuntu 22.04).

## Persistent fix

Copy [`systemd/go2-usbc-front.service`](systemd/go2-usbc-front.service) to `/etc/systemd/system/go2-usbc-front.service`:

```ini
[Unit]
Description=Enable the front USB-C port of the Unitree board (GPIO PP.06 high, as the factory rc.local did)
DefaultDependencies=no
After=sysinit.target
Before=systemd-user-sessions.service multi-user.target

[Service]
Type=simple
ExecStartPre=/usr/bin/busybox devmem 0x02430030 w 0x004
ExecStart=/bin/sh -c 'exec /usr/bin/gpioset --mode=signal $(/usr/bin/gpiofind PP.06)=1'
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

```bash
sudo install -m 644 systemd/go2-usbc-front.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now go2-usbc-front.service
```

To remove it:

```bash
sudo systemctl disable --now go2-usbc-front.service
sudo rm /etc/systemd/system/go2-usbc-front.service
sudo systemctl daemon-reload
```

Result on our unit, after a reboot: the service starts on its own and a RealSense D435i in the front port enumerates on `usb 2-1` as SuperSpeed (5000M).

## Verify on your unit

If you still have the factory system or a disk image of it, confirm it does the same before copying this:

```bash
cat /etc/rc.local
```

If you are about to reflash, **save `/etc/rc.local` and take a full image of the NVMe first**. That file is the only board-specific piece we found in the factory boot sequence.

## What it is not

These are the things we checked and ruled out, in case you are halfway down the same path:

- **Device tree.** The factory system already booted the stock NVIDIA dev-kit device tree, and its USB section (port map, VBUS regulators, Type-C connector node) is identical to the JetPack 6 one.
- **USB role.** Forcing `host` on `/sys/class/usb_role/usb2-0-role-switch/role` does not bring the port back.
- **The `fusb301` error.** `fusb301 1-0025: failed to read device id` appears in the kernel log on both the factory system and JetPack 6. The board does not have that chip; the message is harmless.
- **The rear port we used as a workaround.** It maps to `usb2-2`, which has no USB 3 lane assigned in the device tree, so it can only do USB 2.

## Related: RealSense depth disappears once the camera is on USB 3

This one is about `realsense2_camera`, not about the port, but it shows up right after the fix.

On USB 3 the driver's default profiles change to depth 848x480x30 and color 1280x720x30. With `align_depth.enable:=true` and those defaults, the node published color but **no depth and no aligned depth at all**. Setting explicit profiles fixed it:

```bash
ros2 launch realsense2_camera rs_launch.py align_depth.enable:=true \
  depth_module.depth_profile:=848x480x30 rgb_camera.color_profile:=848x480x30
```

Measured on the Orin NX with alignment enabled (realsense2_camera 4.58.4, librealsense 2.58.4 from the ROS 2 Humble apt packages, stock kernel):

| Color / depth profile | Aligned depth | Color |
|---|---|---|
| 1280x720x30 / 848x480x30 (USB 3 defaults) | none | ~20 Hz |
| 1280x720x15 / 848x480x15 | 9 Hz | 14 Hz |
| 848x480x30 / 848x480x30 | 20 Hz | 30 Hz |
| 640x480x30 / 640x480x30 | 30 Hz | 30 Hz |

We did not need to rebuild librealsense: both the apt build and a source build delivered depth when used directly.

Not fixed by any of this: the D435i IMU stays disabled on the stock JetPack 6 kernel, which is built without `CONFIG_HID_SENSOR_HUB`.

## Credits and license

Written by Javier Abad Hernández. The investigation was done at [CIDAUT](https://github.com/cidautai) on one of its Go2 EDU robots.

If your unit behaves differently (another board revision, another GPIO), please open an issue so the guide can cover it.

Released under the [MIT License](LICENSE). Use it at your own risk: it writes to a pinmux register and drives a GPIO on your robot's computer.
