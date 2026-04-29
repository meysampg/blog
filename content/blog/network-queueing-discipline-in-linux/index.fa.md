---
date: 2026-03-24T16:49:36+03:30
draft: false
title: خرابکاری عمدی شبکه در لینوکس
categories:
  - Network
  - Distributed Systems
tags:
  - Linux
summary: شاید کل ماجرای سیستم‌های توزیع‌یافته این باشه که شبکه ناپایداره و برای کنترل این ناپایداری، روش‌های مختلفی ابداع میشن. در این پست قراره حالت‌های مختلف ناپایداری شبکه رو بدون ابزار اضافی داخل لینوکس ایجاد کنیم.
---
# مقدمه
امروز روز ۲۵م جنگه و مثل بچه‌ی ۱۴ ساله‌ای که داره تو نت می‌چرخه و یدفه میان تو اتاقش و مشغول خوندن هلپ ویندوز میشه (البته این تو نسل من معنا داره و احتمالاً الان معنای خاصی نداشته باشه)، دارم تو گوشه کنارهای لینوکس می‌چرخم که خودم رو سرگرم کنم. این وسط رسیدم به ابزار `tc` (مخفف Network Control) که خیلی کارا میشه باهاش انجام داد که میشه با `man 8 tc` اونا رو دید. از بین اون خیلی کارا ولی چیزی که نظرم رو جلب کرد `qdisc` (مخفف Queueing Discipline) بود. در این پست قراره ببینیم ماجرای صف‌ها چی‌ان، دوتا سیستم مجازی بسازیم و حالت‌های ناپایداری شبکه رو روی اونها شبیه‌سازی و تست کنیم.

# صف
داخل شبکه، هر دیوایس فیزیکی می‌تونه با حداکثر سرعت مشخصی پکت‌ها رو ارسال/دریافت کنه و طبیعیه که اگه سرعت ارسال/دریافت پکت‌ها از حدی بیشتر باشه، پکت‌ها باید منتظر بمونن. برای اینکه در این انتظار پکتی از بین نره، طبیعیه که شکلی از بافرِ پکت‌ها رو داشته باشیم. و صف در `tc` همون بافر هست برای پکت‌هایی که قراره ارسال شن. داخل لینوکس این صف(ها) با `qdisc` کنترل میشن.

صف‌هایی که داخل `qdisc` می‌تونن تعریف شن به دو قسمن: Classful و Classless.
## صف‌های باکلاس
باکلاس‌ها، صف‌هایی هستن که می‌تونن تشکیل یه درخت بدن (شامل صف‌های دیگه باشن). برای مثال وقتی نیازه که یه سری پکت‌ها (پکت‌های `ssh` مثلاً) اولویت بیشتری نسبت به دسته‌ی دوم پکت‌ها (پکت‌های `ICMP` مثلاً) داشته باشن و باقی پکت‌ها اولویت سوم رو داشته باشن یه اینطور درختی میشه ساخت: 
```
qdisc
|- High Priority: SSH Packets
|- Medium Priority: ICMP Packets
|- No Priority: All Other Packets
```
اینطوری مثلاً همیشه میشه مطمئن بود که هر بگایی‌ای هم که پیش بیاد، پکت‌های `SSH` تو صف گیر نمی‌کنن و احتمالن میشه مشکل رو حتی در وقت ترافیک سنگین روی شبکه جمعش کرد.

##  صف‌های بی‌کلاس
در مقابل صف‌های بی‌کلاس هستن که طبیعتاً یه صف ساده‌ن. همونطور که ترافیک میاد، پردازش میشه و میره. و تو این پست قراره بریم سراغ یکی از این بی‌کلاس‌ها: `netem` (مخفف `Network Emulator`) که میشه با `man 8 netem` به داکیومنت‌هاش دسترسی پیدا کرد.

