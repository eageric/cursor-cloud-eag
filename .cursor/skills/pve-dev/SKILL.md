---
name: pve-dev
description: >-
  Creates, connects to, and cleans up disposable test VMs on the EAG Proxmox VE
  server through the Cloudflare Access SSH tunnel at eric-pve-ssh.eag.tools.
  Clones the Windows Server 2025 template ws2025-template (VMID 101) or the
  cloud-init Ubuntu 24.04 template ubuntu-2404 (VMID 100). Uses the environment
  key ~/.ssh/pve_ssh. Covers host SSH, cloning, Ubuntu cloud-init, first-boot
  setup, guest-agent IPs, guest SSH through the tunnel, PowerShell remoting,
  snapshots, and cleanup. Use when the user asks to spin up, test on, or tear
  down a VM on the Cloudflare-connected EAG Proxmox host.
disable-model-invocation: true
---

# EAG Proxmox over the Cloudflare SSH tunnel

The Proxmox VE 9.2 server hosts disposable test VMs. From this environment the host is `eric-pve-ssh.eag.tools`. Agents clone a template, run the test, and then ask before deleting anything.

The templates, clone steps, and guest logins match the LAN environment. What changes is the path: the cloud agent reaches the host through Cloudflare Access, and reaches every guest by jumping through that host.

## Access

The environment start script writes the key pair from `PVE_SSH_PRIVATE_B64` and `PVE_SSH_PUBLIC_KEY`:

| File | Mode | Use |
|---|---|---|
| `~/.ssh/pve_ssh` | `600` | Private key. Proxmox host, Windows `Administrator`, Ubuntu `erolleman`. |
| `~/.ssh/pve_ssh.pub` | `644` | Matching public key. |

`/etc/ssh/ssh_config.d/eric-pve-ssh.conf` already sets `User root`, `IdentityFile ~/.ssh/pve_ssh`, `IdentitiesOnly yes`, and `ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h` for `eric-pve-ssh.eag.tools`. Use that host alias. Leave the system file in place.

```bash
KEY="$HOME/.ssh/pve_ssh"
test -r "$KEY" || { echo "missing $KEY; the start script creates it from PVE_SSH_PRIVATE_B64" >&2; exit 1; }
PVE="ssh -o StrictHostKeyChecking=accept-new eric-pve-ssh.eag.tools"
$PVE 'qm list'
```

Guest SSH, for both `10.115.64.x` (`vmbr0`) and `10.20.0.x` (`nat0`):

```bash
JUMP='ssh -o StrictHostKeyChecking=accept-new -W %h:%p eric-pve-ssh.eag.tools'
VMSSH="ssh -i $KEY -o IdentitiesOnly=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=5 -o ProxyCommand=\"$JUMP\" USER@$IP"
```

Set `USER` to `Administrator` or `erolleman` and `IP` to the guest address before expanding `VMSSH`.

- Windows VM login: `Administrator` / `Change_me`, or `~/.ssh/pve_ssh`.
- Ubuntu VM login: `erolleman` with `~/.ssh/pve_ssh`. `sudo` is passwordless. `docker` needs `sudo`.
- The Proxmox web UI listens on the LAN at `https://10.115.64.44:8006`. This tunnel is SSH only. Administer the host with `qm` and `pvesh` over `$PVE`.
- Keep key material in the files above. Read neither `~/.ssh/pve_ssh` nor `PVE_SSH_PRIVATE_B64` into a command, a log, or a reply. Generate no replacement key.

PowerShell remoting over SSH uses the same jump. Add this to `~/.ssh/config` (the guest `Host` block only; the system file already covers the tunnel host):

```
Host 10.115.64.* 10.20.0.*
  IdentityFile ~/.ssh/pve_ssh
  IdentitiesOnly yes
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  ProxyCommand ssh -o StrictHostKeyChecking=accept-new -W %h:%p eric-pve-ssh.eag.tools
```

```powershell
Enter-PSSession -HostName <ip> -UserName Administrator -KeyFilePath "$HOME/.ssh/pve_ssh"
```

## Hard rules

