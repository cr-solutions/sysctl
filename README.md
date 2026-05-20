# High-Performance Linux sysctl Configuration

Production-ready sysctl.conf optimized for high-performance servers and 100G+ networking environments. Based on NSA security guidelines, Linux network tuning best practices, and performance insights from Netflix/Brendan Gregg.

**Key optimizations:**
- Network security hardening (ICMP, redirects, source routing)
- TCP performance tuning for high-bandwidth, high-latency networks
- Memory management for containerized workloads
- Resource scaling (file descriptors, connection limits)
- DDoS mitigation and connection handling

## Quick Start

```bash
# Backup current configuration
sudo cp /etc/sysctl.conf /etc/sysctl.conf.backup

# Apply new configuration
sudo cp sysctl.conf /etc/sysctl.conf
sudo sysctl --system

# Verify settings
sysctl net.core.rmem_max
sysctl -a | grep tcp | head -10
```

## Performance Tuning by Network Speed

| Network | rmem_max/wmem_max | Use Case |
|---------|-------------------|----------|
| 1GbE    | 16MB             | Standard servers |
| 10GbE   | 32-56MB          | High-performance servers |
| 100GbE  | 2GB              | Ultra-high-speed networks |

## Critical Settings Explained

**Security Hardening:**
- `net.ipv4.tcp_syncookies=1` - SYN flood protection
- `net.ipv4.icmp_echo_ignore_broadcasts=1` - ICMP amplification protection
- `net.ipv4.conf.all.rp_filter=1` - Source validation (strict mode)

**High-Performance Networking:**
- `net.core.somaxconn=262144` - Large TCP accept queue for load balancers
- `net.ipv4.tcp_congestion_control=htcp` - Optimized for high-speed, long-distance networks
- `net.core.default_qdisc=fq` - Fair queueing for reduced latency
- `net.ipv4.tcp_tw_reuse=1` - Efficient TIME_WAIT socket reuse

**Memory Management:**
- `vm.swappiness=10` - Prefer RAM over swap
- `vm.overcommit_memory=1` - Container-friendly memory overcommit
- `vm.min_free_kbytes=1048576` - Keep 1GB free for emergencies

## Compatibility & Warnings

⚠️ **Test in staging before production deployment**

**Kernel Requirements:**
```bash
# Verify congestion control support
cat /proc/sys/net/ipv4/tcp_available_congestion_control

# Check qdisc support
tc qdisc show dev eth0
```

**Environment Considerations:**
- **Routers/BGP**: Set `rp_filter=2` or `rp_filter=0`
- **Memory-constrained systems**: Reduce `vm.min_free_kbytes` and buffer sizes
- **Ubuntu compatibility**: Some parameters may not exist in all kernel versions

## Custom Tuning Guidelines

**Buffer Sizing Formula:**
```
Optimal Buffer = Bandwidth × RTT
Example: 10Gbps × 100ms = 125MB
```

**Inotify Tuning:**
- `fs.inotify.max_user_watches=524288` - Supports IDEs, file watchers, containers, zero-downtime deployments, log/cache management, and process managers/microservices (default 8192 is too low)
- `fs.inotify.max_user_instances=512` - Allows more inotify instances per user (default 128)
- Safe for 4GB+ servers; each watch uses ~1KB of kernel memory (~512MB at max)
- Scaling formula: allocate ~10-15% of RAM for watches max → `watches = (RAM_in_GB × 0.10) × 1024 × 1024`

| RAM   | max_user_watches | max_user_instances | Max memory usage |
|-------|------------------|--------------------|------------------|
| 4GB   | 524288           | 512                | ~512MB           |
| 8GB   | 1048576          | 1024               | ~1GB             |
| 16GB  | 2097152          | 2048               | ~2GB             |

**Swap File Recommendation:**

With `vm.swappiness=10` the system rarely swaps, but a swap file is still recommended as a safety net for OOM situations.

| RAM   | Swap Size |
|-------|-----------|
| 4GB   | 4GB       |
| 8GB   | 4GB       |
| 16GB  | 8GB       |

Setup (example for 4GB swap):
```bash
# 1. Turn off the current swap (if it exists)
sudo swapoff -a

# 2. Create the swap file (4GB = 4096 MB)
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096

# 3. Set secure permissions
sudo chmod 600 /swapfile

# 4. Set up the file as Linux swap area
sudo mkswap /swapfile

# 5. Enable the swap file
sudo swapon /swapfile

# 6. Make it persistent across reboots
echo '/swapfile   swap    swap    defaults    0   0' | sudo tee -a /etc/fstab

# 7. Verify
free -h
```

**High-Traffic Servers:**
- Increase `netdev_max_backlog` and `tcp_max_syn_backlog`
- Ensure `tcp_rmem`/`tcp_wmem` max values match `rmem_max`/`wmem_max`
- Monitor with `ss -s` and adjust accordingly

## Validation & Testing

```bash
# Socket statistics
ss -s

# Network performance testing
iperf3 -c target_host -t 60 -P 4

# Memory monitoring
free -h && cat /proc/meminfo | grep -E "(MemFree|Buffers|Cached)"

# TCP settings verification
sysctl -a | grep -E "(rmem|wmem|tcp_congestion)"
```

## License

Mozilla Public License 2.0 - See [sysctl.conf](sysctl.conf) header for details.

## Contributing

Pull requests welcome. Include performance test results and rationale for changes.

---
**Author:** Ricardo Cescon | [CR-Solutions](https://cr-solutions.net) | [cescon.de](https://cescon.de)