# کارت مجازی شبکه
قبل از اینکه بریم سروقت `netem`، باید به این نکته توجه کنیم که طبق منوال `tc`، وقتی می‌خوایم یه `qdisc` اضافه کنیم، یا باید اسم والد رو بدیم (اضافه کردن به درخت در واقع) و یا باید مستقیم به یه دیوایس ادش کنیم. از اونجا که ما نمی‌خوایم بریم سر وقت صف‌های باکلاس و جدا از اون، قرار هم نیست عملکرد کارت شبکه‌ای که داریم باهاش کار می‌کنیم رو مختل کنیم، لاجرم باید یه کارت شبکه‌ی مجازی بسازیم. با دستور `ip` داخل لینوکس میشه جینگیل پینگیل‌های نتورک رو مدیریت کرد و برای ادامه از `ip help` و `man 8 ip` استفاده می‌کنیم.
## لینک
دستور `ip` به صورت
```shell
ip [ OPTIONS ] OBJECT { COMMAND | help }  
```
استفاده میشه که در اون `OPTIONS` 
```shell
OPTIONS := { 
	-V[ersion] | 
	-h[uman-readable] | 
	-s[tatistics] | 
	-d[etails] | 
	-r[esolve] | 
	-iec | 
	-f[amily] { inet | inet6 | link } | 
	-4 | -6 | -B | -0 | 
	-l[oops] { maximum-addr-flush-attempts } | 
	-o[neline] | 
	-rc[vbuf] [size] | 
	-t[imestamp] | 
	-ts[hort] | 
	-n[etns] name | 
	-N[umeric] | 
	-a[ll] | 
	-c[olor] | 
	-br[ief] | 
	-j[son] | 
	-p[retty]
}
```
هست و ‍`OBJECT` هم که همون چیزیه که ما می‌خوایم تغییر بدیم
```shell
OBJECT := { 
	link | 
	address | 
	addrlabel | 
	route | 
	rule | 
	neigh | 
	ntable | 
	tunnel | 
	tuntap | 
	maddress | 
	mroute | 
	mrule | 
	monitor | 
	xfrm | 
	netns | 
	l2tp | 
	tcp_metrics | 
	token | 
	macsec | 
	vrf | 
	mptcp | 
	ioam | 
	stats
}
```
از بین همه‌ی این آبجکت‌ها ما می‌خوایم یه لینک اضافه کنیم، پس اگه هلپ اون آبجکت رو بگیریم:
```shell
meysam@ubuntu:~/www/test/network$ ip link help
Usage: ip link add [link DEV | parentdev NAME] [ name ] NAME
		    [ txqueuelen PACKETS ]
		    [ address LLADDR ]
		    [ broadcast LLADDR ]
		    [ mtu MTU ] [index IDX ]
		    [ numtxqueues QUEUE_COUNT ]
		    [ numrxqueues QUEUE_COUNT ]
		    [ netns { PID | NAME } ]
		    type TYPE [ ARGS ]
```
همه‌ی گزینه‌های بین براکت اختیاری هستن، پس ما یه اسم نیاز داریم و تایپ و احتمالاً آرگومان‌های این تایپ. که اگه پایین همون دستور بالا رو ببینیم، تایپ‌ها اینان:
```shell
TYPE := { amt | bareudp | bond | bond_slave | bridge | bridge_slave |
          dsa | dummy | erspan | geneve | gre | gretap | gtp | ifb |
          ip6erspan | ip6gre | ip6gretap | ip6tnl |
          ipip | ipoib | ipvlan | ipvtap |
          macsec | macvlan | macvtap |
          netdevsim | nlmon | rmnet | sit | team | team_slave |
          vcan | veth | vlan | vrf | vti | vxcan | vxlan | wwan |
          xfrm | virt_wifi }
```
تقریباً نصف این تایپ‌ها رو نمی‌دونم چی‌ان ولی چیزی که ما الان می‌خوایم یه کارت مجازیه پس `veth` (مخفف Virtual Ethernet) رو انتخاب می‌کنیم و هلپ تایپ رو می‌گیریم که ببینیم آرگومان‌هاش چیان:
```shell
meysam@ubuntu:~/www/test/network$ ip link help veth
Usage: ip link <options> type veth [peer <options>]
To get <options> type 'ip link add help'
```
اپشن‌ها که همون آپشن‌های بالان ولی نکته‌ای که هست `peer`ه. veth مثل وصل کردن کارت شبکه‌های دوتا سیستم با یه کابل به همدیگه‌س. پس طبق چیزی که بالا رفت (با توجه به اختیاری بودن گزینه‌ها و انتخاب تایپ موقع ساختن) ما فقط به اسم نیاز داریم:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip link add name vserver type veth peer name vclient
[sudo] password for meysam:
meysam@ubuntu:~/www/test/network$ ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: vclient@vserver: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether c2:b9:df:90:93:6d brd ff:ff:ff:ff:ff:ff
4: vserver@vclient: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 96:28:e4:c9:4a:35 brd ff:ff:ff:ff:ff:ff
```
خوبه. ما دو تا سیستم رو یه جورایی شبیه‌سازی کردیم ولی انگار روشن نیستن (`state Down`). پس روشن‌شون می‌کنیم:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip link set dev vserver up
meysam@ubuntu:~/www/test/network$ sudo ip link set dev vclient up
meysam@ubuntu:~/www/test/network$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: vclient@vserver: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether c2:b9:df:90:93:6d brd ff:ff:ff:ff:ff:ff
4: vserver@vclient: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 96:28:e4:c9:4a:35 brd ff:ff:ff:ff:ff:ff
```
برای اینکه ازشون استفاده کنیم به آدرس IPهاشون نیاز داریم (مگه اینکه قصد داشته باشیم با مک آدرس‌ها ارتباط بگیریم):
```shell
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
3: vclient@vserver: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether c2:b9:df:90:93:6d brd ff:ff:ff:ff:ff:ff
    inet6 fe80::c0b9:dfff:fe90:936d/64 scope link
       valid_lft forever preferred_lft forever
4: vserver@vclient: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 96:28:e4:c9:4a:35 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::9428:e4ff:fec9:4a35/64 scope link
       valid_lft forever preferred_lft forever
```
که می‌بینیم ندارن، پس نیازه بهشون آدرس `IP` هم بدیم که اگه برگردیم به آبجکت‌های `ip` باید هلپ `address` رو ببینیم:
```shell
meysam@ubuntu:~/www/test/network$ ip address help
Usage: ip address {add|change|replace} IFADDR dev IFNAME [ LIFETIME ] [ CONFFLAG-LIST ]
```
که در اون باید `add` رو انتخاب کنیم، `IFADDR` همون آدرسی هست که می‌خوایم به اینترفیس‌مون بدیم و `IFNAME` هم که اسم اینترفیسه. پس:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip address add 10.20.30.40/24 dev vserver
meysam@ubuntu:~/www/test/network$ sudo ip address add 10.20.30.50/24 dev vclient
meysam@ubuntu:~/www/test/network$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
3: vclient@vserver: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether c2:b9:df:90:93:6d brd ff:ff:ff:ff:ff:ff
    inet 10.20.30.50/24 scope global vclient
       valid_lft forever preferred_lft forever
    inet6 fe80::c0b9:dfff:fe90:936d/64 scope link
       valid_lft forever preferred_lft forever
