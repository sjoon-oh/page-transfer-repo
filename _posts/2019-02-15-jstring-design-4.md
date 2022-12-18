---
title:  "JString Design[4]"
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

# [_jcode mini-project] JString Design(4) (작성중)

## JString(4) 본격적인 operator overloading

여기까지 완벽하지는 않지만 그럭저럭 작동 가능한 문자열 클래스 디자인을 끝냈습니다. 그러나 조금더 욕심을 내 보고 싶습니다. 목표는 `std::string` 스러운 `_jcode::JString` 이니까요. 조금 더 사용하기 편하게, 그리고 조금 더 C++ 스럽게 짜 보겠습니다. 

<!--more-->

앞에서 말했듯이, 내가 작성한 클래스가 완전한 타입이 되려면 조금 더 남았습니다. 원시적인 타입처럼 덧셈, 뺄셈, 곱셈 등의 기본적인 연산이나 비트연산 이라던지, 혹은 변수에 대한 대입까지도 가능해야 완전한 하나의 타입처럼 쓸 수 있을 겁니다. Java, 자바스크립트 등의 언어들은 이러한 객체간의 연산을 잘 지원하지 않습니다. 자바스크립트처럼 언어적으로 일부를 지원해도 변수가 특정한 형(type)을 나타내지 않는다는 것일 뿐, 내가 의도한 바와 다르게 동작하면 의미가 없습니다. (Python은 예외로 하겠습니다. 연산자 정의가 가능한 것으로 알고 있습니다.)

위에서 인터페이스를 만들 때 일부나마 설명했지만 언어적 문법을 잠깐 확인하자면 아래와 같이 함수 형태로 정의합니다. 연산자에 대한 키워드는 operator를 쓰며 함수형태로 다음과 같이 선언합니다. 형태는 동일하게 `[반환값]operator[overload 할 연산자] (인자)`입니다.

```cpp
const JString& operator =(const JString&) noexcept;
const JString& operator =(const char*) noexcept;
```

위에서 정의한 대입 연산자로 예를 들겠습니다. 

연산자를 재정의 할 때에는 기본 연산이 어떻게 이루어지는지 알고 디자인하는 것이 좋습니다. 예를 들어 대입 연산자의 경우에는 

```cpp
SomeThing = “3”;
```

단순히 이렇게도 사용하지만 아래와 같이 사용하기도 합니다. 

```cpp
SomeOtherThing = SomeThing = “3”;
```

그러면 고려해야 할 점이 생깁니다. (1)우선 C++에서는 리턴값으로 오버라이딩 함수를 구별하지 않습니다. 무슨 의미인가 하면, 리턴값만 다르다고 하여 동일한 인자를 인수로 가지는 여러개의 함수를 재정의 할 수 없다는 이야기입니다. 연산자 오버로딩 또한 함수 형태로 정의하기 때문에 반환값만 다르게 하여 재정의 할 수 없습니다. 아래와 같이 말입니다. 

```cpp
void operator =(const JString) noexcept;
JString operator =(const JString) noexcept;
```

위의 구문은 컴파일 에러를 발생시킵니다. 그러면 우리는 어느 부분을 신경써야 할까요? 위의 예제에서는 `SomeThing`이라는 변수에 3을 우선적으로 대입하고, `SomeOtherThing`이라는 변수에 `SomeThing`의 값을 대입합니다. 위의 경우를 나누어서 생각하면 

```cpp
SomeThing = “3”; // SomeThing.operator=(“3”);
SomeOtherThing = SomeThing; // SomeOtherThing.operator=(SomeThing);
```

이 정상작동 해야하니 `operator =()` 함수는 두 가지로 오버로딩 되어야 한다는 것을 알 수 있습니다. 그렇다면 인자는 `SomeObject` 타입을 하나 받는 놈, `int`형 타입을 하나 받는 놈이 존재해야 할겁니다. 여기까지는 쉽습니다. 

그렇다면 리턴 타입은 어떻게 해야 될까요? 결론적으로만 말하면 `SomeObject`형을 리턴하는 것이 좋습니다. 결과가 필요 없으면 버리면 됩니다. 그러면 

```cpp
JString operator =(const char*) noexcept;
JString operator =(const JString) noexcept;
```

과 같이 선언되는 겁니다. 