- Never start, modify, clone over, or delete VM **100** (`ubuntu-2404`) or VM **101** (`ws2025-template`). Both are templates.
- Do not change host configuration (storage, network, SDN, packages) unless the user explicitly asks.
- Do not re-run sysprep, `windows-template/Copy-SysprepFiles.ps1`, or `ubuntu-template/cloud-init-prepare.sh` on a clone. Those scripts prepare the templates and stay on the workstation. They are not in this environment.
- Tag every VM you create with `ai-test`.
- When testing is finished, ask the user whether to destroy the VMs you created. Never destroy a VM without the user's answer.
- Open no direct SSH session to `10.115.64.44`, `10.115.64.x`, or `10.20.0.x`. Those networks are not routed to the cloud agent. Host commands go through `eric-pve-ssh.eag.tools`. Guest commands use the `ProxyCommand` jump above.

## Host facts

The host is a Hyper-V virtual machine. Nested VT-x is exposed. On the LAN its address is `10.115.64.44`.

| Item | Value |
|---|---|
| Hardware | Intel Xeon Gold 6234 (8 cores / 16 threads), 94 GB RAM. Budget VMs accordingly. |
| `vmbr0` | LAN bridge, DHCP. Host address `10.115.64.44/24`, gateway and DNS `10.115.64.1`, search domain `eag.lan`. Both templates' NICs are on this bridge. Guest addresses are `10.115.64.x`. From this environment, SSH to them jumps through `eric-pve-ssh.eag.tools`. |
| `nat0` | SDN VNet (simple zone `nat`), `10.20.0.0/24`. Gateway `10.20.0.1`. DHCP `10.20.0.100`–`10.20.0.200` (12 h) from dnsmasq on the host, DNS `10.20.0.1`. SNAT out `vmbr0` as `10.115.64.44`. The Proxmox host can reach this subnet. From this environment, SSH to `10.20.0.x` uses the same jump. |
| `local-data` | LVM-thin on VG `vmdata`, ~671 GB. VM disks, EFI, TPM, and cloud-init drives. |
| `local-lvm` | LVM-thin on the small PVE system VG, ~23 GB, empty. Leave it unused. |
| `local` | Directory on the root filesystem. Backups, container templates, imports. |
| `isos` | Directory `/mnt/isos`. Install ISOs only. |

## Which template

| Need | Clone |
|---|---|
| Windows, RDP, or Windows PowerShell | `101` `ws2025-template` |
| Linux, Docker, `pwsh`, or Nushell | `100` `ubuntu-2404` |

If the user does not say which OS, ask before cloning.

Windows clones enable RDP inside the guest. The Cloudflare tunnel carries SSH. Use SSH or PowerShell remoting to reach the guest.

## Template: `ws2025-template` (VMID 101)

Windows Server 2025 Datacenter Evaluation. 4 vCPU (`host`), 4 GB RAM, UEFI + Secure Boot + TPM 2.0, 64 GB VirtIO SCSI disk on `local-data`, VirtIO NIC on `vmbr0` (4 queues), VirtIO drivers and QEMU guest agent installed.

On first boot, every clone automatically:

- Gets a random computer name, Pacific time zone, en-US locale, and Administrator password `Change_me`.
- Auto-logs on once and runs `Initialize-WindowsServer.ps1`, which installs the latest PowerShell 7 from GitHub, enables OpenSSH server with a `powershell` subsystem (PS remoting over SSH), adds the SSH key to `administrators_authorized_keys`, enables RDP, sets Public networks to Private, and removes the Ctrl+Alt+Del requirement.
- Removes the auto-logon password and the answer file, writes `C:\ProgramData\TemplateSetup\setup-complete`, then restarts.

First boot takes several minutes and needs internet (PowerShell 7 download). Log: `C:\ProgramData\TemplateSetup\Initialize-WindowsServer.log`.

## Template: `ubuntu-2404` (VMID 100)

