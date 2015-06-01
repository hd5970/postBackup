title: "Python实现全开放和半开放端口扫描"
date: 2015-05-02 16:58:38
tags: [Pyhton, Network]
---
今天在练习python,熟练掌握python脚本在linux上实现自动化或半自动化测试.

这个程序仅适用于Linux,稍微修改可用于Windows,至于OS X,在我上一次写的构造icmp报文中socket.sendto()方法没有出问题,但是这里的sendto方法出了问题.
<!--more-->

OS X上python的RAW Socket用的一直不是很好,Google了一下别人也有同样的问题:Perhaps your issue is related to some Mac OS X brain damage with raw sockets?这个问题在安装Python 2.7.9并更新shell环境之后解决.
```python
#!/usr/bin/env  python
# -*- coding: UTF-8 -*-
__author__ = "SunQi, 2012210465"
import random
import fcntl
import thread
import os
import time
import getopt
import socket
from struct import *
import struct
import sys

open_port_count = 0
start = 0
stop = 65535
socket.setdefaulttimeout(1000)


def checksum(msg):
    s = 0
    for i in range(0, len(msg), 2):
        w = (ord(msg[i]) << 8) + (ord(msg[i + 1]) )
        s += w

    s = (s >> 16) + (s & 0xffff)
    s = ~s & 0xffff
    return s


def create_socket():
    # create a raw socket
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_TCP)
    except socket.error, msg:
        print 'Socket could not be created.  Error: ', str(msg[0]), ' Message: ', msg[1]
        sys.exit()

    s.setsockopt(socket.IPPROTO_IP, socket.IP_HDRINCL, 1)
    return s


def create_ip_header(source_ip, dest_ip):
    packet = ''

    # ip header fields
    header_len = 5
    version = 4
    tos = 0
    tot_len = 20 + 20
    id = random.randrange(18000, 65535, 1)
    frag_off = 0
    ttl = 255
    protocol = socket.IPPROTO_TCP
    check = 10
    saddr = socket.inet_aton(source_ip)
    daddr = socket.inet_aton(dest_ip)
    hl_version = (version << 4) + header_len
    ip_header = pack('!BBHHHBBH4s4s', hl_version, tos, tot_len, id, frag_off, ttl, protocol, check, saddr, daddr)
    return ip_header


def create_tcp_syn_header(source_ip, dest_ip, dest_port):
    # tcp header fields
    source = random.randrange(32000, 62000, 1)  # source port
    seq = 0
    ack_seq = 0
    doff = 5
    # tcp flags
    fin = 0
    syn = 1
    rst = 0
    psh = 0
    ack = 0
    urg = 0
    window = socket.htons(8192)  # maximum window size
    check = 0
    urg_ptr = 0
    offset_res = (doff << 4) + 0
    tcp_flags = fin + (syn << 1) + (rst << 2) + (psh << 3) + (ack << 4) + (urg << 5)
    tcp_header = pack('!HHLLBBHHH', source, dest_port, seq, ack_seq, offset_res, tcp_flags, window, check, urg_ptr)
    # pseudo header fields
    source_address = socket.inet_aton(source_ip)
    dest_address = socket.inet_aton(dest_ip)
    placeholder = 0
    protocol = socket.IPPROTO_TCP
    tcp_length = len(tcp_header)
    psh = pack('!4s4sBBH', source_address, dest_address, placeholder, protocol, tcp_length);
    psh += tcp_header
    tcp_checksum = checksum(psh)
    tcp_header = pack('!HHLLBBHHH', source, dest_port, seq, ack_seq, offset_res, tcp_flags, window, tcp_checksum,
                      urg_ptr)
    return tcp_header


def range_scan(source_ip, dest_ip, start_port, end_port):
    syn_ack_received = []  # store the list of open ports here
    # final full packet - syn packets don't have any data
    for j in range(start_port, end_port):
        s = create_socket()
        ip_header = create_ip_header(source_ip, dest_ip)
        tcp_header = create_tcp_syn_header(source_ip, dest_ip, j)
        packet = ip_header + tcp_header
        s.sendto(packet, (dest_ip, 0))
        data = s.recvfrom(1024)[0][0:]
        ip_header_len = (ord(data[0]) & 0x0f) * 4
        tcp_header_len = (ord(data[32]) & 0xf0) >> 2
        tcp_header_ret = data[ip_header_len:ip_header_len + tcp_header_len - 1]
        if ord(tcp_header_ret[13]) == 0x12:  # SYN/ACK flags set
            print(ip_dst + ":" + str(j))
            syn_ack_received.append(j)
    return syn_ack_received


def _usage():
    print '''
Usage:
    Caution:argument '-c' or '-s' should be given at last

            -c  (Socket Connect) Scan the port of the node, MultiThread
                    python port_scan.py -c <ip>

            -s  (TCP SYN) Scan the port of the node
                    python port_scan.py -s <ip>

            -t  set socket default timeout(t)
                    python port_scan.py -t <time> -c <ip>

            -f  from port <a>
                    python port_scan.py -f <a> -e <a> -s/-c <ip>

            -e  end port <a>
                    default is 65535

            -h  Display this help and exit
    '''
    print '\033[32;1mExit\n\033[0m'
    sys.exit(1)


def socket_port(ip, port):
    global open_port_count
    if port > 65535:
        print 'Port scanning beyond the port range,ending of the scan'
        sys.exit()
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    result = s.connect_ex((ip, port))
    if result == 0:
        print ip, u":", port
        open_port_count += 1
    s.close()


def start_scan(ip, from_port, end_port):
    for port in range(from_port, end_port + 1):
        thread.start_new_thread(socket_port, (ip, int(port)))
        # To ensure that the first operation method in Seeker
        time.sleep(0.01)


def _error(err):
    if __name__ == '__main__':
        print "%s: %s" % (os.path.basename(sys.argv[0]), str(err))
        print "Try `%s --help' for more information." % os.path.basename(sys.argv[0])
        sys.exit(1)
    else:
        raise Exception, str(err)


