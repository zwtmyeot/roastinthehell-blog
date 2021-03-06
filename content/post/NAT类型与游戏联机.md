---
title: "NAT类型与游戏联机"
date: 2020-04-08T03:06:08+08:00
draft: false
tags: [
    "网络"
]
---

介绍一下NAT类型为什么会对游戏联机产生影响 <!--more-->

## NAT转换

在当年设计互联网的时候，没有想到发展会这么迅猛，所以即使ipv4的地址数量有 2^32=4,294,967,296 个 ， 也抵挡不住全球人民的消耗。所以NAT（网络地址转换）就应运而生。

当我们位于内网中，想要和公网的服务器（`目标ip:目标端口`）通信时，在经过最外部的具有公网IP路由的时候，就会经过NAT，把 `内网ip:内部端口` 装换成 `公网ip:外部端口`（这个外部端口可能会和内部端口一样，但是大多数时候是不一样），并记录在NAT表中。然后`公网ip:外部端口`与`目标ip:目标端口`通信。

当`目标ip:目标端口`想要和我们通信时，就会使用我们的 `公网ip:外部端口` 来和我们通信，我们这边的路由收到这个数据的时候，就会把 `公网ip:外部端口` 转换成 `内网ip:内部端口`，发给内网中的`内网ip:内部端口`。

## NAT穿透

先说明一下，NAT穿透都是使用UDP协议，不会用TCP协议。首先是TCP协议的延迟问题，然后还有TCP协议端口重用的问题。而且P2P应用中，例如游戏联机，P2P下载，都是使用UDP协议。所以接下来说的都是使用UDP协议的穿透。

一开始NAT表中是没有记录的，如果两个内网设备想要相互通信，必须约定好端口，然后互相向约定好的对方端口发送一个UDP数据包，这样在自己的NAT表中就会有一条记录。下一次对方发数据过来的时候，就能通过NAT转换进来了，这就是NAT穿透。而这个约定，就是要依靠一个公网服务器来实现。

例如内网1中的A想要另外一个内网2中的B进行 P2P 通信。

1. A向服务器S发一条信息，服务器S收到A的信息，知道了A的`公网ip1:外部端口1`

2. B先服务器S发一条信息，服务器S收到了B的信息，知道了B的`公网ip2:外部端口2`

3. 服务器S向A发送B的`公网ip2:外部端口2`，向B发送A的`公网ip1:外部端口1`

4. A向B的`公网ip2:端口2`发一条消息，此时A这边的路由表会记录下一条：A的`内网ip1:内部端口1` --> `公网ip1:外部端口1`。发往B的信息，因为B那边的路由表中没有记录，A这个数据包会被抛弃。

5. B向A的`公网ip1:外部端口1`，此时B这边的路由表会记录下一条：B的`内网ip2:内部端口2` --> `公网ip2:外部端口2`。因为A的路由表在上面的 4 中已经有记录，所以B发的数据包可以通过A的路由，传递给内网中的A（这个是看A、B谁先给对方发，第二个发的就能通过.但是这个没有所谓，只要在自己的NAT表中留下记录就行）。

6. A和B的路由表中都已经留有记录，之后A和B就可以向对方发送数据了。

## NAT类型

在这个NAT的过程中，因为安全性的问题，会有以下4种NAT类型。而正是不同的安全策略，导致了能不能穿透NAT，就是这样从而影响了联机。

- 全锥型(Full Cone)

    经过NAT转换后，在NAT表中留下了 `内网ip:内部端口 <-> 公网ip:外部端口` 这条记录后，公网中任何一个设备，IP2、IP3... 都可以通过这个 `公网ip:外部端口` 和内网里面的设备通信。

    全锥形是最宽松的NAT类型，在这个类型下，游戏联机、P2P下载都不会遇到问题。

    全锥形在PS4里面称为nat1，switch里称为类型A。

- IP受限锥型(Restricted Cone)， 或者说是受限锥型

    IP受限锥型和全锥形的区别是，在NAT会记录下目标ip，如果不是目标ip和自己的 `公网ip:外部端口`通信，路由就会抛弃这个包。所以称为ip受限。

    例如在PS4里开了派对，开好后朋友想连进来却发现连不上。你没有和朋友通信过一次，所以NAT中没有记录到朋友的IP，因为IP受限，所有朋友发给你的数据包就会被路由抛弃掉。

    IP受限锥型在PS4里面称为nat2，switch里称为类型B。

- IP + PORT受限锥型(Port Restricted Cone), 或者说是端口受限锥型

    IP + PORT受限是在IP受限的基础上，再限制端口。NAT会记录下目标IP、目标端口，如果不是`目标IP:目标端口`发过来的数据包，路由就会抛弃。

    例如在PS4派对中，好友连进来后，说话互相听不见。语音数据的传输是在另外一个端口，因为端口受限，这个端口发的数据就被路由抛弃掉了。

    IP + PORT受限锥型在PS4里面称为nat3，switch里称为类型C。

