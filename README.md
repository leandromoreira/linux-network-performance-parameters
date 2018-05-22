# Introduction

Sometimes people are looking for [sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) cargo-culting values that bring high throughput and low latency with no trade-off and that works on every ocasion. That's not realistic, although we can say that the **newer kernel versions are very well tuned by default**. In fact you might [hurt performance if you mess with the defaults](https://medium.com/@duhroach/the-bandwidth-delay-problem-c6a2a578b211).

This brief tutorial was inspired by [the illustrated guide to Linux networking stack](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/). This one though, we aim to show **where some of the most used and quoted sysctl parameters are located into the Linux network flow**. We're going to "ignore" some optimizations like offloadings (TSO,GRO and etc) and other details for the sake of briviety.

> #### Feel free to send corrections and suggestions! :)

# Linux network queues overview

![linux network queues](/img/linux_network_flow.png "A graphic representation of linux/kernel network main buffer / queues")

# Fitting the sysctl variables into the Linux network flow

## Ingress - they're coming
1. Packets arrive at the NIC
1. NIC will DMA packets at RAM and references to them at receive ring buffer queue `rx` until `rx-usecs` timeout
1. NIC will raise a `hardIRQ`
1. CPU will run the `IRQ handler` that runs the driver's code
1. Driver will `schedule a NAPI`, clear the hardIRQ and return
1. Driver raise a `softIRQ (NET_RX_SOFTIRQ)`
1. NAPI will poll data from receive ring buffer until `netdev_budget_usecs` timeout or `netdev_budget` and `dev_weight` packets
1. Packets are handled to a qdisc sized `netdev_max_backlog` with its algorithm defined by `default_qdisc`
1. Packets are handled to IP and TCP systems
1. Delivered to the receive buffer and sized as `tcp_rmem`
    1. If `tcp_moderate_rcvbuf` is enabled kernel will auto tune the receive buffer
1. Application reads data

![tcp ingress flow](/img/tcp_ingress_flow.png "A graphic representation of tcp ingress flow")

## Egress - they're leaving
1. Application writes data at send buffer of `tcp_wmem` size
1. Packets are handled by socket subsystem (TCP, IP)
1. The socket subsystem feeds the output queue of `txqueuelen` length with its algorithm `default_qdisc`
1. The driver code enqueue the packets at the `ring buffer tx`
1. The driver will do a `softIRQ (NET_TX_SOFTIRQ)` after `tx-usecs` timeout
1. Re-enable hardIRQ to NIC
1. Driver will map all the packets (to be sent) to some DMA'ed region
1. NIC fetches the packets (via DMA) from RAM to transmit
1. After the transmission NIC will raise a `hardIRQ` to signal its completion
1. The driver will handle this IRQ (turn it off)
1. And schedule (`softIRQ`) the NAPI poll system 
1. NAPI will handle the receive packets signaling and free the RAM

# sysctl parameters - a short description

* receive/send `ring buffer rx,tx` - the driver receive/send queue a single or multiple queues with fixed size, usually implemented as FIFO, it is located at RAM and it only links to the real packets (skb_buff)
  * Check command: `ethtool -g interface`
  * Change command: `ethtool -G interface rx value tx value`
* `rx-usecs,tx-usecs` - number of microseconds to wait before raising a hardIRQ, from NIC perspective it'll DMA data packets until this timeout
  * Check command: `ethtool -c interface`
  * Change command: `ethtool -C interface rx-usecs value tx-usecs value`
* `netdev_budget_usecs` - Maximum number of microseconds in one NAPI polling cycle. Polling will exit when either netdev_budget_usecs have elapsed during the poll cycle or the number of packets processed reaches netdev_budget.
  * Check command: `sysctl net.core.netdev_budget_usecs`
  * Change command: `sysctl -w net.core.netdev_budget_usecs value`
* `netdev_budget` - Maximum number of packets taken from all interfaces in one polling cycle (NAPI poll). In one polling cycle interfaces which are registered to polling areprobed in a round-robin manner. Also, a polling cycle may not exceed netdev_budget_usecs microseconds, even if netdev_budget has not been exhausted.
  * Check command: `sysctl net.core.netdev_budget`
  * Change command: `sysctl -w net.core.netdev_budget value`
* `dev_weight` - The maximum number of packets that kernel can handle on a NAPI interrupt, it's a Per-CPU variable. For drivers that support LRO or GRO_HW, a hardware aggregated packet is counted as one packet in this
  * Check command: `sysctl net.core.dev_weight`
  * Change command: `sysctl -w net.core.dev_weight value`
