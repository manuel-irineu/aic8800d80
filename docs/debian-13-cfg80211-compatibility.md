# Debian 13 cfg80211 Compatibility

## Overview

This note documents a compatibility fix maintained in this fork for the
AIC8800D80 Linux driver. The issue was found while using the driver as an
out-of-tree DKMS module on Debian 13 with AIC8800D80 hardware.

The original project remains the upstream AIC8800D80 driver project. This
document covers only the fork-specific Debian cfg80211 compatibility patch
carried on the branch named below.

## Environment

The fix was developed and validated in this environment:

- Debian GNU/Linux 13
- Linux kernel `6.12.100+deb13-amd64`
- Linux kernel `6.12.101+deb13-amd64`
- AIC8800D80 hardware
- DKMS-based out-of-tree module builds

## Problem

During the upgrade from `6.12.100+deb13-amd64` to
`6.12.101+deb13-amd64`, DKMS rebuilt the external AIC8800D80 modules for the
new kernel. The `aic8800_fdrv` build failed because the driver initialized
`cfg80211_ops.set_monitor_channel` with a callback signature that did not match
the cfg80211 headers exposed by the target kernel.

The relevant compiler diagnostic was the incompatible callback type for
`set_monitor_channel`:

```text
initialization of
'int (*)(struct wiphy *, struct net_device *,
         struct cfg80211_chan_def *)'
from incompatible pointer type
'int (*)(struct wiphy *,
         struct cfg80211_chan_def *)'
```

## Root Cause

The Linux wireless stack exposes driver callbacks through `struct cfg80211_ops`.
For monitor channel changes, the driver supplies a `set_monitor_channel`
callback. The callback signature is not identical across all kernel cfg80211
APIs: some headers expect a `struct net_device *dev` argument between
`struct wiphy *wiphy` and `struct cfg80211_chan_def *chandef`, while others do
not.

Before this fix, the driver selected the callback signature by checking
`LINUX_VERSION_CODE`. That assumes the cfg80211 API shape can be inferred from
the nominal kernel version. The Debian `6.12.101+deb13-amd64` headers used in
this failure exposed a `set_monitor_channel` declaration whose signature did
not match the driver's version-based assumption, so the build selected the
wrong function prototype and failed at compile time.

The important compatibility boundary is therefore the cfg80211 header actually
used for the module build, not only the kernel release number. Distribution
kernels may carry API changes or backports that make version-only checks
insufficient for out-of-tree drivers.

## Fix

Commit `af58065` (`fix: detect cfg80211 monitor channel API`) changed the
driver to detect the available cfg80211 monitor-channel API at build time.

The `aic8800_fdrv` Makefile now inspects the kernel's
`include/net/cfg80211.h` and defines
`AIC_CFG80211_SET_MONITOR_CHANNEL_HAS_DEV` when the
`set_monitor_channel` declaration includes the `struct net_device *dev`
argument. The C sources then use that feature macro to select the matching
function prototype and call form.

This is more robust than relying only on `LINUX_VERSION_CODE` because it checks
the API contract that the module is compiled against.

## Validation

The patch was validated on real AIC8800D80 hardware with Debian 13.

Validation covered:

- driver build against `6.12.100+deb13-amd64`
- driver build against `6.12.101+deb13-amd64`
- DKMS build
- DKMS install
- initramfs generation
- recovery of `dpkg` package configuration after the original DKMS failure
- reboot into `6.12.101+deb13-amd64`
- loading `aic8800_fdrv`
- loading `aic_load_fw`
- loading `aic_zlp_quirk`
- Wi-Fi operation with the AIC8800D80 adapter
- Wi-Fi 5 GHz connection
- Bluetooth operation
- runtime validation after reboot

No additional validation is claimed here.

## DKMS Upgrade Behavior

DKMS rebuilds registered out-of-tree modules when a new kernel is installed.
That is why this issue appeared during the kernel upgrade from
`6.12.100+deb13-amd64` to `6.12.101+deb13-amd64`: the module had to be compiled
again against the new kernel headers, and the cfg80211 callback signature
mismatch was detected by the compiler during that rebuild.

## Recovery / Troubleshooting Notes

The original DKMS failure left the new kernel packages pending configuration.
After the driver compatibility issue was fixed and the module was built and
installed successfully, the package configuration could be completed and the
system booted successfully with the new kernel.

This note intentionally avoids destructive recovery commands. Review the DKMS
and package-manager logs for the exact failing module build before applying any
manual recovery steps.

## Patch

- Branch: `fix/debian-cfg80211-monitor-channel`
- Commit: `af58065 fix: detect cfg80211 monitor channel API`

## Upstream

This compatibility patch is maintained in this personal fork. The original
AIC8800D80 project remains upstream at
[shenmintao/aic8800d80](https://github.com/shenmintao/aic8800d80).