Ubuntu 24.04.5 LTS. 2 vCPU (`x86-64-v2-AES`), 4 GB RAM, UEFI (Secure Boot keys not enrolled), no TPM, 32 GB VirtIO SCSI disk on `local-data`, VirtIO NIC on `vmbr0`. QEMU guest agent, OpenSSH, PowerShell 7 (`pwsh`), Nushell (`nu` on `/usr/local/bin`), and Docker are installed. Podman is not installed. Time zone is UTC.

Cloud-init has been cleaned and limited to the NoCloud and ConfigDrive datasources. The disk has no netplan, so a clone has no network until Proxmox cloud-init runs. The account `erolleman` is already in `sudo` and already has the project public key. SSH is socket-activated. The guest agent starts from udev when the virtio port appears.

Proxmox's generated user-data also creates a locked `ubuntu` account with no SSH key. Log in as `erolleman`.

## Workflow: create a test VM

```
- [ ] 1. Confirm the source VM is a template
- [ ] 2. Clone, configure, tag, start
- [ ] 3. Get the IP from the guest agent
- [ ] 4. Wait until first boot has finished
- [ ] 5. Run the test
- [ ] 6. Ask the user whether to destroy the VM
```

**1. Confirm the template.** If this prints nothing, stop and tell the user. Do not clone a VM that is not a template.

```bash
$PVE 'qm config 101 | grep ^template'   # Windows
$PVE 'qm config 100 | grep ^template'   # Ubuntu
```

**2. Clone, configure, tag, start.** Linked clones are the default. Add `--full --storage local-data` only when the VM must outlive the template.

Clones inherit 4 vCPU / 4 GB (Windows) or 2 vCPU / 4 GB (Ubuntu). When the test needs more, `qm set` the clone before starting it. Leave the templates unchanged.

Windows:

```bash
ID=$($PVE 'pvesh get /cluster/nextid')
$PVE "qm clone 101 $ID --name test-<purpose> && qm set $ID --tags ai-test && qm start $ID"
```

Ubuntu: attach the cloud-init drive before the first start. `ide2` on the template is an empty CD; replace it. `--ciupgrade 0` skips the default full package upgrade. Without `--ipconfig0 ip=dhcp` the clone never gets an address.

```bash
ID=$($PVE 'pvesh get /cluster/nextid')
$PVE "qm clone 100 $ID --name test-<purpose> && qm set $ID --tags ai-test --ide2 local-data:cloudinit --citype nocloud --ipconfig0 ip=dhcp --ciupgrade 0 --boot order=scsi0 && qm start $ID"
```

Isolated network, either OS: before start, put the NIC on `nat0`. Keep this only when the user wants the VM off the LAN. The guest address will be `10.20.0.x`. SSH from this environment still uses the jump in Access; the Proxmox host is the one that can reach `nat0`.

```bash
$PVE "qm set $ID --net0 virtio,bridge=nat0"
# Ubuntu only. Windows DHCPs on its own. Point Ubuntu DNS at dnsmasq on the NAT gateway.
$PVE "qm set $ID --ipconfig0 ip=dhcp --nameserver 10.20.0.1"
```

**3. Get the IP.** The guest agent answers once the OS is up. Retry every ~15 s. Use the `10.115.64.x` address for a `vmbr0` NIC, or the `10.20.0.x` address for `nat0`.

```bash
$PVE "qm guest cmd $ID network-get-interfaces" \
  | jq -r '.[]."ip-addresses"[]? | select(."ip-address-type"=="ipv4") | ."ip-address"' \
  | grep -vE '^(127\.|169\.254\.)'
```

**4. Wait for first boot.** Clones get new host keys and addresses get reused, so guest SSH skips host key checking. The jump host key is stored with `accept-new`.

Windows: SSH comes up during setup, and the script restarts the VM when it finishes. The VM is ready once `setup-complete` exists and Windows has booted again since it was written.

