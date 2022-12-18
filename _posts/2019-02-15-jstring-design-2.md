---
title:  "JString Design[2]"
date:   2019-02-15 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - C/C++
tags:
   - C/C++
   - std
toc: true
toc_sticky: true
---

# [_jcode mini-project] JString Design(2)

## JString(1) 문자열 관리는 어떻게?

본격적으로 문자열 디자인을 해 봅시다. 결론적으로 C 스타일의 문자열로 관리합시다. char형 배열입니다.

`std::string` 멤버로 관리할 수야 있겠다만 이건 너무 양아치입니다. 애초에 잘 만들어진 인터페이스가 있으니 디자인 측면에서는 할 게 없어집니다. 따라서 C 스타일의 문자열 멤버를 관리하는 형태로 가는 거로 합시다. 그리고 목표는 `std::string` 에서 지원하는 동작들을 대부분 지원하는 방향으로 하겠습니다. 

<!--more-->

그러면 여러 가지가 필요합니다. `std::string` 레퍼런스를 보면, 

```
(이미지)
```
![reference](/assets/posts/2019-02-15-jstring-design-2/2019-02-15-00.jpg)

지원하는 메서드만 봐도 이만큼이 있습니다. 이름은 같은 것을 쓰는 것으로 하고 중요한 메서드만 지원하는거로 하겠습니다. 물론 동작은 같게 만들어봅시다. 우선 `clear`, `insert`, `erase`, `push_back`, `pop_back`, `compare`, `replace` 정도로만 정합시다.

그러면 선언부터 해 보면,

```cpp

```

악! 제가 작성했지만 지금 보니 너무 많습니다. 보기 쉽게 클래스 다이어그램으로 보면,

![reference](/assets/posts/2019-02-15-jstring-design-2/2019-02-15-01.jpg)

100%는 아니지만 `std::string` 클래스를 따라하려고 노력했습니다. 여기서 확인할 것은 연산자 오버로딩 부분입니다.

또 한번 강조하지만 C++은 모든 것이 타입입니다. 하나의 타입을 구성하기 위해서는 객체 하나를 생성한다고 되는 것이 아닌, 인스턴스에 대한 연산도 필요합니다. 단적인 예로 `std::string`의 경우는, 

대입이나, 

```cpp
someString = "";
```

문자열 연결을 단순히 + 연산자로 이어 붙이는 경우, 

```cpp
someThing = TheString + existingThing“;
```

도 지원해주어야 합니다. 연산자 오버로딩은 조금 뒤에 보도록 하고, 메서드부터 하나씩 추가 하도록 합시다. 

## JString(2) 인스턴스 초기화, 쉽게 쉽게

`JString`은 다음과 같이 선언하고 싶습니다. 쉬운 사용을 위해 아래와 같이 쓰면 됩니다. 그러려면 생성자를 몇 개 디자인해야 겠네요. 세 번째 줄 부터 사용하려면 복사 생성자도 만들어야겠습니다.

```cpp
JString ExampleStr;
JString ExampleStr("Hello world!!");

JString ExampleStr2(ExampleStr);
JString ExampleStr2 = ExampleStr;
```

우선 멤버 선언부터 합시다. C 스타일 문자열 관리이니 `char*` 형 변수가 하나 필요할 테고, 문자열 길이에 따른 메모리 관리를 위해 크기를 나타내는 변수가 필요합니다. 간단간단하게 갑시다. 그려면, 

```cpp
class JString {
private:
	size_t Length;
	
	char* Buffer; // Practical storage use.
```

멤버는 저렇게 되니,

```cpp
public:
	// ctor
	explicit JString();
	explicit JString(const char*);
	JString(const JString&); // copy ctor
	JString(const JString&&); // move semantic
	
	// dtor
	virtual ~JString();
```

위와 같이 작성할겁니다. `explicit`은 뭐 ‘암시적으로 형변환을 하지 말거라’라는 뜻이고, 조잡해서 그럴 일은 없겠지만 혹시 모를 확장을 대비해 `destructor`는 `virtual`로 선언해 줍니다. 

