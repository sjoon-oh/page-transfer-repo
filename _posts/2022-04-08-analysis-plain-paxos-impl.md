---
title:  "[조사 노트]Plain Paxos 구현 분석"
date:   2022-04-08 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
   show_overlay_excerpt: false
categories: 
   - Research
tags:
   - Python
   - Consensus
   - Paxos
toc: true
toc_sticky: true
breadcrumbs: true
---

# [Analysis] Plain Paxos Implementations

*이 포스트는 OSLab 연구 활동에서 작성한 개인 노트의 일부입니다. 공부 목적으로 작성된 내용이므로 많이 부족할 수 있습니다.*

아래의 Repository 에 공개되어 있는 Paxos는 `Java`와 `Python`으로 구현되어 있다. 이는 실제 상용되어 있는 알고리즘이 아니고, 철저히 *교육용 목적* 이라고 명시되어 있다.

[https://github.com/cocagne/paxos](https://github.com/cocagne/paxos)

# Environment Setup

---

테스트 한 환경은 아래와 같다.

- Manjaro Quonos 21.2
- Java: **open-jdk 8**
- Python: **2.7.18**

```bash
$ java -version
openjdk version "1.8.0_332"
OpenJDK Runtime Environment (build 1.8.0_332-b04)
OpenJDK 64-Bit Server VM (build 25.332-b04, mixed mode)
```

```bash
$ python2 -V
Python 2.7.18
```

---

~~테스트는 가상 환경 `virtualenv` 사용하여 진행한다.~~

```bash
$ virtualenv venv && cd venv
$ source bin/activate
$ git clone https://github.com/cocagne/paxos
```

* **주의 사항**: 이 프로젝트는 마지막 커밋 날짜가 9년 전이라 Python 3 으로 돌릴 수 없다. 일부 deprecated 기능을 쓰는 것으로 보이며, 고칠 수 없는 오류가 발생한다. 예를 들어, 정상적으로 선언한 변수임에도 불구하고 `NoneType` 으로 인식한다.

---

테스트 스크립트는 `/test` 폴더 안에 존재한다. `unittest` 라이브러리를 활용하여 테스트를 진행하기 때문에 실행 진행사항은 보여주지 않고 오류가 있는 경우에만 콘솔에 보여준다. 

```bash
$ python2 test_essential.py
......................
----------------------------------------------------------------------
Ran 22 tests in 0.001s

OK
```

# Source Tree

---

## **Python Modules: `/paxos/`**

- **essential.py:** 가장 기본적인 Paxos Role이 구현되어 있다.
- **practical.py:** 이 모듈은 Paxos 알고리즘을 확장하여 Leadership Tracking, NACK, State Persistence가 추가되었다.
- **functional.py:** 이 모듈에서는 Fully-functional Paxos 알고리즘을 구현하고 있다. Heartbeating 메커니즘과 Leadership failure detection, Recovery를 지원한다.
- **external.py:** `[practical](http://practical.py)` 모듈을 확장하여 외부 Failure Detector (Leadership Management)를 지원한다. 이 모듈은 `functional` 모듈과 같이 완전한 (Fully-functional) Leadership Management 를 지원한다기 보다는 하나의 Basis로만 제공된다.
- **durable.py:** 정확한 Paxos 알고리즘의 구현은 `Acceptor` 의 상태를 persistent media 에 저장하는 기능을 요구한다. 이러한 저장 기능은 `Promise`와 `Accepted` 메세지를 보내기 전에 이루어져야 한다. 외부와의 `promise` 는 Crash/Recover의 순간에 거부되지 않도록 보장해야 된다. 이 모듈은 상태를 디스크로 저장하는 가장 기본적인 기능을 제공한다.
- **test_*.py**: 각 모듈을 테스트 하는 스크립트이다. `unittest` 모듈을 이용한다.

---

들어가기 전에, 이 프로젝트에서 구현하는 test 파일의 구조를 간단하게 짚고 넘어가는게 좋다. 우선 개념적으로 설명하기 위해 각 메서드의 자세한 구현은 생략하여 (`# ...`) 표현한다. `test_essential.py` 를 예로 들어 설명한다.

```python
# 1. Messenger Class
class EssentialMessenger (object):
	def setUp(self):
		# ...	
	# ...

# 2. Proposer Test Class
class EssentialProposerTests (object):

    proposer_factory = None 

    def setUp(self):
       # ...
		# ...
        
    def test_set_proposal_no_value(self): # ...
    def test_set_proposal_with_previous_value(self): # ...
    def test_prepare(self): # ...
    def test_prepare_two(self): # ...
    def test_prepare_with_promises_rcvd(self): # ...

	# ...
	
# 3. Test Paxos Class
class TPax(object):
    def __init__(self, messenger, uid, quorum_size):
        self.messenger     = messenger
        self.proposer_uid  = uid
        self.quorum_size   = quorum_size

# 4. Test Proposer Class
class TProposer (essential.Proposer, TPax):
    pass

# 5. Unit-test Class
class EssentialProposerTester(EssentialProposerTests, EssentialMessenger, unittest.TestCase):
    **proposer_factory = TProposer**
```

상속 관계를 간단하게 나타내면 아래와 같다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled.png)

`essential.Proposer` 는 구현한 `essential` 모듈이고, 그 외의 `unittest.TestCase` 모듈 외에는 위 테스트 스크립트 파일에 구현되어 있다. 

---

Python의 `unittest` 는 아래와 같은 방법으로 테스트를 진행한다. **아래의 코드는 사용 예시**이다. 우선 이 예시를 간단하게 이해하고, 이 프로젝트가 어떻게 구성되어 있는지 보도록 한다.

