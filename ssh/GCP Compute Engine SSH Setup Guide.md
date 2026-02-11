# GCP Compute Engine SSH Setup Guide (Manual Connection)

This documentation covers the standard workflow for generating keys in **Windows PowerShell**, linking them to your VM, and managing access directly from the Linux terminal.

## 1. Local Key Generation (Windows PC)

We used **PowerShell** to create a modern **ED25519** key pair. This is the "Identity" your PC uses to prove who you are.

* **Command:** ```powershell
ssh-keygen -t ed25519 -f "$HOME.ssh\gcp" -C "cheska_site2018"
```

```


* **Resulting Files:**
* `gcp`: The **Private Key**. Never share or upload this file.
* `gcp.pub`: The **Public Key**. This is the "lock" you give to the server.



---

## 2. Connecting to the VM

Because the key has a custom name (`gcp`), you must tell PowerShell which identity to use by using the `-i` flag.

* **Command:**
```powershell
ssh -i ~/.ssh/gcp cheska_site2018@35.221.196.21

```



---

## 3. Managing Keys inside the Server (Linux CLI)

Once you are logged in, you can manage who has access by editing the `authorized_keys` file directly. This is useful for adding a second key or a colleague's key.

1. **Open the file with Nano:**
```bash
nano ~/.ssh/authorized_keys

```


2. **Edit the file:**
* **To Add:** Paste a new public key (starting with `ssh-ed25519...`) on a new line at the bottom.
* **To Remove:** Delete the line containing the key you want to revoke.


3. **Save and Exit:**
* Press `Ctrl + O` and `Enter` to save.
* Press `Ctrl + X` to exit the editor.



---

## 4. Troubleshooting: Permissions

SSH is very strict. If your permissions are too "loose," the server will reject the key for security reasons. Use these commands to ensure only your user can see the keys.

### In Windows PowerShell (Local):

```powershell
icacls "$HOME\.ssh\gcp" /inheritance:r
icacls "$HOME\.ssh\gcp" /grant:r "$($env:USERNAME):R"

```

### In Linux Terminal (Remote):

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

```

---

## Summary Table

| File | Location | Role |
| --- | --- | --- |
| `gcp` | Windows PC | Your secret key (stays on your PC). |
| `gcp.pub` | Windows PC | The lock you copy/paste into GCP. |
| `authorized_keys` | Linux VM | The list of all locks allowed to open the server. |

[Generating SSH keys for Google Cloud](https://www.youtube.com/watch?v=a4rldiAoP5A)
This video walks through the exact process of generating keys and adding them to the Google Cloud metadata section, mirroring the steps we followed in PowerShell.
