# Day 0 — One-Time Setup: Build a Reusable Base VM

**Time:** ~20 min, done once for the whole 50-day challenge
**You'll need:** VirtualBox (already installed), ~2GB free disk for the ISO

---

## Why This Matters

If you rebuild a fresh Ubuntu install from scratch every session, most of your time goes to the installer, not the lab. Today you build one clean "golden" VM. Every day after this, you clone it in a single command — boots in seconds, always starts from the same known-good state.

## Setup

**1. Download Ubuntu Server 24.04 LTS**
Get the ISO from ubuntu.com/download/server. Note the path you saved it to.

Step 1 — Install VirtualBox
Download

Go to the official site:

VirtualBox Downloads

Download:

Windows hosts installer

VirtualBox Extension Pack (same version as VirtualBox)

Install

Run the installer.

Keep the default options.

Allow any Windows network driver prompts.

After installation, double-click the Extension Pack file to install it.

Verify

Open PowerShell and run:

VBoxManage --version

You should see a version number such as 7.1.x.

If you get “VBoxManage is not recognized”, use the full path:

"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" --version
Step 2 — Download Ubuntu Server 24.04 LTS

Open:

Ubuntu Server download

Download Ubuntu Server 24.04 LTS (64-bit).

Save it somewhere easy, for example:

C:\ISO\ubuntu-24.04-live-server-amd64.iso

Create the folder C:\ISO first if it doesn’t exist.

Step 3 — Create a lab folder

Create a folder where VirtualBox will store the VM disk:

mkdir C:\vbox-labs

Move into it:

cd C:\vbox-labs

Check where you are:

pwd

You should see:


cd C:\vbox-labs
**2. Create the base VM (run in PowerShell/cmd, from your VirtualBox install folder or with it on PATH)**
```
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\Oracle\VirtualBox", "User")
VBoxManage --version

VBoxManage createvm --name "base-vm" --ostype Ubuntu_64 --register
VBoxManage modifyvm "base-vm" --memory 1024 --cpus 1
VBoxManage createhd --filename "base-vm.vdi" --size 8192
VBoxManage storagectl "base-vm" --name "SATA Controller" --add sata
VBoxManage storageattach "base-vm" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium "base-vm.vdi"
```

**3. Unattended install (no manual clicking through the installer)**
```
VBoxManage unattended install "base-vm" --iso="C:\path\to\ubuntu-24.04-live-server-amd64.iso" --user=labuser --password=labpass --install-additions
```
If this specific flag set errors on your VirtualBox version, run `VBoxManage unattended install --help` — the flags shift slightly between versions and it'll show you the current ones.

**4. Add SSH access**
```
VBoxManage modifyvm "base-vm" --natpf1 "ssh,tcp,,2222,,22"
VBoxManage startvm "base-vm" --type headless
```
Give it a minute to finish booting, then:
```
ssh labuser@127.0.0.1 -p 2222
```

**If SSH won't connect (reset/aborted/unit-not-found errors):** the unattended install doesn't always include `openssh-server` by default. Log into the console directly (`labuser`/`labpass`) and check:
```
systemctl status ssh
```
If you see "unit ssh.service could not be found," install it before going any further:
```
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```
Do this **before** the shutdown step below — fix it once here, and every daily clone inherits a working SSH server automatically.

**5. Shut it down clean — this becomes your template**
```
sudo shutdown now
```

## What You Now Have

A stopped "base-vm" you never boot directly again. From Day 1 onward, each day's lab starts with:
```
VBoxManage clonevm "base-vm" --name "day-N-vm" --register
VBoxManage startvm "day-N-vm" --type headless
```
One command, seconds to boot, always identical starting state.

**Forward note:** when weeks 3–4 need multiple nodes talking to each other (MicroCloud clustering), we'll add a Host-only Network adapter to those clones — NAT-only (what you set up today) keeps each VM isolated from the others, which is fine for solo days but won't let nodes see each other later. Nothing to do about that today, just flagging it so it isn't a surprise.
