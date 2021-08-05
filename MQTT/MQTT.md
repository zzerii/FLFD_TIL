# MQTT



## MQTT란?

MQ Telemetry Transport

매우 간단한 경량의 메세지 프로토콜. TCP/IP 환경에서 작동한다.

> TCP/IP
>
> 전송제어 프로토콜인 TCP와  패킷 통신방식의 인터넷 프로토콜인 IP로 아루어져있다.
> IP는 패킷 전달 여부를 보증하지 않고 패킷을 보낸 순서와 받는 순서가 다를 수 있다.
> TCP는 IP 위에서 동작하는 프로토콜로 데이터의 전달을 보증하고 보낸 순서대로 받게 해준다.
> 즉, TCP/IP는 두개의 프로토콜이다 그 중 IP는 복잡한 네트워크 망을 통하여 가장 효율적인 방법으로 데이터의 작은 조각들을 되도록 빨리 보내는 일을 한다.
> TCP는 데이터를 잘게 잘라 보내면서 순서가 맞지 않거나 중간에 빠진 부분을 점검하여 다시 요청하는 일을 담당한다.

제한적인 자원 환경과 낮은 대역폭, 불안한 네트워크 환경에서 특히 적합하다.
따라서, M2M(machine to machine) 환경이나 IoT 환경에서 자주 사용한다.

## MQTT 동작 원리 

MQTT는 publish/subscribe/broker를 사용하는 프로토콜로서 subscriber는 토픽을 구독하기 위한 목적으로, publisher는 토픽을 발행하는 목적으로 broker와 연결하게 된다

```
  client				Message-Broker			Client
(Publisher)									 (Subscriber)
	|						  |					  |
	|						  |	<--- Connect ---  |
	|  ---- Connect ---->	  |					  |
	|						  |-- Connect_Ack --->|
	|  <-- Connect_Ack ---	  |					  |
	|						  | -- Subscribe ---> |
	|	---- Publish ---->	  |					  |
	|  <-- Publish_Ack ---	  |					  |
	|						  |	--- Publish ----> |
	|						  |<-- Publish_Ack ---|
```



Topic은 /로 구분지어 계층으로 관리할 수 있다.

