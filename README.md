# Arch Linux T2 Suspend/Resume & USB Fix

This guide provides a production-ready fix for Kernel Panics, "Zombie" USB ports (dead xhci), and Wi-Fi authentication failures (wpa_supplicant 2.11 regression) on Apple T2 Macs (MacBook Pro 2019/2020) running Arch Linux.

It specifically addresses race conditions where the T2 bridge (apple-bce) and USB controller fail to initialize simultaneously upon wake, especially when xhci_pci is builtin.


## Environment


```bash
Hardware: MacBook Pro 2019 with Apple T2 Chip (15,2 / 16,1)
OS: Arch Linux
Kernel: 6.17.7-arch1-Watanare-T2-1-t2
Bootloader: systemd-boot / (GRUB not tested yet)
```

## 1. Kernel Parameters (Bootloader Config)
The T2 bridge requires IOMMU to be active to handle DMA requests correctly during resume. The brcmfmac parameter fixes the Wi-Fi connection drop/auth failure.

Add the following options to your bootloader configuration (e.g., /boot/efi/loader/entries/arch.conf for systemd-boot):

```python
intel_iommu=on iommu=pt pcie_ports=compat mem_sleep_default=s2idle brcmfmac.feature_disable=0x82000
```

## 2. The Fix Service: suspend-fix-t2.service

This service uses a Fail-Safe approach. It attempts to unload unstable modules but ignores errors (using the - prefix) if modules are builtin or missing. Crucially, it adds a 3-second delay on resume to allow hardware power-up before reloading drivers.

Create file: /etc/systemd/system/suspend-fix-t2.service

```bash
[Unit]
Description=Fix T2 Mac Hardware (Wifi/Touchbar) on Suspend
Before=sleep.target
StopWhenUnneeded=yes

[Service]
User=root
Type=oneshot
RemainAfterExit=yes

# --- PRE-SLEEP (Unload unstable modules) ---
# The (-) prefix tells systemd to ignore errors if the module is missing or builtin.
# This prevents the script from failing if xhci_pci is compiled into the kernel.

ExecStart=-/usr/bin/systemctl stop tiny-dfr
ExecStart=-/usr/bin/systemctl stop NetworkManager

# Unload Wi-Fi & Touchbar to prevent hang
ExecStart=-/usr/bin/modprobe -r brcmfmac_wcc
ExecStart=-/usr/bin/modprobe -r brcmfmac
ExecStart=-/usr/bin/modprobe -r apple_touchbar

# Try to unload USB/T2 if possible (ignore if builtin)
ExecStart=-/usr/bin/modprobe -r xhci_pci
ExecStart=-/usr/bin/modprobe -r xhci_hcd

# --- POST-RESUME (Reload & Stabilize) ---
# CRITICAL: Wait 3s for hardware power-up to prevent race condition.
# Without this sleep, the system tries to access hardware before it's powered on.
ExecStop=/usr/bin/sleep 3

# Reload modules
ExecStop=/usr/bin/modprobe apple_touchbar
ExecStop=/usr/bin/modprobe brcmfmac
ExecStop=/usr/bin/modprobe brcmfmac_wcc
ExecStop=/usr/bin/modprobe xhci_pci
ExecStop=/usr/bin/modprobe xhci_hcd

# Restart services
ExecStop=-/usr/bin/systemctl start tiny-dfr
ExecStop=-/usr/bin/systemctl start NetworkManager

[Install]
WantedBy=sleep.target
```
## 3. Installation & Enable

This service uses a Fail-Safe approach. It attempts to unload unstable modules but ignores errors (using the - prefix) if modules are builtin or missing. Crucially, it adds a 3-second delay on resume to allow hardware power-up before reloading drivers.

```bash
# 1. Create the service file
sudo nano /etc/systemd/system/suspend-fix-t2.service
# (Paste the content from Section 2 above)

# 2. Reload systemd daemon
sudo systemctl daemon-reload

# 3. Enable the service so it runs on suspend
sudo systemctl enable suspend-fix-t2.service

# 4. Reboot or test immediately
sudo systemctl start suspend-fix-t2.service
```

