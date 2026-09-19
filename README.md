# Nintendo S3S / nxapi Authentication Fix

A practical workaround for the obsolete Nintendo Account authentication flow in `space4y/nxapi-s3s:0.7.0`.

## Why the old Docker flow fails

The old image runs an entrypoint that expects `nxapi nso auth` to print a `session_token:` line and then parses that output.

Current nxapi no longer exposes the session token that way. Authentication is stored internally in nxapi's persistent storage.

Because of that mismatch, the old image can fail with:

```
Error: Remote configuration prevents Coral authentication
```

This does **not** necessarily mean Nintendo Account authentication itself failed.

## Working method

Use **current nxapi** for Nintendo Account / SplatNet authentication, and use the **old image only to run the s3s Python program**.

### 1. Authenticate with current nxapi

PowerShell:

```powershell
docker run --rm -it `
  -v "C:\Users\AMD\Desktop\s3s:/data" `
  ghcr.io/samuelthomas2774/nxapi:ref-main `
  nso auth
```

A successful login includes:

```
Authenticated as Nintendo Account ...
Set as default user
```

### 2. Verify the saved authentication

```powershell
docker run --rm -it `
  -v "C:\Users\AMD\Desktop\s3s:/data" `
  ghcr.io/samuelthomas2774/nxapi:ref-main `
  nso user
```

If Nintendo Account / Nintendo Switch user information is returned, the saved authentication is usable.

> **Note:** In current nxapi, `nso token` is for setting a token; it is not a command for displaying the existing stored session token.

### 3. Generate the s3s configuration

```powershell
docker run --rm -it `
  -v "C:\Users\AMD\Desktop\s3s:/data" `
  ghcr.io/samuelthomas2774/nxapi:ref-main `
  util update-s3s-token /data/config.txt
```

A successful run includes:

```
Authenticating to SplatNet 3
writing s3s config file
```

The generated `config.txt` contains authentication material. **Never commit it to GitHub.**

### 4. Run s3s without the obsolete entrypoint

The old image expects its configuration at `/s3s/config.txt`, while current nxapi writes it to the mounted `/data/config.txt`.

Copy the generated configuration into the old container and invoke `s3s.py` directly:

```powershell
docker run --rm -it `
  -v "C:\Users\AMD\Desktop\s3s:/data" `
  --entrypoint /bin/sh `
  space4y/nxapi-s3s:0.7.0 `
  -c "cp /data/config.txt /s3s/config.txt && cd /s3s && python s3s.py --getseed && mv gear_*.json /data/"
```

Generated `gear_*.json` files are placed in:

```
C:\Users\AMD\Desktop\s3s
```

## Quick flow

```text
Current nxapi
  │
  ├─ nso auth
  ├─ nso user
  └─ util update-s3s-token
             │
             ▼
        /data/config.txt
             │
             ▼
Old nxapi-s3s image
  │
  └─ s3s.py --getseed
             │
             ▼
        gear_*.json
```

## Security

Do not publish or commit:

- `config.txt`
- Nintendo Account session tokens
- JWTs
- SplatNet `gtoken`
- `bulletToken`
- private authentication logs
- local persistent authentication storage

This repository contains the workaround and commands, **not user authentication data**.

## Troubleshooting

### `Remote configuration prevents Coral authentication`

Do not keep retrying the old image's built-in authentication step. Use current nxapi to authenticate and generate the s3s configuration, then bypass the old entrypoint as shown above.

### `nso user` says no token is set

Run `nso auth` again using the same persistent mount, then retry `nso user`.

### `config.txt` is missing

Run `util update-s3s-token /data/config.txt` first. The command creates or updates the file.

## Credits / upstream components

This workaround combines:

- current [nxapi](https://github.com/samuelthomas2774/nxapi)
- the existing [s3s / nxapi-s3s](https://github.com/space4y/nxapi-s3s) Docker image

The purpose of this repository is to document the compatibility workaround so users do not waste time trying to make the obsolete authentication entrypoint work unchanged.
