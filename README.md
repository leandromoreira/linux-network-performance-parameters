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

# What, Why and How - network and sysctl parameters

## Ring Buffer - rx,tx
* **What** - the driver receive/send queue a single or multiple queues with fixed size, usually implemented as FIFO, it is located at RAM
* **Why** - a buffer to smoothly accept bursts of connections without droping them, you might need to increase these queues when you see drops or overrun, aka there are more packets comming than the kernel is able to consume them, the side effect might be increased latency.
* **How:**
  * **Check command:** `ethtool -g ethX`
  * **Change command:** `ethtool -G ethX rx value tx value`
  * **How to monitor:** `ethtool -S ethX | grep -e "err" -e "drop" -e "over" -e "miss" -e "timeout" -e "reset" -e "restar" -e "collis" -e "over" | grep -v "\: 0"`
 
## Interrupt Coalescence (IC) - rx-usecs, tx-usecs, rx-frames, tx-frames (hardware IRQ)
* **What** - number of microseconds/frames to wait before raising a hardIRQ, from the NIC perspective it'll DMA data packets until this timeout/number of frames
* **Why** - Reduce CPUs usage, hard IRQ, might increase throughput at cost of latency.
* **How:**
  * **Check command:** `ethtool -c ethX`
  * **Change command:** `ethtool -C ethX rx-usecs value tx-usecs value`
  * **How to monitor:** `cat /proc/interrupts` 
  
