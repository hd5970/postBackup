title: "Python实现探测链路MTU"
date: 2015-04-23 22:45:49
tags: [Pyhton, Network]
---
经过一次惨痛的手滑,我损失了1TB硬盘上的全部数据,当然也包括这个Hexo的博客目录,以前的博文也没有留下md文件,只有留下的html文件,把html还原到md文件实在太麻烦,并且需要手动调整.现在把hexo目录调整到Dropbox的自动备份目录,以后不会出现这种事情了.从前的文章在http://hd5970.github.io 可以找到
<!--more-->
本来我是觉得这个实验一点意思都没有的,RFC里面PMTUD写的清清楚楚,也没有什么意思.但是看到不少精通网络的业内人士连IP头是什么部分都说不清楚,VLAN为什么只有4094个都不知道.想了想,我好像也不知道IP头具体是什么组成,就拿这个作业复习下IP头吧,顺便练习一下刚学的Python.

##MTU介绍
有的人把MTU分为二层MTU和三层MTU,对于以太网来说,二层MTU就是1518字节,但是这里我指的MTU就是三层MTU,也就是一个以太网帧包含的IP报文大小,是1500字节,去掉20个IP头,8个ICMP头,剩下1472的DATA.

RFC 1191 描述了“路径最大传输单元发现方法”，这是一种确定两个 IP 主机之间路径最大传输单元的技术，其目的是为了避免 IP 分片。在这项技术中，源地址将设置数据报的 DF（Don't Fragment，不要分片）标记位，再逐渐增大发送的数据报的大小——路径上任何需要将分组进行分片的设备都会将这种数据报丢弃并返回一个“数据报过大”的 ICMP 响应到源地址——这样，源主机就“获取”到了不用进行分片就能通过这条路径的最大的最大传输单元了。

一般来说MTU的取值是576到1500字节,那么如果我按照RFC的方法,从576开始向上加,那也太麻烦了,因为我们大多数的环境都是以太网,再加上我们无法构造出比本机的MTU更大的报文,所以我决定从1500开始减小报文长度,一个一个知道找出能ping通的最大地址.OK,既然都到这里了,你不觉得使用这种顺序查找法是一种很麻烦的嘛?不如使用常用的二分法来查找好了.

```Python
#!/usr/bin/env python
# -*- coding: UTF-8 -*-
__author__ = 'SunQi 2012210465 '
import sys
import socket
import struct
import os
import array
import string
import select
import getopt

ICMP_TYPE = 8
ICMP_TYPE_IP6 = 128
ICMP_CODE = 0
ICMP_CHECKSUM = 0
ICMP_ID = 0
ICMP_SEQ_NR = 0
IP_MTU_DISCOVER = 10
IP_PMTUDISC_DONT = 0
IP_PMTUDISC_DO = 2
mid = 0


def _in_cksum(packet):
    if len(packet) & 1:
        packet += '\0'
    words = array.array('h', packet)
    chksums = 0
    for word in words:
        chksums += (word & 0xffff)
    hi = chksums >> 16
    lo = chksums & 0xffff
    chksums = hi + lo
    chksums += chksums >> 16
    return (~chksums) & 0xffff


def _search(size, pong, mid):
    if not (1472 >= size >= 548):
        print "illegal size option"

    while 1:
        pong = pingNode(alive=alive, timeout=timeout, number=count, node=node)
        if not pong:
            mid = (size - 548) // 2
            size -= mid
            print "dead"
            print mid, size
            return size, mid
        else:
            print "live"
            if mid == 0:
            size += 1

        mid //= 2
        size += mid
        print mid, size
        return size, mid


def _error(err):
    if __name__ == '__main__':
        print "%s: %s" % (os.path.basename(sys.argv[0]), str(err))
        print "Try `%s --help' for more information." % os.path.basename(sys.argv[0])
        sys.exit(1)
    else:
        raise Exception, str(err)


def construct(sid, size):
    data = ""
    data += size * "X"
    header = struct.pack('bbHHh', ICMP_TYPE, ICMP_CODE, ICMP_CHECKSUM,
                         ICMP_ID, ICMP_SEQ_NR + sid)
    packet = header + data
    checksum = _in_cksum(packet)
    header = struct.pack('bbHHh', ICMP_TYPE, ICMP_CODE, checksum, ICMP_ID,
                         ICMP_SEQ_NR + sid)
    packet = header + data
    return packet