```bash
VMSSH="ssh -i $KEY -o IdentitiesOnly=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=5 -o ProxyCommand=\"$JUMP\" Administrator@$IP"
# -EncodedCommand (base64 UTF-16LE) avoids nested bash/cmd/PowerShell quoting.
PS_CHECK=$(cat <<'EOF'
$m = Get-Item C:\ProgramData\TemplateSetup\setup-complete -ErrorAction SilentlyContinue
if ($m -and (Get-CimInstance Win32_OperatingSystem).LastBootUpTime -gt $m.LastWriteTime) { 'READY ' + (Get-Content $m.FullName) } else { 'WAITING' }
EOF
)
CHECK="powershell -NoProfile -EncodedCommand $(printf %s "$PS_CHECK" | iconv -t UTF-16LE | base64 -w0)"
until STATUS=$($VMSSH "$CHECK" 2>/dev/null | tr -d '\r'); [[ $STATUS == READY* ]]; do sleep 15; done
echo "$STATUS"
```

`READY succeeded` means setup finished and the VM has restarted. If it prints `READY failed`, read the log and report it:

```bash
$VMSSH 'powershell -NoProfile -Command "Get-Content C:\ProgramData\TemplateSetup\Initialize-WindowsServer.log -Tail 30"'
```

Ubuntu: wait until cloud-init reports `status: done`. `status: error` means stop and read `/var/log/cloud-init-output.log`.

```bash
VMSSH="ssh -i $KEY -o IdentitiesOnly=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=5 -o ProxyCommand=\"$JUMP\" erolleman@$IP"
until STATUS=$($VMSSH 'cloud-init status 2>/dev/null' | tr -d '\r'); [[ $STATUS == status:\ done || $STATUS == status:\ error ]]; do sleep 10; done
echo "$STATUS"
```

**5. Run the test.**

Windows: the default SSH shell is `cmd.exe`.

```bash
$VMSSH 'pwsh -NoProfile -Command "Get-ComputerInfo | Select-Object OsName, CsName"'
```

PowerShell remoting over SSH (from `pwsh` on the cloud agent) uses the `~/.ssh/config` guest block in Access:

```powershell
Enter-PSSession -HostName <ip> -UserName Administrator -KeyFilePath "$HOME/.ssh/pve_ssh"
```

Ubuntu:

```bash
$VMSSH 'pwsh -NoLogo -Command "$PSVersionTable.PSVersion.ToString()"'
$VMSSH 'sudo docker info --format "{{.ServerVersion}}"'
```

**6. Ask before cleaning up.** Report the test results, list the VMs you created (VMID, name, IP), and ask the user whether to destroy them. Use the AskQuestion tool when available, with options "Destroy", "Keep running", and "Shut down but keep" (plus per-VM choices if you created several).

If the user chooses to destroy:

```bash
$PVE "qm stop $ID && qm destroy $ID --purge"
```

If the user chooses to keep but shut down:

```bash
$PVE "qm shutdown $ID --timeout 120 || qm stop $ID"
```

When VMs are kept, tell the user their VMIDs so they can be found later (they keep the `ai-test` tag).

## Snapshots

Both templates' disks are on LVM-thin, so clones support snapshots:

```bash
$PVE "qm snapshot $ID before-test"
$PVE "qm rollback $ID before-test"
$PVE "qm delsnapshot $ID before-test"
```

## Cleanup and inventory

```bash
$PVE 'pvesh get /cluster/resources --type vm --output-format json' \
  | jq -r '.[] | select((.tags // "") | split(";") | index("ai-test")) | "\(.vmid)\t\(.name)\t\(.status)"'
```

Only offer to destroy `ai-test` VMs you created in this session, unless the user asks for a broader cleanup. Always get the user's confirmation first.

## Gotchas

- Windows is an evaluation edition with a 180-day license. Use it for short-lived VMs.
- An Ubuntu clone that boots once without the cloud-init drive comes up with no network, and cloud-init will not retry cleanly. Destroy it and clone again.
- Do not run `sysprep` on Windows clones, and do not convert clones of either template into templates.
- The host address is a DHCP lease. `nat0` SNAT is pinned to `10.115.64.44`. If that address changes, say so; do not edit SDN unless the user asks.
- `cloudflared` is invoked as `/usr/local/bin/cloudflared` by the system SSH config. A missing key file means the start script did not run; leave the secrets untouched and report that.
- Guest host keys change every clone. `UserKnownHostsFile=/dev/null` applies to the guest session. The tunnel host key stays in `~/.ssh/known_hosts`.