생성자를 디자인할 때도 여러 가지 변수를 고려해야 합니다. 우선, ① 인자가 없을 때의 초기화는 어떻게 할 것인가? C 스타일의 문자열이면 `char`형의 배열 형식입니다. 포인터를 선언해 두었으니 그 자리에다가 동적 할당을 하겠다는 의미입니다. 따라서 우선 인자가 없이 선언하는 경우는 아래와 같이 `nullptr`을 먹여줍시다. 만일 Buffer 값이 `nullptr`(GND로 묶어놓는다면)라면 동적할당이 되어있지 않다는 이야기이니까요.

```cpp
// JString class
// Author : SukJoon Oh, KTMO-CELL S/W Support
// over C++11 please.

// constructor
_jcode::JString::JString() : Length(0), Buffer(nullptr) {
	PRINT_FUNCTION_CALL("JString::JString()");
}
```

그러면 

```cpp
JString ExampleStr;
```

과 같이 불러올 때면 멤버는 `{0, nullptr}`으로 초기화 되어 있을 겁니다. (이하에서는 멤버를 `{Length, Buffer}` 형식으로 표기합니다.)

좋습니다. 두 번째 경우를 생각해 봅시다. ② 인스턴스를 하나 생성함과 동시에 어떠한 문자열을 들고 있도록 하고 싶습니다. 이것도 크게 어렵지는 않습니다. 문자열을 하나 받아서 크기를 재고, 동일한 크기만큼 동적할당을 한 후 복사하여 들고 있도록 하면 됩니다. 구현은 아래와 같습니다. 

```cpp
_jcode::JString::JString(const char* argStr) {
	PRINT_FUNCTION_CALL("JString::JString(const char*)");
	// PRINT_LOG(argStr);
	
	Buffer = (Length = std::strlen(argStr)) != 0 ? (char*)malloc(sizeof(char) * (Length + 1)) : nullptr;

	if (Buffer != nullptr)
		std::memcpy(Buffer, argStr, (Length + 1) * sizeof(char));

	update();
};
```

202번째 줄에서 받은 문자열의 길이가 0보다 크다면 동적 할당을 해 주어야 됩니다. 여기서는 `std::strlen()` 함수를 빌려 씁시다. `std::strlen()`은 문자열의 길이를 반환하는 함수입니다. C 스타일 배열의 문자열 종료 ‘`/0`’값은 무시합니다. 우리는 그 종료값도 유의미한 값이므로 그 크기보다 1이 큰 배열을 만들고 Buffer에 동일한 크기만큼 값을 복사합니다. `std::memcpy()`를 씁시다. 

마지막에 `update()` 함수가 보입니다. 요 놈도 별 것 아닙니다. JString이 관리하는 문자열의 크기가 변동이 생기는 경우 `update()` 함수를 호출하여 `Length` 멤버를 업데이트 시킵니다. `private`으로 선언해 두고 구현은 아래와 같습니다. 

```cpp
void _jcode::JString::update() noexcept {
	
	Length = std::strlen(Buffer);
};
```

그러면

```cpp
JString ExampleStr("Hello world!!");
```

과 같이 이용할 수 있습니다. 원래라면 여기서 마치려고 했습니다만 조금 더 욕심이 생깁니다. 이렇게도 써 보고 싶습니다.

```cpp
JString ExampleStr;
ExampleStr = "Hello world!!";
```

사용이 훨씬 더 직관적이고 쉽습니다. 그러려면 대입 연산자를 정의해야합니다. 정의합시다. 

```cpp
const JString& operator =(const JString&) noexcept;
const JString& operator =(const char*) noexcept;
```

이제 조금 C++스러워져 갑니다. 이래야 C++ 답다고 할 수 있습니다. 우리가 볼 것은 174번 줄 놈입니다. 

```cpp
const JString& operator =(const char*) noexcept;
```