* 참고 링크: [Unit testing framework (Python 3)](https://docs.python.org/3/library/unittest.html)

```python
import unittest

class TestStringMethods(unittest.TestCase):

    def test_upper(self):
        self.**assertEqual**('foo'.upper(), 'FOO')

    def test_isupper(self):
        self.**assertTrue**('FOO'.isupper())
        self.**assertFalse**('Foo'.isupper())

    def test_split(self):
        s = 'hello world'
        self.**assertEqual**(s.split(), ['hello', 'world'])
        # check that s.split fails when the separator is not a string
        with self.assertRaises(TypeError):
            s.split(2)

if __name__ == '__main__':
    unittest.main()
```

1. 테스트 케이스는 `unittest.TestCase` 모듈을 상속받아서 작성한다. 각 `test` 로 이름 붙여진 함수는 각각의 테스트 케이스를 나타낸다. 이때 Naming Convention은 prefix으로 `test` 를 사용하며, 이는 테스트 실행기에게 어떠한 함수가 테스트를 나타내는지 알린다. 
2. 테스트는 `assert` 류의 함수로 진행한다. 위의 예시에서는 `assertEqual()`로 예상 값을 지정한다. `assert`는 실행되면 테스트 케이스를 누적하여 계산한다.
3. 테스트 함수를 실행하기 전에 가장 먼저 실행되어야 할 라인이 있다면 `setUp()` 메서드를 상속받아 활용할 수 있다. 테스트를 모두 마치고 실행되어야 되는 라인은 `tearDown()` 을 이용한다.

위의 예시의 결과는 아래와 같이 나타난다.

```bash
test_isupper (__main__.TestStringMethods) ... ok
test_split (__main__.TestStringMethods) ... ok
test_upper (__main__.TestStringMethods) ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.001s

OK
```

**위의 `unittest` 라이브러리의 동작을 바탕으로 위의 코드를 이해하면 아래와 같이 정리된다.**

1. `EssentialMessenger` : 테스트 대상 Role 이 공통적으로 가지고 있는 `Messenger` 클래스이다. `setUp()` 함수를 작성함으로써 `unittest` 내에서 초기화하는 코드를 작성한다. 
2. `EssentialProposerTests` : 테스트를 진행하고 싶은 Test Case를 이 클래스에 작성한다. 
3. `TPax` : Test Paxos 클래스. 테스트 Role에 맞추어 초기화만 담당하는 메서드를 작성한다. 이는 각 모듈에 따라 달라질 수 있다.

여기까지는 기본 클래스 (참고: `Python2` 에서는 클래스 정의 시 `object` 클래스를 명시적으로 상속받아야 클래스로 인식된다. `Python3` 는 이러한 제한이 없다. 이는 `Java`에서의 `Object` 클래스와 동일하다고 생각하면 된다.)의 구조이다.

1. `TProposer`  : 테스트 Role 클래스를 작성한다. 테스트 대상 모듈(클래스)와, 멤버를 초기화하는 함수를 가지고 있는 `TPax` 를 상속받는다.
2. `EssentialProposerTester` : 위에서 설명한 두 가지 클래스와 `unittest.TestCase` 클래스를 상속하고 있기 때문에 테스트 실행이 가능하며, static 멤버로 `TProsper` 오브젝트 `proposer_factory` 변수를 선언하고 있다.

---

### Interface: `essential.py`

`essential` 모듈에서는 주로 인터페이스를 정의한다. 구체적인 클래스의 동작은 구현하지 않고 메서드만 남겨두거나, 기본적인 메서드만 정의해 둠으로써 후술할 `practical` 모듈 등에서 이를 상속받아 사용한다. 따라서 이 모듈에서는 주로 어떠한 인터페이스가 공개되는지 짚고 넘어가면 된다. 

이러한 구조는 `Java` 로 작성된 소스에서도 나타난다. `*Impl` 으로 명명된 클래스에서 인터페이스 상속하여 `implement` 시키는 구조로, `Python` 으로 작성된 구조와 동일하다.

Proposal ID는 아래와 같이 표현된다.

```python
ProposalID = collections.namedtuple('ProposalID', ['number', 'uid'])
```

`collection.namedtuple` 은 변수의 이름으로 튜플 값에 접근하도록 하는 방법이다. 일반적으로 튜플은 인덱스로 접근하는데, 아래 두 줄의 코드는 동일한 효과를 가지고 있다.

```python
print(ProposalID.number, ProposalID[0])  # name와 index 둘다 접근이 가능
print(ProposalID.uid, ProposalID[1])
```

총 4가지의 클래스를 정의한다.

- `Messenger` : 각 클래스의 멤버 변수로 선언된다. Paxos 알고리즘에서 통신이 일어나는 메커니즘을 정의한다. 총 다섯가지 `send_prepare`, `send_promise`, `send_accept`, `send_accepted`, `on_resolution` 인터페이스를 가지고 있고, **실제로 통신이 일어나지 않으므로 구현은 생략되어 있다.**
- `Acceptor` : Acceptor Role
- `Learner` : Learner Role
- `Proposer` : Proposer Role

**1. Messenger: 구현은 생략되어 있다.**

```python
class Messenger (object):
    def send_prepare(self, proposal_id):
		# Broadcasts a Prepare message to all Acceptors

    def send_promise(self, proposer_uid, proposal_id, previous_id, accepted_value):      
    # Sends a Promise message to the specified Proposer

    def send_accept(self, proposal_id, proposal_value):
    # Broadcasts an Accept! message to all Acceptors

    def send_accepted(self, proposal_id, accepted_value):
		# Broadcasts an Accepted message to all Learners

    def on_resolution(self, proposal_id, value):
		# Called when a resolution is reached
```

후술할 모든 Role이 통신을 위해 `messenger` 라는 이름의 변수로 이 상속 클래스 오브젝트를 선언하게 된다. 따라서 이 클래스는 인터페이스만 기억하자.

이 글에서 클래스 관계는 아래와 같은 UML 형태로 나타낸다.

![uml.png](/assets/posts/2022-04-08-analysis-plain-paxos/uml.png)

**2. Proposer**

[[Class] Proposer](https://www.notion.so/Class-Proposer-0b3d19cb2b5a451bb657e13d7daeb4f9)

기본 인터페이스는 아래와 같이 세 가지를 구현한다.

- `set_proposal` : 제안할 값을 설정한다. (Setter)

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%201.png)

- `prepare` :
    1. 우선 현재 요청에 대한 Response (Promise)를 보관할 `set` 초기화 해둔다. 
    2. 다음으로 제안할 Proposal Number를 하나 증가시키고, 
    3. `messenger` 를 통해서 *prepare<proposal_id>* 요청을 전송한다.

![essential.Proposer.prepare.png](/assets/posts/2022-04-08-analysis-plain-paxos/essential.Proposer.prepare.png)

- `recv_promise` :
    1. 이 Promise가 유효한지 검사한다: 
    (1) 이전에 요청을 보냈던 ID 인가? 
    (2) Prepare 보냈던 Proposal number (`proposal_id`)에 해당하는가?
    2. 초기화 했던 `set`에 어디로 부터 도착한 Response인지 기록 후,
    3. 가장 최근 받았던 response number 보다 크다면 `proposed_value` 에 기록.
    4. 모든 Node로 부터 요청이 도착했다면 (Quorum 크기 만큼 요청을 받은 경우) `messenger` 를 통해서 *accept<proposal_id, proposed_value>* 요청을 전송한다.

![essential.Proposer.recv_promise.png](/assets/posts/2022-04-08-analysis-plain-paxos/essential.Proposer.recv_promise.png)

~~이 클래스의 모든 변수는 static 으로 선언되어 있으며, 초기값은 `None` 타입이다.~~ 

static 변수인지 확인이 필요하다. 아래 `Java` 코드의 예시를 보자.

```java
public class EssentialAcceptorImpl implements EssentialAcceptor {
	
	protected EssentialMessenger messenger;
	protected ProposalID promisedID;
// ...

// ...
```

각 클래스는 자신이 필요한 변수를 로컬로 관리한다. `Python2` , `Python3`  문법/구현에 변경이 있었는지 확인한다.

윗 부분에서 언급했던 초기화 코드는 `test_*.py` 에 작성된다. 변수는 걸어둔 링크를 참고한다.

**3. Acceptor**

[[Class] Acceptor](https://www.notion.so/Class-Acceptor-d2da46dc0de04c1595b41d35c773c341)

기본 인터페이스는 아래 두 가지를 구현한다.

- `recv_prepare` : prepare 요청의 proposal_id 를 보고,
    1. 가장 최근의 proposal_id (`self.proposal_id`) 보다 작은 값인 경우 반응하지 않는다.
    2. 같거나 크다면  `messenger` 를 통해서 *promise<proposal_id, accepted_value>* 요청을 전송한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%202.png)

- `recv_accept_request` : 노드가 가지고 있는 상태 (`self.promised_id` , `self.accepted_id` , `self.accepted_value`)를 업데이트 하고 `messenger` 를 통해서 *accepted<proposal_id, accepted_value>* 요청을 전송한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%203.png)

