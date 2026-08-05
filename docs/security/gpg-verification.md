# GPG Verification Guide

!!! note "Prerequisites"
    You need to have GPG (GNU Privacy Guard) installed on your system:

    - Windows: [Gpg4win](https://www.gpg4win.org/)
    - macOS: `brew install gpg`
    - Linux: `sudo apt-get install gnupg` or `sudo yum install gnupg`

## Overview

Starting from version 0.9.2, releases are published to Maven repository and signed with the GPG key:

```text
pub: 3149 FCA5 6377 2D11 2624 9C36 CC3F 8CEA 7B0C 74D6
uid: buildtrident@tron.network
```

## Verification Steps

1. Import the Trident public key:
    ```bash
    gpg --keyserver hkp://keys.openpgp.org --recv-keys 3149FCA563772D1126249C36CC3F8CEA7B0C74D6
    ```

2. Download package files:
    ```bash
    wget https://repo1.maven.org/maven2/io/github/tronprotocol/trident/1.0.0/trident-1.0.0.jar
    wget https://repo1.maven.org/maven2/io/github/tronprotocol/trident/1.0.0/trident-1.0.0.jar.asc
    ```

3. Verify the signature:
    ```bash
    gpg --verify trident-1.0.0.jar.asc trident-1.0.0.jar
    ```

    A successful verification will show:
    ```text
    gpg: Signature made Mon Mar  3 18:50:13 2025 CST
    gpg:                using RSA key 3149FCA563772D1126249C36CC3F8CEA7B0C74D6
    gpg: Good signature from "trident <buildtrident@tron.network>"
    ```

## Security Notes

1. Always verify downloads before using them in production
2. Make sure you're using the correct public key
3. Download files only from official sources
4. Keep your GPG installation up to date

## Troubleshooting

If you encounter issues:

1. Verify you have imported the correct public key
2. Ensure both the JAR and signature files are for the same version
3. Check that the files were downloaded completely
4. Try refreshing the GPG keyring: `gpg --refresh-keys`
