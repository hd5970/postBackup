title: "Maven remote repositories configuration"
date: 2015-05-09 16:08:07
tags: [Maven, XML]
---
今天使用Maven archetype生成工程的时候突然发现无法生成,以为是墙的原因,挂了Proxychains之后还是不行.

打开https://repo.maven.apache.org/maven2/ 才发现,远程库已经移到 http://search.maven.org.
<!--more-->
好吧,来修改mvn中央库的地址,使用settings.xml来配置,settings.xml可以配置在很多地方,下面是优先级:

``` bash
# The following locations are checked for the existence of the settings.xml file
#   * 1. looks for the specified url
#   * 2. if not found looks for ${user.home}/.m2/settings.xml
#   * 3. if not found looks for ${maven.home}/conf/settings.xml
#   * 4. if not found looks for ${M2_HOME}/conf/settings.xml
#
#org.ops4j.pax.url.mvn.settings=
```

```bash
cd ~/.m2
vi settins.xml
```

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <profiles>
    <profile>
      <repositories>
        <repository>
          <id>codehausSnapshots</id>
          <name>Codehaus Snapshots</name>
          <releases>
            <enabled>false</enabled>
            <updatePolicy>always</updatePolicy>
            <checksumPolicy>warn</checksumPolicy>
          </releases>
          <snapshots>
            <enabled>true</enabled>
            <updatePolicy>never</updatePolicy>
            <checksumPolicy>fail</checksumPolicy>
          </snapshots>
          <url>http://search.maven.org/</url>
          <layout>default</layout>
        </repository>
      </repositories>
    </profile>
  </profiles>
</settings>
```
当然你也可以在这个setting.xml中配置一些其他的功能,比如代理,支持socks4,5以及http代理.Maven现在也支持https连接,只要在url中改为https即可.其他介绍请看下面的引用连接.

##Reference:

https://maven.apache.org/settings.html

