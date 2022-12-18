---
title:  "Introduction to Concurrency(Part 1)"
date:   2019-03-20 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - C/C++
tags:
   - C/C++
toc: true
toc_sticky: true
breadcrumbs: true
---

# C++ Multithreading (Part 1)
## 표준 쓰레드 지원 라이브러리 맛보기

## Hello Thread!! - 배경 이야기

처음 전입 왔을 당시인 17년에 제가 받았던 프로젝트가 하나 있었습니다. MFC 프로젝트였습니다. 팀 내부에서 자체적으로 만든 프로그램으로, **외부 업체**에서 제작한 프로그램을 베이스로 한, 일부를 변형시켜 작성한 프로젝트(RTD: 가명)였습니다. 

<!--more-->

간단하게 이야기하자면, 그 동안은 사용자가 정보를 직접 수기로 입력하는 것이었습니다. 예를 들어 한 모니터에 어떠한 정보가 표시되면 그것을 불러주는 사람이 따로 있고 다른 곳(컴퓨터)에 작성하는 사람 따로 있는, 그런 비효율적인 과정이었습니다. 어차피 그 정보들도 어느 곳에서 가져오는 것이니 그 로그를 읽어 자동으로 가져오면 되는 것 아니냐 해서 만들어 진 것이 그 RTD 프로젝트입니다. 조금 안타까운 것이라면 자동화라고는 해도 모든 로그를 긁어와 사람이 선택적으로 체크하고 해제해야 하는 작업도 필요해서 그렇게 많이 쓰이지는 않았던 체계입니다.

지금이야 예전에 비해 각종 의욕을 잃어버린 지 오래이지만 당시에만 해도 무슨 일이든 던져만 주면 해보이겠다는 의지가 충만했습니다. 학교에서 간단하게나마 MFC를 하긴 했지만 거의 보지도 않은 수준이었는데 소스는 받았으니 분석은 해야겠고, 해서 빠른 속도로 MFC 구조를 익히고 소스를 분석하기 시작했습니다. 무슨 생각이었는지 한쇼로 프로그램 작동 구조를 일일이 그리고 도표로 클래스와 메서드를 정리하기 시작했습니다. 새로 프로그램을 작성하는 일 보다 남이 작성한 소스를 쳐다보는 것이 더 고된 일이라는 걸 그때 확실히 피부로 느낀 것 같습니다. 

뒤져보니 아직도 관련 자료가 남아있습니다. 어떻게 일일이 그렸는지 다시 하라면 못 할 짓입니다. 한쇼 파일 여는 데에만 해도 5~6초가 걸립니다.

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-FF.jpg)

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-FE.jpg)


각설하고, RTD 프로젝트는 약간의 문제가 있었습니다. 로그에서 찍히는 정보가 어마무시하다 보니 이놈이 정보를 가져오고 처리하는데 시간을 좀 잡아먹습니다. 시뮬레이터 장비로 정보를 쏘게 되면 가져오고 전시하는 데에만 몇 십초. 어떤 로직으로 짜여져 있는지 그 도중에는 마우스나 각종 입력이 먹히지 않습니다. 

그래서 저에게 던져진 것이 데이터 처리와 사용자 입력 로직을 분리해라. 즉, 데이터 처리 과정을 별도의 쓰레드(Thread)로 만들어 사용자 입력 처리가 가능하게 하라는 것이었습니다. 하나의 쓰레드는 작업(worker thread)을, 다른 하나의 쓰레드는 사용자 입력을 받도록 하도록 만드는 작업이었습니다. 

*예전에 올려 둔 자료에 MFC Thread Control에 대한 내용이 있습니다.*

뭐, 어찌어찌해서 결국 별도의 쓰레드로 돌아가게 만들기는 했습니다. 그런데 소스를 갖다 주기도 전에 개발실장이 바뀌면서 프로젝트가 터졌습니다. 하지 말랍니다. 잠정 중단이죠. 이건 뭐 돌려보지도 못하고 갖다 버린 꼴이 되어버린 비운의 프로젝트입니다. 

Sad 엔딩이지만 이게 제가 처음 멀티쓰레드라는 놈을 만난 계기입니다.


## C++11, 쓰레드 지원 라이브러리

