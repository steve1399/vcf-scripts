# vcf-creds

Display component names, FQDNs, usernames, passwords, and credential types from VCF installer credential JSON files in an aligned terminal table.

With no arguments, the command scans the **current working directory**. No site name or credential file path is hard-coded.

## Requirements

- Linux with Bash and Python 3.
- Read access to the credential JSON files.
- No `jq`, Python packages, or root privileges are needed to run the command.

On Ubuntu/Debian, install Python if needed:

```bash
sudo apt install python3
```

## Installation

From the directory containing the downloaded `vcf-creds` script:

```bash
mkdir -p "$HOME/.local/bin"
install -m 755 ./vcf-creds "$HOME/.local/bin/vcf-creds"

# Add the directory to PATH for future Bash sessions, once.
if ! grep -qF 'export PATH="$HOME/.local/bin:$PATH"' "$HOME/.bashrc"; then
    printf '\nexport PATH="$HOME/.local/bin:$PATH"\n' >> "$HOME/.bashrc"
fi

# Activate it in this terminal.
export PATH="$HOME/.local/bin:$PATH"
command -v vcf-creds
```

To update the installed command, repeat the `install -m 755` command with the new script. The PATH setup only needs to be done once.

## Usage

```bash
vcf-creds [credentials.json]
```

Scan the current directory:

```bash
cd ~/site3-nfs1/scripts
vcf-creds
```

Read one specific file from anywhere:

```bash
vcf-creds ~/site3-nfs1/scripts/site3-creds.json
vcf-creds "/path/with spaces/site5-creds.json"
```

Show help:

```bash
vcf-creds --help
```

## File discovery and input format

Automatic scanning checks files with a `.json` extension, case-insensitively, in the current directory. It does not search subdirectories. It inspects the first 4,096 characters and selects files beginning with a top-level `components` array:

```json
{"components":[{"componentName":"VCF Operations"}]}
```

Whitespace, pretty printing, and a UTF-8 byte-order mark are supported. For automatic discovery, `components` must be the first top-level property. An explicitly supplied file bypasses that opening-text filter, but must still contain a top-level `components` array.

Example credential file using a placeholder password:

```json
{
  "components": [
    {
      "componentName": "VCF Operations",
      "FQDN": "site3-ops.vcf-labs.com",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "ExamplePassword!"
        }
      ]
    }
  ]
}
```

## Example output

```text
Credentials: /home/steve/site3-nfs1/scripts/site3-creds.json

VCF Operations
FQDN: site3-ops.vcf-labs.com
  USERNAME  PASSWORD          CREDENTIAL TYPE
  --------  ----------------  ----------------
  root      ExamplePassword!  Root credentials
```

Each selected file gets its own output section. Missing credentials are shown as `No credentials found.` Password characters such as `$`, `!`, `*`, and `#` are displayed literally, without shell evaluation. Nonprinting characters are escaped for terminal display.

The command displays plaintext passwords by design. Examples in this README use placeholder passwords; the script does not embed credentials or modify the input files.

## Exit codes

| Code | Meaning |
| --- | --- |
| `0` | Completed successfully, help displayed, or no matching files found. |
| `1` | A file could not be read or parsed, its structure was invalid, or Python 3 was missing. Other files are still processed after file errors. |
| `2` | Too many positional arguments supplied. |

## Troubleshooting

| Problem | What to check |
| --- | --- |
| `command not found` | Run `export PATH="$HOME/.local/bin:$PATH"` and verify `command -v vcf-creds`. |
| No matching credential files | Run `pwd` and check the JSON filename and opening structure. Try passing the file explicitly. |
| Error reading a file | Check read permissions, valid JSON syntax, and that `components` and `credentials` are arrays. |
| Table wraps | Widen the terminal; long usernames and passwords determine column widths. |

## Related command

See [vcf-dns-check](README-vcf-dns-check.md) to resolve the same files' FQDNs and compare the host's configured DNS servers.