4: vserver@vclient: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 96:28:e4:c9:4a:35 brd ff:ff:ff:ff:ff:ff
    inet 10.20.30.40/24 scope global vserver
       valid_lft forever preferred_lft forever
    inet6 fe80::9428:e4ff:fec9:4a35/64 scope link
       valid_lft forever preferred_lft forever
```

# فضای نام شبکه
تایتل بخش شبیه وقتیه که یکی از مجاهدین خلق می‌خواد خودش رو منفجر کنه، ولی یه مشکل کمتر جالب هست. تا اینجا ما تونستیم دوتا کارت شبکه مجازی بسازیم. ولی وقتی می‌زنیم که آدرس اونا رو بهمون نشون بده، خیلی تابلو مشخصه که داخل یه سیستم داریم کار می‌کنیم (در این حد که شماره‌ی ۲ که آدرس سیستم خودم هستم رو مجبور شدم پاک کنم). این باحال نیست، ما قرار بود دوتا سیستم مجازی جدا داشته باشیم و روی اون ناپایداری شبکه رو شبیه‌سازی کنیم. از اونجا که ما دوتا سیستم جدا نداریم و می‌خوایم توهم جدا بودن دوتا سیستم رو داشته باشیم، باید از فضای نام شبکه (Network Namespace) استفاده کنیم. به فضای نام‌ها در ادامه‌ی سری [داکر از صفر](series/docker-from-scratch) بر می‌گردیم، ولی داخل [پست پروسه](blog/docker-from-scrach-process-isolation/#چیه-اینا) بررسی کردیم که سیس‌کال `clone` یه سری فلگ می‌گیره که بعداً می‌بینیم که یه سری از این فلگ‌ها مرتبط با فضاهای نام هستن. خیلی الان گیر ندیم و بهش برگردیم اگه بخوایم توضیح بدیم، فضای نام با ایزوله کردن پروسه، بهش توهم تک و تنها بودن میده. اگه به آبجکت دستور لینک برگردیم، `netns` همون چیزیه که در این مورد باید بهش توجه کنیم.

## ساختن شبکه‌های مجزا
هلپ `netns` راه کار با فضای نام شبکه رو میگه:
```shell
meysam@ubuntu:~/www/test/network$ ip netns help
Usage:	ip netns list
	ip netns add NAME
	ip netns attach NAME PID
	ip netns set NAME NETNSID
	ip [-all] netns delete [NAME]
	ip netns identify [PID]
	ip netns pids NAME
	ip [-all] netns exec [NAME] cmd ...
	ip netns monitor
	ip netns list-id [target-nsid POSITIVE-INT] [nsid POSITIVE-INT]