아무리 많은 복잡한 함수를 작성하고 복잡한 클래스를 작성하고, 복잡한 Recursive 동작을 구현해도 대부분 main()안에 구겨 넣는 식의 하나의 쓰레드(Window는 쓰레드 개념 기반)만 만들면 되었습니다. 그러나 MFC 또는 기타 언어(Java runnable 등)으로 멀티쓰레드 기반 프로그램을 작성해 본 경험이 있는 사람들은 아마 느끼시겠지만 멀티쓰레드 프로그램은 고려해야 할 대상이 아주 많습니다. 하나의 쓰레드가 돌아감과 동시에 다른 로직이 작동중이니 다수의 동작을 동시에 고려해야 하는 상황이고, 우리가 작성한 코드는 이제는 단순히 위에서 아래로가 아니라 여기저기에서 동시 다발적으로 작동하기 때문입니다. 

RTD 프로젝트는 Visual Studio 2008에서 작성된 프로젝트입니다. 물론 MFC니 윈도우에서 지원해주는 각종 쓰레드 라이브러리를 이용하는게 훨씬 더 효율적이었겠지만 만일 유닉스/리눅스에서 동일하게 작동하는 로직을 필요로 하는 소요가 들어왔다면 아마 pthread 라이브러리를 이용하여 다시 한 번 더 갈아엎었어야 했을 겁니다. 2008이면 C++11을 지원하지 않기 때문입니다.

C++11은 언어적으로도 그렇지만 표준 라이브러리 측면에서도 추가된 사항이 굉장히 많습니다. “이전의 C++과 C++11이후는 완전히 새로운 언어라고 느껴진다.”라는 표현이 딱 들어 맞는 것 같습니다. C++11에 드디어 쓰레드 지원 라이브러리가 추가되었습니다. 

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-00.jpg)

C++ 표준의 쓰레드 지원은 크게 두 부분으로 구성되는데, 하나는 잘 정의된(well-defined) 메모리 모형이고 또 하나는 표준화된 쓰레드 적용 인터페이스입니다.

다중 쓰레드 적용의 기초는 잘 정의된 메모리 모형입니다. 메모리 모형(Memory model)은 반드시 다음과 같은 사항을 지원해야됩니다. 

- 원자적(atomic) 연산: 가로채기(interrupt)없이 수행할 수 있는 연산.
- 연산들의 부분 순서(partial ordering): 일련의 연산들의 순서를 바꾸지 않아야 한다.
- 연산의 가시적 효과: 공유 변수에 대한 연산의 결과를 다른 쓰레드에서도 볼 수 있음을 보장해야 한다.

C++ 메모리 모형에 대한 내용은 따로 공부 중이니 Part 1에서는 쓰레드 적용 인터페이스부터 알아보는 거로 합시다. 

## 환경

들어가기 전에 컴파일 환경은 Ubuntu 16.04 LTS([구름IDE](https://ide.goorm.io)), gcc/g++ 8.1로 두겠습니다. C++11은 gnu 컴파일러 4.7 이상이면 됩니다. 기본적으로 구름IDE에서 빈 프로젝트로 컨테이너 생성 시 gcc/g++ 4.8.4 버전이 깔려 있습니다. 

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-01.jpg)

쓰레드 라이브러리를 컴파일하기에 충분할 것이라 생각은 됩니다만 추후 C++17 라이브러리를 쓸 수도 있으니 그냥 최신 버전으로 업데이트를 미리 해 두겠습니다. 몇 번 삽질 한 결과 아래와 같이 진행하면 됩니다. 

```bash
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
```

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-02.jpg)

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-03.jpg)

```bash
sudo apt-get install gcc-8 g++-8
```

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-04.jpg)

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-05.jpg)

아직도 4.8.4 버전입니다. 업그레이드 버전으로 잡아 줍니다.

```bash
sudo update-alternatives --install /usr/bin/gcc /usr/bin/gcc-8 800 --slave /usr/bin/g++ /usr/bin/g++-8
```
![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-06.jpg)

확인하고 gdb도 깔아 줍시다. 빈 컨테이너에는 gdb가 없습니다.

```bash
sudo apt-get install gdb
```

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-07.jpg)

그러면 환경 설정은 완료됩니다.

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-08.jpg)

인터넷이 되는 환경이라 자료 찾아보기도 편하고 너무 좋습니다. 

추후 예제에 쓸 함수를 하나 만들어 둡시다. 리눅스에서는 윈도우 VS에서 지원하는 getch() 함수가 존재하지 않습니다. 따라서 아래와 같이 하나 만들어 둡니다. 

