# 流量監控

此SDN專案是以不斷發出```OFPFlowStatsRequest```和```OFPPortStatsRequest```請求，來取得網路狀況的統計資訊。

## 匯入模組
```python
from operator import attrgetter
from ryu.app import simple_switch_13
from ryu.controller import ofp_event
from ryu.controller.handler import MAIN_DISPATCHER, DEAD_DISPATCHER
from ryu.controller.handler import set_ev_cls
from ryu.lib import hub
import json
```
## 模組簡介  

### attrgetter 函數

Python 的 operator 模組提供一種簡潔、有效率的```attrgetter```函數，它是專門用來對輸出的資訊進行排序。在使用```sorted```排序時，搭配```attrgetter```函數可以簡化對參數```key```的描述。使用方式如下：

```python
sorted(objects, key= attrgetter('Attribute_A'))
```

objects 在排序時，會針對 objects 的"屬性"```Attribute_A```進行排序。

### 繼承 SimpleSwitch13

在此SDN專案中，藉由繼承```SimpleSwitch13```的方式來實現 Switch 的基本功能。

### ofp_event

當連上的 Switch 傳送 OpenFlow 訊息時， ofp_event 模組會將該訊息轉換成事件類別。

### Switch 和 Controller 之間的連線狀況

* MAIN_DISPATCHER：一般狀態（完成 Handshaking 的動作）
* DEAD_DISPATCHER：連線中斷 (Disconnected)

### set\_ev\_cls

透過 set\_ev\_cls 裝飾器（decorator），依接收到的參數（事件類別、Switch 狀態），而進行反應。

### hub

是用來負責執行緒（Threads）的工作。

### json
使用者
將統計數據的資料型態轉換成```json```格式，以利於使用者後續的觀察及分析。

## 初始化

```python
class SimpleMonitor(simple_switch_13.SimpleSwitch13):

    def __init__(self, *args, **kwargs):
    
        super(SimpleMonitor, self).__init__(*args, **kwargs)
        self.datapaths = {}
        self.monitor_thread = hub.spawn(self._monitor)
```

### self.datapaths = {}

用來儲存 Switch 在監測中的資訊。

### self.monitor\_thread = hub.spawn(self.\_monitor)

建立執行緒，並將```_monitor```方法放入執行緒中，不斷執行```取得統計狀況請求```。

## EventOFPStateChange 事件 (監視Switch的連線狀態)

在此可以接收到 Switch 狀態改變的訊息，藉由修改```self.datapath```，讓```self.datapath```只存放在目前運作中的 Switch。  

```python
@set_ev_cls(ofp_event.EventOFPStateChange,
            [MAIN_DISPATCHER, DEAD_DISPATCHER])
def _state_change_handler(self, ev):
    datapath = ev.datapath
    if ev.state == MAIN_DISPATCHER:
        if not datapath.id in self.datapaths:
            self.logger.debug('register datapath: %016x', datapath.id)
            self.datapaths[datapath.id] = datapath
    elif ev.state == DEAD_DISPATCHER:
        if datapath.id in self.datapaths:
            self.logger.debug('unregister datapath: %016x', datapath.id)
            del self.datapaths[datapath.id]
```
* 如果狀態是```MAIN_DISPATCHER```且不在```self.datapath```中，則放入```self.datapath```。（Switch 連接中）
* 如果狀態是```DEAD_DISPATCHER```且在```self.datapath```中，則從```self.datapath```中移除。（Switch 中斷連結）

## def \_monitor(self):

執行緒每10秒向監測中的 Switch 發送要求以取得統計資訊。

```python
def _monitor(self):
    while True:
        for dp in self.datapaths.values():
            self._request_stats(dp)
        hub.sleep(10)
```
## def \_request\_stats(self, datapath):
建立```OFPFlowStatsRequest```與```OFPPortStatsRequest```請求訊息並發送。

```python
def _request_stats(self, datapath):
    self.logger.debug('send stats request: %016x', datapath.id)
    ofproto = datapath.ofproto
    parser = datapath.ofproto_parser

    req = parser.OFPFlowStatsRequest(datapath)
    datapath.send_msg(req)

    req = parser.OFPPortStatsRequest(datapath, 0, ofproto.OFPP_ANY)
    datapath.send_msg(req)
```
* ```OFPFlowStatsRequest```：取得 Switch 中的規則資訊。  
* ```OFPPortStatsRequest```：取得 Switch 中每個 Port 的相關資訊和統計數據。  

### 其中```ofproto.OFPP_ANY```代表指定所有的 Port。

```python
req = parser.OFPPortStatsRequest(datapath, 0, ofproto.OFPP_ANY)
``` 

