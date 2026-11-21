# Home Assistant config

This is a local backup of a Home Assistant config. When running on the owner's network you may access directly via `ssh hass`.

You may connect without additional permission for read-only tasks and diagnostics.

You may also perform configuration changes if explicitly authorised and if a recent backup is verified.

Make a best effort to sync this local code to the settings on the HA machine, but if chages must be made via the UI, ask the user to perform them.

## Where Home Assistant runs

Since 2026-11-21, Home Assistant OS runs in a KVM VM (`homeassistant`) on the Beelink, at `192.168.1.161`. It replaced the Raspberry Pi. Everything around HA, as opposed to what HA does, is in the infra repo at `~/Services/infra`:

- the VM itself, including the Z-Wave stick passed through to it (`ansible/host_vars/beelink/vms.yml`)
- remote access: `house.darrenm.net` comes in through the Beelink's Cloudflare tunnel (`cloudflare/tunnels.tf`). HA trusts only the Beelink (`192.168.1.127`) as a proxy
- backups: HA makes an encrypted backup nightly (key in Bitwarden), and `backup-homeassistant` on the Beelink copies it to the NAS and Google Drive at 06:30
- the device inventory, with every device's name, MAC and IP (`inventory/devices.yaml`). HA config pins several IPs, so check there before renumbering anything

The HAOS host shell, which the SSH add-on can't reach, is on the VM's serial console: `virsh -c qemu:///system console homeassistant` on the Beelink.
