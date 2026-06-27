---

# NetBird on Synology NAS — Supplementary Guide

This guide supplements the [official NetBird Synology documentation](https://docs.netbird.io/get-started/install/synology) with additional steps discovered through real-world troubleshooting.

## Why this guide exists

The official documentation covers the basic installation but is incomplete in several areas. Following it alone can lead to a broken reinstallation or unexpected network issues.

## What this guide covers

- **Full uninstallation** including terminating the orphaned daemon process that `netbird service uninstall` does not stop
- **Reinstallation** with verification of the `/dev/net/tun` module before starting
- **Reboot script** to ensure the tun module is loaded automatically after every DSM restart
- **Routing conflict warning** when configuring Synology as a routing peer for its own subnet

## The most important gap in the official docs

After running `netbird service uninstall`, the NetBird daemon process continues running in memory as an orphan. A subsequent reinstallation will fail with:

```
Error: call service setConfig method: rpc error: code = Unknown desc = failed to update profile config: config file /var/lib/netbird/default.json does not exist
```

The fix is to manually identify and kill the process before reinstalling. This guide documents the exact steps.

## Related

- [Official NetBird Synology Documentation](https://docs.netbird.io/get-started/install/synology)
- [GitHub Issue #4950 — Routing conflict when peer advertises its own subnet](https://github.com/netbirdio/netbird/issues/4950)

## Tested on

- Synology DS723+ running DSM 7.x
- NetBird v0.73.2

## Contributing

If you have encountered additional issues or improvements, feel free to open an issue or pull request.