변수는 걸어둔 링크를 참고한다.

**4. Learner**

[[Class] Learner](https://www.notion.so/Class-Learner-87e75dce56d443008217d4171dad7e96)

한 가지 인터페이스를 구성하고 있다.

- `recv_accepted` : Acceptor 로부터 메세지를 받는 경우 호출하는 함수이다. 각 Learner는  내부적으로 Acceptor와 Proposal을 `dict` 형태로 관리한다.
    1.  `from_uid` 의 ID를 가진 Acceptor 로부터 요청이 들어오는 경우 이전에 보냈던 값보다 큰지 검사한다.
    2. 제안한 값에 대한 요청이 Quorum 의 크기와 동일한지 검사한 후 맞다면 값을 `self.final_value` 로 반영시킨다. 합의가 이루어진 순간이므로 내부적으로 관리하고 있던 `self.acceptor` , `self.proposal` 정보는 제거한다.
    3. 합의가 이루어졌다면  `messenger` 를 통해 `on_resolution` 함수를 호출한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%204.png)

** Learner 클래스에서 내부적으로 제안값을 유지시키는 방식은 [내부 링크](https://www.notion.so/Class-Learner-87e75dce56d443008217d4171dad7e96)를 참고한다.*

각 관계를 나타내면 아래와 같다.

![uml.png](/assets/posts/2022-04-08-analysis-plain-paxos/uml%201.png)

---

### Extension 1: `practical.py`

이 `practical` 모듈에서는 앞서 언급했던 `essential` 모듈의 클래스를 상속받아 사용하며, 문자 그대로 practical한 기능이 추가되어 있다. 어떠한 기능이 추가/재정의 되었는지 자세하게 설명하면 다음과 같다.

* *회색으로 표시된 부분은 부모 클래스의 기능을 그대로 가져온다.*

**1. Messenger (essential.Messenger): 여전히 구현은 생략되어 있다.** 기존 클래스에서 확장되어 추가된 몇 가지의 인터페이스가 있다. 이는 아래와 같다.

[상속 인터페이스](https://www.notion.so/Analysis-Plain-Paxos-Implementations-f5f180acafac42b991d7e476e954cedb) 리스트

- `send_prepare` : *Prepare* 메세지를 모든 Acceptor로 브로드캐스트 시킨다.
- `send_promise` : 특정한 Proposer에게 *Promise* 메세지를 전송한다.
- `send_accept` : *Accept* 메세지를 모든 Acceptor에게 전송한다.
- `send_accepted` : 모든 Learner에게 *Accepted* (ACK) 메세지를 전송한다.
- `on_resolution` : 합의에 도달했을 때 이 함수를 호출한다.

**추가된 인터페이스 리스트**

- `send_prepare_nack` : 특정 노드로 *Prepare NACK* 메세지를 전송한다.
- `send_accept_nack` : 특정 노드로 *Accepted NACK* 메세지를 전송한다.
- `on_leadership_acquired` : Leadership을 획득했을 경우 이 함수 (Callback)를 호출한다. 이 함수를 호출하는 것은 특정 노드가 Leader 으로 선출된 것이 확정됨을 보장하지 않는다. 다른 노드가 이 함수를 호출하는 시점에서 Leader 로 선출되었을 가능성이 존재한다. 따라서 이 함수를 호출할 때 세심한 주의가 필요하다.

```python
class Messenger (essential.Messenger):
    
    def send_prepare_nack(self, to_uid, proposal_id, promised_id):
        # Sends a Prepare Nack message for the proposal to the specified node

    def send_accept_nack(self, to_uid, proposal_id, promised_id):
        # Sends a Accept! Nack message for the proposal to the specified node

    def on_leadership_acquired(self):
        '''
        Called when leadership has been acquired. This is not a guaranteed
        position. Another node may assume leadership at any time and it's
        even possible that another may have successfully done so before this
        callback is executed. Use this method with care.

        The safe way to guarantee leadership is to use a full Paxos instance
        with the resolution value being the UID of the leader node. To avoid
        potential issues arising from timing and/or failure, the election
        result may be restricted to a certain time window. Prior to the end of
        the window the leader may attempt to re-elect itself to extend its
        term in office.
        '''
```

마찬가지로, 구현은 생략되어 있으므로 인터페이스를 기억하고 넘어가자.

**2. Proposer (essential.Proposer)**

[[Class] Proposer (essential.Proposer)](https://www.notion.so/Class-Proposer-essential-Proposer-51805dbb32d94906b3bb24e01996fb6e)

이 모듈 기존 `essential.Proposer` 클래스를 상속받고 있고, **다음과 같은 기능이 추가**되었다.

- Paxos 인스턴스에서 **자기 자신이 Leader 인지, 또는 아닌지 추적**할 수 있다.
내부적으로 한 오브젝트가 리더인지 인지하도록 하는 방법으로는 `boolean` 타입의 변수로 판단한다. 따라서, 동시에 여러 Leader가 있을 수 있다. (Attribute *‘leader’*)
- Paxos 인스턴스에서 Active Participation (적극적인 의사 결정 참여)을 비활성화 할 수 있는 옵션이 추가되었다.
Active 속성을 나타내는 것은 마찬가지로 `boolean` 타입의 변수로 결정한다. **Active 하다는 것은 외부로 메세지 전송이 가능**하다는 것을 의미하고, Passive (Not active) 하다면 메세지 수신은 가능하지만 전송은 하지 않는다는 것을 의미한다.

인터페이스는 아래와 같다.

 [**재정의 인터페이스](https://www.notion.so/Analysis-Plain-Paxos-Implementations-f5f180acafac42b991d7e476e954cedb) 리스트**

- `set_proposal` :  제안할 값을 설정한다. 
단순히 제안할 값을 설정하는 기존의 기능에서, **만일 자신이 Leader이고 Active 하다면 (추가)**, (`self.leader` , `self.active` 플래그가 설정되어 있는 경우) `messenger` 를 이용하여 `send_accept` 함수를 호출한다.

![*파란색으로 표시된 부분은 변경된 부분이다.*](/assets/posts/2022-04-08-analysis-plain-paxos/practical.Proposer.set_proposal.png)

*파란색으로 표시된 부분은 변경된 부분이다.*

- `prepare` : 기존의 기능에서, **만일 자신이 Active 하다면 (추가)** `messenger` 를 이용하여 `send_prepare` 함수를 호출한다.

![practical.Proposer.prepare.png](/assets/posts/2022-04-08-analysis-plain-paxos/practical.Proposer.prepare.png)

- `recv_promise`
    1. `**observe_proposal` 을 호출한다. (추가)**
    2. **Leader 가 아닌 경우 그대로 함수를 종료한다. (추가)**
    3. 이 Promise가 유효한지 검사한다: 
    (1) 이전에 요청을 보냈던 ID 인가? 
    (2) Prepare 보냈던 Proposal number (`proposal_id`)에 해당하는가?
    4. 내부 변수 `self.promises_rcvd` 에 어떤 노드로부터 응답이 왔는지 기록 후,
    5. 가장 최근 받았던 response number 보다 크다면 `proposed_value` 에 기록.
    6. 모든 Node로 부터 요청이 도착했다면 (Quorum 크기 만큼 요청을 받은 경우) **자기 자신을 Leader로 표기하고 `messenger`를 통해 `on_leadership_acquired`를 호출한다. (추가)** `messenger` 를 통해서 *accept<proposal_id, proposed_value>* 요청을 전송한다.

![practical.Proposer.recv_promise.png](/assets/posts/2022-04-08-analysis-plain-paxos/practical.Proposer.recv_promise.png)

**추가된 인터페이스 리스트**

- `observe_proposal` : (Optional) 네트워크 상에서 Proposal 값을 관찰하고, 자신이 다음 전송할 값보다 크다면, 다음 값을 관찰한 값보다 1 큰 수로 설정한다. 
이 기능은 부가적인 기능으로, Leadership 획득 시도 시 지연 (제안 값이 너무 작은 경우 NACK을 발생시키는 경우가 반드시 일어나기 때문이다.) 을 방지하는 목적이라고 명시하고 있다.

![practical.Proposer.observe_proposal.png](/assets/posts/2022-04-08-analysis-plain-paxos/practical.Proposer.observe_proposal.png)

- `~~recv_prepare_nack~~` : 단순히 `observe_proposal` 의 Wrapper 함수이다.
- `recv_accept_nack` : 명시적인 NACK을 받았을 때 응답으로 호출되는 함수이다. **구현은 생략되어 있다.**
- `resend_accept` : 자기 자신이 Leader, Active인 경우 `messenger` 를 이용하여 `send_accept` 함수를 호출한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%205.png)

**3. Acceptor (essential.Acceptor):**

[[Class] Acceptor (essential.Acceptor)](https://www.notion.so/Class-Acceptor-essential-Acceptor-d02ad97e7aa341a0b7dc4f420876445b)

이 모듈은 기존 `essential.Acceptor` 클래스를 상속받고 있다. Paxos 알고리즘에서 Acceptor는 마치 Fault-tolerant Memory와 같이 작동하기 때문에, 각 Acceptor는 자신들이 만들었던 Promise를 기억해 둘 필요가 있다. 따라서 `promised_id`, `accepted_id`, `accepted_value` 와 같은 값들은 안정적인 미디어에 기록된 후 전송되어야 한다. 따라서 다음의 기능이 추가되었다.

- `persistance_required` 함수를 호출하여 검사한다.

이와 함께 Paxos 알고리즘은 패킷 유실을 허용하는 알고리즘이고, 모든 요청에 대한 응답을 할 의무는 없다. 때문에 요청이 들어온 후 처음 요청에 대한 응답만 전송하는 경우가 존재할 수 있고, 이렇게 되면 이후에 들어오는 모든 요청은 무시되어버린다. 따라서 Persistent 한 방식에 더해 응답 대기 기능이 추가된다.

또한, `Proposer`과 동일하게,

- Paxos 인스턴스에서 Active Participation (적극적인 의사 결정 참여)을 비활성화 할 수 있는 옵션이 추가되었다.
Active 속성을 나타내는 것은 마찬가지로 `boolean` 타입의 변수로 결정한다. **Active 하다는 것은 외부로 메세지 전송이 가능**하다는 것을 의미하고, Passive (Not active) 하다면 메세지 수신은 가능하지만 전송은 하지 않는다는 것을 의미한다.

[재정의 인터페이스](https://www.notion.so/Analysis-Plain-Paxos-Implementations-f5f180acafac42b991d7e476e954cedb) 리스트

- `recv_prepare` : prepare 요청의 proposal_id 를 보고,
    1. 가장 최근의 proposal_id (`self.proposal_id`) 보다 작은 값인 경우 반응하지 않는다.
    2. **같거나 크다면, 그리고 자신이 Active 하다면**  `messenger` 를 통해서 *promise<proposal_id, accepted_value>* 요청을 **`pending_promise`에 대기시킨다.  (추가 및 변경)**
    3. **그 이외의 경우에는, 그리고 자신이 Active 하다면 `messenger` 를 통해서 `send_prepare_nack`요청을 `pending_accepted`에 대기시킨다. (추가)**

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%206.png)

- `recv_accept_request` :
    1. 노드가 가지고 있는 상태 (`self.promised_id` , `self.accepted_id` , `self.accepted_value`)를 업데이트 하고 , **제안된 값이 `promised_id` 보다 크고, 또 자신이 Active 하다면 (추가)** `messenger` 를 통해서 *accepted<proposal_id, accepted_value>* 요청을 전송한다.
    2. **그 이외의 경우에는, 그리고 자신이 Active 하다면 `messenger` 를 통해서 `send_prepare_nack`요청을 전송한다. (추가)**

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%207.png)

**추가된 인터페이스 리스트**

- `recover` : 함수 인자로 제공된 값들로 복원한다. (Setter)
- `persisted` : 대기중인 메세지 (*Promise* and/or *Accepted*) 를 전송한다.
    1. 내가 Active 하다면,
        1. `pending_promise` 가 있다면  `messenger` 를 통해서 `send_promise` 를 호출한다.
        2. `pending_accepted` 가 있다면  `messenger` 를 통해서 `send_accepted` 를 호출한다.
    2. 그리고 대기 플래그인 `pending_promise`, `pending_accepted` 를 초기화한다.

**4. Learner (essential.Learner):**

[[Class] Learner (essential.Learner)](https://www.notion.so/Class-Learner-essential-Learner-e95868b8436e4d9bb93dee0b8fce900c)

- [재정의 인터페이스](https://www.notion.so/Analysis-Plain-Paxos-Implementations-f5f180acafac42b991d7e476e954cedb) 리스트
    - `recv_accepted` : Acceptor 로부터 메세지를 받는 경우 호출하는 함수이다. 각 Learner는  내부적으로 Acceptor와 Proposal을 `dict` 형태로 관리한다.
        1. **내부 변수 `final_acceptors` 를 두어 요청을 보내는 노드를 관리한다. 
        인자 `accepted_value` 가 현재 가지고 있는 `final_value`와 동일하다면 그 노드에서 요청이 왔음을 기록하고 종료한다. (추가)**
        2. `from_uid` 의 ID를 가진 Acceptor 로부터 요청이 들어오는 경우 이전에 보냈던 값보다 큰지 검사한다.
        3. 제안한 값에 대한 요청이 Quorum 의 크기와 동일한지 검사한 후 맞다면 값을 `self.final_value` 로 반영시킨다. 합의가 이루어진 순간이므로 내부적으로 관리하고 있던 `self.acceptor` , `self.proposal` 정보는 제거한다.
        4. 합의가 이루어졌다면  `messenger` 를 통해 `on_resolution` 함수를 호출한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%208.png)

앞서 정의된 Learner 와 다른 점은 마지막 값을 가지고 있는 노드를 추적하는 기능이 포함되어 있다는 것이다. 하이라이트로 표기된 부분이 추가된 부분이다. 

**5. Node (practical.Proposer, practical.Acceptor, practical.Learner)**

[[Class] Node **(practical.Proposer, practical.Acceptor, practical.Learner)**](https://www.notion.so/Class-Node-practical-Proposer-practical-Acceptor-practical-Learner-b172f9bcfc964efa9059902d5c11c69e)

이 클래스는 단순히 세 개의 Role을 나타내는 클래스를 상속받고 있다. 이에 이 `Node`는 어떠한 역할도 수행이 가능한 Generalized Role을 나타낸다. 인터페이스는 아래와 같다.

**상속된 인터페이스 리스트**

From `practical.Acceptor`

- `recv_accept_request`
- `recover`
- `persisted`

From `practical.Proposer`

- `observe_proposal`
- `recv_prepare_nack`
- `recv_accept_nack`
- `resent_accept`
- `recv_promise`
- `recv_accept_request`
- `prepare`

From `practical.Learner`

- `recv_accepted`

**추가된 인터페이스 리스트**

- `change_quorum_size` : 내부 변수 Quorum 크기를 변경한다.

**재정의 인터페이스 리스트**

- `recv_prepare` : prepare 요청의 proposal_id 를 보고,
    1. `**observe_proposal` 함수를 호출하여 네트워크 상에서 Proposal 값을 관찰한다. (추가)**
    2. 가장 최근의 proposal_id (`self.proposal_id`) 보다 작은 값인 경우 반응하지 않는다.
    3. **같거나 크다면, 그리고 자신이 Active 하다면**  `messenger` 를 통해서 *promise<proposal_id, accepted_value>* 요청을 **`pending_promise`에 대기시킨다.** 
    4. **그 이외의 경우에는, 그리고 자신이 Active 하다면 `messenger` 를 통해서 `send_prepare_nack`요청을 `pending_accepted`에 대기시킨다.**

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%209.png)

`practical` 모듈의 관계를 나타내면 아래와 같다.

* *재정의되거나 상속받은 인터페이스는 생략한다.*

![practical-module.png](/assets/posts/2022-04-08-analysis-plain-paxos/practical-module.png)

---

### Extension 3: `functional.py`

이 `functional` 모듈에서는 앞의 `practical` 모듈의 기능을 확장한다. 이 모듈에서는 두 개의 상속 클래스 `HeartbeatMessenger`와 `HeartbeatNode`를 정의한다.

* *회색으로 표시된 부분은 부모 클래스의 기능을 그대로 가져온다.*

**1. HeartbeatMessenger (practical.Messenger): 여전히 구현은 생략되어 있다.** 기존 클래스에서 확장되어 추가된 몇 가지의 인터페이스가 있다. 이는 아래와 같다.

- [상속 인터페이스](https://www.notion.so/Analysis-Plain-Paxos-Implementations-f5f180acafac42b991d7e476e954cedb) 리스트
    - `send_prepare` : *Prepare* 메세지를 모든 Acceptor로 브로드캐스트 시킨다.
    - `send_promise` : 특정한 Proposer에게 *Promise* 메세지를 전송한다.
    - `send_accept` : *Accept* 메세지를 모든 Acceptor에게 전송한다.
    - `send_accepted` : 모든 Learner에게 *Accepted* (ACK) 메세지를 전송한다.
    - `on_resolution` : 합의에 도달했을 때 이 함수를 호출한다.
    - `send_prepare_nack` : 특정 노드로 *Prepare NACK* 메세지를 전송한다.
    - `send_accept_nack` : 특정 노드로 *Accepted NACK* 메세지를 전송한다.
    - `on_leadership_acquired` : Leadership을 획득했을 경우 이 함수 (Callback)를 호출한다. 이 함수를 호출하는 것은 특정 노드가 Leader 으로 선출된 것이 확정됨을 보장하지 않는다. 다른 노드가
- **추가된 인터페이스 리스트**
    - `send_heartbeat` : 모든 노드에게 Heartbeat 시그널을 보낸다.
    - `schedule` : Leadership을 보유하고 있는 경우 `pulse` 함수에 의해 호출되는 함수이다. (Callback) 이 함수가 만일 상속 클래스에서 올바르게 재정의 (Override) 되지 않는 경우, 자식 클래스에서는 `on_leadership_acquired` 와  `on_leadership_lost`을 명시적으로 호출하여 `pulse` 함수 `hb_period` 주기로 실행되는 것을 보장해야 한다. *
    - `on_leadership_lost` : Leadership을 잃어버리는 경우 호출된다.
    - `on_leadership_change` : Leadership의 변경이 감지되는 경우 호출된다.
    
    ** `schedule` 과 `pulse`의 경우 `HeartbeatNode`의 용례를 확인한다.*
    

```python
class HeartbeatMessenger (practical.Messenger):

    def send_heartbeat(self, leader_proposal_id):
        '''
        Sends a heartbeat message to all nodes
        '''

    def schedule(self, msec_delay, func_obj):
        '''
        While leadership is held, this method is called by pulse() to schedule
        the next call to pulse(). If this method is not overridden appropriately, 
        subclasses must use the on_leadership_acquired()/on_leadership_lost() callbacks
        to ensure that pulse() is called every hb_period while leadership is held.
        '''

    def on_leadership_lost(self):
        '''
        Called when loss of leadership is detected
        '''

    def on_leadership_change(self, prev_leader_uid, new_leader_uid):
        '''
        Called when a change in leadership is detected. Either UID may
        be None.
        '''
```

**2. HeartbeatNode (practical.Node):**

[[Class] HeartbeatNode (practical.Node)](https://www.notion.so/Class-HeartbeatNode-practical-Node-af8422ac98df4b8ca13250db553e0429)

이 클래스는 `practical.Node` 클래스를 상속받는다. Paxos의 기본적인 Heartbeating Mechanism을 구현하고 있으며, Leader 장애와 Leadership 획득을 시도하는 기능을 구현한다.

만약 하나 이상의 Heartbeat 메세지가 `liveness_window` 이내에 도착하지 않는 경우 다음과 같은 단계를 거쳐 Leadership 획득을 시도한다.

1. Phase 1-a의 *Prepare* 메세지를 전송한다.
2. 만일 Leadership 변경에 대한 Quorum 수의 응답이 도착하는 경우 Leadership을 획득하며, Hearbeat 메세지를 보내기 시작한다.
3. Leadership 변경에 대한 Quorum 수의 응답이 도착하지 않는 경우 그 수 만큼의  응답이 도착할 때 까지 `liveness_window` 마다 *Prepare* 메세지를 전송하거나, 받았던 제안 값 이상의 Proposal ID와 Heartbeat 를 같이 보낸다.
    - `hb_period` 와 `liveness_window`는 초 단위이며, 부동소수점으로 설정된다.

Leadership Loss는 다음 두 가지의 방식으로 감지한다.

- Proposer 로부터 제안 값 보다 더 큰 수의 제안 값을 가진 Heartbeat 시그널을 받는 경우
- 또는, *Accept* 요청에 대해 Quorum 수 만큼의 *NACK* 응답을 받는 경우

인터페이스는 아래와 같다.

[상속된 인터페이스](https://www.notion.so/Analysis-Plain-Paxos-Implementations-f5f180acafac42b991d7e476e954cedb) 리스트

- From `practical.Acceptor`
    - `recv_accept_request`
    - `recover`
    - `persisted`
- From `practical.Proposer`
    - `observe_proposal`
    - `recv_prepare_nack`
    - `recv_accept_nack`
    - `resent_accept`
    - `recv_promise`
    - `recv_accept_request`
- From `practical.Learner`
    - `recv_accepted`

**재정의 인터페이스 리스트**

From `practical.Acceptor`

- `recv_prepare` : prepare 요청의 proposal_id 를 보고,
    1. `**observe_proposal` 함수를 호출하여 네트워크 상에서 Proposal 값을 관찰한다.**
    2. 가장 최근의 proposal_id (`self.proposal_id`) 보다 작은 값인 경우 반응하지 않는다.
    3. **같거나 크다면, 그리고 자신이 Active 하다면**  `messenger` 를 통해서 *promise<proposal_id, accepted_value>* 요청을 **`pending_promise`에 대기시킨다.** 
    4. **그 이외의 경우에는, 그리고 자신이 Active 하다면 `messenger` 를 통해서 `send_prepare_nack`요청을 `pending_accepted`에 대기시킨다.**
    5. **가장 최근의 timestamp를 업데이트 한다. (추가)**

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2010.png)

From `practical.Proposer`

- `prepare` : **기록하고 있던 NACK을 초기화 하고 (추가)**, **만일 자신이 Active 하다면** `messenger` 를 이용하여 `send_prepare` 함수를 호출한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2011.png)

- `recv_promise` :
    1. `**observe_proposal` 을 호출한다.**
    2. **Leader 가 아닌 경우 그대로 함수를 종료한다.**
    3. 이 Promise가 유효한지 검사한다: 
    (1) 이전에 요청을 보냈던 ID 인가? 
    (2) Prepare 보냈던 Proposal number (`proposal_id`)에 해당하는가?
    4. 내부 변수 `self.promises_rcvd` 에 어떤 노드로부터 응답이 왔는지 기록 후,
    5. 가장 최근 받았던 response number 보다 크다면 `proposed_value` 에 기록.
    6. 모든 Node로 부터 요청이 도착했다면 (Quorum 크기 만큼 요청을 받은 경우) **자기 자신을 Leader로 표기하고 `messenger`를 통해 `on_leadership_acquired`를 호출한다.**  `messenger` 를 통해서 *accept<proposal_id, proposed_value>* 요청을 전송한다.
    7. **만일 과거에는 자신이 리더가 아니었지만 6번에 의해 현재 리더로 선출된다면, `pulse` 함수를 호출하고 `messenger`를 통해 `on_leadership_change`를 호출한다. (추가)**

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2012.png)

- `recv_prepare_nack` :
    1.  `observe_proposal` 을 호출하여 네트워크를 관찰하고,
    2. **현재 Leadership 획득을 시도 중이라면 `prepare` 함수를 호출한다. (추가)**

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2013.png)

- `recv_accept_nack`
    1. **현재 Proposal에 대한 NACK 을 받았다면 어떤 노드로부터 왔는지 기록한다. (재정의)**
    2. **자신이 현재 Leader이지만 Quorum 크기 만큼의 NACK 응답을 받았다면, Leader 자리를 포기하고,** 
    3. `**messenger`를 통해 아래의 함수를 호출한다. (추가)**
        1. `**on_leadership_lost**`
        2. `**on_leadership_change**`
        3. `**observe_proposal**`

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2014.png)

**추가된 인터페이스 리스트**

- `leader_is_alive` : `liveness_window` 보다 더 많은 시간이 지났는지, 최근의 Heartbeat 값과 비교한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2015.png)

- `observed_recent_prepare` : `liveness_window` 에 1.5배의 가중치를 두어 이보다 더 많은 시간이 지났는지, [최근의 Prepare 시간](https://www.notion.so/Analysis-Plain-Paxos-Implementations-f5f180acafac42b991d7e476e954cedb)과 비교한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2016.png)

- `poll_liveness` : 현재 리더가 Active 한지 체크하고 아니라면 Leadership 획득을 시도한다.
    1. `leader_is_alive` 함수와 `observed_recent_prepare` 함수의 리턴값이 `False` 라면,  
    (리더가 죽었고 `liveness_window` 보다 많은 시간이 지났는지 확인)
    현재 Leadership 획득 시도 중이라면 `prepare` 함수를 호출한다.
    현재 Leadership 획득 시도 중이 아니라면 `acquire_leadership` 함수를 호출한다.
    2. 리더가 죽지 않았다면 아무일도 하지 않는다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2017.png)

- `recv_heartbeat`
    1. 현재 `proposal_id` 가 Leader의 제안 값 보다 크다면 (Leader에 문제가 있는 경우)
        1. **자기 자신이 리더였다면**
            1. Heartbeat 시그널을 보낸 노드를 리더로 인식시키고,
            2. `messenger` 를 통해 `on_leadership_lost` 함수를 호출한다.
            3. `observe_proposal` 함수를 호출한다.
        2. 자신이 리더가 아니었다면 `messenger`를 통해  `on_leadership_change` 함수를 호출한다.
    2. 자기 자신이 여전히 리더라면 가장 최근의 Heartbeat 값을 기록한다. (timestamp)

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2018.png)

