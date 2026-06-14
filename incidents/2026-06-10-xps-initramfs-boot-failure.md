---

# Incident Report: XPS 13 Boot Failure — Broken initramfs After Kernel Update

Date: 2026-06-10
Severity: Medium
Status: Resolved
System: Dell XPS 13 9310 "Edith" — Zorin OS 18.1 Pro, LUKS full-disk encryption

## Summary

System failed to boot after an apt session that installed Docker alongside a new kernel (6.17.0-35-generic). The boot process hung at the splash screen with no visible error. Root cause was an incomplete initramfs generated for the new kernel during the same apt run.

## Timeline

- Apt session — Docker CE installed; kernel 6.17.0-35-generic installed in same session
- Reboot — System hangs at Dell/Zorin splash screen; no progress, no LUKS passphrase prompt
- Diagnosis — GRUB advanced options used to boot older kernel (6.17.0-29-generic); system booted normally
- Root cause confirmed — New kernel had incomplete/broken initramfs; LUKS passphrase prompt was hidden behind plymouth on the broken kernel
- Remediation — sudo update-initramfs -c -k 6.17.0-35-generic run from working older kernel; sudo update-grub run to refresh bootloader
- Resolved — Reboot into 6.17.0-35-generic succeeded

## Root Cause

The initramfs for kernel 6.17.0-35-generic was not correctly generated during the apt session. This is a known risk when multiple large packages (kernel + Docker) are installed in a single apt run — the initramfs generation step can fail silently or produce an incomplete image. With LUKS full-disk encryption, a broken initramfs means the decryption prompt never appears, causing an apparent hang at the splash screen.

## Resolution

From the working older kernel:

sudo dpkg --configure -a
sudo apt update && sudo apt install -f
sudo update-initramfs -c -k 6.17.0-35-generic
sudo update-grub

All commands completed cleanly. Subsequent reboot into the new kernel succeeded.

## Prevention

- Run sudo update-initramfs -u explicitly after any apt session that installs or upgrades a kernel, especially when other large packages are installed in the same run
- On LUKS-encrypted systems, a boot hang with no passphrase prompt is almost always an initramfs issue, not a hardware failure — boot into advanced GRUB options and try the previous kernel first
- Keep at least one prior kernel available in GRUB for recovery

---