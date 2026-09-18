# Shadowrocket configuration

`Default.conf` follows the routing policy of [surge/Default.conf](https://github.com/rsivanov-git/surge/blob/main/Default.conf), adapted for Shadowrocket.

Initial configuration adaptation baseline: Surge commit `4cecfa98db1ca7dbe6395774b424cf7ad6ab721f` (2026-09-18 comparison).

## Routing

- `guzzoni.apple.com` uses PROXY before the existing SYSTEM rule.
- `_test.local` is rejected using pre-matching.
- Explicit local-domain and IPv4/IPv6 rules provide DIRECT access to local networks, loopback, link-local and multicast addresses. They are a portable LAN approximation, not a claim that the two apps' built-in LAN lists are identical.
- Connectivity checks, NTP, the shared Surge `Direct.txt` rule set, Russian domains and already-resolved Russian IPs use DIRECT.
- All remaining traffic uses Shadowrocket's built-in PROXY policy (the selected node).
- The blanket Apple DIRECT rule and obsolete DNS-domain exceptions have been removed.

`Default.conf` loads [Direct.txt from the Surge repository](https://github.com/rsivanov-git/surge/blob/main/Direct.txt) directly:

```ini
RULE-SET,https://raw.githubusercontent.com/rsivanov-git/surge/main/Direct.txt,DIRECT
```

The Surge repository is the single source for this list; there are no local `Direct.list` or `Proxied.list` copies. Changes to `surge/main/Direct.txt` take effect after Shadowrocket refreshes the remote rule set, without copying files or editing this profile. This list follows Surge's `main` branch rather than the initial adaptation commit above. As in the Surge baseline, `Proxied.txt` is not loaded by `Default.conf`.

## General settings and intentional differences

| Surge setting | Shadowrocket adaptation |
| --- | --- |
| `encrypted-dns-server = https://dns.nextdns.io` | `dns-server = https://dns.nextdns.io` |
| `dns-server = system` alongside encrypted DNS | Do not add `system` to the DoH resolver list: Surge uses plain DNS for bootstrap/connectivity, not ordinary-domain fallback. Shadowrocket's fallback points to the same DoH endpoint. This prevents configured plaintext fallback but adds no resolver redundancy. |
| `use-local-host-item-for-proxy = true` | Same setting; preserves the `ntc.party` mapping for proxied connections. |
| UDP unsupported policy | Same `REJECT` behavior. |
| No active `skip-proxy` / `tun-excluded-routes` overrides | Removed the old explicit broad bypass lists, including `100.64.0.0/10`; local traffic is handled by rules. Application/OS defaults still apply. |
| No global real-IP override | Removed `always-real-ip = *`; Shadowrocket uses its normal DNS behavior. |
| Managed-config header | Shadowrocket's existing `update-url`. |
| `PROXY = select, DIRECT, REJECT` placeholder | Keep Shadowrocket's built-in PROXY; copying that group would omit actual proxy nodes. |
| `loglevel`, `proxy-test-url`, `internet-test-url` | Not copied because equivalent General keys were not verified for Shadowrocket. Set its connectivity-test URL in the app to `http://www.gstatic.com/generate_204` if desired. |

`private-ip-answer = true` and `dns-direct-system = false` retain Shadowrocket-specific DNS handling. The explicit `hijack-dns` override was removed to match the Surge baseline. Direct DNS failures no longer trigger proxy fallback. The obsolete `bypass-system` flag and unrelated rewrite/ICMP flags were removed. The pre-existing `RULE-SET,SYSTEM,DIRECT` is retained; its actual contents are app-specific.

No private node credentials, Tailscale configuration, or alternative Surge profiles (`DefaultTailnet`, `DefaultChinese`, `DefaultWhiteLists`) are imported.

## Validation

Static validation should cover rule structure, CIDR validity, ordering and remote rule-set references; local list-parity checks are no longer needed. This repository does not contain a Shadowrocket runtime, so successful import and actual routing must be checked in the app before rollout.

1. Import `Default.conf` and refresh its remote rule sets. Confirm that `Direct.txt` loads from `rsivanov-git/surge/main`. When testing a Shadowrocket branch, keep this shared Surge URL unchanged unless intentionally testing a separate Surge rule-set revision.
2. Select a working proxy node and use configuration-based routing.
3. Check Siri (`guzzoni.apple.com`): PROXY; `habr.com`: DIRECT; `_test.local`: REJECT; a foreign site outside DIRECT lists: PROXY.
4. Check local router/NAS access, connectivity checks and NTP. Confirm no remote rule-set download or parse errors, including SYSTEM.
5. Check DNS and `ntc.party` in the connection log. DoH failure intentionally does not fall back to plaintext system DNS. Check any separately configured Tailscale policy after removal of the old CGNAT bypass.

References: [Surge encrypted DNS](https://manual.nssurge.com/dns/encrypted-dns.html), [Shadowrocket community manual maintained from official-group material](https://github.com/LOWERTOP/Shadowrocket).