def pingNode(alive=0, timeout=1.0, number=sys.maxint, node=None,
             size=1472):
    dst_ip = socket.gethostbyname(node)
    test_mid = 0
    if not node:
        _error("")
    try:
        host = socket.gethostbyname(node)
    except:
        _error("cannot resolve %s: Unknown host" % node)
    if int(string.split(dst_ip, ".")[-1]) == 0:
        _error("no support for network ping")
    if number == 0:
        _error("invalid count of packets to transmit")
    if alive:
        number = 1
    start = 1

    lost = 0

    if not alive:
        print "PING %s (%s): %d data bytes (20+8+%d)" % (str(node), str(dst_ip),
                                                         20 + 8 + size, size)

    try:
        while start <= number:
            try:
                pingSocket = socket.socket(socket.AF_INET, socket.IPPROTO_ICMP)
                pingSocket.setsockopt(socket.SOL_IP, IP_MTU_DISCOVER, IP_PMTUDISC_DO)
            except socket.error, e:
                print "socket error: %s" % e
                _error("You must be root (%s uses raw sockets)" % os.path.basename(sys.argv[0]))

            packet = construct(start, size)
            try:
                pingSocket.sendto(packet, (node, 1))
            except socket.error, e:
                _error("socket error: %s" % e)

            pong = 0
            if size + 28 <= 748:
                    pong = "1"
            """
            iwtd = []
            while 1:
                iwtd, owtd, ewtd = select.select([pingSocket], [], [], timeout)
                break
            if iwtd:
                pong, address = pingSocket.recvfrom(size + 48)
                lost -= 1

                pongHeader = pong[20:28]
                pongType, pongCode, pongChksum, pongID, pongSeqnr = \
                    struct.unpack("bbHHh", pongHeader)

                if not pongSeqnr == start:
                    pong = None

            if not pong:
                if alive:
                    print "no reply from %s (%s)" % (str(node), str(host))
                else:
                    print "ping timeout: %s (icmp_seq=%d) " % (host, start)
                    print "Current mtu is %d" % (size + 28)
                    size, test_mid = _search(size, pong, mid)
                    start += 1
                continue

            size, test_mid = _search(size, pong, test_mid)
            """
    except (EOFError, KeyboardInterrupt):
        start += 1
        pass
    return pong
    pingSocket.close()


def _usage():
    print """link mtu finder
            python ping.py <ip_address>
    """


if __name__ == "__main__":
    """ Main loop
        """
    # version control
    version = string.split(string.split(sys.version)[0][:3], ".")
    if map(int, version) < [2, 3]:
        _error("You need Python ver 2.3 or higher to run")
    try:
        opts, args = getopt.getopt(sys.argv[1:-1], "hat:6c:fs:")
    except getopt.GetoptError:
        _error("fucking illegal option")
    if len(sys.argv) >= 2:
        node = sys.argv[-1:][0]
        if node[0] == '-':
            _usage()
    else:
        _error("No arguments given")
    if args:
        _error("illegal option -- %s" % str(args))
    alive = 0
    timeout = 1.0
    count = sys.maxint
    flood = 0

    sys.exit()
```
##MTU在实际校园网中的问题
前面说到了RFC中规定的MTU方法,也就是PMTUD(Path MTU Discovery)是由内核自动完成的,但是实际生活中MTU还是会造成很多意想不到的问题,我们学校有一栋楼有个很奇怪的问题,都是PPPoE拨号,Windows的机器可以正常拨号上网没有问题,但是Linux/OS X等系统的机器却没有办法上网.实际我发现,Windows的PPPoE
拨号的默认MTU是1480,也就是说,大于1480的包会被Windows内核拆分(DF的丢弃)并不会递交给网卡驱动.而UNIX/LINUX系统的默认PPPoE MTU是1492,非常有趣,这个数值是一个恰好的数值,1492+6个PPPoE头+2个PPP头正好是以太网的MTU.在交换机上使用镜像端口抓包发现,交换机被运维人员配置了QinQ,本来学校的交换机的二层MTU是1522字节,恰好是1500IP+18以太网(包括末尾的CRC校验)+4VLAN tag,但是部署了QinQ之后,又占用了4个字节,也就是说MTU最大只支持到1488字节,所以LINUX系统的报文被丢弃了!虽然主流操作系统都配备了PMTUD模块,但是还是没有正确的侦测到MTU,也许它只侦测到了离主机最近的交换机的MTU.

##MTU在Overlay中的问题
这里我主要说的是MTU设置不当在一中新型的Overlay技术-VXLAN中的问题,如果你是搭建一些云计算平台用到了VXLAN技术,因为VXLAN的封包至少需要1554字节(不包括末尾CRC校验),也就是这个VXLAN报文从VTEP上封装好,交付给物理交换机的时候,可能会被物理交换机丢弃.
你可能会问,VXLAN是一个UDP报文,IP报文不是可以分片的吗?可惜的是,包括现在的Open vSwitch等软件Plugin实现的VTEP,还是一些做Offload,VPEA的VXLAN交换机,都不支持自动分包,我猜想可能是因为性能的原因,咨询了业内人士,得到的答案是,结合自己的推测,第一UDP报文并不像TCP,UDP分包如果没有上层协议的控制,很有可能导致错位.第二交换机芯片并不支持,IP报文分包可能需要交付上层CPU处理,影响了报文转发速度.所以现在的做法是手动调高交换机的MTU,但是这个交换机并不需要支持VXLAN.