# vcf-dns-check

Resolve FQDNs from VCF credential JSON files, compare answers from the Linux host's configured DNS servers, and compare authoritative zone SOA serials.

The default output is a compact FQDN/IP/result table and zone summary. Use `-v` for DNS configuration, per-server answers, and SOA details. All checks run in either mode.

## Requirements

- Linux with Bash and Python 3.
- `dig` for DNS queries.
- Read access to credential JSON files and network access to the discovered DNS servers.
- `resolvectl` or `nmcli` is preferred for discovering upstream servers. `/etc/resolv.conf` is a fallback.

On Ubuntu/Debian:

```bash
sudo apt install python3 dnsutils
```

Running the checker does not require root. It reads configuration and sends DNS queries; it does not change DNS records, host networking, or credential files. Passwords are not displayed.

## Installation

From the directory containing the downloaded script:

```bash
mkdir -p "$HOME/.local/bin"
install -m 755 ./vcf-dns-check "$HOME/.local/bin/vcf-dns-check"

# Add the directory to PATH for future Bash sessions, once.
if ! grep -qF 'export PATH="$HOME/.local/bin:$PATH"' "$HOME/.bashrc"; then
    printf '\nexport PATH="$HOME/.local/bin:$PATH"\n' >> "$HOME/.bashrc"
fi

export PATH="$HOME/.local/bin:$PATH"
command -v vcf-dns-check
```

To update, repeat the `install -m 755` command with the new script. If you installed `vcf-creds` already, the shared PATH setup may already be complete.

## Usage

```bash
vcf-dns-check [-h] [-v] [--interface INTERFACE] [file]
```

| Argument | Purpose |
| --- | --- |
| No file argument | Scan credential JSON files in the current directory. |
| `file` | Check one explicitly supplied credential file. |
| `-v`, `--verbose` | Show configuration, per-server answers, and authoritative SOA details. |
| `--interface ens160` | Limit DNS-server and domain discovery to that interface. The host-resolver query still follows the host's normal resolver configuration. |
| `-h`, `--help` | Show command help. |

Examples:

```bash
cd ~/site3-nfs1/scripts
vcf-dns-check
vcf-dns-check -v
vcf-dns-check ~/site3-nfs1/scripts/site3-creds.json
vcf-dns-check -v --interface ens160
```

## Input discovery

The command scans `.json` files, case-insensitively, directly in the current working directory. It does not recurse. It reads the opening 4,096 characters to select files beginning with a top-level `components` array, allowing whitespace, pretty printing, and a UTF-8 byte-order mark.

Automatic discovery requires `components` to be the first top-level property. Passing a file explicitly bypasses the opening-text filter. Selected files must contain valid JSON with a `components` array and component objects containing `FQDN` values.

```json
{
  "components": [
    {"FQDN": "site3-ops.vcf-labs.com"},
    {"FQDN": "site3-vcsa.vcf-labs.com"}
  ]
}
```

The full installer credential schema also works. FQDNs are lowercased, trailing dots removed, deduplicated across files, validated, and sorted. Empty or missing FQDNs are skipped.

## How the checks work

1. Discover configured DNS servers and domains using `resolvectl`, then `nmcli`, then `/etc/resolv.conf` as needed. No server IPs or domains are hard-coded. Local loopback stub addresses are excluded from direct server comparisons.
2. Query each FQDN's A (IPv4) and AAAA (IPv6) records through `dig` using the host's default resolver configuration.
3. Query every discovered DNS server directly and compare response status and answer content, ignoring TTL and answer order. The host-resolver answer is also compared with the first discovered server.
4. Discover enclosing zones and query their SOA records directly with recursion disabled. Require authoritative replies from at least two configured servers, with matching serials.

Server labels `DNS1`, `DNS2`, and so on follow discovery order. They do not establish which server is the zone's primary or secondary. On hosts with multiple links or split DNS, use `--interface` to avoid comparing unrelated DNS server groups.

