title: "Distributed ONOS Application Tutorial"
date: 2015-04-23 23:17:47
tags: [ONOS, Java, OSGi]
---
ONOS是一款新兴的控制器,也是目前唯一一款支持分布式控制的控制器,关于SDN控制器分布式的优点并不是本文的主题,但是可以肯定的是,考虑到负载均衡和多机备份的情况,在大规模网络中必须要求控制器支持分布式.本文主要引导您体验ONOS的分布式部署和分布式应用开发,至于ONOS选择的分布式协作系统,从ZooKeeper到Hazelcast再到Raft,将在以后的文章中介绍(其实我大三没空也看不懂分布式协作系统...)
<!---more--->
##Dependeces
> * 和ODL一样,ONOS采用了Maven + OSGi框架,本文将带领你构建三个OSGi Bundle.
> * 首先,你需要阅读这篇[官网教程][1],我的代码就是在他们非常非常简陋的代码上一步步Flesh Out
> * 其次,你需要下载ONOS官网提供的虚拟机,因为ONOS在BlackBird版本中放弃使用了Docker部署,所以我原先提取到的Dokcer镜像只有1.0版本,BlackBird 1.1.1版本并没有上传Maven Center,所以我也没法制作1.1.1版本的Docker镜像,这里是我上传的ONOS虚拟机[MEGA][2]网盘分流地址,文件后缀名为".zip".
> * 和其他很多开源项目一样,ONOS的Wiki中也有很多大坑,如果你是新手,这里有我二次开发的APP源码,已经上传到了[GitHub][3],也就是本文中开发APP的答案.注意每个Branch对应一次"MCI"(Maven Clean Install),欢迎Star,Fork,有问题请发Issue.

![GitHub分支][4]

##Verifying that ONOS is deployed
今天我们要做的APP实现功能是创建并管理一个私有的Network,关于这个network到底指的是什么意思?笔者认为这里的Network是指的同一个广播域,VLANID相同的网络,并不是指Subnet(一个IP网段).
首先按照wiki上面的内容在Docker或者LXC,也就是namespace上面创建三个instance,至于namespace和Docker的对比,将在以后的博文中介绍.
![部署][5]
你可能需要在master控制器上安装onos-app-fwd来实现二层转发,不过也许你并不知道哪个是master控制器,那你就要在三个上面全部安装,以免出现仅有部分网络互通的情况.

