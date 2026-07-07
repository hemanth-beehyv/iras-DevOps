# SOPS Age Key Usage

This document explains how to export the SOPS Age key as an environment variable and use SOPS to encrypt and decrypt files.

## Prerequisites

- Install `sops`
- Install `age`

Verify the installation:

```bash
sops --version
age --version
```

---

## Export the Age Key

Export the Age private key file location as an environment variable.

```bash
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/keys.txt"
```

Verify:

```bash
echo $SOPS_AGE_KEY_FILE
```

---

## View the Public Key

To display the Age public key:

```bash
age-keygen -y "$SOPS_AGE_KEY_FILE"
```

Example output:

```text
age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

This public key is used for encryption.

---

## Encrypt a File

Encrypt a file using the Age public key.

```bash
sops --encrypt \
  --age age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  secrets.yaml > secrets.enc.yaml
```

Or overwrite the existing file:

```bash
sops --encrypt \
  --in-place \
  --age age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  secrets.yaml
```

---

## Decrypt a File

Decrypt a file:

```bash
sops --decrypt secrets.enc.yaml > secrets.yaml
```

Or edit the encrypted file directly:

```bash
sops secrets.enc.yaml
```

SOPS automatically decrypts the file, opens it in your editor, and encrypts it again when you save and exit.

---

## Encrypt YAML In-Place

```bash
sops --encrypt --in-place \
  --age age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  values.enc.yaml
```

---

## Decrypt to Standard Output

```bash
sops --decrypt values.enc.yaml
```

---

## Encrypt JSON

```bash
sops --encrypt \
  --age age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  secrets.json > secrets.enc.json
```

---

## Decrypt JSON

```bash
sops --decrypt secrets.enc.json > secrets.json
```

---

## Verify the Age Key Being Used

```bash
echo $SOPS_AGE_KEY_FILE
```

Check the public key:

```bash
age-keygen -y "$SOPS_AGE_KEY_FILE"
```

---

## Common Commands

| Task | Command |
|------|---------|
| Export Age key | `export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/keys.txt"` |
| Show public key | `age-keygen -y "$SOPS_AGE_KEY_FILE"` |
| Encrypt file | `sops --encrypt --age <PUBLIC_KEY> secrets.yaml > secrets.enc.yaml` |
| Encrypt in-place | `sops --encrypt --in-place --age <PUBLIC_KEY> secrets.yaml` |
| Decrypt file | `sops --decrypt secrets.enc.yaml > secrets.yaml` |
| Edit encrypted file | `sops secrets.enc.yaml` |

---

## Notes

- Keep the private Age key (`keys.txt`) secure and never commit it to Git.
- Only share the Age **public key** (`age1...`) for encryption.
- Ensure the `SOPS_AGE_KEY_FILE` environment variable is exported before decrypting files.
- SOPS automatically uses the private key referenced by `SOPS_AGE_KEY_FILE` during decryption.