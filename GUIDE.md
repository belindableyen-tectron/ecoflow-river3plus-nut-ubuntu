# EcoFlow RIVER 3 Plus with NUT 2.8.4 on Ubuntu

A documented working setup for monitoring an **EcoFlow RIVER 3 Plus** via its USB HID interface with **Network UPS Tools (NUT)** on Ubuntu, including a battery-percentage-based graceful shutdown configuration.

> **Community case study, not an official EcoFlow or NUT support guide.** Adapt paths and service names to your system. Test shutdown behavior safely before relying on it. This guide is based on a real installation documented on 29 October 2026.

## Hardware and observations

- UPS: EcoFlow RIVER 3 Plus, 286 Wh LiFePO4, 600 W AC output.
- Connected load: Ubuntu server and router, approximately 104 W under typical conditions.
- USB identification: `3746:ffff` (`EcoFlow EF-UPS-RIVER 3 Plus`).
- The device exposes both a CDC ACM serial interface (for example `/dev/ttyACM0`) and a HID UPS interface.
- Disconnecting AC input did not interrupt the server or router in the observed test.
- An AC-loss / restore test confirmed NUT status transitions `OB DISCHRG` and `OL CHRG`.

## Why NUT 2.8.4?

The distribution's NUT 2.8.1 `nutdrv_qx` did not support this unit in this installation. A locally compiled **NUT 2.8.4** build provided the **EcoFlow HID** subdriver in `usbhid-ups`.

The local installation used:

| Purpose | Location |
|---|---|
| NUT source/build tree | `~/nut-2.8.4` |
| Installed binaries | `/opt/nut-2.8.4` |
| Configuration | `/opt/nut-2.8.4/etc/` |
| Driver/server runtime state | `/var/state/ups` |
| USB permissions rule | `/etc/udev/rules.d/99-ecoflow-nut.rules` |

The NUT service account is `nut`, including ownership of `/var/state/ups`. The USB device must also be accessible to the `nut` group.

### USB permissions

The installation used this udev rule in `/etc/udev/rules.d/99-ecoflow-nut.rules`:

```udev
SUBSYSTEM=="usb", ATTR{idVendor}=="3746", ATTR{idProduct}=="ffff", MODE="0660", GROUP="nut"
```

Apply udev changes appropriately for your system and verify the device permissions. Do not assume the USB bus/device number is stable.

## NUT configuration

The following is an illustrative configuration from the working installation. Replace all credentials and review shutdown thresholds for your own equipment.

### `ups.conf`

```ini
user = nut

[ecoflow]
    driver = usbhid-ups
    port = auto
    vendorid = 3746
    productid = ffff
    subdriver = EcoFlow HID
    ignorelb
    override.battery.charge.low = 25
    override.battery.runtime.low = 0
```

The `user = nut` setting was placed **above** the `[ecoflow]` section. The configuration uses a 25% battery-charge threshold and does not rely on the device's runtime estimate.

### `upsd.conf`

```ini
LISTEN 127.0.0.1 3493
```

This binds the NUT server to loopback only.

### `upsd.users`

Create a dedicated monitor user with a strong, unique password and the appropriate primary-monitor role. Do **not** publish your actual password.

### `upsmon.conf`

```ini
MONITOR ecoflow@localhost 1 upsmon REPLACE_WITH_YOUR_PASSWORD primary
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
FINALDELAY 5
RUN_AS_USER nut
```

The original installation restricted sensitive NUT configuration files to `root:nut` with mode `0640`. Ensure that the monitor account in `upsd.users` matches the configuration above.

**Important:** A configured 25% threshold is not itself proof that the exact low-battery trigger was tested end-to-end. Validate your own shutdown sequence under controlled conditions.

## systemd integration

In the documented installation, these units were enabled:

- `nut-driver@ecoflow`
- `nut-server`
- `nut-monitor`
- `nut-driver.target`
- `nut.target`

The build initially used the `nobody` account by default, which caused USB and state-directory permission failures. The installation explicitly used `nut` instead.

The `/usr/lib/tmpfiles.d/nut-common-tmpfiles.conf` state-directory ownership was changed from `nobody nogroup` to `nut nut`. The NUT server also used a systemd drop-in at `/etc/systemd/system/nut-server.service.d/override.conf`:

```ini
[Service]
ExecStart=
ExecStart=/opt/nut-2.8.4/sbin/upsd -F -u nut
```

The driver instance was registered with:

```bash
sudo /opt/nut-2.8.4/libexec/nut-driver-enumerator.sh
```

A rebuild or reinstall can overwrite installed service units or tmpfiles rules. Review them after maintenance; configuring a future build with `--with-user=nut --with-group=nut` may avoid the original default-user mismatch.

## Verify operation

```bash
systemctl is-active nut-driver@ecoflow nut-server nut-monitor
/opt/nut-2.8.4/bin/upsc ecoflow@localhost ups.status
/opt/nut-2.8.4/bin/upsc ecoflow@localhost
```

Expected status meanings:

| Status | Meaning |
|---|---|
| `OL` | On utility power |
| `OL CHRG` | On utility power, charging |
| `OB DISCHRG` | On battery, discharging |

Inspect the full `upsc` output for:

```text
battery.charge.low: 25
battery.runtime.low: 0
driver.flag.ignorelb: enabled
driver.flag.allow_killpower: 0
```

The source runbook records a successful mains-disconnect/reconnect test on 29 October 2026. A subsequent server reboot also confirmed that the driver, server, and monitor services came back active and the UPS reported `OL`.

## Runtime estimates and limitations

The EcoFlow `battery.runtime` value was unreliable in this installation, sometimes showing an implausibly long estimate while on AC. This configuration therefore relies on charge percentage rather than the reported runtime.

A rough battery-energy calculation is useful for planning, **not** as a guarantee of runtime. Actual runtime depends on inverter efficiency, load, battery condition, and shutdown reserve.

The installation deliberately did **not** use `POWERDOWNFLAG` to turn off the EcoFlow AC output after the host shut down, so the router could remain powered.

Check the EcoFlow app's backup-reserve and discharge-limit settings: an output cutoff before the operating system shuts down can cause abrupt power loss. Validate the interaction on your own device.

## Troubleshooting

| Symptom | What to inspect |
|---|---|
| `insufficient permissions on everything` | HID device permissions and whether the driver runs as `nut` |
| `Can't chdir to /var/state/ups: Permission denied` | State-directory ownership and tmpfiles rule |
| `Driver not connected` | `upsd` user, driver status and logs |
| `/dev/ttyACM0 is locked by another process` | Competing older NUT driver/process |

Relevant logs:

```bash
journalctl -u nut-driver@ecoflow -n 25 --no-pager
journalctl -u nut-server -n 25 --no-pager
journalctl -u nut-monitor -n 25 --no-pager
```

Avoid manually starting a second driver while its systemd service is running.

## Scope of validation

**Observed / documented:** USB HID communication; NUT reporting; mains-loss and restoration detection; active services; successful service restart after server reboot.

**Not independently established by the supplied runbook:** a measured, controlled end-to-end shutdown triggered specifically at exactly 25% battery charge; universal compatibility across firmware versions, Linux distributions, or NUT builds.

## References

- [Network UPS Tools](https://networkupstools.org/)
- [NUT source repository](https://github.com/networkupstools/nut)

---

Documentation contributed from a working home-server setup. No serial numbers, private network addresses, or real credentials are included.