NETNSID := auto | POSITIVE-INT
```
پس برای ساختن فضای نام کافیه یه اسم انتخاب کنیم و اونو `add` کنیم:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns add server
meysam@ubuntu:~/www/test/network$ sudo ip netns add client
meysam@ubuntu:~/www/test/network$ ip netns list
client
server
```
و کارت شبکه‌هامون رو به سرورهای شبه‌مجازی‌مون اد کنیم:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip link set vserver netns server
meysam@ubuntu:~/www/test/network$ sudo ip link set vclient netns client
```
## اجرای دستور در یک فضای شبکه
اگه به هلپ `netns` که در بالاتر هست نگاه کنیم، به ما یه دستور
```shell
ip [-all] netns exec [NAME] cmd ...
```
میده. این دستور این امکان رو برامون فراهم می‌کنه که روی همه یا یکی از فضای نام‌های شبکه‌مون یه دستور رو اجرا کنیم. پس با ترکیب این دستور و دستور `ip address show` که بالا استفاده کردیم، میشه ببینیم کارت شبکه‌های سیستم‌های شبه‌مجازی‌مون در چه حالن:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip -all netns exec ip address show

netns: client
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: vclient@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether c2:b9:df:90:93:6d brd ff:ff:ff:ff:ff:ff link-netns server

netns: server
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: vserver@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 96:28:e4:c9:4a:35 brd ff:ff:ff:ff:ff:ff link-netns client
```
عالی! حتی دیوایس لوپ‌پک رو هم به صورت جدا داریم. فقط مونده دادن آدرس بهشون در این فضای نام‌ها:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns exec server ip address add 10.20.30.40/24 dev vserver
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client ip address add 10.20.30.50/24 dev vclient
meysam@ubuntu:~/www/test/network$ sudo ip netns exec server ip link set vserver up
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client ip link set vclient up
meysam@ubuntu:~/www/test/network$ sudo ip -all netns exec ip address show

netns: client
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: vclient@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether c2:b9:df:90:93:6d brd ff:ff:ff:ff:ff:ff link-netns server
    inet 10.20.30.50/24 scope global vclient
       valid_lft forever preferred_lft forever
    inet6 fe80::c0b9:dfff:fe90:936d/64 scope link
       valid_lft forever preferred_lft forever

netns: server
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: vserver@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 96:28:e4:c9:4a:35 brd ff:ff:ff:ff:ff:ff link-netns client
    inet 10.20.30.40/24 scope global vserver
       valid_lft forever preferred_lft forever
    inet6 fe80::9428:e4ff:fec9:4a35/64 scope link
       valid_lft forever preferred_lft forever
```

## پینگ از یک سرور به سرور دیگه
و خب این تیکه هم چیزی نداره دیگه:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client ping 10.20.30.40
PING 10.20.30.40 (10.20.30.40) 56(84) bytes of data.
64 bytes from 10.20.30.40: icmp_seq=1 ttl=64 time=0.790 ms
64 bytes from 10.20.30.40: icmp_seq=2 ttl=64 time=0.111 ms
64 bytes from 10.20.30.40: icmp_seq=3 ttl=64 time=0.103 ms
64 bytes from 10.20.30.40: icmp_seq=4 ttl=64 time=0.391 ms
64 bytes from 10.20.30.40: icmp_seq=5 ttl=64 time=0.175 ms
64 bytes from 10.20.30.40: icmp_seq=6 ttl=64 time=0.176 ms
64 bytes from 10.20.30.40: icmp_seq=7 ttl=64 time=0.303 ms
64 bytes from 10.20.30.40: icmp_seq=8 ttl=64 time=0.090 ms
^C
--- 10.20.30.40 ping statistics ---
8 packets transmitted, 8 received, 0% packet loss, time 7161ms
rtt min/avg/max/mdev = 0.090/0.267/0.790/0.220 ms
```
ما از شبه‌سرور کلاینت‌مون تونستیم به شبه‌سرور سرور پکت بفرستیم. خیلی هم عالی، حالا می‌تونیم بریم سراغ خراب‌کاری داخل شبکه‌مون.

