---
layout: post
title: "WebRTC 뽀개기 - ICE 편"
description: "ICE 프레임워크가 무엇이고 어떤 방식인지 정리 한 내용 입니다."
thumb_image: "documentation/webrtc-logo.png"
tags: [webrtc]
---

## 목차

- ICE 란?
- ICE는 왜 필요 할까?
  * NAT Traversal
- ICE는 어떻게 동작 할까?
  - SDP
  - Trickle ICE
  - (간략하게) ICE 역사 알아보기
- 정리

---
<br>
**_본 포스팅은 문서와 여러 아티클을 참고해 주관적으로 정리 한 내용이므로 사실과 다를 수도 있습니다.
만약 발견시 피드백 주시면 감사하겠습니다_**
<br>
<br>
## 1. ICE 란?

## <span style="color:orangered">I</span>nteractive <span style="color:orangered">C</span>onnectivity <span style="color:orangered">E</span>stablishment

두 peer 간 서로 통신을 할 수 있도록 최적의 경로를 찾아주는 프레임워크.

"프레임워크" 라고 해서 일반적인 언어의 framework로 생각 했었으나<br>
ICE에서 일어나는 전체적인 과정이 프레임워크 라고 한다.

---
<br>
## 2. ICE는 왜 필요 할까?

두 peer 간 연결을 시도 할때 방화벽 문제 라던지, public IP가 없다던지 등 문제가 발생 했을때
ICE 프레임워크를 이용해 연결을 할 수 있다.<br>
-> (사실 요즘은 사설(내부) IP를 사용하기 때문에 ICE는 필수 사용이 되었다.)<br>
-> (STUN / TURN을 사용하는 과정도 ICE 동작에 포함됨)



### NAT Traversal

 	방화벽 문제, NAT 보안정책 등 연결 시도에 막혔을때 우회해서 연결할 방법을 찾아주는 과정.