여기서 끝은 아닙니다. (2)리턴 타입을 어떤 것으로 선언할지에 대한 세부적 고찰이 필요합니다. 우리의 목적은 연속으로 나타나는 연산자에 대응할 수 있도록 디자인을 하되, 함수를 인자로 구분하므로 언어적 문법 허용 범위 안에서 작성해야 합니다. 다시 한번 봅시다. 궁극적으로는 

```cpp
SomeOtherThing = SomeThing = 3;
```

을 하고 싶으니, 이를 다른 방식으로 작성하면,

```cpp
SomeOtherThing.operator=( SomeThing.operator=(“3”));
```

이렇게 됩니다. 

어떠한 객체를 반환해야 하는 것 까지는 알겠습니다. 그러면 단순히 `JString`형을 복사하여 넘기는 것이 나을까요? 즉, 함수의 형태가 

```cpp
JString operator =(JString) noexcept;
```

이는 좋은 방법이 아닙니다. JString 형이 인자로 들어갈　때 복사 생성자를 호출할 테니 크기가 커지면 오버헤드가 걸립니다. 그러면 레퍼런스로 받는 것이 좋을 것 같습니다. 

```cpp
JString operator =(JString&) noexcept;
```

끝난 것이 아닙니다. 현재 인자로 들어가는 형태가 레퍼런스이니 `SomeThing = 3;` 의 리턴값도 레퍼런스로 맞추어 주어야 `SomeOtherThing( SomeThing.operator=(3));` 의 인자도 레퍼런스로 들어갈 겁니다. 그러면 아래와 같이 됩니다. 여기까지는 좋습니다.

```cpp
JString& operator =(JString&) noexcept;
```

추가적으로 생각해 봅시다. 대입 연산자는 멤버를 수정하지 않습니다. 자기 자신만 수정할 뿐 피연산자를 수정하는 일도 없을뿐더러 대입 시에 수정하면 오작동을 일으킵니다. 따라서 `const` 지정자를 앞에 붙여주어 혹시라도 모를 수정에 대비합니다. 그러면, 궁극적으로

```cpp
const JString& operator =(const JString&) noexcept;
```

(3)마지막으로 우리는 극한의 상황에 대비해야 합니다. 대입 연산자 이므로 다음과 같은 상황에서도 정상 동작을 보장해야 합니다. 예를 들어

```cpp
SomeThing = SomeThing;
```

자기 자신을 대입하는 경우입니다. 만일 함수 내에서 리턴값을 함수 내의 임시 변수를 넘겨주게 되면 오작동을 일으킵니다. 리턴하는 것은 레퍼런스, 즉 존재하는 어떠한 변수에 대한 이름인데 함수 내에서 곧 사라질 임시 변수에 대한 레퍼런스라뇨. 따라서 자기 자신을 넘기는 게 맞습니다. `operator=()` 함수 내에서 다음과 같이 리턴 하라고 지시하면 되겠죠.

```cpp
	return *this;
} // end of function
```
연산자는 대입 연산자만 존재하는 것이 아닙니다. 심지어는 `new`, `delete`와 같은 할당 연산자도 오버로딩이 가능합니다. C++에서 언어적으로 지원하는 계산 방식을 내가 스스로 클래스에 맞게 지원하는 것이므로 클래스에 따라 어느 범위에서 작동시킬건지 확실히 보장해주어야 합니다. 다른 개발자들이 직관적으로 

```cpp
SomeThing = OtherThing + ThisThing * AnythingElse;
```

과 같은 코드를 작성하는데 연산자 자체에 대한 제한이 있으면 안되기 때문입니다. `operator+()`는 인자로 `int`형이 와도 되고, `double`형이 와도 작동하고, 연산자를 겹쳐 쓰는 등 모든 경우의 수를 열어두어야 합니다. 

## ① operator ==()

흔히 == 연산자는 비교하는데 이용합니다. 마찬가지로 문자열을 비교할 때 

```cpp
if(Sentence == lyrics) ;
if(Sentence == "Yeah") ;
```

와 같이 사용하도록 하고 싶습니다. 그러면 아래와 같이 우선적으로 하나만 작성합니다. 

```cpp
// ==
const bool _jcode::JString::operator ==(const JString& argJStr) const noexcept {

	if (preventAccess()) return false;

	for (int Indx = 0; Indx != Length; Indx++)

		if (at(Indx) != argJStr[Indx])
			return false;

	return true;
};
```