```cpp
#include <stdio.h>
#include <termio.h>

int getch(void) {

	int ch;
	struct termios buf, save;
	
	tcgetattr(0, &save);
	
	buf = save;
	
	buf.c_flag &= ~(ICANON | ECHO);
	buf.c_cc[VMIN] = 1;
	buf.c_cc[VTIME] = 0;
	
	tcsetattr(0, TCSAFLUSH, &buf);
	
	ch = getchar();
	tcsetattr(0, TCSAFLUSH, &save);
	
	return ch;

}
```

## Thread 생성

cppreference.com([링크](https://en.cppreference.com/w/))은 참 편리합니다. 방대한 표준 라이브러리 내용을 예제와 함께 담고 있어 사전처럼 찾아보기 좋습니다. 일단 C++ 다중 쓰레드 인터페이스를 사용하려면 thread 헤더가 필요합니다.

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-09.jpg)

쓰레드(Thread)는 하나의 실행 단위입니다. C++에서는 std::thread라는 클래스 객체로 존재하게 됩니다. 기본적으로 쓰레드 객체는 생성 즉시 실행을 시작합니다. 쓰레드가 실행하는 코드는 생성자의 인수로 주어진 호출 가능 단위가 됩니다. 여기서 호출 가능 단위는 __①함수일 수도 있고__  __②함수 객체(Functor)__나  __③람다 함수__일 수도 있습니다. 생성자를 확인해 봅시다.

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-10.jpg)

우리가 주목해야 할 것은 세 번째(3) 생성자의 파라미터입니다. 가변인수 꼴이며 더 자세히는,

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-11.jpg)

라고 되어 있습니다. 호출 가능한 단위와 그 단위에 전달할 파라미터를 가변적으로 받고 있습니다. 조금 더 자세히 들어가면, 

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-12.jpg)

기본적으로 std::thread로 생성된 쓰레드는 리턴값은 무시합니다. 또한 예외를 던지는 경우 std::terminate가 호출된다고 합니다. 쓰레드가 리턴하는 값은 std::promise와 std::future을 이용해야 하는데 이는 Part 2에서 다루겠습니다.

그러면 일단은 한번 만들어 봅시다.

호출 가능 단위는 함수, 함수 객체, 람다가 가능합니다. 그러면 함수 thread_function(), 함수 객체ThreadFunctor를 만들어 둡시다. 

```cpp
// declaration scope
void thread_function() {
    
	APP_THREAD_START // head of a thread.
	
	std::cout << " > thread_function() running as a thread." << std::endl;
	
	APP_THREAD_TERMINATE // end!!
};


class ThreadFunctor {
public:
    void operator()() {
        APP_THREAD_START // head of a thread.
	
		std::cout << " > ThreadFunctor::operator() running as a thread." << std::endl;
	
		APP_THREAD_TERMINATE // end!!
    };
};
```

그리고 main() 함수는 아래와 같습니다.

```cpp
#if defined(_UNIT_TEST_MODE_)

int main() try {
	
	// compilation options
	// g++ -o app ./include/*.h ./src/*.cpp -pthread -std=c++14 -g
	// g++ -o app /workspace/gen_blank/link/include/*.h /workspace/gen_blank/link/src/*.cpp -pthread -std=c++14 -g
	
	namespace jc = _jcode;
	
	APP_BANNER // jc namespace is declared in app.h, for visibility!!
	APP_THREAD_START
	
	auto thread_start = std::chrono::system_clock::now();
	
	
#define _WHAT_ 7
	std::cout << " > Unit test case: " << _WHAT_ << std::endl;
	
	std::cout << " > " << std::thread::hardware_concurrency() << " concurrent threads are supported." << std::endl;

	
#if(_WHAT_ == 1) // part 1 - Create Threads
		
	// What std::thread accepts in constructor?
	// We can attach a callback with the std::thread object, that will be executed when this new thread starts. These callbacks can be,

	// 1.) Function Pointer
	// 2.) Function Objects
	// 3.) Lambda functions
	
	// 1. Creating a thread using Function Pointer
	std::thread worker_thread_1(thread_function);
	
	// 2. Creating a thread using Function Objects
	std::thread worker_thread_2((ThreadFunctor()));
	
	// 3. Creating a thread using Lambda functions
	std::thread worker_thread_3(
		[]() {
			APP_THREAD_START
			std::cout << " > An anonymous lambda running as a thread." << std::endl;
			APP_THREAD_TERMINATE
		}
	);
	
	worker_thread_1.join();
	worker_thread_2.join();
	worker_thread_3.join();
```