- `pulse` : `hb_period` 마다 호출되는 함수
    1. 자기 자신이 리더라면 다음의 함수를 호출한다.
        1. `recv_heartbeat`
        2. `messenger` 의 `send_heartbeat`
        3. `messenger` 의 `schedule`

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2019.png)

- `acquire_leadership` : Leadership 획득을 시도한다.
    1. `leader_is_alive` 하다면 `self._acquiring` 플래그를 `False` 로 세팅한다. 
    (획득 시도 하지 않음을 설정)
    2. 그게 아니라면 `self._acquiring` 플래그를 `True` 로 세팅 후 `prepare` 함수를 호출한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2020.png)

`functional` 모듈의 관계를 나타내면 아래와 같다.

* *재정의되거나 상속받은 인터페이스는 생략한다.*

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2021.png)

---

### Extension 4: `external.py`

이 `external` 모듈에서는 앞의 `practical` 모듈의 기능을 확장한다. 이 모듈에서는 두 개의 상속 클래스 `ExternalMessenger`와 `ExternalNode`를 정의한다. 이 모듈은 외부 Failure Detection을 지원한다.

**1. ExternalMessenger (practical.Messenger): 여전히 구현은 생략되어 있다.** 기존 클래스에서 확장되어 추가된 몇 가지의 인터페이스가 있다. 이는 아래와 같다.

