# packetbeat数据库审计


## 作用
* 全量日志收集
* 追溯SQL来源
* 排查SQL性能
* 图形展示访问趋势
* 合规需求
## 准备环境

* Docker
* Docker-compose
* CentOS 7 
* images（确定能拉取外网源，如果不行，请自行安装源elasticsearch:7.14.2、kibana:7.14.2、elastic/packetbeat:7.14.2）

## 部署

**设置宿主机内核参数**

```
echo vm.max_map_count=655350 >> /etc/sysctl.conf
sysctl -p
```

**启动**

```
mkdir -p /data1/esauditdata && chmod -R 777 /data1/esauditdata
https://github.com/oxff644/packetbeat-dbaudit.git
cd packetbeat-dbaudit
docker-compose up -d
```

**访问服务**

```
http://docker-compose-ip:5601
```

<font color=#FF0000 >**密码**</font> 

```
# 默认密码
elastic:elasticdocker
```
<font color=#FF0000 >**清理**</font> 

```
docker-compose down -v 
rm -r /data1/esautditdata && rm -r packetbeat-dbaudit-compose
```

---

## 项目说明

* 目前使用的版本为7.16.2
* 登陆kibana后，配置kibana步骤：(kibana功能非常丰富)
```
先创建“索引模式”，在通过Discover查看:
Stack Management--->Index Management 能看到相关的es index信息了（如果有流量会自动生成index）
Stack Management--->index Patterns 如果index生成后，通过index pattern，
创建成功，Analytics--->discover 就可以展示相关的信息
```
* 根据需求，修改 packetbeat/config/packetbeat.yml 相关模块的port，进行监听。

## 存在问题

* packetbeat收集了monogdb返回流量，可能造成内存占用过高，可注释packetbeat相关代码(opReplyParse函数)解决该问题
  或者可通过只采集请求流量规避该问题
* mongodb msg 无法正常展示：
mongodb 在3.6版本中，增加了op_msg 协议，目前packetbeat 在msg统计的时候，没有输出msg内的内容。可在
protos/mongodb/mongodb_parser.go [opMsgParse函数]增加对返回的解析即可解决该问题。

## 使用扩展
项目主要为packetbeat+es

### packetbeat+wazuh

只需要将packetbeat.yml 中关于ES、kibana的配置注释掉，将packetbeat的日志存到本地即可
```
packetbeat.interfaces.device: eth0
packetbeat.interfaces.snaplen: 1514
packetbeat.interfaces.type: af_packet
packetbeat.interfaces.buffer_size_mb: 64

tags: ["mongodb", "redis", "mysql","pgsql"]


#================================== Flows =====================================

# Set `enabled: false` or comment out all options to disable flows reporting.
packetbeat.flows:
  # Set network flow timeout. Flow is killed if no packet is received before being
  # timed out.
  timeout: 30s

  # Configure reporting period. If set to -1, only killed flows will be reported
  period: 10s
  enabled: false

#========================== Transaction protocols =============================

packetbeat.protocols:
- type: icmp
  # Enable ICMPv4 and ICMPv6 monitoring. Default: false
  enabled: false

- type: amqp
  # Configure the ports where to listen for AMQP traffic. You can disable
  # the AMQP protocol by commenting out the list of ports.
  #ports: [5672]
  enabled: false

- type: cassandra
  #Cassandra port for traffic monitoring.
  #ports: [9042]
  enabled: false

- type: dhcpv4
  # Configure the DHCP for IPv4 ports.
  #ports: [67, 68]
  enabled: false

- type: dns
  # Configure the ports where to listen for DNS traffic. You can disable
  # the DNS protocol by commenting out the list of ports.
  #ports: [53]
  enabled: false

- type: http
  # Configure the ports where to listen for HTTP traffic. You can disable
  # the HTTP protocol by commenting out the list of ports.
  # ports: [3389, 7001, 8088, 8080, 80, 443]
  enabled: false

- type: memcache
  # Configure the ports where to listen for memcache traffic. You can disable
  # the Memcache protocol by commenting out the list of ports.
  #ports: [11211]
  enabled: false

- type: mysql
  ports: [3306,3872]
  processors:
  - drop_event:
      when:
        or:
          - equals:
             method: SET
          - equals:
             method: SHOW

- type: pgsql
  ports: [5432]

- type: redis
  ports: [6379]

- type: thrift
  enabled: false

- type: mongodb
  ports: [27017]

- type: nfs
  # Configure the ports where to listen for NFS traffic. You can disable
  # the NFS protocol by commenting out the list of ports.
  # ports: [2049]
  enabled: false

- type: tls
  # Configure the ports where to listen for TLS traffic. You can disable
  # the TLS protocol by commenting out the list of ports.
 # ports:
  #  - 443   # HTTPS
  #  - 993   # IMAPS
  #  - 995   # POP3S
  #  - 5223  # XMPP over SSL
  #  - 8443
  #  - 8883  # Secure MQTT
  #  - 9243  # Elasticsearch
  enabled: false
# ------------------------------ File Output -------------------------------
output.file:
  path: "/var/logs/packetbeat"
  filename: packetbeat.log
  rotate_every_kb: 102400
```
然后利用wazuh agent采集即可

* 配置wazuh规则

agent.conf
```
<localfile>
    <log_format>json</log_format>
    <location>/var/logs/packetbeat/packetbeat.log</location>
  </localfile>
```
packetbeat.yml

```
<!-- Modify it at your will. -->
<group name="linux,packetbeat,">
    <rule id="200301" level="3">
        <decoded_as>json</decoded_as>
        <field name="network.protocol">mongodb</field>
        <mitre>
          <id>T1078</id>
        </mitre>
        <description>Notice: Mongodb log</description>
        <options>no_full_log</options>
     </rule>
     <rule id="200302" level="3">
        <decoded_as>json</decoded_as>
        <field name="network.protocol">mysql</field>
        <mitre>
          <id>T1078</id>
        </mitre>
        <description>Notice: mysql log</description>
        <options>no_full_log</options>
     </rule>
    <rule id="200303" level="3">
        <decoded_as>json</decoded_as>
        <field name="network.protocol">redis</field>
        <mitre>
          <id>T1078</id>
        </mitre>
        <description>Notice: redis log</description>
        <options>no_full_log</options>
     </rule>
     <rule id="200304" level="3">
        <decoded_as>json</decoded_as>
        <field name="network.protocol">pgsql</field>
        <mitre>
          <id>T1078</id>
        </mitre>
        <description>Notice: pgsql log</description>
        <options>no_full_log</options>
     </rule>
</group>
```