std::thread::hardware_concurrency()는 CPU의 개수를 리턴한다고 보면 될 것 같습니다. worker_thread_1은 함수를, worker_thread_2는 함수 객체를, worker_thread_3는 람다를 인자로 받았습니다. std::thread는 객체 생성과 동시에 쓰레드를 실행한다고 했으니 확인해 봅시다. 

컴파일은 한 번에 합시다. 이후 사용할 컴파일 명령어는 아래와 같습니다. 

```bash
g++ -o app ./include/*h ./src/*.cpp -pthread -std=c++14 -g
```

실행은 아래와 같습니다.

```bash
./app
```

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-13.jpg)

간단한 예제입니다. [thread_id: XXXXXX] 부분은 단순한 코드입니다. 

```cpp
std::cout << "[thread_id: " << std::this_thread::get_id() << ...
```

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-14.jpg)


즉, 쓰레드를 구분하는 ID값을 반환합니다. 맨 처음 main() 함수[140132807874368]가 시작되고 객체가 생성됨과 동시에 worker_thread_3[140132773603072], worker_thread_2[140132781995776], worker_thread_1[140132790388480] 쓰레드가 작동하는 것을 확인할 수 있습니다.

## Thread 수명

쓰레드의 수명은 쓰레드를 생성한 코드(현재 쓰레드)에서 관리해야 합니다. 생성된 쓰레드의 실행은 해당 호출 가능 단위의 실행이 끝나면 함께 끝납니다. 

생성된 쓰레드 객체가 T라 할 때, T를 생성한 코드는 T의 실행을 마칠 때 까지 기다릴 수도 있고(T.join()), 아니면 명시적으로 T를 자신으로부터 떼어낼 수도 있습니다(T.detach()). join() 이나 detach()가 한 번도 수행되지 않은 쓰레드를 합류 가능한(joinable)이라고 합니다. 합류 가능한 쓰레드는 소멸자에서 std::terminate를 호출하고, 그러면 기본적으로 프로그램이 종료됩니다.

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-15.jpg)

자신을 생성한 코드에서 떨어진 쓰레드는 배경에서 독자적으로 실행됩니다. 그런 쓰레드를 흔히 데몬(daemon)쓰레드라고 부릅니다.

혹시라도 쓰레드 객체를 생성하고 join()이나 detach()를 실행하지 않으면 프로그램이 죽어버립니다. 

## Thread 인수 전달

앞서 이야기 했듯, std::thread는 가변 인수 템플릿(variadic template)입니다. 간단히 말해 이는 클래스의 생성자가 임의의 개수의 인수들을 복사 또는 참조로 전달받을 수 있다는 뜻입니다. std::thread의 경우 첫 인수는 쓰레드가 실행할 호출 가능 단위이고, 그 이후의 임의의 개수의 인수들은 그 호출 가능 단위에 전달됩니다. 물론, 호출 가능 단위가 람다 함수인 경우에는 생성자 인수 대신 람다 capture을 통해 인수를 전달할 수도 있습니다.

```cpp
#elif(_WHAT_ == 2) // part 2 - Joining and Detaching Threads
	
	int divider = 1;
	std::vector<std::thread> worker_threads;
	
	// Once a thread is started then another thread can wait for this new thread to finish. 
	// For this another need need to call join() function on the std::thread object.
		
	for(int i = 0; i < 10; i++) {
		worker_threads.push_back(
			std::thread(
				[=](int arg_div) { // catch by copying.
					APP_THREAD_START
					std::cout << " > An anonymous lambda #" << arg_div << " running as a thread." << std::endl;
					
					APP_THREAD_TERMINATE
				}
			, divider)
		);
		
		divider++;
	}
	
	
	std::this_thread::sleep_for(std::chrono::seconds(1));
	
	std::cout<<" > Waiting for all the worker thread to finish..."<<std::endl;
	for(auto& itor : worker_threads)
		if(itor.joinable())
			itor.join();
```

예를 들어, 132번 줄에 std::thread 생성자에 int형 인자를 받는 람다 함수가 들어가 있습니다. 이에 대한 파라미터는 std::thread 두 번째 파라미터로 들어가고 있습니다. 이를 출력해 보면, 

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-16.jpg)

쓰레드가 수행되는 순서라 함은 OS에서 알아서 정하는 것이니 생성한 순서대로 수행되지는 않습니다. 어쨌든 인자가 잘 들어가고 있는 건 확인할 수 있습니다.


## Thread 연산

뭐, 종합하면 아래와 같이 됩니다. 


