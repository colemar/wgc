# ![WireGuard](images/wireguard_logo.png) wgc - WireGuard Connection Manager

![Made with Bash](https://img.shields.io/badge/Made%20with-Bash-1f425f.svg)

**[🇬🇧 English](README.md) | [🇮🇹 Italiano](README.it.md)**

> Run and monitor multiple isolated WireGuard tunnels using Linux network namespaces.

`wgc` is a bash script for managing multiple, simultaneous WireGuard connections on a Linux system. Its core feature is the ability to run VPNs either in **isolated network namespaces** (`ip netns`) or in the default namespace with policy-based routing.

Each VPN connection can be brought up in an isolated namespace with its own network interface, routing table, and DNS configuration, or in the default namespace using source-based routing rules. This allows multiple VPNs to be active concurrently without route conflicts.

---

## Features

* **Dual Operation Modes:** 
  * **Namespace mode (`nup`):** Complete isolation with dedicated namespace, perfect for running specific applications through the VPN.
  * **Default mode (`up`):** Uses the system's default namespace with policy-based routing, ideal for system-wide VPN usage.
* **Total Isolation:** Run multiple VPNs simultaneously in namespace mode. Each VPN's traffic is completely separate from the host and other VPNs.
* **Targeted Execution:** Run specific commands or applications (like `curl`, `ssh`, `bash`, a browser, a service) *inside* a VPN namespace. This routes only that application's traffic through the tunnel.
* **Automatic DNS:** Automatically configures DNS servers specified in the `.conf` file (via the `DNS =` key).
* **Smart Process Management:** Graceful termination of processes running in namespaces with timeout and progress indication.
* **Flexible Stopping:** Option to force-stop VPNs even when processes are running in their namespace.
* **Simple Interface:** A single script with clear commands to start, stop, list, and monitor tunnels.

## Requirements

* `bash`
* `sudo` access (the script auto-elevates if not run as root) 
* `wireguard-tools` (provides the `wg` command) 
* `iproute2` (provides the `ip` command)
* `openresolv` (provides the `resolvconf` command for **exclusive** DNS management to prevent leaks)

## Installation

1. Download file [wgc](https://github.com/colemar/wgc/raw/refs/heads/main/wgc)

2. Make the script executable:
   
   ```bash
   chmod +x wgc
   ```

3. Move the script to a directory in your `$PATH`, such as `/usr/local/bin`:
   
   ```bash
   sudo cp wgc /usr/local/bin/wgc
   ```

## Configuration

Your WireGuard configuration files (`.conf`) must be placed in `/etc/wireguard/` or in another directory pointed to by the `WGC_CFGDIR` environment variable.

```bash
export WGC_CFGDIR=/path/to/your/configs
wgc list
```

The script uses the filename (without the `.conf` extension) as the VPN identifier. For example, a file at `$WGC_CFGDIR/proton-it.conf` will be managed as the `proton-it` VPN.

**Note:** Config files named with the pattern `wg[0-9]+` (e.g., `wg0.conf`, `wg1.conf`) are excluded from management to avoid conflicts with system interfaces.

The script parses the file and expects standard WireGuard keys:

* **Required Keys:** The script will exit if any of these is missing:
  * `Address` 
  * `PrivateKey` 
  * `PublicKey` 
  * `Endpoint` 
  * `AllowedIPs` 
* **Optional Keys:** The script also supports:
  * `ListenPort`
  * `DNS`
  * `MTU` 
  * `PresharedKey` 
  * `PersistentKeepalive` 

## DNS Configuration & Requirements

To ensure **zero DNS leaks**, `wgc` relies on the `openresolv` implementation of `resolvconf` (specifically its `-x` exclusive flag).

**Check your installation:**
Run `resolvconf -h`. If the output mentions `-x ... Mark the interface as exclusive`, you are good to go.
If the output says "This is a compatibility alias for...", you are using the systemd wrapper, which is **not supported**.

### Installing openresolv

**Debian / Ubuntu 22.04 and older:**

```bash
sudo apt update && sudo apt install openresolv
```

**Ubuntu 24.04+ (Noble) and newer:** Canonical has removed `openresolv` from the official repositories. You must install it manually:

1. Download the package (from Debian or older Ubuntu archives):
   
   ```bash
   wget http://ftp.debian.org/debian/pool/main/o/openresolv/openresolv_3.13.2-1_all.deb
   ```

2. Install it (ignoring conflicts with systemd-resolved):
   
   ```bash
   sudo dpkg -i --force-conflicts openresolv_*.deb
   ```

3. Configure it to use systemd as the fallback DNS (to keep internet working when VPN is off):
   
   ```bash
   echo "name_servers=127.0.0.53" | sudo tee -a /etc/resolvconf.conf
   sudo resolvconf -u
   ```

---

## Usage

The general syntax is `wgc [command] <vpn_name>`.

The script requires `sudo` or root access because it manipulates network interfaces and namespaces.

  ![](images/wgc.png)

### Main Commands

* **`up <vpn>`**
  Starts the VPN connection in the **default namespace** using policy-based routing. The VPN's routes are applied based on the source address, allowing the VPN to coexist with your normal network connection.
  
  This means that an application (e.g. qBittorrent) must bind to the VPN interface or ip address to have its network traffic redirected through the tunnel.
  
  This also means that if you are connected via ssh, your session will not close when you start the VPN.
  
  **Important (DNS):** To prevent privacy leaks, **DNS resolution is switched globally** to the VPN's DNS server. This means that even applications using your normal internet connection (not bound to the VPN) will rely on the active VPN tunnel to resolve domain names.
  
  ```bash
  wgc up proton-it
  ```
  
  ![](images/start.png)

* **`upd <vpn>`**
  Starts the VPN connection in the **default namespace** using policy-based routing **and** adding a ***split route*** for the tunnel (0.0.0.0/1, 128.0.0.0/1).
  
  This means that any application will have its network traffic routed through the VPN tunnel.
  
  This also means that if you are connected via ssh your session will be terminated, unless before starting the VPN you manually add a specific route to your ip address.
  
  ```bash
  wgc upd proton-it
  ```

* **`nup <vpn>`**
  Starts the VPN connection in its **own isolated namespace**. This provides complete network isolation and is the recommended mode for running specific applications through the VPN.
  
  A default route (0.0.0.0/0) will be added for the VPN tunnel, so any application will have its network traffic routed through it.
  
  **Note:** Applications with network listeners (e.g., web interfaces on TCP ports) will be unreachable from outside the VPN namespace unless you set up port forwarding (built in support coming soon) or a *veth* (virtual ethernet) connecting the default namespace with the VPN namespace.  Applications with GUI or terminal interfaces remain fully accessible.
  
  ```bash
  wgc nup proton-it
  ```
  
  ![](images/nup.png)

* **`down <vpn> [force]`**
  Stops the VPN connection.
  
  * If the VPN is active in the default namespace (started with `up/upd`), stops it.
  * If the VPN is active in its own namespace (started with `nup`):
    * If no processes are running in the same namespace, stops the VPN.
    * If some processes are running in the same namespace, shows the process list and refuses to stop the VPN.  If you specify `force`, gracefully terminates all processes in the namespace (SIGTERM), waits up to 10 seconds with a progress bar, then forcibly kills remaining processes (SIGKILL). Finally, stops the VPN.
  
  ```bash
  wgc down proton-it
  wgc down proton-it force
  ```
  
  ![](images/stop.png)

* **`status <vpn>`**
  Shows the detailed status of the connection, including:
  
  * Connection state and namespace
  * WireGuard interface details
  * IP addresses and routes
  * DNS configuration
  * Running processes (for namespace mode)
  
  ```bash
  wgc status proton-it
  ```
  
  ![](images/status.png)

* **`exec <vpn> <command...>`**
  Executes a command *inside* the VPN's namespace. This only works for VPNs started with `nup`.
  
    *Example: Check your public IP as seen by the VPN.*
  
  ```bash
  wgc exec proton-it curl ipinfo.io
  ```
  
    *Example: Start an interactive shell that only sees the VPN network.*
  
  ```bash
  wgc exec proton-it bash
  ```
  
  ![](images/exec.png)

* **`list`**
  Lists all available `.conf` files found in config directory with their key configuration details (Address, AllowedIPs, Endpoint). 
  
  ```bash
  wgc list
  ```
  
  ![](images/list.png)

* **`active`**
  Lists all *currently active* VPNs, showing both those in the default namespace and those in isolated namespaces.
  
  ```bash
  wgc active
  ```
  
  ![](images/active.png)

### Command Shortcuts

Some commands support prefix matching. For example:

* `nup`, `nu`, `n` → `nup`
* `down`, `dow`, `do`, `d` → `down`
* `status`, `stat`, `st`, `s` → `status`
* `exec`, `exe`, `ex`, `e` → `exec`
* `active`, `activ`, `act`, `ac`, `a` → `active`
* `list`, `lis`, `li`, `l` → `list`

### Bash Completion

The script can install its own bash completion file with intelligent suggestions:

1. Run the following command:
   
   ```bash
   wgc completion
   ```

2. This will create the file `/etc/bash_completion.d/wgc`. 

3. Source the file or start a new shell to use the completion:
   
   ```bash
   source /etc/bash_completion.d/wgc
   ```

4. The completion provides:
   
   * Command name completion
   * VPN name completion based on available config files (for `up`/`nup`)
   * Active VPN completion (for `down`/`status`/`exec`)
   * System command completion after `exec <vpn>` (Note: Issuing a double TAB without providing a command prefix to narrow the search results in a significant delay)

5. To avoid sudo password prompts during completion, the installer provides optional `sudoers` rules.

6. If `WGC_CFGDIR` is changed, the `completion` command must be run again to update its knowledge of the config dir path.

## Operation Modes Comparison

| Feature           | Default Namespace (`up`)              | Isolated Namespace (`nup`)             |
| ----------------- | ------------------------------------- | -------------------------------------- |
| Network isolation | Partial (routing rules).              | Complete.                              |
| DNS configuration | System-wide.                          | Namespace-specific.                    |
| Process execution | Direct.                               | Via `wgc exec`.                        |
| Multiple VPNs     | Possible, with distinct ip addresses. | Simple and clean.                      |
| Use case          | System-wide VPN.                      | Application-specific VPN.              |
| **DNS scope**     | **System-wide** (Overrides host DNS). | **Isolated** (Affects only namespace). |

## Examples

### Running a browser through VPN

```bash
# Start VPN in namespace
wgc nup proton-it

# Launch Firefox in the VPN namespace
wgc exec proton-it firefox

# When done, stop the VPN (will ask to force if Firefox is running)
wgc down proton-it
```

### Checking VPN connectivity

```bash
# Your real IP
curl ipinfo.io

# IP as seen through the VPN, if started with 'up'
curl --interface proton-it ipinfo.io
curl --interface 10.2.0.2 ipinfo.io

# IP as seen through the VPN, if started with 'nup'
wgc exec proton-it curl ipinfo.io
```

### Multiple simultaneous VPNs

```bash
# Start multiple VPNs in their own namespaces
wgc nup vpn1
wgc nup vpn2
wgc nup vpn3

# List all active VPNs
wgc active

# Use different VPNs for different applications
wgc exec vpn1 firefox
wgc exec vpn2 bash # then whatever command suits you
```

## Troubleshooting

### Custom configuration directory

If you want to use a configuration directory other than `/etc/wireguard/`, set the `WGC_CFGDIR` environment variable:

```bash
export WGC_CFGDIR=$HOME/.config/wireguard
wgc list
```

Make sure the directory:

- Exists and is readable
- Contains your `.conf` files with appropriate permissions
- Has the same access rights as `/etc/wireguard/` would have

**Important:** When using `sudo`, environment variables are not always preserved. You can either:

- Add `WGC_CFGDIR` to your sudoers configuration
- Or the script will automatically pass it through when elevating privileges

### Config file parsing

The script validates configuration files and reports:

* Missing required keys
* Unknown sections or keys (as warnings)
* Malformed lines
* Invalid DNS server addresses

### Process management

When stopping a namespace VPN with running processes:

1. First attempt shows the list of processes
2. Use `force` parameter to terminate them
3. Script tries graceful termination (SIGTERM)
4. Progress bar shows 10-second timeout
5. Remaining processes are forcibly killed (SIGKILL)

### DNS issues

* **DNS Leaks (Default Mode):** When running in the default namespace (`up`/`upd`), `wgc` requires `openresolv` to guarantee exclusive DNS priority. Standard `systemd-resolved` wrappers usually lack the true exclusive mode (`-x`), leading to DNS leaks. **This requirement does not apply to namespace mode (`nup`).**
* **DNS in Namespace Mode:** DNS is configured directly in `/etc/netns/<vpn_name>/resolv.conf` without external tools.
* **DNS in Default Mode:** DNS is managed via `resolvconf -x`, which registers the VPN DNS with **exclusive priority**. Depending on your system configuration, this effectively overrides other DNS settings either by communicating with the system resolver (e.g., `systemd-resolved`) or by updating `/etc/resolv.conf` directly.
* Malformed DNS addresses in `.conf` files are detected and skipped with warnings.

## License

This project is licensed under the GNU General Public License v3.0 (GPL-3.0).
See the [LICENSE](LICENSE) file for details.