C++에서는 한 클래스에 대한 연산자도 정의할 수 있는데, 함수 형태로 정의합니다. 연산자에 대한 키워드는 `operator`를 쓰며 함수형태로 위와 같이 선언합니다. 형태는 동일하게 
> `[반환값]operator[overload 할 연산자][인자]`

입니다.

호출하는 경우는 아래 두 줄, 161번과 162번이 동일한 효과를 냅니다. 

```cpp
_jcode::JString TryWhat;

TryWhat = "Try this!!";
TryWhat.operator=("Try this!!");
```

앞의 객체가 멤버 `operator`를 부르는 놈이 되고 연산자 뒤에 오는 피연산자가 인자로 들어갑니다. 

```cpp
"Try" = TryWhat;
```

조금 이상하지만 이런식으로 부르면 호출이 되지 않을 것입니다. 이런 경우는 전역 함수로 인자를 두 개 설정해 주어야 합니다. 아래와 같이요. 

```cpp
const JString& operator =(const char*, const JString&) noexcept;
```

이번 프로젝트에서는 이러한 전역 함수는 일단 고려하지 않는 거로 합시다. 다시 돌아가서 세 번째 경우를 생각해 보도록 합니다.

③ 자신과 동일한 타입인 JString을 인자로 받아 초기화 하는 경우가 있을겁니다. 그러면 간단합니다. JString형을 인자로 받으면 됩니다. 그런데 일반적인 JString형을 인자로 넣을 수는 없습니다. 그렇게 되면 인자를 가져갈 때 복사를 하기 때문에 부담이 됩니다. 따라서 레퍼런스(reference)로 가져갈 겁니다. 이러한 특별한 생성자를 복사 생성자라고 부릅니다.

```cpp
_jcode::JString::JString(const JString& argJStr) {
	PRINT_FUNCTION_CALL("JString::JString(const JString& argJStr)");

	if (argJStr.Buffer != nullptr) {
		
		Length = argJStr.Length;

		this->Buffer = (char*)malloc(sizeof(char) * (Length + 1)); // first allocate,
		std::memcpy(Buffer, argJStr.Buffer, (Length + 1) * sizeof(char)); // copy that.

	} else
		this->Buffer = nullptr;
};
```

내용은 동일합니다. 어차피 동일한 타입이므로 안에 Buffer 멤버가 있을 것이고, 그렇다면 Buffer가 동적할당 되어 있는 상태, 즉 문자열을 인자로 받는 생성자를 호출한 상태면 그 안의 내용물의 크기만큼 동적할당 후 복사해버리면 됩니다. 만일 기본 생성자 (default constructor)를 호출하여 동적 할당이 되어있지 않다면 복사할 내용물이 없으므로 `nullptr`로 묶어 버립니다. 

여기까지 되었다면 다음과 같은 코드가 성공적으로 작동할 겁니다. 

```cpp
JString ExampleStr;
  JString ExampleStr("Hello world!!");
  JString ExampleStr2(ExampleStr);
  JString ExampleStr2 = ExampleStr; // Copy constructor called.
```

파괴자(destructor)는 더욱 더 간단할 겁니다. 동적할당 되어 있다면 해제하면 되고, 그게 아니라면 그냥 내버려 두면 됩니다. 구현은 아래와 같습니다.

```cpp
// destructor
_jcode::JString::~JString() {
	PRINT_FUNCTION_CALL("JString::~JString()");

	if(Buffer != nullptr)
		free(Buffer); // Memory free!!

	else
		PRINT_NORMAL_MSG("Not allocated.");
};
```


> 필자는 공군 작전정보통신단 체계개발실에서 복무('17~'19)하였습니다. 이 포스트는 작전정보통신단 병사 **프로그래밍 동아리(LINK)** 에서의 활동을 바탕으로 작성한 내용입니다.

[다음 포스트: JString Design[3]](https://sjoon-oh.github.io/archivers/Jstring_Design_3)