| 메서드 | 설명 |
| ---
| t.join() | 쓰레드 t(호출 가능 단위)가 실행을 마칠 때까지 기다린다. |
| t.detach() | 생성된 쓰레드 t를 그것을 생성한 쓰레드와 독립적으로 실행되게 한다. |
| t.joinable() | 쓰레드 t에 대해 join이나 detach를 호출할 수 있는지의 여부를 돌려준다. |
| t.get_id(), std::this_thread::get_id() | 쓰레드 식별자를 돌려준다. |
| std::thread::hardware_concurrency() | 병렬로 실행할 수 있는 쓰레드 개수를 돌려준다. |
| std::this_thread::sleep_until(absTime) | 쓰레드 t를 absTime(절대 시간)이 될 때까지 재운다. |
| std::this_thread::sleep_for(relTime) | 쓰레드 t를 지금부터 relTime 시간이 흐를 때까지 재운다. |
| std::this_thread::yield() | 운영체제에게 실행권을 양보한다(다른 쓰레드를 실행 할 수 있도록).  |
| t.swap(t2), std::swap(t1, t2) | 두 쓰레드를 맞바꾼다. |

t.join()과 t.detach()는 하나의 t에 대해 딱 한 번만 호출할 수 있습니다. 이 메서드들을 여러 번 호출하면 std::system_error 예외가 발생합니다. 한편으로는 컴파일은 에러를 내놓지 않고 정상적으로 완료됩니다. 실행 시 프로그램이 자꾸 죽으니 디버깅 시 애를 먹이게 됩니다.

std::thread::hardware_concurrency()는 CPU 코어의 개수를 돌려줍니다. 단, 실행 시점 모듈이 코어 개수를 알아낼 수 없는 상황이면 0을 돌려줍니다. sleep_until과 sleep_for 연산은 시점(time point, chrono헤더에 정의) 또는 기간을 받습니다. 

한편 쓰레드는 복사할 수 없고 이동(std::move())만 가능합니다. swap함수는 이동이 가능한 대상에 대해서는 이동을 수행해줍니다.


## 공유 변수와 경쟁 조건(Race condition)

여기서부터 모든 것을 main()에 때려 박았던 습관이 발목을 잡기 시작합니다. 이전에는 하나의 함수가 배타적인 동작을 보장하는 로직을 작성했기 때문에 변수에 대해 생각할 필요가 없었습니다. 단지 함수 내부에 있는 지역 또는 전역 변수의 여부나 메모리 할당과 해제에 대해서만 고려하면 됩니다.

**그런데 하나의 변수를 둘 이상의 쓰레드가 공유한다면 문제가 발생하게 됩니다.**

![race_condition](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-17.jpg "「C++ Concurrency in Action」 P.35")

이중 연결 리스트를 생각해 봅시다. 한 노드에는 양 방향의 노드를 가리키고 있는 포인터가 있습니다. 만약 한 노드를 지우는 과정을 실행한다면 지워지는 노드의 양 쪽에 있는 노드의 포인터는 업데이트가 되어야 할 겁니다. 일련의 과정을 간단히 나타내면 아래와 같습니다.

1. Identify the node to delete (N).
2. Update the link from the node prior to N to point to the node after N.
3. Update the link from the node after N to point to the node prior to N.
4. Delete node N.

예를 들어 쓰레드 A가 한 node를 읽는 반면, 쓰레드 B는 하나의 node를 지운다고 생각해 봅시다. 지우는 과정 중인 그림의 b)나 c)의 과정에서 데이터를 읽는 다면 쓰레드 A는 원하는 데이터를 읽을 수도 없을 뿐더러 전체적인 구조를 망가뜨릴 수 있습니다. 이러한 경우와 같이 적어도 두 개의 쓰레드가 동시에 하나의 공유 자료에 접근하되 둘 중 적어도 하나가 기록자(write)인 상황을 경쟁 조건(race condition)이라고 합니다. 경쟁 조건이 발생하는 경우 프로그램의 행동은 정의되지가 않습니다.

그럼 이중 연결 리스트보다는 더 간단한 예제를 만들어 봅시다. 하나는 counter 변수를 증가시키는 놈이고 counter 변수를 감소시키는 놈입니다. 동일한 횟수로 1씩 증가/감소 시키는 과정입니다. 