# شبیه‌ساز خرابی شبکه
گفتیم که قراره از `netem` متعلق به `qdisc` دستور `tc` استفاده کنیم. اول ببینیم دستور `tc` خودش چه شکلیه:
```shell
tc [ OPTIONS ] OBJECT { COMMAND | help }
```
که در اون آبجکت‌ها اینان:
```shell
OBJECT := { 
	qdisc | 
	class | 
	filter | 
	chain |
	action | 
	monitor | 
	exec 
}
```
و ما دنبال `qdisc` هستیم:
```shell
tc qdisc [ add | del | replace | change | show ] dev STRING
       [ handle QHANDLE ] [ root | ingress | clsact | parent CLASSID ]
       [ estimator INTERVAL TIME_CONSTANT ]
       [ stab [ help | STAB_OPTIONS] ]
       [ ingress_block BLOCK_INDEX ] [ egress_block BLOCK_INDEX ]
       [ [ QDISC_KIND ] [ help | OPTIONS ] ]
```
که در اون:
```shell
QDISC_KIND := { [p|b]fifo | tbf | prio | cbq | red | etc. }
OPTIONS := ... try tc qdisc add <desired QDISC_KIND> help
STAB_OPTIONS := ... try tc qdisc add stab help
QDISC_ID := { root | ingress | handle QHANDLE | parent CLASSID }
```
چون ما می‌خوایم `netem` رو بریم سروقتش، به نظر باید دنبال `QDISC_KIND` باشیم و طبق راهنمایی خودش برا گرفتن آپشن‌هاش:
```shell
meysam@ubuntu:~/www/test/network$ tc qdisc add netem help
Usage: ... netem [ limit PACKETS ]
                 [ delay TIME [ JITTER [CORRELATION]]]
                 [ distribution {uniform|normal|pareto|paretonormal} ]
                 [ corrupt PERCENT [CORRELATION]]
                 [ duplicate PERCENT [CORRELATION]]
                 [ loss random PERCENT [CORRELATION]]
                 [ loss state P13 [P31 [P32 [P23 P14]]]
                 [ loss gemodel PERCENT [R [1-H [1-K]]]
                 [ ecn ]
                 [ reorder PERCENT [CORRELATION] [ gap DISTANCE ]]
                 [ rate RATE [PACKETOVERHEAD] [CELLSIZE] [CELLOVERHEAD]]
                 [ slot MIN_DELAY [MAX_DELAY] [packets MAX_PACKETS] [bytes MAX_BYTES]]
                 [ slot distribution {uniform|normal|pareto|paretonormal|custom}
                   DELAY JITTER [packets MAX_PACKETS] [bytes MAX_BYTES]]
```
به به به به! هر گندی که بشه تو شبکه زد اینجا هست :)). با `man 8 netem` میشه توضیح هر کدوم از این گندکاری‌ها رو دید. تنها نکته‌ای که باید بهش توجه کنیم اینه که `netem` روی ترافیک خروجی اثر می‌ذاره و اگه قراره یه سناریویی باشه که روی ورودی و خروجی اثر بذاره، باید روی دو طرف اعمال بشه.

