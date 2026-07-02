# Ansible Role — Firewall (UFW)

Declarative UFW firewall management. Define rules in `host_vars` or `group_vars` — the role ensures the firewall matches your declared state.

## Quick Start

```yaml
# host_vars/hsrv01.yaml
firewall_rules:
  - { rule: allow, port: 22, proto: tcp, comment: "SSH" }
  - { rule: allow, port: 80, proto: tcp, comment: "HTTP" }
  - { rule: allow, port: 443, proto: tcp, comment: "HTTPS" }

firewall_rate_limit_rules:
  - { port: 22, proto: tcp, comment: "SSH brute force" }
```

```bash
ansible-playbook site.yaml -i inventory.ini
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `firewall_state` | `present` | `present` or `absent` |
| `firewall_enabled` | `true` | Enable UFW after configuring |
| `firewall_default_incoming` | `deny` | Default incoming policy |
| `firewall_default_outgoing` | `allow` | Default outgoing policy |
| `firewall_default_routed` | `deny` | Default forwarding policy |
| `firewall_logging` | `on` | `on`, `off`, `low`, `medium`, `high`, `full` |
| `firewall_rules` | `[{SSH on 22}]` | Port/service rules (supports `interface_in`/`interface_out` to scope rules to a specific NIC, e.g. `eth0`, `tailscale0`) |
| `firewall_rate_limit_rules` | `[]` | Rate-limited rules |
| `firewall_route_rules` | `[]` | Forwarding/route rules |
| `firewall_raw_rules` | `[]` | Raw NAT lines for before.rules |

## Rule Formats

### Port Rules

```yaml
firewall_rules:
  - { rule: allow, port: 22, proto: tcp, comment: "SSH" }
  - { rule: allow, port: 80, proto: tcp, comment: "HTTP", interface_in: eth0 }
  - { rule: allow, port: 443, proto: tcp, comment: "HTTPS", interface_in: tailscale0 }
  - { rule: allow, port: 51820, proto: udp, comment: "WireGuard" }
  - { rule: allow, port: 22, proto: tcp, from: "10.0.0.0/8", comment: "SSH from internal" }
```

The `interface_in` and `interface_out` keys may be added to any rule item; they are passed directly to the `ufw` module and restrict the rule to traffic arriving on or leaving via the named interface.

### Rate Limiting

Automatically denies connections from IPs that exceed 6 connections in 30 seconds. Interface selectors are also supported:

```yaml
firewall_rate_limit_rules:
  - { port: 22, proto: tcp, comment: "SSH brute force", interface_in: eth0 }
  - { port: 2222, proto: tcp, comment: "Alt SSH brute force", interface_in: tailscale0 }
```

### Forwarding Rules (Containers, VPN)

```yaml
firewall_route_rules:
  - { rule: allow, interface_in: br0, interface_out: enp1s0, comment: "Containers outbound" }
  - { rule: allow, interface_in: enp1s0, interface_out: br0, comment: "Containers return" }
```

### Raw NAT Rules

Inserted into `/etc/ufw/before.rules` for masquerade/DNAT. The role now also removes the block when `firewall_raw_rules` is emptied, keeping the file tidy.

```yaml
firewall_raw_rules:
  - "-A POSTROUTING -s 192.168.100.0/24 -o enp1s0 -j MASQUERADE"
```

## Example: Server with Containers

```yaml
# host_vars/hsrv01.yaml
firewall_default_incoming: deny
firewall_default_routed: allow

firewall_rules:
  - { rule: allow, port: 22, proto: tcp, comment: "SSH" }
  - { rule: allow, port: 80, proto: tcp, comment: "HTTP" }
  - { rule: allow, port: 443, proto: tcp, comment: "HTTPS" }

firewall_rate_limit_rules:
  - { port: 22, proto: tcp, comment: "SSH brute force" }

firewall_route_rules:
  - { rule: allow, interface_in: br0, interface_out: enp1s0, comment: "nspawn outbound" }
  - { rule: allow, interface_in: enp1s0, interface_out: br0, comment: "nspawn return" }

firewall_raw_rules:
  - "-A POSTROUTING -s 192.168.100.0/24 -o enp1s0 -j MASQUERADE"
```

## Tags

| Tag | Controls |
|---|---|
| `firewall` | Entire role |
| `install` | Package installation |
| `configure` | Rules and policies |
| `uninstall` | Removal |

## File Structure

```
firewall/
├── site.yaml
└── roles/
    └── firewall/
        ├── defaults/main.yaml
        ├── handlers/main.yaml
        └── tasks/
            ├── main.yaml
            ├── install.yaml
            ├── configure.yaml
            └── uninstall.yaml
```

## Requirements

- Ansible ≥ 2.9
- `community.general` collection (for `ufw` module)
- Target hosts running Debian/Ubuntu