## FlowStatsReply 事件 (從Switch的Flow Entry取得資料)

對收到的 Switch 規則資訊進行處理，也就是剛剛```OFPFlowStatsRequest```所請求 Switch 中的規則資訊，並將其顯示。

```python
@set_ev_cls(ofp_event.EventOFPFlowStatsReply, MAIN_DISPATCHER)
def _flow_stats_reply_handler(self, ev):
    body = ev.msg.body
    self.logger.info('%s',json.dumps(ev.msg.to_jsondict(), ensure_ascii=True, indent=3, sort_keys=True))
```
OFPFlowStatsReply 的內容被轉換成為 JSON 的格式進行輸出，以利於進行後續的觀察和分析。

## PortStatsReply 事件

對收到的 Port 的資訊進行處理，也就是剛剛```OFPPortStatsRequest```所請求每個 Port 的相關資訊和統計數據，並將其顯示。
```python
#...

@set_ev_cls(ofp_event.EventOFPPortStatsReply, MAIN_DISPATCHER)
def _port_stats_reply_handler(self, ev):
    body = ev.msg.body
    self.logger.info('%s',json.dumps(ev.msg.to_jsondict(), ensure_ascii=True, indent=3, sort_keys=True))
```

## 執行程式


```shell
$ ryu-manager --verbose ./SimpleMonitor.py
```

## 執行結果
```shell
lab@ubuntu:~$ ryu-manager Desktop/SimpleMonitor.py
loading app Desktop/SimpleMonitor.py
loading app ryu.controller.ofp_handler
instantiating app Desktop/SimpleMonitor.py of SimpleMonitor
instantiating app ryu.controller.ofp_handler of OFPHandler
packet in 0000000000000001 00:00:00:00:00:02 33:33:00:00:00:02 2
packet in 0000000000000001 00:00:00:00:00:01 33:33:00:00:00:02 1
packet in 0000000000000001 00:00:00:00:00:02 33:33:00:00:00:02 2
packet in 0000000000000001 00:00:00:00:00:01 33:33:00:00:00:02 1
```

## 取得Switch回傳的規則資訊(Flowstats)
```shell
{
   "OFPFlowStatsReply": {
      "body": [
         {
            "OFPFlowStats": {
               "byte_count": 280, 
               "cookie": 0, 
               "duration_nsec": 997000000, 
               "duration_sec": 14, 
               "flags": 0, 
               "hard_timeout": 0, 
               "idle_timeout": 0, 
               "instructions": [
                  {
                     "OFPInstructionActions": {
                        "actions": [
                           {
                              "OFPActionOutput": {
                                 "len": 16, 
                                 "max_len": 65535, 
                                 "port": 4294967293, 
                                 "type": 0
                              }
                           }
                        ], 
                        "len": 24, 
                        "type": 4
                     }
                  }
               ], 
               "length": 80, 
               "match": {
                  "OFPMatch": {
                     "length": 4, 
                     "oxm_fields": [], 
                     "type": 1
                  }
               }, 
               "packet_count": 4, 
               "priority": 0, 
               "table_id": 0
            }
         }
      ], 
      "flags": 0, 
      "type": 1
   }
}
```

## 接收Switch中連接埠號儲存接收端的資訊
```shell
{
   "OFPPortStatsReply": {
      "body": [
         {
            "OFPPortStats": {
               "collisions": 0, 
               "duration_nsec": 713000000, 
               "duration_sec": 18, 
               "port_no": 4294967294, 
               "rx_bytes": 0, 
               "rx_crc_err": 0, 
               "rx_dropped": 4, 
               "rx_errors": 0, 
               "rx_frame_err": 0, 
               "rx_over_err": 0, 
               "rx_packets": 0, 
               "tx_bytes": 0, 
               "tx_dropped": 0, 
               "tx_errors": 0, 
               "tx_packets": 0
            }
         }, 
         {
            "OFPPortStats": {
               "collisions": 0, 
               "duration_nsec": 723000000, 
               "duration_sec": 18, 
               "port_no": 1, 
               "rx_bytes": 696, 
               "rx_crc_err": 0, 
               "rx_dropped": 0, 
               "rx_errors": 0, 
               "rx_frame_err": 0, 
               "rx_over_err": 0, 
               "rx_packets": 8, 
               "tx_bytes": 3247, 
               "tx_dropped": 2, 
               "tx_errors": 0, 
               "tx_packets": 23
            }
         }, 
         {
            "OFPPortStats": {
               "collisions": 0, 
               "duration_nsec": 723000000, 
               "duration_sec": 18, 
               "port_no": 2, 
               "rx_bytes": 872, 
               "rx_crc_err": 0, 
               "rx_dropped": 0, 
               "rx_errors": 0, 
               "rx_frame_err": 0, 
               "rx_over_err": 0, 
               "rx_packets": 10, 
               "tx_bytes": 3337, 
               "tx_dropped": 0, 
               "tx_errors": 0, 
               "tx_packets": 24
            }
         }
      ], 
      "flags": 0, 
      "type": 4
   }
}
```
## 從 host 1 向 host 2 執行 ping 的指令