## پکت‌لاس
سه تا مدلی که برای پک‌لاس داریم رندوم مستقل، زنجیره‌ی مارکوف ۴ مرحله‌ای و مدل Gilbert-Eliot هستن. دوتای آخری رو چون نت نداریم ببینیم چیان ول می‌کنیم و همون رندوم مستقل رو پیش می‌بریم. می‌خوایم ۵۰ درصد پکت‌ها از دست برن.
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client tc qdisc add dev vclient root netem loss random 50
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client ping -c 10 10.20.30.40
PING 10.20.30.40 (10.20.30.40) 56(84) bytes of data.
64 bytes from 10.20.30.40: icmp_seq=1 ttl=64 time=0.061 ms
64 bytes from 10.20.30.40: icmp_seq=2 ttl=64 time=0.094 ms
64 bytes from 10.20.30.40: icmp_seq=4 ttl=64 time=0.097 ms
64 bytes from 10.20.30.40: icmp_seq=6 ttl=64 time=0.108 ms
64 bytes from 10.20.30.40: icmp_seq=7 ttl=64 time=0.177 ms
64 bytes from 10.20.30.40: icmp_seq=8 ttl=64 time=0.109 ms
64 bytes from 10.20.30.40: icmp_seq=10 ttl=64 time=0.108 ms

--- 10.20.30.40 ping statistics ---
10 packets transmitted, 7 received, 30% packet loss, time 9216ms
rtt min/avg/max/mdev = 0.061/0.107/0.177/0.032 ms
```
و اگه به اطلاعات لینک برگردیم، جلوی `qdisc` می‌تونیم `netem` رو پیدا کنیم:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: vclient@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc netem state UP mode DEFAULT group default qlen 1000
    link/ether c2:b9:df:90:93:6d brd ff:ff:ff:ff:ff:ff link-netns server
```
کامند پاک کردن همه‌ی خرابکاری‌ها هم مثل همون اد کردنشونه، به جز جابجایی `add` با `del`:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: vclient@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether c2:b9:df:90:93:6d brd ff:ff:ff:ff:ff:ff link-netns server
```

## وقفه
وقفه اینطور تعریف میشه:
```
DELAY := delay TIME [ JITTER [ CORRELATION ]]]
              [ distribution { uniform | normal | pareto |  paretonormal } ]
```
مثلا اضافه کردن ۱۰ میلی‌ثانیه دیلی به پکت‌های خروجی:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client tc qdisc add dev vclient root netem delay 10ms
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client ping -c 20 10.20.30.40
PING 10.20.30.40 (10.20.30.40) 56(84) bytes of data.
64 bytes from 10.20.30.40: icmp_seq=1 ttl=64 time=10.1 ms
64 bytes from 10.20.30.40: icmp_seq=2 ttl=64 time=12.0 ms
64 bytes from 10.20.30.40: icmp_seq=3 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=4 ttl=64 time=11.3 ms
64 bytes from 10.20.30.40: icmp_seq=5 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=6 ttl=64 time=12.6 ms
64 bytes from 10.20.30.40: icmp_seq=7 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=8 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=9 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=10 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=11 ttl=64 time=10.6 ms
64 bytes from 10.20.30.40: icmp_seq=12 ttl=64 time=10.2 ms
64 bytes from 10.20.30.40: icmp_seq=13 ttl=64 time=12.6 ms
64 bytes from 10.20.30.40: icmp_seq=14 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=15 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=16 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=17 ttl=64 time=10.6 ms
64 bytes from 10.20.30.40: icmp_seq=18 ttl=64 time=12.5 ms
64 bytes from 10.20.30.40: icmp_seq=19 ttl=64 time=12.2 ms
64 bytes from 10.20.30.40: icmp_seq=20 ttl=64 time=12.7 ms

--- 10.20.30.40 ping statistics ---
20 packets transmitted, 20 received, 0% packet loss, time 19058ms
rtt min/avg/max/mdev = 10.081/11.989/12.692/0.859 ms
```
مقایسه‌ی `rtt` این مثال و مثال قبل، شمایل وقفه رو مشخص می‌کنه.

## تکراری
۲۰ درصد پکت‌های دو بار فرستاده شن:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client tc qdisc add dev vclient root netem duplicate 20
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client ping -c 10 10.20.30.40
PING 10.20.30.40 (10.20.30.40) 56(84) bytes of data.
64 bytes from 10.20.30.40: icmp_seq=1 ttl=64 time=0.052 ms
64 bytes from 10.20.30.40: icmp_seq=2 ttl=64 time=0.076 ms
64 bytes from 10.20.30.40: icmp_seq=2 ttl=64 time=0.083 ms (DUP!)
64 bytes from 10.20.30.40: icmp_seq=3 ttl=64 time=0.103 ms
64 bytes from 10.20.30.40: icmp_seq=4 ttl=64 time=0.096 ms
64 bytes from 10.20.30.40: icmp_seq=5 ttl=64 time=0.127 ms
64 bytes from 10.20.30.40: icmp_seq=6 ttl=64 time=0.110 ms
64 bytes from 10.20.30.40: icmp_seq=7 ttl=64 time=0.075 ms
64 bytes from 10.20.30.40: icmp_seq=8 ttl=64 time=0.075 ms
64 bytes from 10.20.30.40: icmp_seq=9 ttl=64 time=0.092 ms
64 bytes from 10.20.30.40: icmp_seq=9 ttl=64 time=0.098 ms (DUP!)
64 bytes from 10.20.30.40: icmp_seq=10 ttl=64 time=0.105 ms