같은지 아닌지에 대한 판단이므로 반환값은 `const bool` 타입이 가장 이상적입니다. 또한 자기 자신과 같은 타입에 대한 비교가 가장 기본적이기 때문에 인자는 `const JString&` 형으로 맞춰줍니다. 

동작은 어렵지 않습니다. 완전탐색으로 배열의 동일한 위치에 동일한 값이 들어가 있는지 체크한 후 값을 리턴합니다. 내부 멤버를 수정하지 않으므로 `const` 함수로 둡니다. 그러면 

```cpp
if(Sentence == lyrics) ;
```

과 같은 코드는 정상 동작 합니다.

두 번째로 인자가 `const char*(문자열)`일 경우를 생각해봅시다. 작동은 완전히 동일하니 방금 작성해 둔 o`perator==()`를 참고하면

```cpp
const bool _jcode::JString::operator ==(const char* argStr) const noexcept {

	return this->operator==(JString(argStr));
};
```

과 같습니다. 테스트 해 봅시다. 

```cpp
PRINT_NORMAL_MSG(std::to_string(
		life_is_a_highway_lyrics_1 == life_is_a_highway_lyrics_1
	));

PRINT_NORMAL_MSG(std::to_string(
		life_is_a_highway_lyrics_1 == life_is_a_highway_lyrics_2
	));
	
PRINT_NORMAL_MSG(std::to_string(
		life_is_a_highway_lyrics_1 == "Yeah"
	));
```

```
이미지
```

정상작동 합니다. 

## ② operator !=()

다른지 판단하는 연산자입니다. 이 연산자는 `==`의 결과와 완전히 반대되는 값을 내놓습니다. 그렇다면 따로 무엇인가 작성할 필요 없이 아래와 같이 쓰면 되겠네요?

```cpp
// !=
const bool _jcode::JString::operator !=(const JString& argJStr) const noexcept {

	return !(this->operator ==(argJStr));
};


const bool _jcode::JString::operator !=(const char* argStr) const noexcept {

	return !(this->operator ==(argStr));
};
```

단순한 반전입니다. 테스트는 건너뛰겠습니다. 

## ③ operator =()
대입 연산자입니다. 이는 윗 장에서 자세히 훑었으므로 건너뛰겠습니다. 

## ③ operator +()
연산자는 클래스 특성에 따라 디자인의 방향이 달라집니다. 덧셈 연산자를 보겠습니다. 문자열의 덧셈은 단순하게 

```cpp
JString SomeString("I ");
JString Operand("want ");

SomeString + Operand; // JString type + JString type
SomeString + "you"; // JString type + const char*
```

이렇게 쓰고 싶습니다. 

그러면 인자로 `JString`형과 `const char*` 형이 들어가도록 파라미터를 구성해야됩니다. 리턴 값은 어떻게 구성하는 것이 좋을까요? 리턴값은 구현에 따라 다르게 구성해야할 듯 싶습니다. 일단 구현해봅시다.

```cpp
// user-defined ops
const _jcode::JString _jcode::JString::operator +(const JString& argOperand) noexcept {
	PRINT_FUNCTION_CALL("JString::operator +(const JString&)");

	if (preventAccess()) 
		return JString(argOperand);

	// Reallocation
	JString tbuf(*this);
	tbuf.insert(argOperand, tbuf.sizelength());

	return tbuf;
};


const _jcode::JString _jcode::JString::operator +(const char* argOpStr) noexcept {

	return this->operator+(JString(argOpStr));
};

```

덧셈으로 들어가는 파라미터는 레퍼런스 값이 좋을 듯 싶습니다. 다음과 같이 적어보면 조금 더 직관적입니다.

```cpp
SomeString + Operand; // JString type + JString type
SomeString.operator+(Operand); // JString type + JString type
```

어차피 인자로 들어가는 놈은 변하지 않습니다. 덧셈 하기 위해 전달되는 값은 변하면 안되므로 const 지정자를 붙여줍니다. 리턴값은 JString이 좋을 것 같습니다. 예를 들어

```cpp
SomeString + Operand_1 + Operand_2;
```

라면 

```cpp
SomeString + Operand_1.operator+(Operand_2);
```