[상속 인터페이스](https://www.notion.so/Analysis-Plain-Paxos-Implementations-f5f180acafac42b991d7e476e954cedb) 리스트

- `send_prepare` : *Prepare* 메세지를 모든 Acceptor로 브로드캐스트 시킨다.
- `send_promise` : 특정한 Proposer에게 *Promise* 메세지를 전송한다.
- `send_accept` : *Accept* 메세지를 모든 Acceptor에게 전송한다.
- `send_accepted` : 모든 Learner에게 *Accepted* (ACK) 메세지를 전송한다.
- `on_resolution` : 합의에 도달했을 때 이 함수를 호출한다.
- `send_prepare_nack` : 특정 노드로 *Prepare NACK* 메세지를 전송한다.
- `send_accept_nack` : 특정 노드로 *Accepted NACK* 메세지를 전송한다.
- `on_leadership_acquired` : Leadership을 획득했을 경우 이 함수 (Callback)를 호출한다. 이 함수를 호출하는 것은 특정 노드가 Leader 으로 선출된 것이 확정됨을 보장하지 않는다. 다른 노드가

**추가된 인터페이스 리스트**

- `send_leadership_proclamation` : Leadership Proclamation을 모든 노드에게 알린다.
- `on_leadership_lost` : Leadership을 잃어버리는 경우 호출된다.
- `on_leadership_change` : Leadership의 변경이 감지되는 경우 호출된다.

```python
class ExternalMessenger (practical.Messenger):

    def send_leadership_proclamation(self):
        '''
        Sends a leadership proclamation to all nodes
        '''
    
    def on_leadership_lost(self):
        '''
        Called when loss of leadership is detected
        '''

    def on_leadership_change(self, prev_leader_uid, new_leader_uid):
        '''
        Called when a change in leadership is detected. Either UID may
        be None.
        '''
```

2**. ExternalNode (practical.Messenger):**

[[Class] ExternalNode (practical.Node)](https://www.notion.so/Class-ExternalNode-practical-Node-bf6bd56a70ae456fb0412b6b2a1fdab7)

이 클래스의 구현은 Passive 하다고 설명하고 있다. 즉, 외부 개체가 Failure를 관찰해야 하고 Paxos 인스턴스가 Leadership을 획득하려 할때 `prepare` 을 호출한다는 것을 가정하고 있다. 또한, `prepare` 함수 호출과 Leadership Battle (Race Condition) 방지는 호출부의 책임으로 둔다.

Leadership Proclamation은 Leadership을 획득했을 때 브로드캐스트된다. 

인터페이스는 아래와 같다.

**상속된 인터페이스 리스트**

From `practical.Acceptor`

- `recv_accept_request`
- `recover`
- `persisted`

From `practical.Proposer`

- `observe_proposal`
- `recv_prepare_nack`
- `recv_accept_nack`
- `resent_accept`
- `recv_promise`
- `recv_accept_request`
- `prepare`
- From `practical.Learner`
    - `recv_accepted`

**재정의 인터페이스 리스트**

From `practical.Proposer`

- `recv_promise`
    1. `**observe_proposal` 을 호출한다.**
    2. **Leader 가 아닌 경우 그대로 함수를 종료한다.**
    3. 이 Promise가 유효한지 검사한다: 
    (1) 이전에 요청을 보냈던 ID 인가? 
    (2) Prepare 보냈던 Proposal number (`proposal_id`)에 해당하는가?
    4. 내부 변수 `self.promises_rcvd` 에 어떤 노드로부터 응답이 왔는지 기록 후,
    5. 가장 최근 받았던 response number 보다 크다면 `proposed_value` 에 기록.
    6. 모든 Node로 부터 요청이 도착했다면 (Quorum 크기 만큼 요청을 받은 경우) **자기 자신을 Leader로 표기하고 `messenger`를 통해 `on_leadership_acquired`를 호출한다.**  `messenger` 를 통해서 *accept<proposal_id, proposed_value>* 요청을 전송한다.
    7. **만일 과거에는 자신이 리더가 아니었지만 6번에 의해 현재 리더로 선출된다면, `messenger`를 통해`send_leadership_proclamation` 함수를 호출하고 `on_leadership_change`를 호출한다. (추가)**

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2022.png)

- `prepare` :
**기록하고 있던 NACK을 초기화 하고 (추가)**, **만일 자신이 Active 하다면** `messenger` 를 이용하여 `send_prepare` 함수를 호출한다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2023.png)

