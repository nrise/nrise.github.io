---
title: "RabbitMQ 를 이용한 서버-클라이언트 양방향 통신 구현"
date: 2020-01-02T12:13:58+09:00
draft: false
---
# 개요
다양한 목적으로 인해 서버-클라이언트 간 양방향 통신(duplex communication system)
을 필요로 하는 경우가 있습니다. 엔라이즈에서 서비스 하는 WIPPY 와 MOCI 역시
서버-클라이언트 간 양방향 통신을 구현해야 하는 이슈들이 있었으며, 우리는
[RabbitMQ](https://www.rabbitmq.com/) 를 이용하여 양방향 통신을 성공적으로 구축하여
다양한 용도로 사용하고 있습니다. 우리의 사례를 소개함으로써 양방향 통신 구축을 고민하는
분들에게 작게나마 도움이 될 수 있다면 좋을 것 같습니다.

### 양방향 통신이 필요한 이유
서버-클라이언트 간 양방향 통신 기능이 최초로 요구되던 시기는 2015년으로 거슬러 올라갑니다.

양방향 통신은 iOS/Android 에서 지원하는 푸시 메시지를 통해서도 부분적으로 구현할 수 있겠으나,
이 경우 사용자가 푸시를 허용하지 않았을 경우 무용지물이 되는 단점이 존재합니다.
만일 [Polling](https://en.wikipedia.org/wiki/Polling_(computer_science))
형태로 구현할 경우엔 양방향 통신을 위한 중계 서버 자체는 필요 없겠지만 Polling 주기에 따라
약화되는 응답성, 트래픽의 낭비라는 문제점이 발생하며, 그로 인하여 불필요한 서버/클라이언트의
자원이 소비될 것입니다.

즉 서버-클라이언트 양방향 통신 기능이 구현된다면 클라이언트의 네트워크 상태에만 의존적이면서도
자원의 낭비를 최소화 하는 구성이 가능해 집니다. 또한 서버는 접속한 클라이언트에게 언제든지
원하는 메시지를 보낼 수 있으며, 실시간 응답이 중요시 되는 기능들의 구현을 통해 더욱 좋은 사용자
경험을 제공할 수 있게 됩니다.

### RabbitMQ 에 관하여
Python 으로 서버를 개발하는 분들 중 비동기 작업을 위해 [Celery](http://www.celeryproject.org/)
를 도입해 본 경험이 있다면 RabbitMQ 라는 메시지 브로커 서버에 대해서 들어보셨을 것입니다.
생소하신 분들을 위해 간단히 설명하자면 RabbitMQ 는 [Erlang](https://www.erlang.org/) 으로 작성된 메시지
브로커입니다. [AMQP](https://www.amqp.org/) 를 지원하며, 다양한 라우팅 모델을 지원하는
오픈소스 소프트웨어입니다.

우리는 서버 개발 언어로 Python 을 사용하고 있으며, [Celery 의 메시지 브로커](http://docs.celeryproject.org/en/latest/getting-started/brokers/index.html)
로 RabbitMQ 를 선택하여 오랜 기간 운용해 왔습니다. 많은 검토와 고민 끝에 RabbitMQ 를 서버-클라이언트
양방향 통신 수단으로 충분히 활용할 수 있겠다는 생각을 했었고, 성공적으로 적용하여 다양한 곳에서
사용할 수 있게 되었습니다.

# 솔루션 검토
최초 양방향 통신 시스템을 구축하기 위해 다양한 방법들을 고민해 보았습니다.

### WebSocket
양방향 통신 설계로써 대표적으로 생각해 볼 만한 방법은 socket.io 를 위시한 WebSocket 서버를
구축하여 서버-클라이언트 간 통신을 구현하는 것입니다. 하지만 다음 이유들로 인해
최종적으로는 WebSocket 서버를 채택하지 않기로 결정하였습니다.

* 당시에는 만족할 만한 [WebSocket](https://en.wikipedia.org/wiki/WebSocket)
서버는 [Node.js](https://nodejs.org/) 와 [socket.io](https://socket.io/) 를 기반으로 서버를 구현하는 방법이
주류였는데, 이 경우 Python 으로 구현되어 있는 비즈니스 로직이 여러 곳으로 분산될 수 있는
문제점이 예상 되었습니다. 현재 MOCI 와 WIPPY 는 단일체 구조(Monolithic Architecture) 로
설계되어 있으며, 이 구조를 현재까지 유지하고 있는 이유에 대해서는 추후 다른 글을 통해
이야기 해 보도록 하겠습니다.
* Node.js 나 [Django Channels](https://channels.readthedocs.io/) 등의
WebSocket 서버를 구축하여 운용할 경우 발생하게 될 서버 유지보수 비용에 대해 고민하게 되었습니다. 
여기서 비용이라 함은 단순히 금액적 비용이 아닌 관리 측면에 대한 비용을 의미합니다.
Node.js 나 Python 모두 기본적으로 단일 프로세스 기반으로 동작합니다.
물론 필요하다면 프로세스 클러스터링을 할 수 있습니다만 프로세스가 예기치 못한 종료로 장애가
발생할 경우에 대한 Failover 처리, Auto Scaling  등을 고려해 보았을 때 손쉽고 안정적으로 관리하기엔
어려운 점이 많을 것이라 생각했으며, 명확한 해결 방안을 찾지 못하였습니다.
* WebSocket 의 경우 양방향 통신을 지원합니다만, 네트워크 환경에 따라 이벤트가 유실될 수 있는
위험이 존재합니다.[^1]

[^1]: 우아한 형제들의 경우 결과적으로 WebSocket 을 선택하였습니다만 비슷한 고민을
했던 것 같습니다. [참고](http://woowabros.github.io/woowabros/2017/09/12/realtime-service.html)

### Redis
[Redis](https://redis.io/) 는 [자체적인 Pub/Sub 모델을 지원합니다](https://redis.io/topics/pubsub).
Redis 의 Pub/Sub 을 이용하게 되면 WebSocket 서버 에서 예상되던 문제점을 부분적으로 해결할 수
있습니다만, 여전히 단일 프로세스 기반 서버라는 이슈는 해결이 되지 않습니다. 거기에 덧붙여
Redis 는 Single Threaded[^2] 형태로 동작하기 때문에 대규모의 Pub/Sub 처리가 잘 될 수 있을지에
대한 보장이 없었습니다.

[^2]: "Redis is, mostly, a single-threaded server from the POV of commands execution 
(actually modern versions of Redis use threads for different things). It is not designed
to benefit from multiple CPU cores. People are supposed to launch several Redis instances to
scale out on several cores if needed. It is not really fair to compare one single Redis
instance to a multi-threaded data store." - [원문](https://redis.io/topics/benchmarks)

### RabbitMQ
결과적으로 RabbitMQ 는 다음과 같은 특징을 가지고 있기에 최종적으로 선택하게 되었습니다.

* 기본적으로 메시지 큐(Message Queue) 서버이기 때문에 메시지가 누락될 위험이 거의 없습니다.
설령 연결이 끊어졌다 하더라도 다시 연결하는 순간 큐에 저장된 메시지를 구독할 수 있으며, 순서 역시 보장됩니다.
* 내결함성(Fault tolerance)이 극도로 뛰어납니다[^3]. 이는 RabbitMQ 의 특징이라기 보다
Erlang 언어의 특징이기도 합니다. 실제로 RabbitMQ 서버의 안정성에 대해 충분히 느끼고 있었기 때문에
신뢰가 갔습니다.
* 외부 의존성이 없습니다. 내장된 [Mnesia](https://erlang.org/doc/man/mnesia.html)
및 [ETS](https://erlang.org/doc/man/ets.html), [DETS](https://erlang.org/doc/man/dets.html)
모두 강력한 성능과 안정성을 자랑합니다. 별도의 외부 의존성 없이 구동, 운용이 가능한 점은 큰 장점을 가집니다.
* 이 역시 Erlang 의 특징이기도 한 부분입니다만, [동시성(Concurrency) 성능이 매우 뛰어납니다](https://erlang.org/doc/reference_manual/processes.html).
따라서 비슷하게 동시성 성능이 뛰어나다면 굳이 단일 프로세스로 동작하는 Node.js 나 Python
을 고려해야 할 이유가 사라지게 됩니다.
* 수평적 확장(Horizontal Scale Out) 이 쉽습니다. 여러 RabbitMQ 서버를 [클러스터로 묶어버릴 수도 있습니다](https://www.rabbitmq.com/clustering.html).
* 뛰어난 성능을 보여줍니다. 초당 수만건의 메시지는 큰 문제 없이 전송이 가능합니다. [Apache Kafka](https://kafka.apache.org/)
정도의 성능은 아니겠습니다만 실 사용에는 전혀 문제가 없었습니다.
* WebSocket/[STOMP](https://stomp.github.io/) 등의 웹 기반 소켓 프로토콜 역시 지원합니다.
MOCI 와 WIPPY 모두 네이티브 앱이기에 당장은 필요 없을지라도 웹 기반 관리자 사이트에서 요긴하게 사용할 수 있습니다.
* RabbitMQ 는 다양한 언어로 개발 라이브러리를 제공합니다. 만일 iOS/Android 클라이언트 라이브러리가 없었다면
연동하기가 쉽지 않을 것입니다.

[^3]: Erlang 을 흔히 "[write once, run forever](https://en.wikipedia.org/wiki/Erlang_(programming_language)#cite_note-14)"
한 언어라고 표현하기도 합니다.

즉 메시지 브로커 기능을 하는 안정적인 서버가 클라이언트와 Pub/Sub 관계를 전담하게 되면
시스템 설계 복잡도를 크게 높이지 않고도 쉽게 서버-클라이언트 간 양방향 통신을 구현할 수 있겠다는
결론을 내리게 되었습니다.

# 서비스 구성

RabbitMQ 가 적용된 서버-클라이언트 간 양방향 통신이 위한 구성은 다음과 같습니다.

![](/images/20200102/architecture.png)

1. 클라이언트와 서버는 API 서버와 동기화(Synchronous)된 요청과 응답을 주고 받습니다.(그림 1)
1. 클라이언트는 RabbitMQ 서버와 AMQP 연결을 맺고 있습니다.(그림 2)
1. 서버에서 특정 클라이언트에 메시지를 전달하고자 할 때 RabbitMQ 를 통해 메시지를 Publish 하면 클라이언트는 해당 메시지를 받습니다.(그림 3, 2)
1. 클라이언트는 받은 메시지를 해석(Parsing)해서 취해야 할 행동을 합니다.

이 예제 코드는 AMQP 서버에 연결하여 서버의 메시지를 기다리는 것을 구현한 간단한 안드로이드 예제입니다.(iOS 도 크게 다르지 않습니다.)
주의할 점은 앱의 생명주기에 맞춰서 클라이언트의 AMQP 연결도 같이 활성/비활성화 처리가 되어야 하며,
중복 연결을 방지하기 위해 Singleton Pattern 으로 설계하는 것이 좋습니다.

```kotlin
// AMQPManager.kt
// Android Client Example
import com.rabbitmq.client.*
import org.jetbrains.anko.doAsync

open class AMQPManager {
	var exchangeName = "exchange_001" // RabbitMQ 에 연결할 exchange 명
	var queueName = "user_0001" // RabbitMQ 에 연결할 queue 명
	var serverUri = "amqps://amqpmanager.example.com" // RabbitMQ Server URI

	fun connect() {
		doAsync {
			factory = ConnectionFactory()
            factory.setUri(serverUri)

			connection = factory.newConnection()
            channel = connection!!.createChannel()
            channel!!.queueDeclare(queueName, false, true, true, null)
            channel!!.queueBind(queueName, exchangeName, queueName)
            channel!!.basicConsume(queueName, true, object : Consumer {
				override fun handleDelivery(
						consumerTag: String?,
						envelope: Envelope?,
						properties: AMQP.BasicProperties?,
						body: ByteArray?) {
					// 서버로부터 온 body 를 이곳에서 처리합니다.
				}
			}
		}
	}
}
```

다음 코드는 queue "user_0001" 을 Subscribe 하고 있는 사용자에게 'hello, client!' 라는
메시지를 보내는 Python 예제입니다. 예제 코드를 보면 알 수 있듯이 서버는 특정 클라이언트가 어떤 이름의
exchange 와 queue 를 사용하는지를 알고 있어야 합니다.

```python
# pika 를 이용한 python 예제입니다.
import pika

exchange_name = 'exchange_001'
queue_name = 'user_0001'
server_uri = 'amqps://amqpmanager.example.com'

rq_uri = pika.URLParameters(server_uri)
connection = pika.BlockingConnection(rq_uri)
channel = connection.channel()

channel.basic_publish(
	exchange=exchange_name,
	routing_key=queue_name,
	body='hello, client!'
)
```


# 활용 예
서버-클라이언트에 구축된 양방향 통신을 어떻게 사용하는지에 대한 활용 예를 소개하겠습니다.

## 1. 채팅
WIPPY 에서 이용자들 간에 1:1 채팅, 혹은 단체 채팅을 할 때 사용됩니다. API Server 를 통해 채팅 메시지를
보내면, 서버는 대화 수신자에게 RabbitMQ 서버를 통해 메시지를 Publishing 합니다. 클라이언트는 해당 메시지를
즉각적으로 받아 처리하게 됩니다. 실제로 WIPPY 에서는 200 명 정도의 대화방을 테스트 해 보았을 때도 별도의
최적화 없이 처리 가능했습니다. 그 외에 사용자가 채팅 리스트에 머무르고 있을 때 새로운 친구 요청이나,
기존 친구들이 보내는 새로운 메시지들을 받아서 실시간으로 리스트를 갱신하고자 할 때 굉장히 유용하게
사용이 가능합니다.

## 2. 상단 알림
![](/images/20200102/notification.gif)

WIPPY 에는 '상단 알림' 이라는 기능이 존재합니다. 서버에서 메시지를 보내면 클라이언트에서 즉각적으로
메시지를 받아 상단 알림을 노출해 주는 기능입니다. 이 역시 양방향 통신을 이용하면 손쉽게 구현이 가능합니다.

## 3. 각종 진행 상황 알림
예를 들어 WIPPY 에 있는 '내 프로필 평가받기' 의 경우 특정 사용자가 나의 프로필을 조회할 경우 실시간으로
진행률이 올라가야 하는 기능이 있습니다. 이 경우 양방향 통신을 이용하여 구현이 가능한 부분입니다.

## 4. Signaling Server
WIPPY 에는 보이스톡이라고 하는 [WebRTC](https://webrtc.org/) 기반의 음성 채팅 기능이 있습니다.
WebRTC 의 경우 최초 두 Peer 가 연결을 하기 위해서는 중개자를 통해 [SDP(Session Description Protocol)](https://tools.ietf.org/html/rfc4566)
와 [ICE Candidate](https://tools.ietf.org/id/draft-ietf-ice-rfc5245bis-14.html) 정보를 교환해야 합니다.
이 경우 양방향 통신 서버를 이용하여 Signaling Server 의 역할을 수행하게 할 수 있습니다.

이 외에도 WIPPY 와 MOCI 에서는 다양한 목적으로 RabbitMQ 를 통한 양방향 통신을 활용하고 있습니다.

# 마치며
RabbitMQ 를 이용하여 서버-클라이언트 간 양방향 통신을 구현함으로써 WIPPY 와 MOCI 서비스에 많은 재미있는
기능과 실시간을 강조하는 기능들을 성공적으로 추가할 수 있었습니다. 물론 우리가 구현한 방법이 Best Practice
는 아닐 것입니다. 하지만,

* 이미 사용해 보면서 운용 노하우를 쌓은 서버를 기반으로
* 서버의 코드 및 비즈니스 로직이 별도로 분리되지 않게 하며
* 다양한 규모에서도 안정적으로 높은 성능을 발휘하며
* 손쉽게 양방향 통신을 구축 및 지속적인 개량이 가능할 수 있었던 것은

역시 RabbitMQ 라는 소프트웨어 덕분일 것입니다.

흥미가 느껴지나요? 만일 우리와 함께 재미있고 흥미로운 코드를 개발해 보고 싶다면 언제든지
[채용 페이지](https://www.rocketpunch.com/companies/nrise/jobs) 를 보고 주저없이 지원해 주세요!

감사합니다.