![image-20210805204114248](https://user-images.githubusercontent.com/61822411/128344207-2314dc33-27f1-4ca6-9b82-10aad4225621.png)

MQTT는 메시지 버스 시스템으로서 MQTT Broker가 메시지 버스를 만들고 여기에 메시지를 흘려보내면, 버스에 붙은 애플리케이션들이 메시지를 읽어가는 방식이다. 메시지 버스에는 다양한 주제의 메시지들이 흐를 수 있는데, 메시지를 구분하기 위해 "Topic"을 이름으로 하는 메시지 체널을 만든다.



#### MQTT Broker

MQTT Pub와 Sub가 메시지를 주고받을 수 있도록 다리를 놔주는 역할을 한다.

##### 종류

mosquitto

Hive MQ

Rabbit MQ

IBM MQ

Vertx



#### 터미널에서 통신하기

Server A 에서 브로커 실행

`sudo mosquitto -c /etc/mosquitto/mosquitto.conf`



Server B에서 topic 구독

 -h : broker 주소				-t:  topic 

`mosquitto_sub -h '192.168.0.16' -t 'things/1/temperature'`



Server A에서 publish

  -h : broker 주소				-t:  topic 		 -m: publish할 message

`mosquitto_pub -h '192.168.0.16' -t 'things/1/temperature' -m '36.5'`



Server B 터미널 관찰

```
# 15번 server
kay@ubuntu:~$ mosquitto_sub -h '192.168.0.16' -t 'things/1/temperature'
36.5
```





#### python으로 통신하기

Broker로 HiveMQ사용

HiveMQ 클라우드 브로커를 사용했기 때문에 별도의 설치가 필요없다.





python에서 mqtt를 통신하기 위해 paho-mqtt설치

ubuntu 20.04에서

`apt install python-paho-mqtt`



**Server B에서 subscriber 구성**

sub.py

```python
import paho.mqtt.client as mqtt


def on_connect(client, userdata, flags, rc):
    if rc == 0:
        print("connected OK")
    else:
        print("Bad connection Returned code=", rc)


def on_disconnect(client, userdata, flags, rc=0):
    print(str(rc))


def on_subscribe(client, userdata, mid, granted_qos):
    print("subscribed: " + str(mid) + " " + str(granted_qos))


def on_message(client, userdata, msg):
    print(str(msg.payload.decode("utf-8")))


# 새로운 클라이언트 생성
client = mqtt.Client()
# 콜백 함수 설정 on_connect(브로커에 접속), on_disconnect(브로커에 접속중료), on_subscribe(topic 구독),
# on_message(발행된 메세지가 들어왔을 때)
client.on_connect = on_connect
client.on_disconnect = on_disconnect
client.on_subscribe = on_subscribe
client.on_message = on_message
# 로컬 아닌, 원격 mqtt broker에 연결
# address : broker.hivemq.com
# port: 1883 에 연결
client.connect('broker.hivemq.com', 1883)
# test/hello 라는 topic 구독
client.subscribe('test/hello', 1)
client.loop_forever()
```

`python sub.py` 로 실행



**Server A에서 publisher 구성**

```python
import paho.mqtt.client as mqtt
import json


def on_connect(client, userdata, flags, rc):
    # 연결이 성공적으로 된다면 완료 메세지 출력
    if rc == 0:
        print("completely connected")
    else:
        print("Bad connection Returned code=", rc)

# 연결이 끊기면 출력
def on_disconnect(client, userdata, flags, rc=0):
    print(str(rc))


def on_publish(client, userdata, mid):
    print("In on_pub callback mid= ", mid)


# 새로운 클라이언트 생성
client = mqtt.Client()
# 콜백 함수 설정 on_connect(브로커에 접속), on_disconnect(브로커에 접속중료), on_publish(메세지 발행)
client.on_connect = on_connect
client.on_disconnect = on_disconnect
client.on_publish = on_publish
# 로컬 아닌, 원격 mqtt broker에 연결
# address : broker.hivemq.com
# port: 1883 에 연결
client.connect('broker.hivemq.com', 1883)
client.loop_start()
# 'test/hello' 라는 topic 으로 메세지 발행
client.publish('test/hello', "Hello", 1)
client.loop_stop()
# 연결 종료
client.disconnect()
```

`python pub.py` 로 실행



**#Broker로 HiveMQ를 사용하지 않고 mosquitto를 사용할 경우**

client.connect('broker.hivemq.com', 1883) 부분의 broker주소를

mosquitto broker가 있는 곳의 IP를 적어주면 된다.





### influxDB 연동해서 mqtt 테스트하기



**Server B (subscriber)**

sub.py

```python
#!/usr/bin/env python3
# subscriber_influxdb.py
import paho.mqtt.client as mqtt
import datetime
import time
import json
from influxdb import InfluxDBClient

def on_connect(client, userdata, flags, rc):
    if rc == 0:
        print("connected OK")
    else:
        print("Bad connection Returned code=", rc)


def on_disconnect(client, userdata, flags, rc=0):
    print(str(rc))


def on_subscribe(client, userdata, mid, granted_qos):
    print("subscribed: " + str(mid) + " " + str(granted_qos))

def on_message(client, userdata, msg):
    # Use utc as timestamp
    receiveTime=datetime.datetime.utcnow()
    message=msg.payload.decode("utf-8")
    print(str(message))
    print(str(receiveTime) + ": " + msg.topic + " payload : " + str(message))
    msgDict = json.loads(message)

    json_body = [
        {
            "measurement": msg.topic,
            "time": receiveTime,
            "fields": msgDict
        }
    ]

    dbclient.write_points(json_body)

# Set up a client for InfluxDB
dbclient = InfluxDBClient('192.168.56.23', 8086, 'root', 'root', 'test_mqtt')


client = mqtt.Client()
client.on_connect = on_connect
client.on_disconnect = on_disconnect
client.on_subscribe = on_subscribe
client.on_message = on_message
client.connect('192.168.56.11', 1883)
client.subscribe('test', 1)
client.loop_forever()
~                      
```



**Server A (publisher+broker)**

pub.py

```python
import paho.mqtt.client as mqtt
import json

def on_connect(client, userdata, flags, rc):
    if rc == 0:
        print("completely connected")
    else:
        print("Bad connection Returned code=", rc)

def on_disconnect(client, userdata, flags, rc=0):
    print(str(rc))


def on_publish(client, userdata, mid):
    print("In on_pub callback mid= ", mid)


client = mqtt.Client()
client.on_connect = on_connect
client.on_disconnect = on_disconnect
client.on_publish = on_publish
client.connect('192.168.56.11', 1883)
client.loop_start()

#pub에서 json형식으로 보내야 sub에서 json으로 제대로 받는다.

dict1={'name':'zzerii'}
json_test=json.dumps(dict1)

client.publish('temp', json_test, 1)
client.loop_stop()
client.disconnect()
```



**Server C (DB)**

`wget https://dl.influxdata.com/influxdb/releases/influxdb_1.7.8_amd64.deb`
`dpkg -i influxdb_1.7.8_amd64.deb`

`systemctl enable --now influxdb`

`influx -precision rfc3339`

`CREATE DATABASE test_mqtt`

`SHOW DATABASES`

`CREATE USER root WITH PASSWORD 'root' WITH ALL PRIVILEGES`

`GRANT ALL ON test_mqtt TO root`

`USE test_mqtt`

`SHOW measuerments`

`select * from [topic]`



![image-20210805200950120](https://user-images.githubusercontent.com/61822411/128344031-da6976ba-686d-498e-a881-e88600f29877.png)