- `recv_accept_nack`
    1. **현재 Proposal에 대한 NACK 을 받았다면 어떤 노드로부터 왔는지 기록한다. (재정의)**
    2. **자신이 현재 Leader이지만 Quorum 크기 만큼의 NACK 응답을 받았다면, Leader 자리를 포기하고,** 
    3. `**messenger`를 통해 아래의 함수를 호출한다. (추가)**
        1. `**on_leadership_lost**`
        2. `**on_leadership_change**`
        3. `**observe_proposal**`

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2024.png)

**추가된 인터페이스 리스트**

- `recv_leadership_proclamation`
    1. 현재 `proposal_id` 가 Leader의 제안 값 보다 크다면 (Leader에 문제가 있는 경우)
        1. `observe_proposal` 함수를 호출하고,
        2. **자기 자신이 리더였다면** 
            1. `messenger` 를 통해 `on_leadership_lost` 를 호출한다.
        3. 그리고 `messenger` 를 통해 `on_leadership_change` 로 Leader의 변경을 알린다.
    2. 현재 `proposal_id` 가 Leader의 제안 값 보다 같거나 작다면 (Leader에 문제가 없는 경우)
    아무 반응을 하지 않는다.

![Untitled](/assets/posts/2022-04-08-analysis-plain-paxos/Untitled%2025.png)

최종적으로 각 모듈의 관계를 나타내면 아래와 같다. (드디어...)

* *재정의되거나 상속받은 인터페이스는 생략한다.*

![uml-extern.png](/assets/posts/2022-04-08-analysis-plain-paxos/uml-extern.png)

---