##Writing 'Build Your Own Network
第一次MCI对应的分支是SimpleImplement,比较简单这里就不再说了,第二次MCI对应的是SimpleNetworkStore.
```bash
onos> list-networks
onos> create-network test
Created network test   
onos> add-host test 00:00:00:00:00:01/-1
Added host 00:00:00:00:00:01/-1 to test  
onos> add-host test 00:00:00:00:00:02/-1 #fixme
Added host 00:00:00:00:00:02/-1 to test
onos> list-networks
test
    00:00:00:00:00:01/-1
    00:00:00:00:00:02/-1
onos> intents
id=0x0, state=INSTALLED, type=HostToHostIntent, appId=org.onos.byon
    constraints=[LinkTypeConstraint{inclusive=false, types=[OPTICAL]}]
```
这里管理网络的ID就是mac地址,在onos中定义为hostID,是mac地址和vlan ID两部分组成,"-1"表示VLAN为NONE.
如果你是按照官网的addToMesh方法完成项目,这里当你输入intents命令的时候,显示的状态是:
```bash
onos> intents
id=0x0, state=FAILED, type=HostToHostIntent, appId=org.onos.byon
    constraints=[LinkTypeConstraint{inclusive=false, types=[OPTICAL]}]
```
而且在mininet运行ping发现h1 ping h2根本就不通,交换机也没有装载二层转发相关的流表.
这里我们需要修复一下Intent,修改NetworkManager.java的addToMesh方法.
```java
    private Set<Intent> addToMesh(HostId src, Set<HostId> existing) {
        if (existing.isEmpty()) {
            return Collections.emptySet();
        }
        Set<Intent> submitted = new HashSet<>();
        existing.forEach(dst -> {
            if (!src.equals(dst)) {
                TrafficSelector selector = DefaultTrafficSelector.emptySelector();
                TrafficTreatment treatment = DefaultTrafficTreatment.emptyTreatment();

                Intent intent = new HostToHostIntent(appId, src, dst, selector, treatment);
                submitted.add(intent);
                intentService.submit(intent);
            }
        });
        return submitted;
    }
```
ok,现在应该就可以了,试试h1 ping h2吧,你可以在交换机上看到对应的二层转发流表,至于Intent FrameWork,将在后续的博文中详细介绍.
![flows][6]
可以清楚的看到二层互通的流表.
你可能还无法让mac地址为1和为2的两个主机互通,或者h1 和h2只能ping通前几个包,后面就无法ping通,那么这到底是为什么呢?因为我们的程序仅仅做了流表下发,其实并没有ARP Handle的模块,h1无法获取到h2的mac地址,所以我要求你先安装onos-app-fwd来为h1和h2做arp处理,让h1,h2相互获取到对方的mac地址.但是mininet里面的host毕竟不是真实的主机,里面的arp表项老化的非常快,所以就会出现只能ping通几个包的问题.那么如何确认是我们的程序下发了流表呢?如果你没有运行我的程序直接卸载了fwd模块,那么交换机的二层相关流表被清空了,这时候迅速的添加私有网络,会看到h1和h2的通信又恢复了正常.
##Flesh Out the CLI
超简单的模块,只要按照addHost等命令添加类并重新修改对应的方法就好了,记得在shell-config里面声明,并且别忘记修改注释!这里面唯一能让我觉得兴奋的就只有这个类了
```java
public class NetworkCompleter implements Completer {

    @Override
    public int complete(String buffer, int cursor, List<String> candidates) {
        // Delegate string completer
        StringsCompleter delegate = new StringsCompleter();

        NetworkService service = AbstractShellCommand.get(NetworkService.class);
        delegate.getStrings().addAll(service.getNetworks());

        // Now let the completer do the work for figuring out what to offer.
        return delegate.complete(buffer, cursor, candidates);

    }

}
```
能不能猜到这个类是做什么的?在Shell里面Tab补全的功能,如果你用的是remove-host的方法,还需要一个hostIdCompleter的类.
##Network Events
 A delegate is a manager which is receiving events from a neighbouring store for the purpose of either taking action on the store event or notifying listeners. So we are going to add NetworkListener.
 Now since the store is making use of the delegates, we need to make the stores' implementation delegate-aware by notifying the delegates when network elements are added or removed. For example, the putNetwork method no notifies the delegate that a new network has been added.
这里有用到Java8的[Stream][7] API
```java
    public Set<Intent> removeIntents(String network, HostId hostId) {
        Set<Intent> intents = checkNotNull(intentsPerNet.get(network).stream().map(intent -> (HostToHostIntent)intent)
                .filter(intent -> intent.one().equals(hostId) || intent.two().equals(hostId)).collect(Collectors.toSet()));
        intentsPerNet.get(network).remove(intents);
        return intents;
    }
```
###Delegate-Aware
```java
public Set<HostId> addHost(String network, HostId hostId) {
        Set<HostId> hosts = checkNotNull(networks.get(network),"Please create the network first");
        boolean added = hosts.add(hostId);
        if (added){
            notifyDelegate(new NetworkEvent(NetworkEvent.Type.NETWORK_UPDATED, network));
        }
        return added ? ImmutableSet.copyOf(hosts) : Collections.emptySet();
    }
```
##Distributed Store && Eventlistener
注释掉SimpleNetworkStore中的OSGi声明
代码改动实在太多了,请移步上文的github地址中的EventListener分支.
运行结果在onos log里面可以查看
```bash
$ol
```
![安装][8]

![onos-log][9]

![removed][10]

![操作][11]

![updated][12]

![added][13]

---


  [1]: https://wiki.onosproject.org/display/ONOS/Distributed+ONOS+Tutorial
  [2]: https://mega.co.nz/#!WptxCbxS!ez8HoC-LS4k0kKe-G0JE4jZahYFX6dGGYYDqeSNFQFk
  [3]: https://github.com/hd5970/onos-byon
  [4]: http://sunqi-me.qiniudn.com/onos-github.png
  [5]: http://sunqi-me.qiniudn.com/onos3.png
  [6]: http://sunqi-me.qiniudn.com/onos-flows.png
  [7]: http://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/
  [8]: http://sunqi-me.qiniudn.com/r3.png
  [9]: http://sunqi-me.qiniudn.com/r4.png
  [10]: http://sunqi-me.qiniudn.com/r1.png
  [11]: http://sunqi-me.qiniudn.com/r2.png
  [12]: http://sunqi-me.qiniudn.com/r6.png
  [13]: http://sunqi-me.qiniudn.com/r8.png