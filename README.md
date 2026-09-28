# fix_mc_auth_pls — TOTP codes on your computer

You ever try to lock in without your phone, but MC Auth makes u scan a code or smth, and it gets rly annoying? Well, heres:
A hotkey that generates your TOTP (6-digit authenticator app) code on your
computer, without needing your phone. macOS (Raycast / Hammerspoon) and
Windows (PowerShell / AutoHotkey) versions included.

Useful if your organization's MFA is normally tied to a phone-based
authenticator app (Microsoft Authenticator, Google Authenticator, etc.) and
you want to sign in from your computer even when your phone isn't handy, as
long as your MFA provider lets you register a generic/third-party
authenticator app (most do, via an "I want to use a different app" or "can't
scan the QR code" option during setup).

> **Read the [security trade-offs](#%EF%B8%8F-security-trade-offs--read-before-using-this)
> section before setting this up.** This is a convenience tool, not a
> security upgrade.

## Quick links

| What | Link |
| --- | --- |
| Download everything (ZIP) | [main.zip](https://github.com/Sathvik21/mc_auth_sucks/archive/refs/heads/main.zip) |
| macOS script | [totp.sh](https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/raycast-scripts/totp.sh) |
| macOS auto-detect (optional) | [init.lua.example](https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/hammerspoon/init.lua.example) |
| Windows script | [totp.ps1](https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/windows/totp.ps1) |
| Windows hotkey | [hotkey.ahk](https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/windows/hotkey.ahk) |
| Homebrew (macOS) | [brew.sh](https://brew.sh) |
| Raycast (macOS) | [raycast.com](https://www.raycast.com) |
| AutoHotkey v2 (Windows) | [autohotkey.com](https://www.autohotkey.com/) |
| Microsoft security info page | [mysignins.microsoft.com/security-info](https://mysignins.microsoft.com/security-info) |

## How it works

TOTP codes are generated from a shared secret plus the current time. Your
device and the server both compute the same 6-digit code independently, with
no network call required. This project stores that secret in your OS's
credential store (macOS Keychain / Windows Credential Manager, encrypted and
unlocked when you're logged in) and gives you a hotkey that reads it,
computes the current code, and copies it to your clipboard.

**Do steps 1 through 3 as one continuous pass, in the same browser tab.**
The secret is shown once. If you close or restart the registration dialog,
your provider generates a new secret and the old one stops working.

---

## macOS Setup

### Before you start

You need:

- [Homebrew](https://brew.sh). Check with `brew --version`. If it's missing,
  install it from the link.
- [Raycast](https://www.raycast.com) (or use the Hammerspoon variant below).

### Step 1: Install the tools and download the script

Open Terminal and run:

```bash
brew install oath-toolkit

mkdir -p ~/raycast-scripts
curl -fsSL -o ~/raycast-scripts/totp.sh \
  https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/raycast-scripts/totp.sh
chmod +x ~/raycast-scripts/totp.sh
```

No `git` needed. The `curl` line downloads the script straight into place.

### Step 2: Get your TOTP secret

1. Go to [mysignins.microsoft.com/security-info](https://mysignins.microsoft.com/security-info)
   (or your provider's security settings page).
2. Click **Add sign-in method**.
3. Pick **Microsoft Authenticator** (or "Authenticator app").
4. It will ask you to install the app. Ignore that and click **"I want to
   use a different authenticator app"** near the bottom.
   - If this link isn't there, your account is locked to the official app
     and this approach won't work for it.
5. On the QR code screen, click **"Can't scan the QR code?"**
6. Copy the **Secret key** (and the full `otpauth://` Setup URI if shown).
7. **Leave this tab open.** Don't click Next yet.

### Step 3: Store the secret in Keychain

In Terminal, run this. It waits for input and shows no prompt:

```bash
read -rs OTPURL && security add-generic-password -a totp-seed -s totp -w "$OTPURL" && unset OTPURL
```

Paste the `otpauth://` URL and press Enter. Nothing will echo to the screen;
that's expected.

If you only have a bare secret key, build the URL yourself:

```
otpauth://totp/ACCOUNT_NAME?secret=YOUR_SECRET_KEY&issuer=Microsoft
```

`ACCOUNT_NAME` is what the dialog showed next to **Account name**, usually
`domain.com:you@domain.com`. Example:

```
otpauth://totp/contoso.com:jsmith@contoso.com?secret=abcd1234efgh5678&issuer=Microsoft
```

Verify it saved (this prints the secret, so don't paste the output anywhere):

```bash
security find-generic-password -a totp-seed -s totp -w
```

### Step 4: Generate a code and finish registration

```bash
~/raycast-scripts/totp.sh
```

It prints a 6-digit code and copies it to your clipboard. Go back to the
browser tab **right away** (codes expire after 30 seconds), paste the code,
and submit.

**Use the code this script printed, not one from your phone.** Your phone's
authenticator uses a *different* secret and its codes won't match.

> **Keychain prompt:** The first time the script runs, macOS may ask for
> Touch ID or your password to allow access to the Keychain item. Approve
> it. Choosing **Always Allow** stops the prompt from appearing every time.

### Step 5: Set up the Raycast hotkey

In Raycast: **Settings → Extensions → (+) → Add Script Directory**, and
select `~/raycast-scripts`. The "TOTP Code" command should appear. Click the
hotkey field next to it and press your combo (avoid ones already in use,
like ⌘V).

Test it: press the hotkey, then paste anywhere. You should get a fresh
6-digit code that changes every 30 seconds.

### Optional: Hammerspoon auto-detect variant

[`hammerspoon/init.lua.example`](https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/hammerspoon/init.lua.example)
watches your browser and shows a notification when you land on a matching
login page, instead of requiring a hotkey press. Install
[Hammerspoon](https://www.hammerspoon.org/), then:

```bash
mkdir -p ~/.hammerspoon
curl -fsSL -o ~/.hammerspoon/init.lua \
  https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/hammerspoon/init.lua.example
```

This overwrites any existing `~/.hammerspoon/init.lua`, so back yours up
first. See the comments in the file for setup and for why you might *not*
want it running all the time (it polls your active browser tab continuously).

---

## Windows Setup

Windows uses Credential Manager for storage and a pure-PowerShell TOTP
implementation (no external binary needed).

### Step 1: Download the files

In PowerShell:

```powershell
mkdir $HOME\totp -Force
cd $HOME\totp
Invoke-WebRequest https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/windows/totp.ps1 -OutFile totp.ps1
Invoke-WebRequest https://raw.githubusercontent.com/Sathvik21/mc_auth_sucks/main/windows/hotkey.ahk -OutFile hotkey.ahk
```

Both files must stay in the **same folder**. Install
[AutoHotkey v2](https://www.autohotkey.com/) if you don't have it.

### Step 2: Get your TOTP secret

Same as macOS Step 2 above.

### Step 3: Store the secret in Credential Manager

```powershell
cmdkey /generic:totp-seed /user:totp /pass:"otpauth://totp/ACCOUNT_NAME?secret=YOUR_SECRET_KEY&issuer=Microsoft"
```

`ACCOUNT_NAME` is what the dialog showed next to **Account name**, usually
`domain.com:you@domain.com`. Example:

```powershell
cmdkey /generic:totp-seed /user:totp /pass:"otpauth://totp/contoso.com:jsmith@contoso.com?secret=abcd1234efgh5678&issuer=Microsoft"
```

This puts the secret briefly in your PowerShell history. Run `Clear-History`
afterward, or use the `CredentialManager` module, which doesn't echo or log it:

```powershell
Install-Module CredentialManager -Scope CurrentUser
$secureUrl = Read-Host -AsSecureString "Paste otpauth:// URL"
New-StoredCredential -Target "totp-seed" -UserName "totp" -SecurePassword $secureUrl -Persist LocalMachine
```

### Step 4: Generate a code and finish registration

```powershell
cd $HOME\totp
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\totp.ps1
```

If you get a script-execution error, run this once, then retry:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

It prints a 6-digit code and copies it to your clipboard. Go back to the
browser tab **immediately**, paste it, and submit. Use the code from
`totp.ps1`, not from your phone.

### Step 5: Set up the AutoHotkey hotkey

1. Open `hotkey.ahk` in Notepad if you want to change the hotkey. The line
   to edit and a symbol reference table are at the top. Default is
   `Ctrl+Alt+T`.
2. Right-click `hotkey.ahk` → **Run Script** (not double-click, which can
   open the AutoHotkey dashboard instead).
3. A green **H** icon should appear in the system tray.
4. Press your hotkey anywhere. You should get a toast with a fresh code.

To run at login: right-click `hotkey.ahk` → Create shortcut → move the
shortcut into the Startup folder (Win+R, type `shell:startup`, Enter).

---

## Troubleshooting

Run these in order and stop at the first failure.

### macOS

| Check | Command | If it fails |
| --- | --- | --- |
| `oathtool` installed | `which oathtool` | `brew install oath-toolkit` |
| Secret is in Keychain | `security find-generic-password -a totp-seed -s totp > /dev/null && echo OK` | The secret is gone. Do a [reset](#reset-start-over). |
| Script exists and is executable | `ls -l ~/raycast-scripts/totp.sh` | Re-run the `curl` and `chmod` from Step 1. |
| Script produces a code | `~/raycast-scripts/totp.sh` | Read the error. If it mentions the Keychain, approve the prompt. |
| Hotkey works | Press it, then paste | In Raycast, re-add `~/raycast-scripts` and reassign the hotkey. |

Other common issues:

- **Code is rejected by the website.** Make sure your Mac's clock is set
  automatically (System Settings → General → Date & Time). TOTP depends on
  accurate time.
- **Code works during setup but not later.** You may have used a code from
  your phone during registration, or restarted the dialog. Do a
  [reset](#reset-start-over).
- **Rejected on sensitive resources only.** Your organization's Conditional
  Access policy may require phone-based or phishing-resistant MFA there.

### Windows

- **Credential exists?** Open Credential Manager → Windows Credentials and
  look for `totp-seed`. If it's missing, do a [reset](#reset-start-over).
- **Script errors.** Run `totp.ps1` directly (Step 4) and read the message.
- **Hotkey does nothing.** Check for the green **H** tray icon. If it's
  missing, run `hotkey.ahk` again. Make sure `totp.ps1` is in the same
  folder.

## Reset (start over)

Use this if anything is broken and you'd rather begin cleanly.

**macOS**

```bash
security delete-generic-password -a totp-seed -s totp
rm -f ~/raycast-scripts/totp.sh
```

**Windows**

```powershell
cmdkey /delete:totp-seed
Remove-Item $HOME\totp -Recurse -Force
```

Then:

1. Go to your provider's security page and delete the old authenticator
   entry you made for this. **Keep your phone one.**
2. Redo Setup from the beginning to get a fresh secret.

---

## ⚠️ Security trade-offs — read before using this

This works, but it changes what your "second factor" actually protects
against. Know this before you rely on it:

- **Factor collapse.** If your TOTP secret and your password both end up
  reachable from the same unlocked computer (e.g. both in the same password
  manager or credential store), your two-factor login is now protected by
  one thing: your computer being unlocked. That's a real reduction in
  security compared to a code source on a separate physical device.
- **TOTP is phishable.** A fake login page can ask for your password and
  your 6-digit code, then relay both to the real service within the
  ~30-second validity window. Push notifications with number matching, and
  passkeys/security keys, resist this because they're cryptographically
  bound to the real domain. If your account offers a passkey or hardware
  security key (e.g. YubiKey) and phishing resistance matters to you, that's
  the stronger option.
- **Some resources may reject it anyway.** Organizations with Conditional
  Access / phishing-resistant-MFA policies on sensitive resources (VPN,
  admin tools, financial systems) may reject a TOTP code even if it's
  accepted for everyday sign-in. You may still need your phone for those.
- **Local storage is still a target.** Anything readable by your user
  account from the credential store (Keychain or Credential Manager) is
  readable by any process running as you. The encryption protects against
  someone copying the raw file off disk, not against malicious software
  already running as you.
- **Never share your secret.** Don't paste the `otpauth://` URL or the
  secret key into chats, tickets, or screenshots. If you do, reset it.
- **Keep a backup MFA method registered.** Don't delete your phone-based
  authenticator entry. If you lose access to this computer and its stored
  credential, you want another way back in that doesn't require an
  account-recovery process.

This is a convenience tool, not a security upgrade. Use it for lower-stakes,
frequent logins where phone friction is the main problem, and prefer
phishing-resistant methods (passkeys, hardware keys) wherever your
organization supports them.

## License

MIT. See [LICENSE](https://github.com/Sathvik21/mc_auth_sucks/blob/main/LICENSE).