## Interrupt Coalescing (soft IRQ) and Ingress QDisc
* **What** - maximum number of microseconds in one [NAPI](https://en.wikipedia.org/wiki/New_API) polling cycle. Polling will exit when either `netdev_budget_usecs` have elapsed during the poll cycle or the number of packets processed reaches  `netdev_budget`.
* **Why** - instead of reacting to tons of softIRQ, the driver keeps polling data, keep an eye on `dropped` and `squeezed`, dropped  # of packets that were dropped because `netdev_max_backlog` was exceeded and squeezed  # of times ksoftirq ran out of `netdev_budget` or time slice with work remaining.
* **How:**
  * **Check command:** `sysctl net.core.netdev_budget_usecs`
  * **Change command:** `sysctl -w net.core.netdev_budget_usecs value`
  * **How to monitor:** `cat /proc/net/softnet_stat; or a better tool https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh`  
* **What** - `netdev_budget` is the maximum number of packets taken from all interfaces in one polling cycle (NAPI poll). In one polling cycle interfaces which are registered to polling areprobed in a round-robin manner. Also, a polling cycle may not exceed `netdev_budget_usecs` microseconds, even if `netdev_budget` has not been exhausted.
* **How:**
  * **Check command:** `sysctl net.core.netdev_budget`
  * **Change command:** `sysctl -w net.core.netdev_budget value`
  * **How to monitor:** `cat /proc/net/softnet_stat; or a better tool https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh`
* **What** - `dev_weight` is the maximum number of packets that kernel can handle on a NAPI interrupt, it's a Per-CPU variable. For drivers that support LRO or GRO_HW, a hardware aggregated packet is counted as one packet in this.
* **How:**
  * **Check command:** `sysctl net.core.dev_weight`
  * **Change command:** `sysctl -w net.core.dev_weight value`
  * **How to monitor:** `cat /proc/net/softnet_stat;o r a better tool  https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh`
* **What** - `netdev_max_backlog` is the maximum number  of  packets,  queued  on  the  INPUT side (_the ingress qdisc_), when the interface receives packets faster than kernel can process them.
* **How:**
  * **Check command:** `sysctl net.core.netdev_max_backlog`
  * **Change command:** `sysctl -w net.core.netdev_max_backlog value`
  * **How to monitor:** `cat /proc/net/softnet_stat;or a better tool  https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh`
  
## Egress QDisc - txqueuelen and default_qdisc
* **What** - `txqueuelen` is the maximum number of packets, queued on the OUTPUT side.
* **Why** - a buffer/queue to face connection burst and also to apply [tc (traffic control).](http://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html)
* **How:**
  * **Check command:** `ifconfig ethX`
  * **Change command:** `ifconfig ethX txqueuelen value`
  * **How to monitor:** `ip -s link` 
* **What** - `default_qdisc` is the default queuing discipline to use for network devices.
* **Why** - each application has different load and need to traffic control and it is used also to fight against [bufferfloat](https://www.bufferbloat.net/projects/codel/wiki/)
* **How:**
  * **Check command:** `sysctl net.core.default_qdisc`
  * **Change command:** `sysctl -w net.core.default_qdisc value`
  * **How to monitor:**   `tc -s qdisc ls dev ethX`

## TCP Read and Write Buffers/Queues
* **What** - `tcp_rmem` - min (size used under memory pressure), default (initial size), max (maximum size) - size of receive buffer used by TCP sockets.
* **Why** - the application buffer/queue to the write/send data, it's provided to the userspace from the skb_buff.
* **How:**
  * **Check command:** `sysctl net.ipv4.tcp_rmem`
  * **Change command:** `sysctl -w net.ipv4.tcp_rmem="min default max"; when changing default value remember to restart your user space app (ie: your web server, nginx and etc)`
  * **How to monitor:** `cat /proc/net/sockstat`
* **What** - `tcp_wmem` - min (size used under memory pressure), default (initial size), max (maximum size) - size of send buffer used by TCP sockets.
* **How:**
  * **Check command:** `sysctl net.ipv4.tcp_wmem`
  * **Change command:** `sysctl -w net.ipv4.tcp_wmem="min default max"; when changing default value remember to restart your user space app (ie: your web server, nginx and etc)`
  * **How to monitor:** `cat /proc/net/sockstat`
* **What** `tcp_moderate_rcvbuf` - If set, TCP performs receive buffer auto-tuning, attempting to automatically size the buffer.
* **How:**
  * **Check command:** `sysctl net.ipv4.tcp_moderate_rcvbuf`
  * **Change command:** `sysctl -w net.ipv4.tcp_moderate_rcvbuf value`
  * **How to monitor:** `cat /proc/net/sockstat`

## Honorable mentions - TCP FSM and congestion algorithm
* `somaxconn` - Limit of socket [listen() backlog](https://eklitzke.org/how-tcp-sockets-work), known in userspace as SOMAXCONN. When you change this value you should change your application to the same value, for example [nginx backlog](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) should be changed.
* `tcp_fin_timeout` - this specifies how many seconds to wait for a final FIN packet before the socket is forcibly closed.  This is strictly a violation of the TCP specification, but required to prevent denial-of-service attacks.
* `tcp_available_congestion_control` - shows the available congestion control choices that are registered.
* `tcp_congestion_control` - set the congestion control algorithm to be used for new connections.
* `tcp_max_syn_backlog` - the maximum number of queued connection requests which have still not received an acknowledgement from the connecting client.  If this number is exceeded, the kernel will begin dropping requests.
* `tcp_slow_start_after_idle` - enable/disable tcp slow start

**How to monitor:** 
* `netstat -atn | awk '/tcp/ {print $6}' | sort | uniq -c ; summary by state`
* `ss -neopt state time-wait | wc -l; count for a specific state: established, syn-sent, syn-recv, fin-wait-1, fin-wait-2, time-wait, closed, close-wait, last-ack, listening, closing`
* `netstat -st; tcp stats summary`
* `nstat -a; human friendly tcp stats summary`
* `cat /proc/net/sockstat; summary`
* `cat /proc/net/tcp; detailed, see each field meaning at https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt`
* `cat /proc/net/netstat; ListenOverflows and ListenDrops are fields to closely monitor;`
  * `cat /proc/net/netstat | awk '(f==0) { i=1; while ( i<=NF) {n[i] = $i; i++ }; f=1; next} \
(f==1){ i=2; while ( i<=NF){ printf "%s = %d\n", n[i], $i; i++}; f=0} ' | grep -v "= 0"; from https://sa-chernomor.livejournal.com/9858.html`

![tcp finite state machine](https://upload.wikimedia.org/wikipedia/commons/a/a2/Tcp_state_diagram_fixed.svg "A graphic representation of tcp tcp finite state machine")
Source: https://commons.wikimedia.org/wiki/File:Tcp_state_diagram_fixed_new.svg

# References

* https://www.kernel.org/doc/Documentation/sysctl/net.txt
* https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
* https://www.kernel.org/doc/Documentation/networking/scaling.txt
* https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt
* http://man7.org/linux/man-pages/man7/tcp.7.html
* http://man7.org/linux/man-pages/man8/tc.8.html
* http://www.ece.virginia.edu/cheetah/documents/papers/TCPlinux.pdf
* https://netdevconf.org/1.2/papers/bbr-netdev-1.2.new.new.pdf
* https://blog.cloudflare.com/how-to-receive-a-million-packets/
* https://blog.cloudflare.com/how-to-achieve-low-latency/
* https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/
* https://www.youtube.com/watch?v=6Fl1rsxk4JQ
* https://www.intel.com/content/dam/www/public/us/en/documents/reference-guides/xl710-x710-performance-tuning-linux-guide.pdf
* https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf
* https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/multi-core-processor-based-linux-paper.pdf
* http://syuu.dokukino.com/2013/05/linux-kernel-features-for-high-speed.html
* https://software.intel.com/en-us/articles/setting-up-intel-ethernet-flow-director
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
