# Оглавление

* [Введение](#Введение)
* [Обзор сетевых очередей Linux](#Обзор-сетевых-очередей-Linux)
* [Применение переменных sysctl в сетевом потоке Линукс](#Применение-переменных-sysctl-в-сетевом-потоке-Линукс)
  * Входящие пакеты
  * Исходящие исходящие
* [Что, Почему и Как - сеть и  sysctl параметры](#Что-Почему-и-Как-сеть-и--sysctl-параметры)
  * Кольцевой буффер (Ring Buffer) - rx,tx
  * Слияние прерываний (Interrupt Coalescence - IC) - rx-usecs, tx-usecs, rx-frames, tx-frames (аппаратные прерывания IRQ)
  * Коалисцирование прерываний (программные прерывания - soft IRQ) и входящие QDisc
  * Исходящие QDisc - txqueuelen (длина очереди tx) и default_qdisc 
  * TCP Чтение и Запись Буфер/Очеререди (TCP Read and Write Buffers/Queues)
  * Почетное упоминания - TCP FSM и алгоритм перегрузки (Honorable mentions - TCP FSM and congestion algorithm)
* [Сетевые утилиты](#сетевые-утилиты-для-тестирования-и-мониторинга)
* [Рекомендации](#рекомендации)

> От переводчика (@zersh).

>Это адаптированный перевод [работы](https://github.com/leandromoreira/linux-network-performance-parameters). 
>Для понимания некоторых моментов использовалась [статья.](https://habr.com/ru/company/mailru/blog/314168/). Принимаются любые замечания и предложения.
>Спасибо zizmo и @servers(Artem) - за помощь и конструктивную критику )

# Введение

Иногда, люди пытаются найти некие универсальные значения для  [sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) параметров, применение которых во всех случаях позволит добиться и высокой пропускной способности, и низкой задержки при обработке. К сожалению, это не возможно, хотя стоит отметить, что современные версии ядер **по умолчанию уже неплохо настроены**. Важно понимать, что изменение заданных по умолчанию настроек, может [ухудшить производительность] (https://medium.com/@duhroach/the-bandwidth-delay-problem-c6a2a578b211).

Это краткое руководство, в которм приведены **часто используемые параметры  sysctl касающиеся настрое сети Linux**, вдохновленный [илюстрированным реководством сетевого стэку Linux](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/) и многими [постами Marek Majkowski's](https://blog.cloudflare.com/how-to-achieve-low-latency/).

> #### Не стесняйтесь присылать пожелания и предложения! :)

# Обзор сетевых очередей Linux

![linux network queues](/img/linux_network_flow.png "Графическое представление linux/ядра  основного сетевого буфера / очередей")

# Применение переменных sysctl в сетевом потоке Линукс

## Входящие пакеты
1. Пакеты прибывают в NIC(сетевой адаптер)
2. NIC проверяет `MAC` (если не включен "неразборчивый"(promiscuous) режим) и `FCS (Frame check sequence)` и принимает решение отбросить пакет или продолжить обработку.
3. NIC используя [DMA](https://en.wikipedia.org/wiki/Direct_memory_access), помещает пакеты в RAM регионе, ранее подготовленном (mapped) драйвером 
4. NIC ставит ссылки в очередь на пакеты при получении [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer) очередь `rx` до истичения `rx-usecs` таймаута или `rx-frames`
5. NIC Генерируется аппаратное прерывание, чтобы система узнала о появлении пакета в памяти `hard IRQ`
6. CPU запустит `IRQ handler` который запускает код драйвера
7. Драйвер вызовет  `планировщик NAPI`, очистит `hard IRQ` 
8. Драйвер будит подсистему NAPI с помощью `soft IRQ (NET_RX_SOFTIRQ)`
9. NAPI опрашивает данные полученные из кольцевого буфера до тех пор пока не истечет `netdev_budget_usecs` таймаут, или `netdev_budget` и `dev_weight` пакета
10. Linux также аллоцирует память в `sk_buff`
11. Linux заполняет метаданные: протокол, интерфейс, устанавливает мак адрес (setmacheader), удаляет ethernet
12. Linux передает skb (данные) в стэк ядра (`netif_receive_skb`)
13. Установит сетевые заголовки, клонирует `skb` ловушкам (вроде tcpdump) и передаст на вход
14. Пакеты обрабатываются в qdisc (Queueing discipline) размера `netdev_max_backlog`, алгоритм которого определяется `default_qdisc`
15. Вызывает `ip_rcv` и пакеты обрабатываются в IP
16. Вызывает netfilter (`PREROUTING`)
17. Проверяет маршрутизацию, кому предназначен пакет: переслать (forwarding) или локально (local)
18. Если локально, вызывает netfilter (`LOCAL_IN`)
19. Это вызовет протокол L4 (для примера `tcp_v4_rcv`)
20. Находит нужный сокет
21. Он переходит на конечный автомат tcp
22. Поставит пакет в входящий буффер размер которого определяется правилами `tcp_rmem`
    1. Если `tcp_moderate_rcvbuf` включен, ядро будет автоматически тюнить пходящий (receive) буфер
23. Ядро сигнализирует приложению что доступны данные (epoll или другая  polling система)
24. Приложение просыпается и читает данные

## Исходящие пакеты
1. Приложение отправляет сообщение (`sendmsg` или другие)
2. TCP отправка сообщения аллоцирует skb_buff
3. Он поместит skb в сокет буфера, размером `tcp_wmem`
4. Создаст TCP заголовки (источник и порт назначения, чексумма)
5. Вызов обработчика L3 (в данном случае `ipv4` в `tcp_write_xmit` и `tcp_transmit_skb`)
6. L3 (`ip_queue_xmit`) сделает свою работу: построит заголовок IP и вызовет netfilter (`LOCAL_OUT`)
7. Вызывает действие выходного маршрута (Calls output route action)
8. Вызов netfilter (`POST_ROUTING`)
9. Фрагментирует пакет (`ip_output`)
10. Вызов функции отправки L2 (`dev_queue_xmit`)
11. Подает на выход (QDisc) очередь длинной `txqueuelen` с алгоритмом `default_qdisc`
12. Код драйвера помещает пакеты в `ring buffer tx`
13. Драйвер генерирует `soft IRQ (NET_TX_SOFTIRQ)` после `tx-usecs` таймаута или `tx-frames`
14. Реактивирует апаратное прерывание (IRQ) в NIC
15. Драйвер сопоставит все пакеты (для отправки) в некоторую область DMA
16. NIC получит пакеты (через DMA) из RAM для передачи
17. После передачи NIC поднимет `hard IRQ` сигнал о его завершении
18. Драйвер обработает это прерываение IRQ (выключит)
19. И планировщик (`soft IRQ`) NAPI poll system 
20. NAPI будет обрабатывать сигналы приема пакетов и освобождать ОЗУ

# Что, Почему и Как - сеть и  sysctl параметры

## Ring Buffer - rx,tx
* **Что** - драйвер очереди приема / отправки одной или нескольких очередей с фиксированным размером, обычно реализованный как FIFO, находится в ОЗУ
* **Почему** - буфер для плавного приема пакетов соединений без их отбрасывания. Возможно, вам понадобится увеличить эти очереди, когда вы увидите сбросы (drops) или переполнение, то есть, поступает больше пакетов, чем ядро может обработать, побочным эффектом может быть увеличение задержки.
* **Как:**
  * **Команда проверки:** `ethtool -g ethX`
  * **Как изменить:** `ethtool -G ethX rx значение tx значение`
  * **Как мониторить:** `ethtool -S ethX | grep -e "err" -e "drop" -e "over" -e "miss" -e "timeout" -e "reset" -e "restar" -e "collis" -e "over" | grep -v "\: 0"`
 
## Слияние прерываний (Interrupt Coalescence - IC) - rx-usecs, tx-usecs, rx-frames, tx-frames (аппаратные IRQ)
* **Что** - количество микросекунд / кадров, ожидающих перед поднятием hard IRQ, с точки зрения сетевого адаптера это будет пакеты данных DMA до этого тайм-аута / количества кадров
* **Почему** - сокращение использования CPUs, аппаратных IRQ, может увеличить пропускную способность за счет задержки.
* **Как:**
  * **Команда проверки:** `ethtool -c ethX`
  * **Как изменить:** `ethtool -C ethX rx-usecs value tx-usecs value`
  * **Как мониторить:** `cat /proc/interrupts` 
  
## Коалесцирование прерываний (soft IRQ) и вход QDisc
* **Что** - максимальное число микросекунд в одном [NAPI](https://en.wikipedia.org/wiki/New_API) цикле опроса. Опрос завершится когда, либо `netdev_budget_usecs` истечет по временя цикла опроса или количество обработанных пакетов достигнет `netdev_budget`.
* **Почему** - вместо того чтобы обрабатывать тонны softIRQ, драйвер сохраняет данные в пуле (polling data); следите за `dropped` (# пакеты которые были отброшены потому, что `netdev_max_backlog` был превышен) и  `сжат`(`squeezed`) (# число раз когда ksoftirq превысил `netdev_budget` или отрезок времени с оставшейся с работой).
* **Как:**
  * **Команда проверки:** `sysctl net.core.netdev_budget_usecs`
  * **Как изменить:** `sysctl -w net.core.netdev_budget_usecs value`
  * **Как мониторить:** `cat /proc/net/softnet_stat`; или [скриптом](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **Что** - `netdev_budget` максимальное количество пакетов, взятых со всех интерфейсов за один цикл опроса (NAPI poll). В одном цикле опроса интерфейсы, которые зарегистрированы для опроса, зондируются круговым способом. Кроме того, цикл опроса не может превышать `netdev_budget_usecs` микросекунд, даже если `netdev_budget` не был исчерпан.
* **Как:**
  * **Команда проверки:** `sysctl net.core.netdev_budget`
  * **Как изменить:** `sysctl -w net.core.netdev_budget value`
  * **Как мониторить:** `cat /proc/net/softnet_stat`; или [скриптом](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **Что** - `dev_weight` максимальное количество пакетов, которое ядро может обработать при прерывании NAPI, это переменная для каждого процессора. Для драйверов, которые поддерживают LRO или GRO_HW, аппаратно агрегированный пакет считается в этом пакете одним.
* **Как:**
  * **Команда проверки:** `sysctl net.core.dev_weight`
  * **Как изменить:** `sysctl -w net.core.dev_weight value`
  * **Как мониторить:** `cat /proc/net/softnet_stat`; или [скриптом](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
* **Что** - `netdev_max_backlog` максимальное количество пакетов, находящихся в очереди на стороне INPUT (входной qdisc_), когда интерфейс получает пакеты быстрее, чем ядро может их обработать.
* **Как:**
  * **Команда проверки:** `sysctl net.core.netdev_max_backlog`
  * **Как изменить:** `sysctl -w net.core.netdev_max_backlog value`
  * **Как мониторить:** `cat /proc/net/softnet_stat`; или [скриптом](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
  
## Исходящие QDisc - txqueuelen (длина очереди tx) и default_qdisc
* **Что** - `txqueuelen` максимальное количество пакетов, поставленных в очередь на стороне вывода.
* **Почему** - buffer/queue появление разрывов соединений, а также примением контроля трафика [tc (traffic control).](http://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html)
* **Как:**
  * **Команда проверки:** `ifconfig ethX`
  * **Как изменить:** `ifconfig ethX txqueuelen value`
  * **Как мониторить:** `ip -s link` 
* **Что** - `default_qdisc` дисциплина очереди по умолчанию, используемая для сетевых устройств.
* **Почему** - Каждое приложение имеет разную нагрузку и требует контроля трафика, оно также используется для борьбы с  излишней сетевой буферизацией [bufferbloat](https://www.bufferbloat.net/projects/codel/wiki/)
* **Как:**
  * **Команда проверки:** `sysctl net.core.default_qdisc`
  * **Как изменить:** `sysctl -w net.core.default_qdisc value`
  * **Как мониторить:**   `tc -s qdisc ls dev ethX`

## TCP Чтение и Запись Буфера/Очеререди (TCP Read and Write Buffers/Queues)
* **Что** - `tcp_rmem` - min (минимальный размер доступный при создании сокета), default (начальный размер), max (максимальный размер) - максимальный размер приемного буфера TCP.
* **Почему** - буфер/очередь приложения для записи/отправки данных, понять последствия может помочь [статья](https://blog.cloudflare.com/the-story-of-one-latency-spike/).
* **Как:**
  * **Команда проверки:** `sysctl net.ipv4.tcp_rmem`
  * **Как изменить:** `sysctl -w net.ipv4.tcp_rmem="min default max"`; когда меняете значение по умолчанию, не забудьте перезагрузить приложения в пользовательском окружении (т.е. ваш веб-сервер, nginx, и т.п.)
  * **Как мониторить:** `cat /proc/net/sockstat`
* **Что** - `tcp_wmem` - min (минимальный размер доступный при создании сокета), default (начальный размер), max (максимальный размер) - размер буфера отправки, используемого сокетами TCP.
* **Как:**
  * **Команда проверки:** `sysctl net.ipv4.tcp_wmem`
  * **Как изменить:** `sysctl -w net.ipv4.tcp_wmem="min default max"`; когда меняете значение по умолчанию, не забудьте перезагрузить приложения в пользовательском окружении (т.е. ваш веб-сервер, nginx, и т.п.)
  * **Как мониторить:** `cat /proc/net/sockstat`
* **Что** `tcp_moderate_rcvbuf` - Если установлено, TCP выполняет автонастройку приемного буфера, пытаясь автоматически определить размер буфера.
* **Как:**
  * **Команда проверки:** `sysctl net.ipv4.tcp_moderate_rcvbuf`
  * **Как изменить:** `sysctl -w net.ipv4.tcp_moderate_rcvbuf value`
  * **Как мониторить:** `cat /proc/net/sockstat`

## Почетное упоминание - TCP FSM и алгоритм перегрузки (Honorable mentions - TCP FSM and congestion algorithm)
* `sysctl net.core.somaxconn` - обеспечивает верхний предел значения параметра backlog, передаваемого в функцию [`listen()`](https://eklitzke.org/how-tcp-sockets-work), известный пользователям как `SOMAXCONN`. Если вы меняете это значение, вы также должны изменить в своем приложении совместимые значения (т.е. [nginx backlog](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen)).
* `cat /proc/sys/net/ipv4/tcp_fin_timeout` - указывает количество секунд ожидания окончательного пакета FIN, прежде чем сокет будет принудительно закрыт. Это строго является нарушением спецификации TCP, но требуется для предотвращения атак типа «отказ в обслуживании».
* `cat /proc/sys/net/ipv4/tcp_available_congestion_control` - показывает доступные варианты управления перегрузкой, которые зарегистрированы.
* `cat /proc/sys/net/ipv4/tcp_congestion_control` - устанавливает алгоритм управления перегрузкой, используемое для новых соединений.
* `cat /proc/sys/net/ipv4/tcp_max_syn_backlog` - задает максимальное число запросов подключения в очереди, которые еще не получили подтверждения от подключающегося клиента; если это число будет превышено, ядро начнет отбрасывать запросы.
* `cat /proc/sys/net/ipv4/tcp_syncookies` - включен/выключен [syn cookies](https://en.wikipedia.org/wiki/SYN_cookies), полезен для защиты от [syn flood атак](https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/).
* `cat /proc/sys/net/ipv4/tcp_slow_start_after_idle` - включен/выключен медленный старт tcp.

**Как мониторить:** 
* `netstat -atn | awk '/tcp/ {print $6}' | sort | uniq -c` - общая сводка
* `ss -neopt state time-wait | wc -l` - счетчики по определенному состоянию: `established`, `syn-sent`, `syn-recv`, `fin-wait-1`, `fin-wait-2`, `time-wait`, `closed`, `close-wait`, `last-ack`, `listening`, `closing`
* `netstat -st` - tcp статистика
* `nstat -a` - "человекопонятная" tcp статистика
* `cat /proc/net/sockstat` - обобщенная ститистика сокетов
* `cat /proc/net/tcp` - детальная статистика, описание полей смотрите: [kernel docs](https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt)
* `cat /proc/net/netstat` - `ListenOverflows` и `ListenDrops` важные поля для наблюдения
  * `cat /proc/net/netstat | awk '(f==0) { i=1; while ( i<=NF) {n[i] = $i; i++ }; f=1; next} \
(f==1){ i=2; while ( i<=NF){ printf "%s = %d\n", n[i], $i; i++}; f=0} ' | grep -v "= 0`;  [человекочитаемый `/proc/net/netstat`](https://sa-chernomor.livejournal.com/9858.html)

![tcp finite state machine](https://upload.wikimedia.org/wikipedia/commons/a/a2/Tcp_state_diagram_fixed.svg "Графическое представление конечного автомата tcp tcp")
Источник: https://commons.wikimedia.org/wiki/File:Tcp_state_diagram_fixed_new.svg

# Сетевые утилиты для тестирования и мониторинга

* [iperf3](https://iperf.fr/) - пропускная способность сети
* [vegeta](https://github.com/tsenart/vegeta) - нагрузочное тестирование HTTP 
* [netdata](https://github.com/firehol/netdata) - система распределенного мониторинга производительности и работоспособности в реальном времени

# Рекомендации

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