```cpp
#elif(_WHAT_ == 4) // part 4 - Data Sharing and Race Conditions
	
		
	int counter = 0;
	
	std::thread worker_thread_1([&]() {
		APP_THREAD_START
		std::cout << " > An augmenting lambda running as a thread." << std::endl;
				
		for(int itor = 0; itor < 100; itor++) {
			
			std::this_thread::sleep_for(std::chrono::milliseconds(1));
			counter++;
		}
		
		APP_THREAD_TERMINATE
		
	});
	
	std::thread worker_thread_2([&]() {
		APP_THREAD_START
		std::cout << " > An decreasing lambda running as a thread." << std::endl;
				
		for(int itor = 0; itor < 100; itor++) {
			
			std::this_thread::sleep_for(std::chrono::milliseconds(1));
			counter--;
		}
		
		APP_THREAD_TERMINATE
		
	});
	
	
	if(worker_thread_1.joinable())
		worker_thread_1.join();
	
	if(worker_thread_2.joinable())
		worker_thread_2.join();

	std::cout << " > Racing condition result is " << counter << std::endl;
```

동일하게 100번씩 증가시키고 감소시킵니다. 그러면 우리가 기대하는 값은 0이 됩니다.

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-18.jpg)

결과는 그렇지 않습니다. 프로그램을 실행시킬 때마다 한번은 3, 한번은 -1을 출력하고 있습니다. 우리가 원하는 값은 아닙니다. 


## Mutex

뮤텍스(mutex)는 Mutual Exclusion(상호 배제)를 줄인 말입니다. 뮤텍스는 한 번에 한 쓰레드만 임계 영역(critical region)에 접근할 수 있게 하는 수단입니다. Mutex를 사용하려면 mutex 헤더를 포함시켜야 합니다.

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-19.jpg)

단순합니다. m이 하나의 std::mutex 객체라고 하면 m.lock()은 임계 영역을 잠그고 m.unlock()은 임계 영역을 풀어줍니다. 그러면 하나의 변수에 대해 다수의 쓰레드가 동시 접근하지 못하도록 만드는 게 가능합니다. 만일 lock()을 호출하고 unlock()을 호출하지 않았다면 교착 상태(deadlock)가 발생합니다. 

따라서 뮤택스는 자물쇠(lock)안에 캠슐화하여 뮤텍스가 자동으로 잠기고 풀리게 하는 것이 바람직합니다. 그러한 자물쇠 클래스는 RAII 관용구를 이용하여 뮤텍스의 수명을 자신의 수명에 묶습니다. std::lock_guard는 이를 지원합니다.

> RAII는 Resource Acquisition Is Initialization(자원 획득은 초기화)를 줄인 것으로 자원 획득과 해제를 객체의 수명에 연관시키는 C++ 기법입니다. 예를 들어 스마트 포인터가 RAII를 따른다는 것은 간단히 말해 메모리를 생성자에서 할당하고 소멸자에서 해제한다는 뜻입니다. C++에서는 scope를 벗어날 때 자동으로 소멸자가 호출되기 때문에 이러한 기법이 가능합니다. 

```cpp
#elif(_WHAT_ == 5) // part 5 - Using mutex to fix Race Conditions
		
	int counter = 0;
	std::mutex mutex;
	
	std::thread worker_thread_1([&]() {
		APP_THREAD_START
		std::cout << " > An augmenting lambda running as a thread." << std::endl;
				
		mutex.lock();
		
		for(int itor = 0; itor < 100; itor++) {
			
			
			
			std::this_thread::sleep_for(std::chrono::milliseconds(1));
			counter++;
		}
		
		mutex.unlock();
		
		APP_THREAD_TERMINATE
		
	});
	
	std::thread worker_thread_2([&]() {
		APP_THREAD_START
		std::cout << " > An decreasing lambda running as a thread." << std::endl;
		
		// std::lock_guard is a class template, which implements the RAII for mutex.
		// It wraps the mutex inside it’s object and locks the attached mutex in its constructor. 
		// When it’s destructor is called it releases the mutex.
		
		std::lock_guard<std::mutex> local_lock_guard(mutex);
		
		for(int itor = 0; itor < 100; itor++) {
			
			std::this_thread::sleep_for(std::chrono::milliseconds(1));
			counter--;
		}
		
		APP_THREAD_TERMINATE
		
	});
	
	
	if(worker_thread_1.joinable())
		worker_thread_1.join();
	
	if(worker_thread_2.joinable())
		worker_thread_2.join();

	std::cout << " > Racing condition result is " << counter << std::endl;
```

예제에서는 worker_thread_1은 lock(), unlock()으로, worker_thread_2는 lock_guard 객체를 이용해봤습니다. 두 가지 모두 동일하며 실행하면 우리가 원하는 결과를 보장하는 것을 확인할 수 있습니다. 

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-20.jpg)

