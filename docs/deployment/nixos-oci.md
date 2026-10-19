# NixOS on OCI: Architecture, Benefits, and Operations

This document details running **NixOS** as an operating system for Passly and its edge reverse proxy on Oracle Cloud Infrastructure (OCI), specifically on Always Free instances (`VM.Standard.E2.1.Micro` and `VM.Standard.A1.Flex`).

---

## 1. Why NixOS in Cloud Environments?

While standard distributions like Ubuntu or Debian are straightforward to deploy, **NixOS** provides distinct operational advantages for infrastructure reliability and maintenance:

1. **Complete Declarative Configuration**:
   The entire OS state — packages, kernel parameters, firewall rules, user accounts, Nginx vhosts, PHP-FPM pools, and database tuning — is declared in code (`/etc/nixos/configuration.nix`). The server state is version-controlled and auditable.
2. **Atomic Upgrades & Instant Rollback**:
   Every rebuild creates an immutable new system generation. If an OS update or Nginx syntax error causes an outage, running:
   ```bash
   sudo nixos-rebuild switch --rollback
   ```
   instantly returns the machine to the previous working state without manual package uninstalls or filesystem restores.
3. **Zero Configuration Drift**:
   NixOS eliminates ad-hoc `/etc/` changes and forgotten manual tweaks. If a service or configuration directive is not declared in the Nix expression, it does not exist in the active runtime.
4. **Reproducible Deployments**:
   Spinning up a replacement VM or standing up a staging clone takes minutes by applying the same configuration files.
5. **Isolated Toolchains**:
   Combined with Nix Flakes (`flake.nix`), local development environments match production runtimes down to the exact library versions.

---

## 2. Installing NixOS on OCI

OCI does not provide an official out-of-the-box NixOS platform image. The standard method for provisioning NixOS on an OCI compute instance:

1. Launch a standard `VM.Standard.E2.1.Micro` (or `VM.Standard.A1.Flex`) instance running **Ubuntu 24.04**.
2. Connect via SSH and run [`nixos-infect`](https://github.com/elitak/nixos-infect):
   ```bash
   curl https://raw.githubusercontent.com/elitak/nixos-infect/master/nixos-infect | NIXOS_RELEASE=25.11 bash -x
   ```
3. Once the conversion script finishes, the instance will automatically reboot into NixOS.

---

## 3. Critical Operational Rule: Memory Management on 1 GB Micro VMs

> [!WARNING]
> **Evaluation Memory Exhaustion (OOM)**:
> Evaluating and building a NixOS system closure locally on an x86_64 machine requires approximately 1 GB of RAM during `nixos-rebuild switch`. On a 1 GB Micro VM (`VM.Standard.E2.1.Micro`), a rebuild without swap and memory protection can trigger the Linux OOM killer, causing SSH and Nginx to hang or crash.

### Mandatory Protection Steps for Micro VMs:

#### 1. File-Backed Disk Swap
Do not rely exclusively on `zram` (compressed RAM). While zram is efficient for small anonymous pages, it cannot absorb a single process demanding 1 GB of memory. 

Declare a file-backed swapfile in `/etc/nixos/configuration.nix`:
```nix
swapDevices = [
  {
    device = "/var/swapfile";
    size = 2048; # 2 GB
  }
];
```

*Note: The declarative config cannot help the very first rebuild that applies it. Create the swapfile manually before running your first rebuild:*
```bash
sudo fallocate -l 2G /var/swapfile || sudo dd if=/dev/zero of=/var/swapfile bs=1M count=2048 status=none
sudo chmod 600 /var/swapfile
sudo mkswap -q /var/swapfile
sudo swapon /var/swapfile
```

#### 2. Enforce Single-Job Evaluation
Always constrain the builder to a single core and a single job to avoid concurrency memory spikes:
```bash
sudo nixos-rebuild switch --max-jobs 1 --cores 1
```

#### 3. Protect Running Services via `systemd-run`
To ensure that an unexpected build spike never terminates active production services (Nginx, PHP-FPM, or MySQL), run `nixos-rebuild` inside a transient systemd slice with strict memory caps:

```bash
sudo systemd-run --unit=passly-rebuild --collect \
  -p MemoryHigh=400M -p MemoryMax=700M -p MemorySwapMax=4G -p CPUQuota=70% \
  nixos-rebuild switch --max-jobs 1 --cores 1
```

If memory consumption approaches the limit, the rebuild process will be throttled or swapped, ensuring that Nginx and SSH continue serving traffic uninterrupted.

---

## 4. Sample Declarative Configuration: Edge Reverse Proxy

The following sanitized module (`/etc/nixos/site.nix`) illustrates an edge reverse proxy terminating TLS and forwarding traffic to an internal Passly backend:

```nix
{ pkgs, config, ... }:

{
  networking.firewall.allowedTCPPorts = [ 80 443 ];

  security.acme = {
    acceptTerms = true;
    defaults.email = "admin@example.com";
  };

  services.nginx = {
    enable = true;
    recommendedGzipSettings = true;
    recommendedOptimisation = true;
    recommendedProxySettings = true;
    recommendedTlsSettings = true;

    virtualHosts."passly.example.com" = {
      enableACME = true;
      forceSSL = true;
      locations."/" = {
        proxyPass = "http://10.0.0.20"; # Internal private IP of Passly backend
        proxyWebsockets = true;
      };
    };

    # Drop connections to unknown Host headers or IP scanners
    virtualHosts."_" = {
      default = true;
      rejectSSL = true;
      extraConfig = "return 444;";
    };
  };
}
```

---

## 5. Summary & Related Guides

- For complete step-by-step setup on OCI Free Tier with standard Linux, see [OCI Always Free Tier Deployment Blueprint](oci-free-tier.md).
- For the Two-Tier DMZ architecture, consult Section 4 of `oci-free-tier.md`.
