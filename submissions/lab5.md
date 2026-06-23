# Lab 5 — Virtualization: QuickNotes in a Vagrant VM

## Task 1 — Vagrantfile + QuickNotes

### Vagrantfile

The Vagrantfile is located at the repository root.

```ruby
Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/jammy64"
    config.vm.hostname = "quicknotes-vm"
    
    config.vm.network "forwarded_port",
    guest: 8080,
    host: 18080,
    host_ip: "127.0.0.1"
    
    config.vm.synced_folder "./app", "/home/vagrant/app"
    
    config.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
    vb.cpus = 2
    end
    
    config.vm.provision "shell", inline: <<-SHELL
    set -e
    
    sudo apt-get update
    sudo apt-get install -y curl tar
    
    cd /tmp
    
    if ! /usr/local/go/bin/go version 2>/dev/null | grep -q "go1.24.5"; then
      curl -LO https://go.dev/dl/go1.24.5.linux-amd64.tar.gz
      sudo rm -rf /usr/local/go
      sudo tar -C /usr/local -xzf go1.24.5.linux-amd64.tar.gz
    fi
    
    echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/go.sh >/dev/null
    
    export PATH=$PATH:/usr/local/go/bin
    
    go version
    
    SHELL
    end
    
```

---

### Vagrant up output

First 10 lines:

```text
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Box 'ubuntu/jammy64' could not be found. Attempting to find and install...
    default: Box Provider: virtualbox
    default: Box Version: >= 0
==> default: Loading metadata for box 'ubuntu/jammy64'
    default: URL: https://vagrantcloud.com/api/v2/vagrant/ubuntu/jammy64
==> default: Adding box 'ubuntu/jammy64' (v20241002.0.0) for provider: virtualbox
    default: Downloading: https://vagrantcloud.com/ubuntu/boxes/jammy64/versions/20241002.0.0/providers/virtualbox/unknown/vagrant.box
    default:
==> default: Successfully added box 'ubuntu/jammy64' (v20241002.0.0) for 'virtualbox'!
==> default: Importing base box 'ubuntu/jammy64'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ubuntu/jammy64' version '20241002.0.0' is up to date...
==> default: Setting the name of the VM: DevOps-Outro_default_1782226527130_32141
Vagrant is currently configured to create VirtualBox synced folders with
the `SharedFoldersEnableSymlinksCreate` option enabled. If the Vagrant
guest is not trusted, you may want to disable this option. For more
information on this option, please refer to the VirtualBox manual:
```

---

### Verification

Go version inside VM:

```bash
vagrant ssh -c "go version"
```

Output:

```text
go version go1.24.5 linux/amd64
```

---

VM curl:

Command:

```bash
vagrant ssh -c "curl -s http://localhost:8080/health"
```

Output:

```json
{"notes":6,"status":"ok"}
```

---

Host curl:

Command:

```bash
curl.exe http://localhost:18080/health
```

Output:

```json
{"notes":6,"status":"ok"}
```

---

## Design Questions

### a) Synced folders

I used VirtualBox shared folders because they require no additional host configuration and work reliably on Windows. The trade-off is that performance can be slower than alternatives such as NFS or rsync.

### b) Networking

The VM uses NAT networking with port forwarding. Binding the forwarded port to 127.0.0.1 keeps the service accessible only from the local machine. A bridged network would expose the VM directly to the local network.

### c) Provisioning

I used the shell provisioner because it is simple and built into Vagrant. It is enough for installing Go automatically during VM creation.

### d) Why pin Go 1.24.5?

A specific patch version ensures reproducible builds. Installing only Go 1.24 could result in different versions being installed later.

---

# Task 2 — Snapshots

Snapshot created:

```bash
vagrant snapshot save clean-working
```

Snapshot list:

```text
==> default:
clean-working
```

---

Break command:

```bash
vagrant ssh -c "sudo rm -rf /usr/local/go"
```

Verification:

```bash
vagrant ssh -c "go version"
```

Output:

```text
bash: line 1: go: command not found
```

---

Restore:

```bash
time vagrant snapshot restore clean-working
```

Output:

```text
real    0m27.039s
user    0m0.000s
sys     0m0.031s
```

---

Recovery:

```bash
vagrant ssh -c "go version"
```

Output:

```text
go version go1.24.5 linux/amd64
```

---

## Snapshot Questions

### e) Why snapshots are not backups

Snapshots are stored with the VM and depend on the same storage system. If the host disk fails or the VM files are deleted, the snapshot is also lost.

### f) Copy-on-write

Copy-on-write means snapshots initially share unchanged data with the original disk. New disk space is only used when changes are made after the snapshot.

### g) Snapshot chains

Long snapshot chains can consume storage and reduce performance. They also make recovery and management more complicated.