```shell
h1 ping h2 -c1
```

可以發現flowentry的回傳資訊開始有了變化

```shell
{
   "OFPFlowStatsReply": {
      "body": [
         {
            "OFPFlowStats": {
               "byte_count": 182, 
               "cookie": 0, 
               "duration_nsec": 234000000, 
               "duration_sec": 12, 
               "flags": 0, 
               "hard_timeout": 0, 
               "idle_timeout": 0, 
               "instructions": [
                  {
                     "OFPInstructionActions": {
                        "actions": [
                           {
                              "OFPActionOutput": {
                                 "len": 16, 
                                 "max_len": 65509, 
                                 "port": 1, 
                                 "type": 0
                              }
                           }
                        ], 
                        "len": 24, 
                        "type": 4
                     }
                  }
               ], 
               "length": 104, 
               "match": {
                  "OFPMatch": {
                     "length": 32, 
                     "oxm_fields": [
                        {
                           "OXMTlv": {
                              "field": "in_port", 
                              "mask": null, 
                              "value": 2
                           }
                        }, 
                        {
                           "OXMTlv": {
                              "field": "eth_src", 
                              "mask": null, 
                              "value": "00:00:00:00:00:02"
                           }
                        }, 
                        {
                           "OXMTlv": {
                              "field": "eth_dst", 
                              "mask": null, 
                              "value": "00:00:00:00:00:01"
                           }
                        }
                     ], 
                     "type": 1
                  }
               }, 
               "packet_count": 3, 
               "priority": 1, 
               "table_id": 0
            }
         }, 
         {
            "OFPFlowStats": {
               "byte_count": 140, 
               "cookie": 0, 
               "duration_nsec": 232000000, 
               "duration_sec": 12, 
               "flags": 0, 
               "hard_timeout": 0, 
               "idle_timeout": 0, 
               "instructions": [
                  {
                     "OFPInstructionActions": {
                        "actions": [
                           {
                              "OFPActionOutput": {
                                 "len": 16, 
                                 "max_len": 65509, 
                                 "port": 2, 
                                 "type": 0
                              }
                           }
                        ], 
                        "len": 24, 
                        "type": 4
                     }
                  }
               ], 
               "length": 104, 
               "match": {
                  "OFPMatch": {
                     "length": 32, 
                     "oxm_fields": [
                        {
                           "OXMTlv": {
                              "field": "in_port", 
                              "mask": null, 
                              "value": 1
                           }
                        }, 
                        {
                           "OXMTlv": {
                              "field": "eth_src", 
                              "mask": null, 
                              "value": "00:00:00:00:00:01"
                           }
                        }, 
                        {
                           "OXMTlv": {
                              "field": "eth_dst", 
                              "mask": null, 
                              "value": "00:00:00:00:00:02"
                           }
                        }
                     ], 
                     "type": 1
                  }
               }, 
               "packet_count": 2, 
               "priority": 1, 
               "table_id": 0
            }
         }, 
         {
            "OFPFlowStats": {
               "byte_count": 602, 
               "cookie": 0, 
               "duration_nsec": 957000000, 
               "duration_sec": 39, 
               "flags": 0, 
               "hard_timeout": 0, 
               "idle_timeout": 0, 
               "instructions": [
                  {
                     "OFPInstructionActions": {
                        "actions": [
                           {
                              "OFPActionOutput": {
                                 "len": 16, 
                                 "max_len": 65535, 
                                 "port": 4294967293, 
                                 "type": 0
                              }
                           }
                        ], 
                        "len": 24, 
                        "type": 4
                     }
                  }
               ], 
               "length": 80, 
               "match": {
                  "OFPMatch": {
                     "length": 4, 
                     "oxm_fields": [], 
                     "type": 1
                  }
               }, 
               "packet_count": 9, 
               "priority": 0, 
               "table_id": 0
            }
         }
      ], 
      "flags": 0, 
      "type": 1
   }
}
```




## 參考
[(GitHub)TrafficMonitor](https://github.com/YanHaoChen/Learning-SDN/tree/master/Controller/Ryu/TrafficMonitor)

[Ryu 應用程式開發介面(API)](https://ryu-zhdoc.readthedocs.io/ryu_app_api.html)


