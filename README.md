# Fixing "Where Winds Meet" Instant Crash on openSUSE with Flatpak Steam

## The Problem

When trying to run "Where Winds Meet" on openSUSE Tumbleweed with Steam installed via Flatpak, the game crashed instantly without any visible error message. The root cause was **SELinux** (Security-Enhanced Linux) blocking Wine/Proton from marking memory regions as executable, which is required for Windows games to run under Linux.

### Error Symptoms

- Game launches but crashes immediately
- No error dialog appears
- Steam shows the game as "running" briefly, then stops

### Root Cause

openSUSE Tumbleweed switched to SELinux as the default security framework in February 2025. SELinux's default policies prevent Wine/Proton (running inside Flatpak's sandboxed environment) from performing memory protection operations needed for Windows executable compatibility.

---

## How to Get the Log File

With Flatpak Steam, logs are **not** stored in your regular home directory. Instead, they're located in Flatpak's isolated container directory.

### Log File Location

```
~/.var/app/com.valvesoftware.Steam/home/steam-<APPID>.log
```


For "Where Winds Meet" specifically (App ID: 3564740):
```
~/.var/app/com.valvesoftware.Steam/home/steam-3564740.log
```


### How to Enable Proton Logging

By default, Proton doesn't create detailed logs. To enable logging:

1. **Right-click the game in Steam** → Properties
2. **Add to Launch Options:**


```
PROTON_LOG=1%command%
```

3. **Launch the game** (it will crash, but create the log)
4. **View the log:**

```
cat ~/.var/app/com.valvesoftware.Steam/home/steam-3564740.log
```


### Key Error Indicators in the Log

Look for these critical errors:

```
err:virtual:map_image_into_view failed to set 60000020 protection on L"\??\C:\windows\system32\ntdll.dll" section .text, noexec filesystem?
err:virtual:virtual_setup_exception stack overflow
```


These indicate SELinux is blocking memory execution permissions.

---

## The Fix

### Quick Test (Temporary Solution)

Temporarily disable SELinux to verify it's the problem:


```
sudo setenforce 0
```


Try launching the game. If it works, SELinux was definitely the issue. 

**Warning:** This disables a security layer and should only be used for testing.

Re-enable SELinux after testing:


```
sudo setenforce 1
```



### Permanent Solution (Recommended)

Install the **SELinux Gaming Policy Package** that specifically allows Wine, Proton, Steam, and Lutris to run properly while keeping SELinux protection active:

```
sudo zypper in selinux-policy-targeted-gaming
```


After installation:

1. Restart Steam (close it completely, not just minimize)
2. Launch the game normally

This package adds targeted SELinux exceptions for gaming software without compromising overall system security.

### Verification

After applying the fix, check that SELinux is still enforcing:


```
sudo getenforce
```


Should return: `Enforcing`

Launch the game—it should now work without crashes.

---

## Additional Context

### Why This Happened

- **Flatpak uses bubblewrap** to create isolated mount namespaces for sandboxing
- **SELinux blocks Wine/Proton** from marking memory regions as executable within these namespaces
- **The gaming policy package** specifically allows this behavior while maintaining security for the rest of the system

### System Information

- **Distribution:** openSUSE Tumbleweed
- **Steam:** Installed via Flatpak (`com.valvesoftware.Steam`)
- **Proton Version:** 10.0 (or Experimental)
- **Security Framework:** SELinux (default since February 2025 on Tumbleweed)

### Important Note

When installing Steam as an RPM package (not Flatpak), the `selinux-policy-targeted-gaming` package is automatically installed as a dependency. However, for Flatpak installations, this must be manually installed.

---

## Summary

The problem was SELinux blocking Proton's memory execution operations. The log file is located in `~/.var/app/com.valvesoftware.Steam/home/` (not your regular home directory) because Flatpak sandboxes applications. The proper fix is installing `selinux-policy-targeted-gaming`, which allows gaming while maintaining security. **Never leave SELinux in permissive mode permanently**—it's a significant security risk.

---

## References

- [openSUSE SELinux Gaming Documentation](https://security.opensuse.org/2025/06/06/selinux-gaming.html)
- [SELinux Policy Package Info](https://software.opensuse.org/package/selinux-policy-targeted-gaming)