std::lock_guard는 주어진 뮤텍스를 생성자에서 잠그고 소멸자에서 해제하는 역할만 합니다. 따라서 lock_guard를 생성하기만 하면 뮤텍스가 자동으로 잠기고 풀립니다. 

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-21.jpg)

추가적으로 std::unique_lock도 한번 확인합시다. std::unique_lock은 std::lock_guard보다 많은 기능을 제공하는, 따라서 좀 더 복잡한 용도로 사용할 수 있는 자물쇠입니다. std::lock_guard 와는 달리 std::unique_lock은 뮤텍스 없이도 생성할 수 있고, 뮤텍스를 명시적으로 잠그구 푸는 기능과 일정 시간 동안 뮤텍스 잠금을 시도하는 기능도 제공합니다. 
여기까지 아주 빠르게 표준 라이브러리 thread 관련 기본만을 다루었습니다. 활용해 봅시다.

‘y' 키가 입력받기 전까지 0.5초 마다 "I am listening!!" 이라는 문구를 출력하는 프로그램을 만들어 봅시다. 그러려면 *①키 입력을 기다리는 쓰레드 하나*, *②0.5초마다 키가 입력되었는지 검사하는 쓰레드 하나*, 두 개의 쓰레드를 만들어 두면 될 겁니다. 맞는 키가 입력되었는지의 여부는 bool형 keep_listener_running 변수에 저장해 두면 되고, 두 개의 쓰레드는 이 변수를 읽으며 자신이 계속 실행될지, 또는 종료할 지의 여부를 결정하도록 하면 될 겁니다. RTD 프로젝트에서 저에게 떨어진 과제도 이런 종류였으니 감회가 새롭네요.

key_event_listener 쓰레드 객체 하나와 listener_alert 쓰레드 객체 하나 선언해 둡니다. 아래와 같이 될겁니다.

```cpp
#elif(_WHAT_ == 6) // part 6 - Need of Event Handling
	
	bool keep_listener_running = true;
	
	// customed example
	std::thread key_event_listener(
		[&]() {
			
			APP_THREAD_START
				
			int key = 0;
			
			while(keep_listener_running) {
				
				std::cout << " > Listening for 'y' key input..." << std::endl;
				
				key = getch();
				
				if(key != 'y')
					continue;
				
				else {
					keep_listener_running = false;
					break;
				}
				
			}
			
			APP_THREAD_TERMINATE
		}
	);
	
	std::thread listener_alert(
		[&]() {
			
			APP_THREAD_START
				
			while(keep_listener_running) {
				
				std::this_thread::sleep_for(std::chrono::milliseconds(500));
				std::cout << " > I am listening!!" << std::endl;
				
			}
				
			APP_THREAD_TERMINATE
		}
	);
	
	
	if(key_event_listener.joinable())
		key_event_listener.join();
	
	if(listener_alert.joinable())
		listener_alert.join();
```

실행하면 0.5초마다 키를 입력받으라고 징징대는 것을 확인할 수 있습니다.

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-22.jpg)

’y‘를 입력해줍시다. 그러면 쓰레드 두 개가 모두 종료됩니다. 만일 다른 키를 입력하면 그 키가 아니라고 다시 입력하라고 징징거립니다.

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-23.jpg)

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-24.jpg)


하지만 이건 한 쓰레드가 지속적으로 0.5초마다 bool형 플래그를 검사하는 과정이 필요합니다. 만일 한 쓰레드가 다른 쓰레드에게 받았다는 신호를 보내면 그때 쓰레드가 반응하도록 하는 것이 더 좋을 겁니다.


## 조건 변수

조건 변수(condition variable)을 이용하면 메시지를 통해서 쓰레드들을 동기화 할 수 있습니다. C++에서 이 기능을 사용하려면 <condition_variable> 헤더를 include해 줍니다. 조건 변수를 이용한 동기화에서는 한 쓰레드는 메시지 송신자(sender) 역할을 하고 다른 한 쓰레드는 메시지 수신자(receiver)역할을 할 겁니다. 수신자는 송신자의 통지를 기다립니다. 조건 변수의 전형적인 용도는 생산자-소비자(product-consumer) 작업흐름을 구현하는 것입니다.

하나의 조건 변수는 메시지 송신과 수신 모두에 사용할 수 있습니다. 

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-25.jpg)