![STUN, TURN](https://mdn.mozillademos.org/files/6117/webrtc-turn.png)

​	가정용 개인 PC를 예로 들자면 기본적으로 Public IP를 ISP(인터넷 서비스 공급자)에서 할당 받아
​	공유기를 사용해 내부 사설 IP로 변환해 사용 하고 있을 것이다.
​	기본적으로 사설 IP를 사용하고 있는 장비에서는 자신의 Public IP를 모르기 때문에,
​	STUN서버를 이용해 response로 Public IP, Port를 받고, 원격 peer에게 연결 할 준비를 한다.

```
<STUN 흐름도>
You                        NAT                     STUN server          
|------ STUN request ----->|                          |                   
|   src:198.145.1.2:3000   |                          |                   
|                          |------ STUN request ----->|                   
|                          |   src:128.11.22.13:8888  |                   
|                          |                          |                   
|                          |<------STUN response------|                   
|                          |   ext:128.11.22.13:8888  |                   
|<------STUN response------|                          |                   
|   ext:128.11.12.13:8888  |                          |
```


​	
​	하지만 NAT 정책상 문제로 자신의 주소 찾기에 실패 할 경우<br>
​	_(ex: symmetric NAT의 경우 port를 동적으로 할당해 STUN은 찾을 수 없다.)_<br>
​	TURN 서버를 대안으로 사용 할 수 있다.

​	TURN 서버는 미디어들을 중계 해주는 역할로,
​	모든 미디어 트래픽은 TURN을 거쳐 각 peer에게 전송 된다.
​	(사실 TURN을 사용하게 되면 webRTC에 장점인 p2p 방식이 아니게되고,
​	그만큼의 서버 유지 비용이 늘어나게 되지만 어쩔수 없는 최후의 방법 이다.)

---
<br>

## 3. ICE는 어떻게 동작 할까?

ICE 과정에서 STUN, TURN 등으로 찾아낸 연결 가능한 네트워크 주소들을 **Candidate** / **후보** 라고 부르며 Candidate을 찾기
_(Finding Candidate)_ 까지 해준다.<br>
(Finding Candidate은 **동기적으로 진행**된다. peer A에서 Candidate을 찾은 뒤 완료 되면 peer B 시작.<br>
ICE의 큰 단점인 속도가 **느리다는 점**인데 위 부분이 원인이다. 뒤에서 다시 언급 할 예정.)<br>

그렇다면 적절한 Candidate를 찾았다면 그 다음은 어떻게 진행 할까?

```
<peer A의 흐름>
1. peer A가 로컬 SDP가 포함된 제안 생성 (create offer)
2. RTCPeerConnection 객체에 *offer를 첨부
3. 시그널링 서버에 offer가 포함된 RTCPeerConnection객체를 peer B에게 전송

<peer B의 흐름>
1. peer B는 시그널링서버로 부터 peer A의 제안을 받음
2. peer A의 제안에 대한 답변(SDP 포함) 생성
3. (peer A의 1번 항목과 같이) RTCPeerConnection 객체에 *answer를 첨부
4. 시그널링 서버에 answer가 포함된 RTCPeerConnection객체를 peer A에게 전송
```
<br>
**​ICE 동작 부분에서 SDP 라는 단어가 나왔다. 이건 뭘까?<br>**
### SDP 란?

​	ICE를 통해 찾은 정보들을 서로 교환할때 사용하는 규격 *(Session Description Protocol)*	
​	해상도나 형식, 코덱, 암호화등의 멀티미디어 컨텐츠의 연결을 설명하기 위한 표준. 
​	**미디어 컨텐츠에 대한 메타데이터.**

​위 설명에 나와있지만 감이 안온다. 아래 그림으로 보면 이렇게 생겼다.<br>

![SDP](https://velog.velcdn.com/images%2Fjodmsoluth%2Fpost%2F5762315c-83fe-4d84-8437-ce348c7bd836%2Fimage.png)

​	복잡해보이지만 규칙이 있고 각 줄 마다 의미가 있다.
​	구글링 해보면 자세하게 나와있는 자료들이 많으니 찾아보는 것을 추천.


​	**SDP는 제안/수락 모델 구조이다. (Offer / Answer)**
​	(아래 그림 첨부)
​	![offer/answer](https://velog.velcdn.com/images%2Fgojaegaebal%2Fpost%2Fa19b9710-b249-494a-aa30-4c6dddc0586f%2Fimage.png "SDP offer/answer")

​	결론은 peer 간 미디어컨텐츠를 직접 전송하는 것이 아닌, 메타데이터로 표현된 문서를 연결을 위해 
​	교환할 목적으로 만들어졌다.



### 	Trickle ICE

![trickle ice](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcR2VF7zyJi6JVPXAMz5LtTZ6OQZgOa3h0HbDA&usqp=CAU)

​	**Trickle ICE** or **ICE trickling**은 _초기 offer 혹은 answer를 다른 유저에게 이미 전달 했음에도_
​	_계속해서 candidate를 보내는 과정 이다._

​	이미지와 같이 해석하면 물방울, 천천히 흐르는 이라는 뜻을 가지고 있다.
​	키워드만 보면 ICE와 대치 되는 다른 방식의 ICE인가? 하고 생각 할 수 있다.
​	먼저 이 부분은 ICE에 대한 히스토리를 간략하게나마 알아볼 필요가 있다.<br>



### ICE의 역사

[VoIP](https://www.3cx.com/global/kr/voip-sip-webrtc/) : _Voice Over Internet Protocol의 약자로 인터넷을 이용한 음성 전송을 뜻합니다. 이 기술은 인터넷 프로토콜(IP) 네트워크를 통해 음성 통신과 멀티미디어 세션(동영상 등)을 전달 합니다._

예전에는 VoIP 방식의 프로토콜을 사용 했던것 같다.
이런 방식은 간단하다.

A가 B에게 전화 하려고할때 A의 IP 주소를 B에게 보내고 B가 수락하면 B의 IP로 응답 후 서로 연결하면 된다. 하지만 NAT가 도입 된 후 위의 상황은 어렵게 됨.
기존 처럼 A와 B를 직접적으로 연결 되도록 하고싶어 STUN 방식을 생각해냄.
자신의 ip를 알아내어 서로에게 초대 메세지를 보내는 방식. ([my ip address](https://www.findip.kr/) 와 같은)

그러나 NAT 정책에 따라 STUN방식이 실패 하는 경우도 발생.
그래서 미디어 트래픽을 중계 하는 기술에 의존하게됨.
(릴레이 관련 솔루션을 사용하거나 TURN 방식)

또 한가지 문제는 SIP프로토콜은 제안/응답 방식.
**SIP:** SIP는 요청 및 응답 모델을 따르는 HTTP 작동 방식과 비슷한 방식 (일회성) 
<u>_(SIP 프로토콜에서 일회성(stateless)로 인해 정확이 어떤 문제인지는 저도 아직 답을 찾지 못했습니다.
혹시 아시는 분이 계시다면 공유 해주시면 감사하겠습니다)_</u>

이후 새로운 해결책이 생김. interactive connectivity establishment 개념.
<br>
*(IETF에 의해  [ICE](https://tools.ietf.org/html/rfc5245) 프로토콜로 표준화되었으며 그 이전에도 Skype 및 Google Talk과 같은 프로그램에서 사용되어짐)*
(VoIP에서는 SIP 방식을, 이후에는 SDP 방식을 사용 하는것으로 이해 했습니다)

다시 돌아와서 ICE를 보자.
클라이언트에서 연결을 시도 하기 전 모든 주소들을 수집 (ip,port 정보 등)
따라서 모든 후보자가 수집되면 우선순위에 따라 목록으로 정렬됨. (후보는 여러개가 될 수 있다.)
(여기서 우선순위 기준은 가장 적은 오버헤드를 가진 후보)

이후 여러개의 후보 중 그나마 안정적으로 네트워크와 통신 할 수 있는 후보를 1개로 추려
Default 값으로 설정 후 SDP 제안에 맞게 준비를 함.

정리하자면 클라이언트에서 ICE 연결 준비를 할때 후보들을 수집하고, 내부적으로도 STUN 바인딩 등등 여러 절차들을 실행을 함. 
관련 작업들이 끝나면 클라이언트는 위의 종합된 결과를 원격 클라이언트에게 보내고 원격에서도
동일한 방식 진행.

이런 여러가지 복잡한 방식을 거쳐 ICE가 동작한다.
계속해서 ICE에는 많은 api들이 추가가 됨. (효율성, 안정성을 위해)
하지만 고질적인 특정 문제가 지속 됨. --> ICE 과정으로 인해 **속도 느림**

> 사람들: ICE 너무 복잡해요 ㅡ,ㅡ
>
> IETF: _우리가 복잡한 세상에 살고 있기 때문에 ICE도 복잡 할 수 밖에 없습니다_.

(위 내용은 IETF의 답변이라고 하는데 사실인지는 모르겠음..)

그래서 ICE [RFC 5245](https://tools.ietf.org/html/rfc5245) 는 위의 후보 정보들을 모두 기다리는 대신 후보가 조금이라도 준비 되면 바로 ICE 연결 진행을 하는 방식 권장함.

*[Trickle ICE](https://tools.ietf.org/id/draft-ietf-mmusic-trickle-ice) 의 컨셉은 후보의 모든 정보들이 모일때까지 기다리기 보다 발견되는 후보마다 조금씩 교환을 시작. (조금씩 흐르게)*
병렬로 프로세스를 진행하기때문에 속도 향상에 상당히 도움.
모든 클라이언트가 후보 수집을 마치면 “end-of-candidates” 신호를 보냄. 신호가 없으면 실패로 간주.

_<u>Trickle ICE는 ICE 프로토콜의 개선 사항 중 하나 이다.</u>_



---
<br>

## 정리

ICE는 두 peer 간에 통신을 할수 있도록 네트워크 최적의 경로를 찾아주는 역할.
이 과정에서 STUN, TURN을 사용.

데이터를 교환하기 위해 미디어콘텐츠를 메타데이터 형식으로 구현해 전송하는 규약이 있음.
-> **SDP 프로토콜**.

WebRTC를 구현하면서 ICE 부분을 정확히 이해 해야 하는 것이 필수 인 것 같다.
STUN, TURN, NAT 부분도 추후 포스팅으로 다룰 것.

---
<br>


## Reference

[참고링크1](https://webrtc.org/getting-started/peer-connections#ice_candidates)

[참고링크2](https://blog.sessionstack.com/how-javascript-works-webrtc-and-the-mechanics-of-peer-to-peer-connectivity-87cc56c1d0ab)

[참고링크3](https://webrtchacks.com/trickle-ice/)




