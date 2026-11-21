# Detection

## IOC list

[`iocs.json`](iocs.json) contains the package, hashes, domains, Firebase project, bridge names, paths, and behavior summary.

## YARA

[`dv38.yara`](dv38.yara) targets an **extracted `classes.dex`**, because APK entries are compressed and plaintext strings inside the ZIP will not match.

```bash
# extract the DEX
unzip -o dv38usdf1_2.4.69_20250926222217.apk classes.dex

# scan
yara detection/dv38.yara classes.dex
```

Expected output:

```text
Android_DV38_WebView_Grayware_DEX classes.dex
```

The rule requires the DEX magic at offset 0 and at least five of the package, merchant, domain, backup, update, bridge, install, and FileProvider strings.

## Quick triage

```bash
# hash
sha256sum dv38usdf1_2.4.69_20250926222217.apk

# package and metadata
aapt2 dump badging dv38usdf1_2.4.69_20250926222217.apk | head -20

# signing certificate
apksigner verify --print-certs dv38usdf1_2.4.69_20250926222217.apk

# search for key strings
unzip -p dv38usdf1_2.4.69_20250926222217.apk classes.dex | strings | grep -E "dv38usdf1|gfbwff|854819|window.Android"
```

## Network monitoring

Watch for DNS or TLS SNI matching:

- `12249046.com`
- `dv38.info`, `dv38.dev`, `dv38.pro`, `dv38.blog`
- `www2.854819.com`
- `www.gfbwff.com`

Also alert on `prod.app_dv38usdf1.com` installs outside managed distribution and on `ACTION_INSTALL_PACKAGE` intents originating from that package.

## Note

Static indicators can change between builds. Domain rotation and remote content mean network IOCs have a limited lifetime; the package name, certificate, and code patterns are more stable.
