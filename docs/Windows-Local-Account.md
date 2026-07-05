# Windows Local Account (OOBE Bypass)

This guide outlines how to bypass the forced Microsoft Account login requirement during a fresh Windows installation (Out-of-Box Experience, or OOBE).

By executing a custom deployment script, you can skip the Microsoft account prompt and automatically decline all telemetry/privacy questions. This method creates a standard local account while still allowing you to select your preferred language, region, and keyboard layout.

??? info "Source & Credits"
    The scripts utilized below are based on the [bypassnro project by xEska1337](https://github.com/xEska1337/bypassnro), incorporating configurations from ChrisTitusTech and my own modifications.

---

## Prerequisites

* A fresh Windows installation currently waiting at the initial OOBE setup screen.
* An **active internet connection**. *(Note: Unlike the standard `OOBE\BYPASSNRO` method which requires you to disconnect from the internet, this specific method requires a connection to download the script via `curl`).*

---

## Deployment Steps

### 1. Open the Command Prompt

When you reach the very first Windows setup screen (usually asking for your Language or Region), do not click "Next". Instead, open the hidden command prompt:

* Press **`Shift + F10`** on your keyboard.

### 2. Execute the Bypass Script

In the black command prompt window that appears, you need to download and run the bypass script. You have two options depending on which configuration you want to load.

Type **one** of the following sets of commands and press `Enter`.

**Option A: Standard ChrisTitusTech Configuration**
This uses the widely trusted baseline configuration to skip OOBE.

```bat
curl -L mountainash.net/bypass -o skip.cmd
skip.cmd

```

**Option B: Custom Modified Configuration**
This executes my personally modified version of the bypass script.

```bat
curl -L mountainash.net/bypassmy -o skip.cmd
skip.cmd

```

---

## How it Works (Under the Hood)

The method utilized above is built entirely on Microsoft's native **Windows Unattended Setup** framework.

When you run the `skip.cmd` file in the steps above, the script executes two primary commands behind the scenes:

```bat
curl -L -o C:\Windows\Panther\unattend.xml https://raw.githubusercontent.com/xEska1337/bypassnro/refs/heads/main/unattend.xml
%WINDIR%\System32\Sysprep\Sysprep.exe /oobe /unattend:C:\Windows\Panther\unattend.xml /reboot

```

1. **The Download (`curl`):** The first command downloads a pre-configured `unattend.xml` file from a remote repository and places it in the `C:\Windows\Panther\` directory (the default location Windows checks for setup instructions).
2. **The Execution (`Sysprep.exe`):** The second command calls the System Preparation Tool. It forces Windows to restart the OOBE process, but this time it instructs it to strictly follow the rules defined in the downloaded `unattend.xml` file, which includes instructions to skip Microsoft account linking and telemetry generation.

!!! info "Custom Unattended Generators"
    If you want to create your own custom `unattend.xml` file with specific user accounts, pre-installed software, or custom regional settings already baked in, you can generate your own version using the **[Schneegans Windows Unattend Generator](https://schneegans.de/windows/unattend-generator/)**.