이므로 SomeString.operator()의 인자로 const JString&이 들어가야 합니다. 그러나 구현부를 확인하면 임시로 JString형 tbuf라는 변수를 만들고 그 객체를 리턴하는 것을 볼 수 있습니다. 만일 리턴값이 레퍼런스로 지정되어 있다면 함수가 종료되면서 곧 사라질 tbuf에 대한 레퍼런스를 리턴하는 것과 같습니다. 따라서 여기에서는 값을 복사하여 넘기는 것이 현명해 보입니다. (리턴값에const 지정자를 붙여도 상관 없습니다.)

구현은 어렵지 않습니다. 덧셈 연산자는 단순히 뒤에 갖다 붙이는 역할을 하므로 애써 만들어둔 insert() 함수를 안써먹을 이유가 없습니다. 따라서 내부에 임시변수인 tbuf를 하나 만들어두고 *this를 인자로 복사합니다. 그리고 제일 끝에 삽입하면 됩니다. 

```cpp
SomeString + Operand_1 + “Something”;
```

과 같은 연산도 어렵지 않습니다. 어차피 문자열이므로 JString형으로 감싸고 위에 만들어둔 operator+()를 호출합니다. 537번 줄에서 확인할 수 있습니다.

테스트는 아래와 같습니다. 

```cpp
jc::JString Test_Phase2;

Test_Phase2 = life_is_a_highway_lyrics_1 + life_is_a_highway_lyrics_2;
PRINT_NORMAL_MSG(Test_Phase2);
```

```
이미지
```

## ④ operator +=()

다른 익숙한 연산자입니다. 위에서 operator +()를 구현해 두었으므로 아주 간단히 아래와 같이 작성 가능합니다. 바로 들어가보겠습니다. 다른 점이라면 리턴값이 없습니다. 대부분

```cpp
SomeThing += OtherThing;
```

과 같이 사용하지 아래와 같이

```cpp
OneThing = SomeThing += OtherThing;
```

과 같이 사용하지는 않으니까요. 구현해봅시다.

```cpp
void _jcode::JString::operator +=(const char* argStr) noexcept {

	*this = *this + argStr;
};


void _jcode::JString::operator +=(const JString& argJStr) noexcept {

	*this = *this + argJStr;
};

```

아주 단순합니다. 자기 자신과 피연산자를 operator+()로 묶고 있습니다. 테스트 해보면,


```cpp
jc::JString Test_Phase2 = life_is_a_highway_lyrics_1;

Test_Phase2 += life_is_a_highway_lyrics_3;
PRINT_NORMAL_MSG(Test_Phase2);
```

```
이미지
```

## ⑤ operator <<(), operator >>()

아주 좋습니다. 조금씩 끝이 보입니다. `std::string`에서 이러한 shift 연산자를 지원하는지는 확인하지 못했습니다. 재미로 정의해본 연산자입니다. `>>`의 경우 피연산자 앞에 호출한 주체 인스턴스의 값을 붙이고, `<<`의 경우 호출 인스턴스 뒤에 값을 붙이는(`operator +=()`와 동일) 것으로 만들어 봅시다. 

`<<`은 `+=`와 동작이 완전히 동일하므로 단순 호출하면 될 것이고, `>>`의 경우는 피연산자의 앞에 붙이는 것이므로 제일 앞의 인덱스인 0에 `insert` 해주면 됩니다. 표현은 단순합니다.

```cpp
// <<
void _jcode::JString::operator <<(const JString& argJStr) noexcept  {

	this->operator+=(argJStr);
}


void _jcode::JString::operator <<(const char* argStr) noexcept  {

	this->operator+=(argStr);
};


// >>
void _jcode::JString::operator >>(JString& argJStr) noexcept  {

	//
	if (preventAccess()) {
		return;

	} else {
		if (argJStr.preventAccess()) {

			argJStr.insert(*this, 0);

		} else
			argJStr.insert(*this, 0);
	}
}
```

```cpp
jc::JString Test_Phase2 = life_is_a_highway_lyrics_1;

Test_Phase2 >> life_is_a_highway_lyrics_3;
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_3);

life_is_a_highway_lyrics_3 << Test_Phase2;
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_3);
```

```
image
```




> 필자는 공군 작전정보통신단 체계개발실에서 복무('17~'19)하였습니다. 이 포스트는 작전정보통신단 병사 **프로그래밍 동아리(LINK)** 에서의 활동을 바탕으로 작성한 내용입니다.