def get_ip_address(interface):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    return socket.inet_ntoa(fcntl.ioctl(
        s.fileno(),
        0x8915,
        struct.pack('256s', interface[:15])
    )[20:24])


if __name__ == '__main__':
    t = 0
    try:
        # opts = arguments recognized,
        # args = arguments NOT recognized (leftovers)
        opts, args = getopt.getopt(sys.argv[1:-1], "tcs")
    except getopt.GetoptError:
        # print help information and exit:
        _error("illegal option(s) -- " + str(sys.argv[1:]))

    if len(sys.argv) >= 2:
        node = sys.argv[-1:][0]  # host to be pinged
        if node[0] == '-' or node == '-h' or node == '--help':
            _usage()

    for o, a in opts:
        if o == "-h" or o == "--help":  # display help and exit
            _usage()
            sys.exit(0)

        if o == "-t":
            try:
                timeout = int(a)
                socket.setdefaulttimeout(timeout)
            except:
                _error("invalid timeout: '%s'" % str(a))
        if o == "-f":

            if a < 0:
                _error("port error")
            start = int(a)
        if o == "-e":

            if a > 65535:
                _error("port error")
            stop = int(a)

        if o == "-c":
            t = time.time()
            start_scan(node, start, stop)
            print ''
            print '\033[32;1mOpen port count %s,Duration: %f\n\033[0m' % (open_port_count, time.time() - t)

        if o == "-s":
            t = time.time()
            open_port_list = []
            ip_src = get_ip_address("eth0")
            ip_dst = node
            step = 1
            scan_ports = range(start, stop, step)
            if scan_ports[len(scan_ports) - 1] < stop:
                scan_ports.append(stop)
            for i in range(len(scan_ports) - 1):
                opl = range_scan(ip_src, ip_dst, scan_ports[i], scan_ports[i + 1])
                open_port_list.append(opl)
            print 'A list of all open ports found: '
            for i in range(len(open_port_list)):
                for j in range(len(open_port_list[i])):
                    print open_port_list[i][j]
```