- 对称型(Symmetric)

    如果你是对称型，那么很遗憾，基本是告别P2P了。

    前面3中都是锥型NAT，可以归为同一类。锥型的意思就是内网中，同一个端口和外部多个`目标IP:目标端口` 的通信，NAT时都会绑定同一个外部端口来通信。

    例如，内网一个设备使用 `内网ip:内部端口` 与 公网的 `目标IP1:目标端口1` 、 `目标IP2:目标端口2` 通信时，NAT都会转换到同一个 `外部端口` 。

    然后`目标IP1:目标端口1` 、 `目标IP2:目标端口2`都可以通过 `公网IP:外部端口`和内网里的设备通信。

    而对称型则是，对于每一个不同的`目标IP:目标端口`，都会绑定不同的外部端口。

    例如，内网一个设备使用 `内网ip:内部端口` 与 公网的 `目标IP1:目标端口1` 通信，会进行：`内网ip:内部端口` --> `公网ip:外部端口1`。与`目标IP2:目标端口2`通信时，会进行另外一个转换：`内网ip:内部端口` --> `公网ip:外部端口2`

    这样外部端口是变化的，而且是你不可控的，内网1中的A不知道NAT会转换成什么样的外部端口，所以内网2中的B也就没有办法知道A的外部端口了。虽然可以通过猜端口或者生日算法攻击来进行外部端口的确定，但是这并不是一个可靠的方法。

    对称型在PS4里面是nat4，在switch中就是类型D。

## 游戏联机

主机上很多游戏的联机都是P2P模式，毕竟能省一笔是一笔，厂商怎么可能给你多出一份服务器和维护的钱(`ヮ´)。不过即使是P2P模式，那么也是需要有一个进行预先通信，也就是做NAT穿透作用的服务器。PSN、Switch会员服务就是提供了这样的作用。

游戏房间里面有一个房主，玩家A进入房间时，房主和玩家A都和服务器通信，告知自己的外部端口。然后服务器进行广播，告诉各个玩家，别的玩家的IP和端口。当有新玩家进来时，再次进行广播。

所以如果你是对称型，那么是不可能加的进去房间的。

如果你是端口受限型，可以加入房间，却发现不能语音，可能是语音数据在另外一个端口，因为端口受限，就没办法发送接受发送语音数据了。

还要消除一个误区，NAT类型只是决定了你连不连得通，联机是否顺畅还是要看网络线路质量。如果在公网上绕来绕去的，频繁丢包，即使你是全锥形，那么联机体验还是会非常差，感觉很卡，经常掉线。

网上会有很多教你改善联机的方法，例如改DNS、设置路由器，获取公网IP等。其实大多数只能算是锦上添花，不能作雪中送炭，一步到位解决你的联机困难问题。

改DNS只能改善你的游戏下载速度。因为现在游戏下载都是通过CND服务器下载，而不是厂商自己的下载服务器。CND服务器就是在你附近，上面存有游戏软件的服务器。

DNS要做的就是通过域名，即网址，转换一个CND服务器的IP给你，通过换DNS服务器，可能给你指向一个新的CND服务器。而这个CND服务器你连接速度是比较快的。所以有些人能通过该DNS加速游戏下载。

改路由器设置是减少家庭内NAT的转换，一般来说现在的路由器都不会那么蠢，很少给你使绊子的。

获取公网IP则是打客服电话，说家里要装摄像头。这样一般就会给安排公网IP。这样会减少一次网络运营商那里的 内网-->公网IP 的NAT转换。但是NAT策略还是取决于运营商的设置，和是否是公网IP无关。

想要体验流畅的游戏联机体验花钱是最便捷的选择。

> 加钱，世界触手可及

在某些地区会提供所谓的游戏宽带，或者精品线路。这些线路就是直连外国，不会给你拐得七荤八素的。不过这种线路不是很靠谱，一是价格贵，二是有可能运营商给你用着用着就降线路优先级，和普通宽带没什么差别。

比较靠谱的就是游戏加速器。多数游戏加速器只要一个月30。

游戏加速器的原理就是提供一个加速服务器作为流量转发。因为服务器拥有公网ip，所以不用担心NAT穿透问题，我们只要和加速服务器通信就行。

我们无法控制自己的网络运营商NAT策略，但是加速器商肯定是可以控制自己的NAT策略，不然开什么加速器服务。加速服务器就作为一个正常的玩家一样，和别的玩家通信就行了。把我们的数据包转发给别的玩家，把别的玩家的数据包转发给我们。

像网易UU和奇游都有联机盒子，只要主机连上游戏盒子的wifi信号就能进行游戏加速，使用上还是很方便的。其实这种联机盒子就是一个定制的路由器，加速器商在上面加上了自己的软件，对流量进行分析，如果是游戏流量那么就转发到加速服务器上。

所以我们也可以在自己的可以刷固件的路由器上部署SS或者V2ray服务，用作游戏加速。但是这需要自己选择线路搭建，然后对游戏流量筛选转发。总体来说不如直接买一个加速器算了。