* `netdev_max_backlog` - Maximum number  of  packets,  queued  on  the  INPUT  side, when the interface receives packets faster than kernel can process them.
  * Check command: `sysctl net.core.netdev_max_backlog`
  * Change command: `sysctl -w net.core.netdev_max_backlog value`
* `txqueuelen` - Maximum number of packets, queued on the OUTPUT side.
  * Check command: `ifconfig interface`
  * Change command: `ifconfig interface txqueuelen value`
* `default_qdisc` - The default queuing discipline to use for network devices.
  * Check command: `sysctl net.core.default_qdisc`
  * Change command: `sysctl -w net.core.default_qdisc value`
* `tcp_rmem` - min (size used under memory pressure), default (initial size), max (maximum size) - size of receive buffer used by TCP sockets.
  * Check command: `sysctl net.ipv4.tcp_rmem`
  * Change command: `sysctl -w net.ipv4.tcp_rmem="min default max"`
* `tcp_wmem` - min (size used under memory pressure), default (initial size), max (maximum size) - size of send buffer used by TCP sockets.
  * Check command: `sysctl net.ipv4.tcp_wmem`
  * Change command: `sysctl -w net.ipv4.tcp_wmem="min default max"`
* `tcp_moderate_rcvbuf` - If set, TCP performs receive buffer auto-tuning, attempting to automatically size the buffer.
  * Check command: `sysctl net.ipv4.tcp_moderate_rcvbuf`
  * Change command: `sysctl -w net.ipv4.tcp_moderate_rcvbuf value`

## Honorable mentions: (more related to TCP FSM queues and algorithms)
* `somaxconn` - Limit of socket listen() backlog, known in userspace as SOMAXCONN.
* `tcp_fin_timeout` - this specifies how many seconds to wait for a final FIN packet before the socket is forcibly closed.  This is strictly a violation of the TCP specification, but required to prevent denial-of-service attacks.
* `tcp_available_congestion_control` - shows the available congestion control choices that are registered.
* `tcp_congestion_control` - set the congestion control algorithm to be used for new connections.
* `tcp_max_syn_backlog` - the maximum number of queued connection requests which have still not received an acknowledgement from the connecting client.  If this number is exceeded, the kernel will begin dropping requests.
* `tcp_slow_start_after_idle` - enable/disable tcp slow start

# References

* https://www.kernel.org/doc/Documentation/sysctl/net.txt
* https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
* https://wiki.mikejung.biz/OS_Tuning
* https://www.coverfire.com/articles/queueing-in-the-linux-network-stack/
* https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/
* http://vger.kernel.org/~davem/skb.html
* https://www.missoulapubliclibrary.org/ftp/LinuxJournal/LJ13-07.pdf
* https://opensourceforu.com/2016/10/network-performance-monitoring/
* http://man7.org/linux/man-pages/man7/tcp.7.html
* https://www.yumpu.com/en/document/view/55400902/an-adventure-of-analysis-and-optimisation-of-the-linux-networking-stack
* https://lwn.net/Articles/616241/
* http://man7.org/linux/man-pages/man8/tc.8.html
* https://medium.com/@duhroach/tools-to-profile-networking-performance-3141870d5233
* https://www.lmax.com/blog/staff-blogs/2016/05/06/navigating-linux-kernel-network-stack-receive-path/
* https://es.net/host-tuning/100g-tuning/
* http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm
* http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html
* https://people.cs.clemson.edu/~westall/853/tcpperf.pdf
* http://tldp.org/HOWTO/Traffic-Control-HOWTO/classless-qdiscs.html
* https://es.net/assets/Papers-and-Publications/100G-Tuning-TechEx2016.tierney.pdf
* https://www.kernel.org/doc/Documentation/networking/scaling.txt
* https://netdevconf.org/1.2/papers/bbr-netdev-1.2.new.new.pdf
* http://www.ece.virginia.edu/cheetah/documents/papers/TCPlinux.pdf
* https://www.kernel.org/doc/ols/2009/ols2009-pages-169-184.pdf
* https://devcentral.f5.com/articles/the-send-buffer-in-depth-21845
* http://packetbomb.com/understanding-throughput-and-tcp-windows/
* https://www.speedguide.net/bdp.php
* https://www.switch.ch/network/tools/tcp_throughput/
* https://www.ibm.com/support/knowledgecenter/en/SSQPD3_2.6.0/com.ibm.wllm.doc/usingethtoolrates.html