--- 10.20.30.40 ping statistics ---
10 packets transmitted, 10 received, +2 duplicates, 0% packet loss, time 9253ms
rtt min/avg/max/mdev = 0.052/0.091/0.127/0.019 ms
```

## خرابی
خرابی و لاس‌پکت اثر نهایی‌شون مثل همدیگه‌ست (کلاینت جوابی نمی‌گیره)، ولی تفاوت‌شون در اینه که در لاس‌پکت، پکت اصن به دست سرور نمی‌رسه، ولی در خرابی، پکت می‌رسه، ولی خرابه. اینطور میشه ۸۰ درصد از پکت‌ها رو خراب کرد:
```shell
meysam@ubuntu:~/www/test/network$ sudo ip netns exec client tc qdisc add dev vclient root netem corrupt 80
```
و اگه قبل از خرابی ۵ بار از سرور پینگ بگیریم و بعد از خرابی هم ۵ بار، نتیجه اینطور چیزی میشه:
```shell
meysam@ubuntu:~$ sudo tcpdump -i vserver
listening on vserver, link-type EN10MB (Ethernet), snapshot length 262144 bytes
17:54:01.254269 IP 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2107, seq 1, length 64
17:54:01.254455 IP 10.20.30.40 > 10.20.30.50: ICMP echo reply, id 2107, seq 1, length 64
17:54:02.304731 IP 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2107, seq 2, length 64
17:54:02.304770 IP 10.20.30.40 > 10.20.30.50: ICMP echo reply, id 2107, seq 2, length 64
17:54:03.328815 IP 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2107, seq 3, length 64
17:54:03.328864 IP 10.20.30.40 > 10.20.30.50: ICMP echo reply, id 2107, seq 3, length 64
17:54:04.355430 IP 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2107, seq 4, length 64
17:54:04.355458 IP 10.20.30.40 > 10.20.30.50: ICMP echo reply, id 2107, seq 4, length 64
17:54:05.379301 IP 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2107, seq 5, length 64
17:54:05.379475 IP 10.20.30.40 > 10.20.30.50: ICMP echo reply, id 2107, seq 5, length 64
************************
17:54:19.616064 IP 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2113, seq 1, length 64
17:54:19.616086 IP 10.20.30.40 > 10.20.30.50: ICMP echo reply, id 2113, seq 1, length 64
17:54:20.675536 IP 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2113, seq 2, length 64
17:54:21.700633 c2:b9:df:90:93:6d (oui Unknown) > 96:28:e4:c9:4a:35 (oui Unknown), ethertype Unknown (0x0900), length 98:
	0x0000:  4500 0054 6e2a 4000 4001 7bfd 0a14 1e32  E..Tn*@.@.{....2
	0x0010:  0a14 1e28 0800 fafe 0841 0003 cdcf c269  ...(.....A.....i
	0x0020:  0000 0000 9bb0 0a00 0000 0000 1011 1213  ................
	0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223  .............!#
	0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233  $%&()*+,-./0123
	0x0050:  3435 3637                                4567
17:54:22.723632 IP truncated-ip - 8 bytes missing! 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2113, seq 4, length 72
17:54:23.746556 IP 10.20.30.50 > 10.20.30.40: ICMP echo request, id 2113, seq 5, length 64
17:54:23.746606 IP 10.20.30.40 > 10.20.30.50: ICMP echo reply, id 2113, seq 5, length 64
```

# بعدش؟
احتمالن نت وصل شه، باحاله اگه چندتا مکانیزم failure detection رو تو این فضا تست کنیم، یا یه الگوریتم اجماع مثل رفت رو بیاریم داخل این فضا و گند بزنیم به شبکه ببینیم چیکار می‌خواد بکنه.