The host-resolver check uses DNS through `dig`; it does not test `/etc/hosts` or the complete NSS lookup path used by `getent`.

## Example output

```text
FQDN                        RESULT       IP ADDRESS(ES)
------------------------------------------------------------------------------
site3-ops.vcf-labs.com      MATCH        192.168.30.4
site3-vcsa.vcf-labs.com     MATCH        192.168.30.2

Zone sync: vcf-labs.com MATCH (serial 323)
RESULT: PASS — listed FQDN answers and authoritative SOA serials agree.
```

Verbose output adds configuration and per-server answers:

```text
  DNS1 192.168.1.2: Resolves to 192.168.30.4
  DNS2 192.168.1.3: Resolves to 192.168.30.4
```

Query statuses appear when a query fails, labeled by record type:

```text
    Query status: A=NOERROR, AAAA=SERVFAIL
```

`NOERROR` means the query completed successfully, even if that record type has no answers. A successful empty AAAA answer is normal for an IPv4-only host. At least one IP address must resolve, and both query types must complete successfully for a resolution check to pass.

## Results and exit codes

| Result | Meaning |
| --- | --- |
| FQDN `MATCH` | Host and configured servers return matching successful answers with at least one address. |
| FQDN `MISMATCH` | Answer content differs between servers or from the host resolver. |
| FQDN `FAIL` | Resolution failed, a query errored, or an endpoint returned no IP addresses. |
| FQDN `ONE SERVER` | Only one upstream server was discovered; comparison is incomplete. |
| Zone `MATCH` | All discovered servers returned authoritative SOA answers with the same serial. |
| Zone `MISMATCH` | Authoritative SOA serials differ. |
| Zone `UNCONFIRMED` | Fewer than two usable authoritative SOA replies are available. |

| Exit code | Meaning |
| --- | --- |
| `0` | PASS, or help displayed. |
| `1` | Resolution, comparison, file, discovery, or missing-`dig` error. |
| `2` | INCOMPLETE, invalid command-line arguments, or missing Python 3. |

Errors take precedence over incomplete checks. Inspect the printed message to distinguish setup errors from DNS results.

Matching listed records and authoritative serials is a consistency check, not a complete zone transfer audit. Forwarding or caching servers may return matching IPs without authoritative SOA data; the checker reports that as incomplete rather than confirming replication. It does not test reverse DNS, every zone record, service availability, or DNSSEC validation.

## Search-domain configuration example

The checker discovers search domains but does not change them. For the NetworkManager profile `Wired connection 1` on `ens160`, the following manually places `vcf-labs.com` before `stevelab.local`:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.dns-search "vcf-labs.com,stevelab.local"
sudo nmcli device reapply ens160
resolvectl domain ens160
```

Use your own active profile and interface names. This sets that profile's manual IPv4 search list. Verify the effective list after applying it; automatically supplied domains may also be present. Fully qualified record checks do not depend on search-domain order.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| `command not found` | Verify `~/.local/bin` is in PATH and run `command -v vcf-dns-check`. |
| No credential FQDNs found | Check `pwd`, the JSON opening structure, and the exact `FQDN` property spelling. Try an explicit file. |
| No upstream servers discovered | Inspect `resolvectl status` and `nmcli device show`; a loopback stub alone is insufficient. |
| `FAIL` or `MISMATCH` | Run `vcf-dns-check -v` to identify the differing server or query type. |
| Zone sync unconfirmed | Check whether configured servers host the zone authoritatively or merely forward/cache it. |
| Host differs from direct servers | Inspect host resolver caching, split DNS, and interface routing; use verbose output. |
| Slow checks | Unreachable servers incur timeouts. Each `dig` query uses a two-second DNS timeout and one try, with a 15-second subprocess limit. |

## Related command

See [vcf-creds](README-vcf-creds.md) to display the credential files' usernames and passwords.

