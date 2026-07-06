# chezmoi-fido2-bridge

A lightweight bash wrapper for `age` and `chezmoi` that automates FIDO2 hardware key integration for secret encryption, decryption, and fallback recovery.

## The Problem
While `chezmoi` natively supports `age` for secret management, integrating hardware-backed encryption via `age-plugin-fido2-hmac` introduces friction. It requires explicitly passing identity files for every operation. Furthermore, managing disaster recovery—ensuring you don't permanently lock yourself out of your dotfiles if you lose your hardware key—requires manually appending a backup recipient to every single encrypt command. 

## The Solution
This repository provides drop-in wrapper scripts that intercept `age` commands invoked by `chezmoi`. 

It automatically:
1. Injects the required FIDO2 identity flags.
2. Ensures all secrets are simultaneously encrypted to both your primary FIDO2 hardware token and a designated backup recovery key.
3. Eliminates the need for manual flag configuration when managing hardware-backed dotfile secrets across multiple machines.

## Prerequisites

* [`chezmoi`](https://chezmoi.io/)
* [`age`](https://github.com/FiloSottile/age)
* [`age-plugin-fido2-hmac`](https://github.com/str4d/age-plugin-fido2-hmac)
* A generated FIDO2 identity file.

## Quick Start & Installation

You can either clone this repository or create the scripts directly using the code blocks below. Place them in a directory included in your `$PATH` (e.g., `~/.local/bin/`).

Make sure all scripts are executable:
```bash
chmod +x ~/.local/bin/age-fido2-*
```

### 1. `age-fido2-wrapper`
This is the primary interceptor for `chezmoi`.
```bash
#!/bin/bash

# Ensure age is in the path
export PATH="$PATH:$HOME/go/bin:$HOME/.local/bin"
AGE_BIN=$(command -v age || echo "/usr/bin/age")

# Use Environment Variables or fallback to standard chezmoi paths
FIDO2_IDENTITY="${AGE_FIDO2_IDENTITY:-$HOME/.config/chezmoi/age-identity.txt}"
RECOVERY_RECIPIENT="${AGE_RECOVERY_RECIPIENT}"

INPUT_FILE="${@: -1}"
ARGS=("${@:1:$(($# - 1))}")

if [[ "$*" == *"--decrypt"* ]] || [[ "$*" == *"-d"* ]]; then
    NEW_ARGS=()
    for arg in "${ARGS[@]}"; do
        NEW_ARGS+=("$arg")
    done
    exec "$AGE_BIN" "${NEW_ARGS[@]}" -i "$FIDO2_IDENTITY" "$INPUT_FILE"
else
    NEW_ARGS=()
    SKIP_NEXT=false
    for arg in "${ARGS[@]}"; do
        if [ "$SKIP_NEXT" = true ]; then
            SKIP_NEXT=false
            continue
        fi
        if [ "$arg" = "-r" ] || [ "$arg" = "--recipient" ]; then
            SKIP_NEXT=true
            continue
        fi
        NEW_ARGS+=("$arg")
    done
    
    # If a recovery recipient is defined in the environment, append it
    if [ -n "$RECOVERY_RECIPIENT" ]; then
        exec "$AGE_BIN" "${NEW_ARGS[@]}" -r "$RECOVERY_RECIPIENT" -i "$FIDO2_IDENTITY" "$INPUT_FILE"
    else
        exec "$AGE_BIN" "${NEW_ARGS[@]}" -i "$FIDO2_IDENTITY" "$INPUT_FILE"
    fi
fi
```

### 2. `age-fido2-encrypt`
Standalone script for manual encryption outside of `chezmoi`.
```bash
#!/bin/bash

export PATH="$PATH:$HOME/go/bin:$HOME/.local/bin"
AGE_BIN=$(command -v age || echo "/usr/bin/age")
FIDO2_IDENTITY="${AGE_FIDO2_IDENTITY:-$HOME/.config/chezmoi/age-identity.txt}"
RECOVERY_RECIPIENT="${AGE_RECOVERY_RECIPIENT}"

if [ -n "$RECOVERY_RECIPIENT" ]; then
    exec "$AGE_BIN" -e -i "$FIDO2_IDENTITY" -r "$RECOVERY_RECIPIENT" "$@"
else
    exec "$AGE_BIN" -e -i "$FIDO2_IDENTITY" "$@"
fi
```

### 3. `age-fido2-decrypt`
Standalone script for manual decryption outside of `chezmoi`.
```bash
#!/bin/bash

export PATH="$PATH:$HOME/go/bin:$HOME/.local/bin"
AGE_BIN=$(command -v age || echo "/usr/bin/age")
FIDO2_IDENTITY="${AGE_FIDO2_IDENTITY:-$HOME/.config/chezmoi/age-identity.txt}"

exec "$AGE_BIN" -d -i "$FIDO2_IDENTITY" "$@"
```

## Configuration

### 1. Environment Variables
The wrappers read from environment variables to locate your hardware identity and your backup recovery key. Add the following to your `.bashrc`, `.zshrc`, or environment configuration:

```bash
# Path to your FIDO2 identity file. 
# If unset, defaults to: ~/.config/chezmoi/age-identity.txt
export AGE_FIDO2_IDENTITY="$HOME/.config/chezmoi/age-identity.txt"

# Your standard age public key for disaster recovery
export AGE_RECOVERY_RECIPIENT="age1yourrecoverykeyhere..."
```

### 2. Chezmoi Configuration
Update your `~/.config/chezmoi/chezmoi.toml` file to point `chezmoi`'s encryption engine at the wrapper script instead of the default `age` binary:

```toml
encryption = "age"
[age]
    command = "age-fido2-wrapper"
    args = []
```

## Usage

Once installed and configured in your `chezmoi.toml`, no change to your workflow is required. 

Run `chezmoi` commands exactly as you normally would:
```bash
# Encrypts a new secret to both the FIDO2 key and the recovery key
chezmoi add --encrypt ~/.secret-file

# Prompts your hardware key for decryption during editing
chezmoi edit ~/.secret-file

# Applies configuration, decrypting on the fly
chezmoi apply
```

If you need to use the wrappers manually outside of `chezmoi`:
```bash
# Encrypt a file
age-fido2-encrypt target-file.txt > secret.age

# Decrypt a file
age-fido2-decrypt secret.age > target-file.txt
```
