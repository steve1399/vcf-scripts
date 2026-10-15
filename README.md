# VCF Scripts

Three Linux command-line tools for VCF installer credentials, DNS consistency checks, and SecureCRT password exports. All settings and inputs come from the current directory, explicit arguments, or the host configuration; no deployment-specific addresses or accounts are embedded in the scripts.

| Command | Purpose | With no arguments |
| --- | --- | --- |
| `vcf-creds` | Display credentials grouped by component. | Scan credential JSON files in the current directory. |
| `vcf-dns-check` | Resolve FQDNs and compare DNS answers and authoritative SOA serials. | Scan credential JSON files in the current directory. |
| `vcf-password-tool` | List combined JSON/API credentials or export a SecureCRT CSV. | Print help and exit successfully. |

## Contents

- [Common setup](#common-setup)
- [Common JSON discovery](#common-json-discovery)
- [vcf-creds](#vcf-creds)
- [vcf-dns-check](#vcf-dns-check)
- [vcf-password-tool](#vcf-password-tool)
- [Exit codes](#exit-codes)
- [Troubleshooting](#troubleshooting)

## Common setup

### Requirements

- Linux, Bash, and Python 3.8 or later.
- Read access to the credential files.
- `dig` for `vcf-dns-check`; DNS server reachability is also required.
- Optional API use requires HTTPS access to SDDC Manager and an authorized account.

All tools use the Python standard library. No `requests`, `jq`, or pip packages are required. Root privileges are not required to run them.

On Ubuntu/Debian:

```bash
sudo apt install python3 dnsutils
```

### Install or update all commands

Run from the repository directory containing the three scripts:

```bash
mkdir -p "$HOME/.local/bin"
install -m 755 vcf-creds vcf-dns-check vcf-password-tool "$HOME/.local/bin/"

# Add the directory to PATH for future Bash sessions, once.
if ! grep -qF 'export PATH="$HOME/.local/bin:$PATH"' "$HOME/.bashrc"; then
    printf '\nexport PATH="$HOME/.local/bin:$PATH"\n' >> "$HOME/.bashrc"
fi

# Activate PATH in this terminal.
export PATH="$HOME/.local/bin:$PATH"

command -v vcf-creds
command -v vcf-dns-check
command -v vcf-password-tool
```

Repeat the `install` command after updating the repository. The PATH setup only needs to be done once. Installed commands run from any directory without a `python3` prefix; their shebang starts the interpreter automatically.

### Help and quick start

```bash
vcf-creds -h
vcf-dns-check -h
vcf-password-tool -h
vcf-password-tool
```

From the directory containing credential files:

```bash
vcf-creds
vcf-dns-check
vcf-dns-check -v
vcf-password-tool list
vcf-password-tool export
```

Only `vcf-password-tool export` writes a deliverable: one CSV. The other actions read inputs and display results. Credential displays and CSV exports contain plaintext passwords by design. Examples below use placeholder passwords and documentation addresses.

## Common JSON discovery

All three tools scan `.json` files, case-insensitively, in the **current working directory**, without recursion. They inspect the first 4,096 characters and select files beginning with a top-level `components` array. Whitespace, pretty printing, and a UTF-8 byte-order mark are supported.

For automatic discovery, `components` must be the first top-level property. Unrelated JSON files are skipped. Explicitly selected JSON files bypass the opening-text filter but must still contain a top-level `components` array.

A shared example input:

```json
{
  "components": [
    {
      "componentName": "SDDC Manager",
      "FQDN": "manager.example.com",
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

| Tool | Explicit JSON selection | Data used |
| --- | --- | --- |
| `vcf-creds` | One filename. | Components and their credential arrays. |
| `vcf-dns-check` | One filename. | Component FQDNs; passwords are not displayed. |
| `vcf-password-tool` | Any number of filenames. | Password-bearing accounts; optionally combined with API credentials. |

Quote filenames containing spaces. Nonprinting characters are escaped on screen; passwords are not evaluated as shell commands. The password tool's CSV preserves original password content.

## vcf-creds

### Usage

```bash
vcf-creds [credentials.json]
```

Scan the current directory:

```bash
vcf-creds
```

Select specific files by filename:

```bash
vcf-creds credentials.json
vcf-creds "additional credentials.json"
```

Show help:

```bash
vcf-creds --help
```

### Example output

```text
Credentials: credentials.json

VCF Operations
FQDN: operations.example.com
  USERNAME  PASSWORD          CREDENTIAL TYPE
  --------  ----------------  ----------------
  root      ExamplePassword!  Root credentials
```

Each selected file gets its own output section. Missing credentials are shown as `No credentials found.` Password characters such as `$`, `!`, `*`, and `#` are displayed literally, without shell evaluation. Nonprinting characters are escaped for terminal display.

The command displays plaintext passwords by design. Examples in this README use placeholder passwords; the script does not embed credentials or modify the input files.

## vcf-dns-check

Default output is compact. `-v` shows configuration, per-server answers, and SOA details; all checks run in both modes. FQDNs are normalized, validated, deduplicated, and sorted. Missing or empty FQDNs are skipped.

### Usage

```bash
vcf-dns-check [-h] [-v] [--interface INTERFACE] [file]
```

| Argument | Purpose |
| --- | --- |
| No file argument | Scan credential JSON files in the current directory. |
| `file` | Check one explicitly supplied credential file. |
| `-v`, `--verbose` | Show configuration, per-server answers, and authoritative SOA details. |
| `--interface eth0` | Limit DNS-server and domain discovery to that interface. The host-resolver query still follows the host's normal resolver configuration. |
| `-h`, `--help` | Show command help. |

Examples:

```bash
vcf-dns-check
vcf-dns-check -v
vcf-dns-check credentials.json
vcf-dns-check -v --interface eth0
```

### How the checks work

1. Discover configured DNS servers and domains using `resolvectl`, then `nmcli`, then `/etc/resolv.conf` as needed. No server IPs or domains are hard-coded. Local loopback stub addresses are excluded from direct server comparisons.
2. Query each FQDN's A (IPv4) and AAAA (IPv6) records through `dig` using the host's default resolver configuration.
3. Query every discovered DNS server directly and compare response status and answer content, ignoring TTL and answer order. The host-resolver answer is also compared with the first discovered server.
4. Discover enclosing zones and query their SOA records directly with recursion disabled. Require authoritative replies from at least two configured servers, with matching serials.

Server labels `DNS1`, `DNS2`, and so on follow discovery order. They do not establish which server is the zone's primary or secondary. On hosts with multiple links or split DNS, use `--interface` to avoid comparing unrelated DNS server groups.

The host-resolver check uses DNS through `dig`; it does not test `/etc/hosts` or the complete NSS lookup path used by `getent`.

### Example output

```text
FQDN                        RESULT       IP ADDRESS(ES)
------------------------------------------------------------------------------
operations.example.com      MATCH        192.0.2.10
vcenter.example.com         MATCH        192.0.2.11

Zone sync: example.com MATCH (serial 323)
RESULT: PASS — listed FQDN answers and authoritative SOA serials agree.
```

Verbose output adds configuration and per-server answers:

```text
  DNS1 192.0.2.53: Resolves to 192.0.2.10
  DNS2 192.0.2.54: Resolves to 192.0.2.10
```

Query statuses appear when a query fails, labeled by record type:

```text
    Query status: A=NOERROR, AAAA=SERVFAIL
```

`NOERROR` means the query completed successfully, even if that record type has no answers. A successful empty AAAA answer is normal for an IPv4-only host. At least one IP address must resolve, and both query types must complete successfully for a resolution check to pass.

### Results and limitations

| Result | Meaning |
| --- | --- |
| FQDN `MATCH` | Host and configured servers return matching successful answers with at least one address. |
| FQDN `MISMATCH` | Answer content differs between servers or from the host resolver. |
| FQDN `FAIL` | Resolution failed, a query errored, or an endpoint returned no IP addresses. |
| FQDN `ONE SERVER` | Only one upstream server was discovered; comparison is incomplete. |
| Zone `MATCH` | All discovered servers returned authoritative SOA answers with the same serial. |
| Zone `MISMATCH` | Authoritative SOA serials differ. |
| Zone `UNCONFIRMED` | Fewer than two usable authoritative SOA replies are available. |

Matching listed records and authoritative serials is a consistency check, not a complete zone transfer audit. Forwarding or caching servers may return matching IPs without authoritative SOA data; the checker reports that as incomplete rather than confirming replication. It does not test reverse DNS, every zone record, service availability, or DNSSEC validation.

### Host configuration

The checker reads the host's configured DNS servers and search domains automatically. It does not change them. Fully qualified record checks do not depend on search-domain order.

## vcf-password-tool

Only `list` and `export` are supported. Both display FQDN, username, password, and source. `export` also writes one CSV. Running with no arguments or with `-h`/`--help` prints the same help and exits with code `0`. No input files are scanned and no API calls are made in those help cases.

Explicit JSON selection:

```bash
vcf-password-tool list credentials.json deferred-credentials.json
vcf-password-tool export credentials.json
```

Environment-file discovery is independent of JSON selection. Matching JSON and API sources are combined. For JSON-only use, run without matching API environment files and without `VCF_SDDC_*` settings in the process environment.

### Quick start

Run from the directory containing your credential files:

```bash
vcf-password-tool list
vcf-password-tool export
```

Export creates `./vcf-securecrt-import.csv`. Existing files are not overwritten; choose another output name for a subsequent export:

```bash
vcf-password-tool export --output securecrt-updated.csv
```

### Optional API environment files

No environment file is required for JSON-only use.

For live API retrieval, automatically discover `.env`, `.env.*`, files ending in `.env`, and `.vcf-sddc*` files in the current directory. Files are selected when they contain a supported `VCF_SDDC_*` assignment. Each matching file configures one manager. Files are parsed on every run, so edits take effect without sourcing them. If a file omits the host, the tool uses the unique component named `SDDC Manager` in the loaded JSON; multiple candidate hosts require explicit configuration.

Create, for example, `manager.env`:

```dotenv
VCF_SDDC_HOST=manager.example.com
VCF_SDDC_USER=admin@local
VCF_SDDC_PASSWORD='ExamplePassword!'
```

The host may be a hostname, IP address, or HTTPS URL with an optional port. HTTP, URL credentials, and extra URL paths are rejected.

| Variable | Purpose |
| --- | --- |
| `VCF_SDDC_HOST` | Manager endpoint. If omitted, use `--sddc` or the unique SDDC Manager component in the loaded JSON. |
| `VCF_SDDC_USER` | Login account; required with password or API-key authentication. |
| `VCF_SDDC_PASSWORD` | Login password. |
| `VCF_SDDC_API_KEY` | API key instead of a password. |
| `VCF_SDDC_TOKEN` | Existing bearer token instead of login credentials. |
| `VCF_SDDC_CA_FILE` | Optional trusted issuing CA bundle in PEM format. |
| `VCF_SDDC_VERIFY_SSL` | `true` (default) or `false` to explicitly disable certificate verification. |

Authentication preference is bearer token, then API key, then password. There is no default username and no interactive password prompt. Use an account authorized on SDDC Manager. `administrator@vsphere.local` is the vCenter SSO account and requires appropriate SDDC Manager access; `admin@local` is the SDDC Manager local administrator. Use the matching account password, not another appliance's password.

Values are parsed literally. Optional `export ` prefixes and matching outer quotes are supported. `$`, `!`, `#`, and other password characters are not expanded or executed. Comments must be on their own lines; inline comments are treated as part of the value. Do not source the file to run the tool.

Select an environment file explicitly instead of discovering it:

```bash
vcf-password-tool list --env-file manager.env
vcf-password-tool export --env-file manager.env
```

An existing file with only user/password settings can be used by supplying its manager:

```bash
vcf-password-tool list --env-file settings.env --sddc manager.example.com
```

`--sddc` supports at most one selected environment file. Environment variables already set in the calling process supply defaults; each file's settings override those defaults for that manager, and `--sddc` overrides its host. If no environment files are selected, process settings can configure a manager directly.

HTTPS certificate verification is enabled by default. To trust an issuing CA without disabling verification:

```bash
vcf-password-tool list --ca-file issuing-ca.pem
```

For a diagnostic run that bypasses certificate verification:

```bash
vcf-password-tool list --env-file manager.env --insecure
```

To persist that explicit choice in a selected environment file, set `VCF_SDDC_VERIFY_SSL=false`. The tool prints a warning whenever verification is disabled. `--ca-file` overrides `VCF_SDDC_CA_FILE`; `--insecure` overrides verification settings.

API retrieval uses `POST /v1/tokens` for login and `GET /v1/credentials?pageNumber=0&pageSize=0` for all credentials. A supplied bearer token skips login. API requests have a 60-second timeout and do not follow redirects.

### Options

```text
vcf-password-tool {list,export} [JSON ...]
                  [--env-file FILE] [--sddc HOST]
                  [--insecure] [--ca-file PEM] [--output CSV]
```

| Option | Purpose |
| --- | --- |
| `JSON ...` | Select explicit JSON files instead of automatic JSON discovery. |
| `--env-file FILE` | Explicit API environment file; default: discover `.vcf-sddc*` and `.env` files in the current directory. |
| `--sddc HOST` | Override the manager endpoint. |
| `--insecure` | Disable API HTTPS certificate verification. |
| `--ca-file PEM` | Trust the specified issuing CA certificate bundle for API HTTPS. |
| `--output CSV` | Export path; parent directory must already exist. Default: `vcf-securecrt-import.csv`. |
| `-h`, `--help` | Display help. |

### List output and duplicate accounts

```text
Passwords found: 1

FQDN                 Username  Password          Source
-------------------  --------  ----------------  ----------------
manager.example.com  root      ExamplePassword!  credentials.json
```

The `Source` column shows the JSON filename (for example, `credentials.json`) or `sddcm_api` for a live API result. When API data overrides a JSON account, the retained row shows `sddcm_api`. This column is displayed by both commands but is not added to the six-field SecureCRT CSV.

A source summary shows how many JSON files and API managers loaded successfully. If any source fails, `Status: PARTIAL` appears above the table so local JSON output cannot be mistaken for a successful API retrieval.

Rows without a host, username, or nonempty plaintext password are skipped. Entries are deduplicated by normalized host plus case-sensitive username. Hostnames are lowercased; usernames retain their case. Nonprinting characters are escaped on screen, while CSV passwords retain their original content.

Automatic JSON files are processed in filename order; explicit files in argument order. API sources are processed after JSON, so a returned live password wins over an installer password for the same account. Environment files follow filename order when discovered, or argument order when explicit. If competing sources contain different passwords, a warning identifies the account and the winning source without repeating the passwords.

### SecureCRT CSV format

The export uses this import layout: **no header**, six comma-separated fields, all fields quoted, UTF-8 encoding.

| Position | Field |
| --- | --- |
| 1 | Host/FQDN |
| 2 | Folder |
| 3 | Session name |
| 4 | Username |
| 5 | Plaintext password |
| 6 | Protocol: `SSH2` |

```csv
"manager.example.com","vcf-password-tool\example.com\SDDC Manager","manager.example.com (root)","root","ExamplePassword!","SSH2"
```

Folders follow `vcf-password-tool\<domain>\<component>`. The fixed root is `vcf-password-tool`; the domain is derived from the FQDN by removing its first label. The final folder is a readable component name. Known API resource types and installer names are normalized: `NSXT_MANAGER` and `Existing NSX Manager` become `NSX Manager`; `VCENTER`, `PSC`, and `Existing vCenter` become `vCenter`; `SDDC_MANAGER` and `BACKUP` become `SDDC Manager`; `ESXI` becomes `Hosts`. Unrecognized component names are preserved. A host without a dot uses `VCF` as its domain folder. Hosts sharing a domain are grouped together, even when queried through different environment files. The CSV Folder field contains only the three folder levels, `vcf-password-tool\<domain>\<component>`. The separate Session name field is the final item within that folder. Session names use the full host and username to avoid short-name collisions. CSV quoting preserves commas, quotes, and password metacharacters.

Use the same SecureCRT CSV import script/mapping that accepts this six-field layout. This CSV is not a native SecureCRT XML configuration file; whether its password field is saved depends on that importer. The exporter includes every password-bearing account, including accounts that may be intended for application/UI login rather than SSH.

Export prints the list first and writes exactly one CSV. It creates no report, log, XML, or timestamped directory. On Linux, the new CSV is requested with mode `0600`; filesystem permissions may behave differently on network mounts. It contains plaintext credentials.

If any source fails, successfully collected credentials are still listed, but no CSV is created. An existing CSV is left intact. If no passwords are found, no empty CSV is created.

### Diagnose HTTPS before authentication

These commands send no login credentials:

```bash
curl --noproxy '*' --connect-timeout 5 --max-time 10 -I https://manager.example.com/v1/tokens
curl --noproxy '*' --connect-timeout 5 --max-time 10 -k -I https://manager.example.com/v1/tokens
```

If verification fails but the second command reaches an HTTP response, HTTPS is reachable and certificate validation is the immediate blocker. HTTP 401, 403, or 405 still demonstrates that an HTTP endpoint responded; these HEAD requests do not test login credentials. Inspect the certificate if needed:

```bash
openssl s_client -connect manager.example.com:443 -servername manager.example.com -showcerts </dev/null
```

If curl succeeds with the proxy bypass but the tool fails, check the host's HTTPS proxy settings and whether an appropriate `NO_PROXY` entry is needed. The tool follows standard urllib proxy settings.

A `self-signed certificate in certificate chain` error means the HTTPS certificate chain could not be validated against the host's trusted CAs. Authentication has not yet been attempted. Test with `--insecure`, or obtain the verified issuing CA certificate bundle and use `--ca-file`. Trusting the issuer resolves the issue only if other certificate checks, including hostname and validity dates, also pass.

## Exit codes

| Code | vcf-creds | vcf-dns-check | vcf-password-tool |
| --- | --- | --- | --- |
| `0` | Success, help, or no matching files. | PASS or help. | Success, help (including no arguments), or empty result. |
| `1` | File/JSON errors or missing Python. | Resolution, comparison, file, discovery, or missing-`dig` error. | Source, API, or export error. |
| `2` | Too many positional arguments. | INCOMPLETE, invalid arguments, or missing Python. | Invalid arguments. |
| `130` | — | — | Interrupted with Ctrl+C. |

For the DNS checker, errors take precedence over incomplete checks. For the password tool, a source failure still permits listing successfully collected rows but blocks CSV export. Inspect the printed message to distinguish setup errors from check results.

## Troubleshooting

| Problem | Action |
| --- | --- |
| `command not found` | Check PATH and run `command -v` for the affected command. |
| No matching credential files/accounts | Check `pwd`, the JSON extension and opening structure, and read permissions. Try an explicit filename. |
| JSON parsing error | Check valid JSON syntax and that `components` and `credentials` are arrays. |
| Table wraps | Widen the terminal; column widths follow the displayed values. |
| DNS FAIL or MISMATCH | Run `vcf-dns-check -v` to identify the server or query type. |
| No upstream DNS servers | Inspect `resolvectl status` and `nmcli device show`; a loopback stub alone is insufficient. |
| Zone sync unconfirmed | Check whether configured servers host the zone authoritatively or forward/cache it. |
| Host DNS differs from direct servers | Inspect resolver caching, split DNS, and interface routing; use `--interface` when appropriate. |
| Slow DNS checks | Each query uses a two-second timeout and one try, with a 15-second subprocess limit. |
| Missing API manager host | Set `VCF_SDDC_HOST`, pass `--sddc`, or load JSON containing one SDDC Manager component. |
| Unexpected API connection | Check discovered environment files and process `VCF_SDDC_*` settings. |
| API HTTP 401/403 | Verify the account/password or token and SDDC Manager authorization. |
| API TLS/connection error | Inspect the classified error; ping does not test HTTPS or certificate trust. See the HTTPS diagnostic commands above. |
| CSV already exists | Choose another filename with `--output`; existing exports are not overwritten. |
| SecureCRT passwords not imported | Verify the importer's six-field mapping and password-saving behavior. |
