# TOC

* [Introduction](#introduction)
* [Linux network queues overview](#linux-network-queues-overview)
* [Fitting the sysctl variables into the Linux network flow](#fitting-the-sysctl-variables-into-the-linux-network-flow)
  * Ingress - they're coming
  * Egress - they're leaving
* [What, Why and How - network and sysctl parameters](#what-why-and-how---network-and-sysctl-parameters)
  * Ring Buffer - rx,tx
  * Interrupt Coalescence (IC) - rx-usecs, tx-usecs, rx-frames, tx-frames (hardware IRQ)
  * Interrupt Coalescing (soft IRQ) and Ingress QDisc
  * Egress QDisc - txqueuelen and default_qdisc
  * TCP Read and Write Buffers/Queues
  * Honorable mentions - TCP FSM and congestion algorithm
* [Network tools](#network-tools-for-testing-and-monitoring)
* [References](#references)

# Introduction

Sometimes people are looking for [sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) cargo cult values that bring high throughput and low latency with no trade-off and that works on every occasion. That's not realistic, although we can say that the **newer kernel versions are very well tuned by default**. In fact, you might [hurt performance if you mess with the defaults](https://medium.com/@duhroach/the-bandwidth-delay-problem-c6a2a578b211).

This brief tutorial shows **where some of the most used and quoted sysctl/network parameters are located into the Linux network flow**, it was heavily inspired by [the illustrated guide to Linux networking stack](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/) and many of [Marek Majkowski's posts](https://blog.cloudflare.com/how-to-achieve-low-latency/). 

> #### Feel free to send corrections and suggestions! :)

# Linux network queues overview

![linux network queues](/img/linux_network_flow.png "A graphic representation of linux/kernel network main buffer / queues")

# Fitting the sysctl variables into the Linux network flow

## Ingress - they're coming
1. Packets arrive at the NIC
1. NIC will verify `MAC` (if not on promiscuous mode) and `FCS` and decide to drop or to continue
1. NIC will [DMA packets at RAM](https://en.wikipedia.org/wiki/Direct_memory_access), in a region previously prepared (mapped) by the driver
1. NIC will enqueue references to the packets at receive [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer) queue `rx` until `rx-usecs` timeout or `rx-frames`
1. NIC will raise a `hard IRQ`
1. CPU will run the `IRQ handler` that runs the driver's code
1. Driver will `schedule a NAPI`, clear the `hard IRQ` and return
1. Driver raise a `soft IRQ (NET_RX_SOFTIRQ)`
1. NAPI will poll data from the receive ring buffer until `netdev_budget_usecs` timeout or `netdev_budget` and `dev_weight` packets
1. Linux will also allocate memory to `sk_buff`
1. Linux fills the metadata: protocol, interface, setmacheader, removes ethernet
1. Linux will pass the skb to the kernel stack (`netif_receive_skb`)
1. It will set the network header, clone `skb` to taps (i.e. tcpdump) and pass it to tc ingress
1. Packets are handled to a qdisc sized `netdev_max_backlog` with its algorithm defined by `default_qdisc`
1. It calls `ip_rcv` and packets are handled to IP
1. It calls netfilter (`PREROUTING`)
1. It looks at the routing table, if forwarding or local
1. If it's local it calls netfilter (`LOCAL_IN`)
1. It calls the L4 protocol (for instance `tcp_v4_rcv`)
1. It finds the right socket
1. It goes to the tcp finite state machine
1. Enqueue the packet to  the receive buffer and sized as `tcp_rmem` rules
    1. If `tcp_moderate_rcvbuf` is enabled kernel will auto-tune the receive buffer
1. Kernel will signalize that there is data available to apps (epoll or any polling system)
1. Application wakes up and reads the data

## Egress - they're leaving
1. Application sends message (`sendmsg` or other)
1. TCP send message allocates skb_buff
1. It enqueues skb to the socket write buffer of `tcp_wmem` size
1. Builds the TCP header (src and dst port, checksum)
1. Calls L3 handler (in this case `ipv4` on `tcp_write_xmit` and `tcp_transmit_skb`)
1. L3 (`ip_queue_xmit`) does its work: build ip header and call netfilter (`LOCAL_OUT`)
1. Calls output route action
1. Calls netfilter (`POST_ROUTING`)
1. Fragment the packet (`ip_output`)
1. Calls L2 send function (`dev_queue_xmit`)
1. Feeds the output (QDisc) queue of `txqueuelen` length with its algorithm `default_qdisc`
1. The driver code enqueue the packets at the `ring buffer tx`
1. The driver will do a `soft IRQ (NET_TX_SOFTIRQ)` after `tx-usecs` timeout or `tx-frames`
1. Re-enable hard IRQ to NIC
1. Driver will map all the packets (to be sent) to some DMA'ed region
1. NIC fetches the packets (via DMA) from RAM to transmit
1. After the transmission NIC will raise a `hard IRQ` to signal its completion
1. The driver will handle this IRQ (turn it off)
1. And schedule (`soft IRQ`) the NAPI poll system 
1. NAPI will handle the receive packets signaling and free the RAM

# What, Why and How - network and sysctl parameters

## Ring Buffer - rx,tx
* **What** - the driver receive/send queue a single or multiple queues with a fixed size, usually implemented as FIFO, it is located at RAM
* **Why** - buffer to smoothly accept bursts of connections without dropping them, you might need to increase these queues when you see drops or overrun, aka there are more packets coming than the kernel is able to consume them, the side effect might be increased latency.
* **How:**
  * **Check command:** `ethtool -g ethX`
  * **Change command:** `ethtool -G ethX rx value tx value`
  * **How to monitor:** `ethtool -S ethX | grep -e "err" -e "drop" -e "over" -e "miss" -e "timeout" -e "reset" -e "restar" -e "collis" -e "over" | grep -v "\: 0"`
 
## Interrupt Coalescence (IC) - rx-usecs, tx-usecs, rx-frames, tx-frames (hardware IRQ)
* **What** - number of microseconds/frames to wait before raising a hardIRQ, from the NIC perspective it'll DMA data packets until this timeout/number of frames
* **Why** - reduce CPUs usage, hard IRQ, might increase throughput at cost of latency.
* **How:**
  * **Check command:** `ethtool -c ethX`
  * **Change command:** `ethtool -C ethX rx-usecs value tx-usecs value`
  * **How to monitor:** `cat /proc/interrupts` 
  
## Interrupt Coalescing (soft IRQ) and Ingress QDisc
* **What** - maximum number of microseconds in one [NAPI](https://en.wikipedia.org/wiki/New_API) polling cycle. Polling will exit when either `netdev_budget_usecs` have elapsed during the poll cycle or the number of packets processed reaches  `netdev_budget`.
* **Why** - instead of reacting to tons of softIRQ, the driver keeps polling data; keep an eye on `dropped` (# of packets that were dropped because `netdev_max_backlog` was exceeded) and `squeezed` (# of times ksoftirq ran out of `netdev_budget` or time slice with work remaining).
* **How:**
  * **Check command:** `sysctl net.core.netdev_budget_usecs`
  * **Change command:** `sysctl -w net.core.netdev_budget_usecs value`
  * **How to monitor:** `cat /proc/net/softnet_stat`; or a [better tool](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **What** - `netdev_budget` is the maximum number of packets taken from all interfaces in one polling cycle (NAPI poll). In one polling cycle interfaces which are registered to polling are probed in a round-robin manner. Also, a polling cycle may not exceed `netdev_budget_usecs` microseconds, even if `netdev_budget` has not been exhausted.
* **How:**
  * **Check command:** `sysctl net.core.netdev_budget`
  * **Change command:** `sysctl -w net.core.netdev_budget value`
  * **How to monitor:** `cat /proc/net/softnet_stat`; or a [better tool](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **What** - `dev_weight` is the maximum number of packets that kernel can handle on a NAPI interrupt, it's a Per-CPU variable. For drivers that support LRO or GRO_HW, a hardware aggregated packet is counted as one packet in this.
* **How:**
  * **Check command:** `sysctl net.core.dev_weight`
  * **Change command:** `sysctl -w net.core.dev_weight value`
  * **How to monitor:** `cat /proc/net/softnet_stat`; or a [better tool](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **What** - `netdev_max_backlog` is the maximum number  of  packets,  queued  on  the  INPUT side (_the ingress qdisc_), when the interface receives packets faster than kernel can process them.
* **How:**
  * **Check command:** `sysctl net.core.netdev_max_backlog`
  * **Change command:** `sysctl -w net.core.netdev_max_backlog value`
  * **How to monitor:** `cat /proc/net/softnet_stat`; or a [better tool](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
  
## Egress QDisc - txqueuelen and default_qdisc
* **What** - `txqueuelen` is the maximum number of packets, queued on the OUTPUT side.
* **Why** - a buffer/queue to face connection burst and also to apply [tc (traffic control).](http://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html)
* **How:**
  * **Check command:** `ifconfig ethX`
  * **Change command:** `ifconfig ethX txqueuelen value`
  * **How to monitor:** `ip -s link` 
* **What** - `default_qdisc` is the default queuing discipline to use for network devices.
* **Why** - each application has different load and need to traffic control and it is used also to fight against [bufferbloat](https://www.bufferbloat.net/projects/codel/wiki/)
* **How:**
  * **Check command:** `sysctl net.core.default_qdisc`
  * **Change command:** `sysctl -w net.core.default_qdisc value`
  * **How to monitor:**   `tc -s qdisc ls dev ethX`

## TCP Read and Write Buffers/Queues

> The policy that defines what is [memory pressure](https://wwwx.cs.unc.edu/~sparkst/howto/network_tuning.php) is specified at tcp_mem and tcp_moderate_rcvbuf.

* **What** - `tcp_rmem` - min (size used under memory pressure), default (initial size), max (maximum size) - size of receive buffer used by TCP sockets.
* **Why** - the application buffer/queue to the write/send data, [understand its consequences can help a lot](https://blog.cloudflare.com/the-story-of-one-latency-spike/).
* **How:**
  * **Check command:** `sysctl net.ipv4.tcp_rmem`
  * **Change command:** `sysctl -w net.ipv4.tcp_rmem="min default max"`; when changing default value, remember to restart your user space app (i.e. your web server, nginx, etc)
  * **How to monitor:** `cat /proc/net/sockstat`
* **What** - `tcp_wmem` - min (size used under memory pressure), default (initial size), max (maximum size) - size of send buffer used by TCP sockets.
* **How:**
  * **Check command:** `sysctl net.ipv4.tcp_wmem`
  * **Change command:** `sysctl -w net.ipv4.tcp_wmem="min default max"`; when changing default value, remember to restart your user space app (i.e. your web server, nginx, etc)
  * **How to monitor:** `cat /proc/net/sockstat`
* **What** `tcp_moderate_rcvbuf` - If set, TCP performs receive buffer auto-tuning, attempting to automatically size the buffer.
* **How:**
  * **Check command:** `sysctl net.ipv4.tcp_moderate_rcvbuf`
  * **Change command:** `sysctl -w net.ipv4.tcp_moderate_rcvbuf value`
  * **How to monitor:** `cat /proc/net/sockstat`

## Honorable mentions - TCP FSM and congestion algorithm

> Accept and SYN Queues are governed by net.core.somaxconn and net.ipv4.tcp_max_syn_backlog. [Nowadays net.core.somaxconn caps both queue sizes.](https://blog.cloudflare.com/syn-packet-handling-in-the-wild/#queuesizelimits)

* `sysctl net.core.somaxconn` - provides an upper limit on the value of the backlog parameter passed to the [`listen()` function](https://eklitzke.org/how-tcp-sockets-work), known in userspace as `SOMAXCONN`. If you change this value, you should also change your application to a compatible value (i.e. [nginx backlog](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen)).
* `cat /proc/sys/net/ipv4/tcp_fin_timeout` - this specifies the number of seconds to wait for a final FIN packet before the socket is forcibly closed.  This is strictly a violation of the TCP specification but required to prevent denial-of-service attacks.
* `cat /proc/sys/net/ipv4/tcp_available_congestion_control` - shows the available congestion control choices that are registered.
* `cat /proc/sys/net/ipv4/tcp_congestion_control` - sets the congestion control algorithm to be used for new connections.
* `cat /proc/sys/net/ipv4/tcp_max_syn_backlog` - sets the maximum number of queued connection requests which have still not received an acknowledgment from the connecting client; if this number is exceeded, the kernel will begin dropping requests.
* `cat /proc/sys/net/ipv4/tcp_syncookies` - enables/disables [syn cookies](https://en.wikipedia.org/wiki/SYN_cookies), useful for protecting against [syn flood attacks](https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/).
* `cat /proc/sys/net/ipv4/tcp_slow_start_after_idle` - enables/disables tcp slow start.

**How to monitor:** 
* `netstat -atn | awk '/tcp/ {print $6}' | sort | uniq -c` - summary by state
* `ss -neopt state time-wait | wc -l` - counters by a specific state: `established`, `syn-sent`, `syn-recv`, `fin-wait-1`, `fin-wait-2`, `time-wait`, `closed`, `close-wait`, `last-ack`, `listening`, `closing`
* `netstat -st` - tcp stats summary
* `nstat -a` - human-friendly tcp stats summary
* `cat /proc/net/sockstat` - summarized socket stats
* `cat /proc/net/tcp` - detailed stats, see each field meaning at the [kernel docs](https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt)
* `cat /proc/net/netstat` - `ListenOverflows` and `ListenDrops` are important fields to keep an eye on
  * `cat /proc/net/netstat | awk '(f==0) { i=1; while ( i<=NF) {n[i] = $i; i++ }; f=1; next} \
(f==1){ i=2; while ( i<=NF){ printf "%s = %d\n", n[i], $i; i++}; f=0} ' | grep -v "= 0`; a [human readable `/proc/net/netstat`](https://sa-chernomor.livejournal.com/9858.html)

![tcp finite state machine](https://upload.wikimedia.org/wikipedia/commons/a/a2/Tcp_state_diagram_fixed.svg "A graphic representation of tcp tcp finite state machine")
Source: https://commons.wikimedia.org/wiki/File:Tcp_state_diagram_fixed_new.svg

# Network tools for testing and monitoring

* [iperf3](https://iperf.fr/) - network throughput
* [vegeta](https://github.com/tsenart/vegeta) - HTTP load testing tool
* [netdata](https://github.com/firehol/netdata) - system for distributed real-time performance and health monitoring

# References

* https://www.kernel.org/doc/Documentation/sysctl/net.txt
* https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
* https://www.kernel.org/doc/Documentation/networking/scaling.txt
* https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt
* https://www.kernel.org/doc/Documentation/networking/multiqueue.txt
* http://man7.org/linux/man-pages/man7/tcp.7.html
* http://man7.org/linux/man-pages/man8/tc.8.html
* http://www.ece.virginia.edu/cheetah/documents/papers/TCPlinux.pdf
* https://netdevconf.org/1.2/papers/bbr-netdev-1.2.new.new.pdf
* https://blog.cloudflare.com/how-to-receive-a-million-packets/
* https://blog.cloudflare.com/how-to-achieve-low-latency/
* https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/
* https://www.youtube.com/watch?v=6Fl1rsxk4JQ
* https://oxnz.github.io/2016/05/03/performance-tuning-networking/
* https://www.intel.com/content/dam/www/public/us/en/documents/reference-guides/xl710-x710-performance-tuning-linux-guide.pdf
* https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf
* https://medium.com/@matteocroce/linux-and-freebsd-networking-cbadcdb15ddd
* https://blogs.technet.microsoft.com/networking/2009/08/12/where-do-resets-come-from-no-the-stork-does-not-bring-them/
* https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/multi-core-processor-based-linux-paper.pdf
* http://syuu.dokukino.com/2013/05/linux-kernel-features-for-high-speed.html
* https://www.bufferbloat.net/projects/codel/wiki/Best_practices_for_benchmarking_Codel_and_FQ_Codel/
* https://software.intel.com/en-us/articles/setting-up-intel-ethernet-flow-director
* https://courses.engr.illinois.edu/cs423/sp2014/Lectures/LinuxDriver.pdf
* https://www.coverfire.com/articles/queueing-in-the-linux-network-stack/
* http://vger.kernel.org/~davem/skb.html
* https://www.missoulapubliclibrary.org/ftp/LinuxJournal/LJ13-07.pdf
* https://opensourceforu.com/2016/10/network-performance-monitoring/
* https://www.yumpu.com/en/document/view/55400902/an-adventure-of-analysis-and-optimisation-of-the-linux-networking-stack
* https://lwn.net/Articles/616241/
* https://medium.com/@duhroach/tools-to-profile-networking-performance-3141870d5233
* https://www.lmax.com/blog/staff-blogs/2016/05/06/navigating-linux-kernel-network-stack-receive-path/
* https://es.net/host-tuning/100g-tuning/
* http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm
* http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html
* https://people.cs.clemson.edu/~westall/853/tcpperf.pdf
* http://tldp.org/HOWTO/Traffic-Control-HOWTO/classless-qdiscs.html
* https://es.net/assets/Papers-and-Publications/100G-Tuning-TechEx2016.tierney.pdf
* https://www.kernel.org/doc/ols/2009/ols2009-pages-169-184.pdf
* https://devcentral.f5.com/articles/the-send-buffer-in-depth-21845
* http://packetbomb.com/understanding-throughput-and-tcp-windows/
* https://www.speedguide.net/bdp.php
* https://www.switch.ch/network/tools/tcp_throughput/
* https://www.ibm.com/support/knowledgecenter/en/SSQPD3_2.6.0/com.ibm.wllm.doc/usingethtoolrates.html
* https://blog.tsunanet.net/2011/03/out-of-socket-memory.html
* https://unix.stackexchange.com/questions/12985/how-to-check-rx-ring-max-backlog-and-max-syn-backlog-size
* https://serverfault.com/questions/498245/how-to-reduce-number-of-time-wait-processes
* https://unix.stackexchange.com/questions/419518/how-to-tell-how-much-memory-tcp-buffers-are-actually-using
* https://eklitzke.org/how-tcp-sockets-work
* https://www.linux.com/learn/intro-to-linux/2017/7/introduction-ss-command
* https://staaldraad.github.io/2017/12/20/netstat-without-netstat/
* https://loicpefferkorn.net/2016/03/linux-network-metrics-why-you-should-use-nstat-instead-of-netstat/
* http://assimilationsystems.com/2015/12/29/bufferbloat-network-best-practice/
* https://wwwx.cs.unc.edu/~sparkst/howto/network_tuning.php
