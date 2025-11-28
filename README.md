# CR-Solutions sysctl tuning

This repository contains a curated sysctl.conf tuned for server workloads and high-performance networking (including 100G with higher RTTs). The settings are a consolidation of recommendations from NSA security guidance, Linux network tuning sources (2013+), and performance tuning guidance from experts like Brendan Gregg / Netflix.

The included sysctl.conf focuses on:
- Basic networking hardening (ICMP protections, rp_filter, disabling redirects)
- Defensive defaults (syncookies, martian logging)
- TCP performance and buffer tuning for high-bandwidth, high-latency paths
- Resource limits (file descriptors, PID limits, shared memory)
- Memory and VM behavior suitable for servers and containerized workloads

Source sysctl file: https://github.com/cr-solutions/sysctl/blob/c9c0459e2cb6de51a1a83eb251ad1affb60bbf75/sysctl.conf

## Quick apply (example)

1. Review the file and adjust values that are environment-specific (notes below).
2. Backup your current sysctl.conf:
   sudo cp /etc/sysctl.conf /etc/sysctl.conf.bak
3. Copy this file to /etc and reload:
   sudo cp sysctl.conf /etc/sysctl.conf
   sudo sysctl --system
   or
   sudo sysctl -p /etc/sysctl.conf

To check a single value:
   sysctl net.core.rmem_max

To show all TCP-related settings:
   sysctl -a | grep tcp

## Key settings and rationale

- net.ipv4.icmp_echo_ignore_broadcasts = 1  
  Protects against some ICMP-based amplification attacks.

- net.ipv4.tcp_syncookies = 1  
  Helps mitigate SYN flood attacks.

- net.core.somaxconn = 262144  
  Allows large TCP accept queue, useful for high-connection-rate services / LBs.

- net.core.rmem_max / wmem_max and net.ipv4.tcp_rmem / tcp_wmem  
  Increased receive/send buffers to support very high bandwidth-delay product (BDP) links.
  Default values here are tuned for very high throughput (10G–100G class). Adjust to your NIC, memory, and expected RTTs.

- net.ipv4.tcp_congestion_control = htcp  
  HTCP recommended here for high-speed, long-fat networks. Validate kernel support on your systems.

- net.core.default_qdisc = fq  
  fq (Fair Queueing) helps reduce latency and improve fairness for modern workloads.

- vm.swappiness = 10, vm.vfs_cache_pressure = 50  
  Suggests preference for keeping application memory and inode/dentry caches appropriately.

- vm.overcommit_memory = 1, vm.oom_kill_allocating_task = 1  
  Helpful in container environments—allows overcommit and kills the allocating process on OOM.

- kernel.shmmax / shmall  
  Shared memory limits (512M in the file). Increase if your workloads (databases, IPC) require more.

- net.ipv4.tcp_tw_reuse = 1 and tcp_fin_timeout=10  
  Help reduce TIME_WAIT pressure on busy servers; tcp_tw_recycle is explicitly disabled due to NAT/load-balancer problems.

## Warnings and compatibility

- These settings are opinionated and intended for servers or network devices where you can control the environment. Test in staging before production.
- Some tunables are only appropriate for hosts (not routers) — e.g., rp_filter. On routers or BGP environments, rp_filter may need to be 2 or disabled.
- tcp_congestion_control and qdisc require kernel support. Verify available congestion control algorithms and qdiscs on your kernel:
  cat /proc/sys/net/ipv4/tcp_available_congestion_control
  tc qdisc show dev <interface>
- Very large buffer values consume memory; ensure your system has adequate RAM before applying maximums.
- vm.min_free_kbytes is set to a large value (1GB) here — be cautious on memory-constrained systems.

## Tuning guidance / suggestions

- For 1GbE, lower extremes are usually appropriate (e.g., rmem_max/wmem_max ~ 16MB).
- For 10GbE, consider rmem_max/wmem_max 32–56MB.
- For 100GbE with high RTTs, larger values may be required (and are set in this file).
- Increase net.core.netdev_max_backlog and tcp_max_syn_backlog for high packet/connection bursts.
- If you change buffer sizes, review tcp_rmem and tcp_wmem min/default/max triplets to ensure they match rmem_max/wmem_max.

## Testing & verification

After applying:
- Use `ss -s` and `ss -t` to inspect socket states and tune effects.
- Measure throughput with iperf3 across expected network paths and RTTs.
- Monitor system memory and swap usage; ensure no undesirable memory pressure is introduced.

## License

This repository and the included sysctl.conf are distributed under the Mozilla Public License 2.0. See the header comments in sysctl.conf for more details.

## Contributing

Contributions, improvements, and pull requests are welcome. If you submit changes, include rationale and test results where applicable.

## Author / Contact

CR-Solutions / Ricardo Cescon  
https://cr-solutions.net