| 메서드 | 설명 |
| ---
| cv.notify_one() | 대기 중인 한 쓰레드에 통지한다.  |
| cv.notify_all() | 대기 중인 모든 쓰레드에 통지한다. |
| cv.wait(lock, ...) | 자물쇠(std::unique_lock)를 잠근 상태에서 통지를 기다린다. |
| cv.wait_for(lock, relTime, ...) | 자물쇠(std::unique_lock)를 잠근 상태에서 일정 기간이 흐를 때까지 통지를 기다린다. |
| cv.wait_until(lock, absTime, ...) | 자물쇠(std::unique_lock)를 잠근 상태에서 특정 시각이 될 때까지 통지를 기다린다. |

레퍼런스는 너무 친절합니다. 사용법을 봅시다. 

![reference](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-26.jpg)

'intends to modify the variable'이라 함은 메시지 송신자를 의미하는 것이고, ’intends to wait on std::condition_variable’은 메시지 통지를 기다리는 쓰레드를 의미할겁니다. ①송신자는 그냥 자물쇠를 한 번만 잠그고 풀면 되므로 std::lock_guard로 충분합니다. 레퍼런스에도 전형적으로 std::lock_guard를 사용한다고 나와 있습니다. lock을 획득한 상태에서 일정한 로직을 수행한 후 notify_all() 또는 notify_one()을 통해 wait()하고 있는 수신자에게 통보합니다. 

②수신자는 뮤텍스를 여러 번 잠그고 풀어야 하는 경우가 많으므로 std::unique_lock이 필요합니다. 레퍼런스에도 maximal efficiency를 위해 std::unique_lock<std::mutex>에서만 동작한다고 명시되어 있습니다. 

그러면 예제를 작성해 봅시다.

#1 부터 #10까지의 쓰레드는 키 입력을 기다립니다. 키 입력을 기다리는 key_event_listener 쓰레드를 하나 생성하고, 어떤 쓰레드가 반응할지 대기하는 쓰레드를 10개 만들어둡니다. 만일 3을 입력했다면 #3 쓰레드가 반응하도록 만들면 됩니다. 

```cpp
#elif(_WHAT_ == 7)	// part 7 - Condition Variables 
	

	std::vector<std::thread> worker_threads; // list of threads

	std::mutex mutex;
	std::condition_variable cv;
	
	
	int wt_signal = 0;
	bool signal_received = false;
		
	std::thread key_event_listener(
		[&]() {
			
			APP_THREAD_START
			
			//while(1) {
				
				std::cout << " > Listening for id of a thread to run..." << std::endl;
				
				std::lock_guard<std::mutex> guard(mutex); // sender
				
				wt_signal = getch() - '0';
			
				signal_received = true;
				cv.notify_all();			
				
			//}
			
			APP_THREAD_TERMINATE
		}
	);
	
	
	
	for(int i = 0; i < 10; i++) {
		worker_threads.push_back(
			std::thread(
				[&](int arg_div) { // catch by copying.
					APP_THREAD_START
					
					const int self_signal_id = arg_div + 1;
						
					//while(1) {
						
						std::unique_lock<std::mutex> lock(mutex);
						// m_condVar.wait(mlock, std::bind(&Application::isDataLoaded, this));
						cv.wait(lock, [&]() { return signal_received; });
						
						if(wt_signal == self_signal_id) {
		
							std::cout << " > An worker thread #" << arg_div + 1 << " catched signal." << std::endl;
						}						
						
						lock.unlock();
					//}					
					
					APP_THREAD_TERMINATE
				}
			, i)
		);
	}
	
	for(auto& itor : worker_threads)
		if(itor.joinable())
			itor.join();
	
	
	if(key_event_listener.joinable())
		key_event_listener.join();	
```

한번 돌려 봅시다. 쓰레드가 시작되고 모두 대기하고 있습니다.

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-27.jpg)

7을 누르면 #7 쓰레드가 반응하고 3을 누르면 #3 쓰레드가 반응합니다. 

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-28.jpg)

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-29.jpg)

만일 다른 키를 누르면 아무런 반응 없이 모든 쓰레드가 종료될겁니다. 

![bash](/assets/posts/2019-03-20-concurrency-introduction-part-1/2019-03-20-30.jpg)

Part 2에서는 원자적 연산을 보장하는 atomic 헤더와 쓰레드가 리턴하는 값을 이용하기 위한 std::future, std::promise, std::async를 보겠습니다. 


> 필자는 공군 작전정보통신단 체계개발실에서 복무('17~'19)하였습니다. 이 포스트는 작전정보통신단 병사 **프로그래밍 동아리(LINK)** 에서의 활동을 바탕으로 작성한 내용입니다.