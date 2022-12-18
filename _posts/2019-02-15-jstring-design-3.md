---
title:  "JString Design[3]"
date:   2019-02-15 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
   show_overlay_excerpt: false
categories: 
   - C/C++
tags:
   - C/C++
   - std
toc: true
toc_sticky: true
breadcrumbs: true
---

# [_jcode mini-project] JString Design(3) (작성중)

## JString(3) 인터페이스를 만들자

이제 본격적으로 JString 클래스의 동작을 정의해 줄 때입니다. 우선 at() 부터 만들기로 합시다.

## ① at()

함수를 만들어 봅시다. 레퍼런스를 보면, 

![reference](/assets/posts/2019-02-15-jstring-design-3/2019-02-15-00.jpg)

<!--more-->

> "accesses the specified character with bounds checking" 

이라고 쓰여 있습니다. 특정 문자를 범위를 검사하면서 리턴하는 함수입니다. 어차피 Buffer는 C 스타일 배열이니 단순히 리턴만 해 주면 됩니다. 구현은 쉽습니다. 

```cpp
// interfaces
const char _jcode::JString::at(const int& argIdx) const noexcept {

	if (preventAccess())
		return -0x01;

	if (Length - 1 >= argIdx && argIdx >= 0) {
		
		return Buffer[argIdx];

	} else
		return -0x01; // Regarded as an index error.
};
```

접근이 가능한지, 즉 동적할당이 되어 있는 상태인지 검사하는 함수는 preventAccess()입니다. 이 또한 간단하게 아래와 같이 구현해 두면 여러 군데에서 써먹을 수 있습니다. 

```cpp
// inner
const bool _jcode::JStringItor::preventAccess() const noexcept {

	// nullptr of LocationAddr means that
		// this instance does not know the existance of the JString instance.
	return LocationAddr == nullptr ? true : false;
};
```

테스트를 해 봅시다. 아래는 제가 좋아하는 노래인 Rascal Flatts의 'Life is a highway'라는 곡의 가사 일부입니다. 이 JString 문자열들은 아래에서의 테스트에서 계속 사용하겠습니다. 

```cpp
namespace jc = _jcode;

// This is my favorite song.
jc::JString life_is_a_highway_lyrics_1("#1: Life's like a road that you travel on,");
jc::JString life_is_a_highway_lyrics_2("#2 : When there's one day here, and the next day gone.");
jc::JString life_is_a_highway_lyrics_3("#3: We won't hesitate, break down the garden gate");
jc::JString life_is_a_highway_lyrics_4("#4: there's not much time left today~");
// These guys calls their constructors, JString(const char*).
```

아래 코드를 돌려보면, 

```cpp
// interface testing...
	// at
	
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_2.at(1));
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_2.at(2);
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_2.at(3);
```

![console](/assets/posts/2019-02-15-jstring-design-3/2019-02-15-01.jpg)

짜자자잔, 정상 작동 확인 완료했습니다. 

## ② operator[] ()

  STL의 vector에서는 멤버에 접근할 때 at() 뿐만 아니라 vector_thing[45]처럼 사용하기도 합니다. JString도 이렇게 사용하고 싶습니다. 예를 들어, 

```cpp
JString ExampleStr("Hello world!!");
std::cout << ExampleStr[3]; // ‘l’ 이 프린트 되겠네요
```

그러면 아까와 같이 연산자 오버로딩을 해 봅시다. 

```cpp
const char _jcode::JString::operator[](const int& argIdx) const noexcept {

	return this->at(argIdx);
}
```

보통 [] 연산자 안에는 int형 정수가 들어갑니다. 디자인하기 나름이지만 만일 something["this?"]와 같이 map의 key처럼 작동하는 연산자를 만들고 싶다면 인자가 std::string이나 const char*가 들어가도 됩니다. JString은 인덱스로 받을 것이기 때문에 const int형을 인자로 받습니다.

여기서 참고할 점은, 내부에 아주 잘 만들어진 인터페이스나 동작이 이미 있다면 갖다 쓰면 됩니다. 다른 클래스를 디자인할 때도 마찬가지이지만 동일한 동작인데 같은 코드를 적어 넣을 필요가 없습니다. operator[]의 경우 at()과 같은 동작을 하므로 단순히 at(argIdx)를 리턴시키면 됩니다. 

추가적으로, C++ 개발자는 const 키워드를 사랑해야 합니다. const는 남발하면 남발할수록 더 안전한 코드를 작성하는 틀을 만들어주기 때문입니다. const를 더 썼다고 하여 작동 방식에 크게 영항을 주지 않으며 오히려 컴파일 타임에 개발자가 실수한 부분을 잡아주기도 합니다.

위의 operator[] 함수를 예로 들어 디자인 방향을 봅시다. 우선 (1) operator[] 함수는 내부 멤버를 수정하지 않습니다. 따라서 함수 정의에 const를 붙여 멤버 값 수정을 막아야 합니다. (2) operator[] 함수에 쓰이는 인자는 절대 수정될 일이 없습니다. 어떤 인덱스 값을 읽겠다고 했는데 함수 내부에서 파라미터가 수정되어 버리면 다른 인덱스를 읽어 버립니다. 마지막으로 (3) operator[] 함수의 반환값은 어디까지나 그냥 값입니다. 그냥 값이라 함은 수정될 일도 없을 뿐더러 호출부에서 필요가 없다면 임시값으로 처리하여 버려버릴겁니다. operator[] 함수 입장에서는 어차피 넘겨준 값을 어떻게 쓰든 알 바 아닙니다. 따라서 리턴값도 const로 처리하면 좋습니다. 

테스트 해 볼까요? 잘 되는군요!!

![console](/assets/posts/2019-02-15-jstring-design-3/2019-02-15-02.jpg)

## ③ front(), back()

간단합니다. 가장 앞의 문자를 반환하고 가장 뒤에 있는 문자를 반환하는 함수입니다. 

```cpp
const char _jcode::JString::front() const noexcept {

	if (preventAccess())
		return -0x01;
	
	else
		return Buffer[0];
};


const char _jcode::JString::back() const noexcept {

	if (preventAccess())
		return -0x01;

	else
		return Buffer[Length - 1];
};

```

vector 내부의 함수라면 설정한 템플릿 인자의 레퍼런스를 리턴 하지만 문자열은 꼴랑 1 byte 밖에 안되는 문자 하나를 반환하기 때문에 레퍼런스 까지는 필요 없습니다. 따라서 일반 값인 const char를 넘겨주는 것으로 합니다. 

```cpp
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_1.front());
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_1.back());
```

테스트 해 봅시다. 잘 나옵니다. 참고로 맨 앞 글자는 공백입니다.

![console](/assets/posts/2019-02-15-jstring-design-3/2019-02-15-03.jpg)

## ③ data(), c_str()

data() 함수를 봅시다. std::string 레퍼런스를 보면 "returns a pointer to ther first character of a string"이라고 쓰여 있습니다. 가장 첫 글자의 포인터를 돌려준다는 이야기입니다. c_str()도 비슷합니다. const char* 형태를 리턴하지만 C 스타일의 배열 형태로 돌려준다고 쓰여 있습니다. 

```
(이미지)
```

중요한 이야기는 아니지만 알고 넘어가도록 합시다. C++11 이전에는 data()가 null 문자를 자동으로 추가하지 않았습니다. C++11부터 data()와 c_str()은 사실상 같은 멤버 함수가 되었는데, 그렇다고 둘 중 하나가 폐기되지는 않았습니다. data()의 현재 서명은 const 문자_형식_포인터 data() const(상수 C 문자열을 돌려주는 상수 멤버 함수)인테, C++17에서는 또 다른 버전인 문자_형식_포인터 data()(수정 가능한 C 문자열을 돌려주는 비상수 멤버 함수)가 추가되었습니다. 

*(여기서 문자_형식 이라고 함은 문자에 char 형만 존재하는 것이 아닌 char16_t, char32_t 등 std::string, std::wstring과 같은 유니코드를 지원하는 클래스가 존재하기 때문에 표기한 방법입니다.)*

뭐가 어찌 됐든 data()나 c_str()나 안에 있는 내용물을 뱉어내는 함수라는 거지요. 따라서 어렵게 생각할 것 없이 그냥 C++11 이후처럼 동일한 동작을 하게끔 만들어 봅시다. 그러면 아래와 같이 만들어 주면 됩니다.

```cpp
const char* _jcode::JString::data() const noexcept {
	PRINT_FUNCTION_CALL("JString::data()");

	return Buffer == nullptr ? "" : Buffer;
};


const char* _jcode::JString::c_str() const noexcept {
	PRINT_FUNCTION_CALL("JString::c_str()");

	return this->data();
};
```

먼저 data()를 정성들여 만들어 줍니다. 동적 할당 되어있지 않다면 뱉어낼 문자열이 없습니다. 그렇다고 nullptr로 초기화 되어 있는 것을 던져주기는 좀 그러니 그냥 빈 문자열을 리턴하도록 했습니다. c_str()은 data()와 동작이 완전히 동일하니 정성들여 만들어 둔 data()를 그냥 호출해 주면 됩니다. 테스트 하기에는 너무 싱거운 코드지만 한번 해 봅시다. 

```cpp
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_1.data());

_jcode::JString Blank;
PRINT_NORMAL_MSG(Blank.c_str());
```
![console](/assets/posts/2019-02-15-jstring-design-3/2019-02-15-04.jpg)

잘 동작합니다!!

## ④ empty()

레퍼런스를 봅시다. std::string의 empty() 함수는 문자열을 가지고 있는지 확인하는 함수입니다. 그렇지만 JString에서는 조금 동작을 달리 하고 싶습니다. JString에서의 empty() 함수는 말 그대로 문자열을 비우는 동작을 하도록 고안하겠습니다. 단, 메모리 크기는 동일하게 유지하는 상태입니다. 개인적으로 is_empty()라는 이름이었다면 동일하게 동작을 구현했겠지만 empty()와 clear() 요 두 놈이 굉장히 헷갈립니다. 마음에 안 듭니다. 따라서 여기서는 말 그대로 empty 하도록 구현하겠습니다. 

처음에는 0으로 값을 채울까도 생각했으나 0도 어쨌든 값입니다. 따라서 쓰레기 값 일지언정 동일한 크기로 다시 메모리를 할당 받도록 합니다, 구현은 간단하니 아래와 같이 작성하면 될 듯 합니다. 

```cpp
const bool _jcode::JString::empty() noexcept {
	PRINT_FUNCTION_CALL("JString::empty()");

	if(preventAccess()) return false;
	
	free(Buffer);
	Buffer = (char*)malloc(sizeof(char) * Length);
	
	return true;
}
```

테스트가 기대됩니다. 어떤 쓰레기 값이 들어가려나...

```cpp
life_is_a_highway_lyrics_3.empty();
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_3);
```
![console](/assets/posts/2019-02-15-jstring-design-3/2019-02-15-05.jpg)

袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴袴羲羲硼??N 찾아보면, 

![console](/assets/posts/2019-02-15-jstring-design-3/2019-02-15-06.jpg)

> “사타구니사타구니사타구니사타구니사타구니사타구니사타구니사타구니사타구니사타구니사타구니사타구니사타구니숨결소리커”

그렇답니다.

## ⑤ sizelength() 

직관적인 이름입니다. 레퍼런스를 볼 것도 없이 문자열의 크기를 리턴하는 메서드 입니다. 애초에 멤버로 길이 정보를 가지고 있는 변수가 있었으니, 바로 들어갑시다. 

```cpp
const int _jcode::JString::sizelength() const noexcept {

	return this->Length;
}
```

테스트 할 것도 없어 보입니다. 

## ⑥ shrink_to_fit()
  
할당된 메모리 크기와 문자열의 크기가 상이할 때 그에 맞게 메모리를 줄여주는 역할을 하는 메서드입니다. 길이 정보는 항상 가지고 있으니 Length 멤버를 가지고 메모리를 줄여주면 됩니다. 메모리 할당이 안 되어 있다면 아무 짓도 안하면 되는 겁니다. 간단한 함수입니다. 메모리 재지정은 realloc() 함수가 담당합니다. 

애초에 다른 함수들이 호출부로 되돌아 가기 직전 update()를 이용하거나 메모리 관리를 하기 때문에 shrink_to_fit()을 사용하는 경우가 있을지 의문이지만 어쨌든 구현은 해 보았습니다. 이 함수 또한 간단하므로 테스트는 스킵하겠습니다. 

```cpp
const bool _jcode::JString::shrink_to_fit() noexcept {
	PRINT_FUNCTION_CALL("JString::shrink_to_fit()");
	
	if(preventAccess()) return false;
	
	Buffer = (char*)realloc(Buffer, (Length + 1) * sizeof(char));

	return true;
}
```

## ⑦ clear()

`clear()` 함수입니다. 간단하게 가지고 있는 문자열을 비우는 동작을 하면 됩니다. Buffer 멤버에 동적 할당이 되어 있다면 메모리를 해제하면 되고, `nullptr`로 애초에 문자열 할당이 되어있지 않은 상태라면 아무 짓도 하지 않도록 하면 됩니다. 마지막으로 길이는 없으니 `Length`는 0으로 설정 해 주면 되겠군요. 코드는 아래와 같습니다.

```cpp
// alteration
void _jcode::JString::clear() noexcept {
	PRINT_FUNCTION_CALL("JString::clear()");

	if(Buffer != nullptr) free(Buffer);

	Buffer = nullptr;
	Length = 0;
};
```

clear 해 보도록 합시다. 

```cpp
life_is_a_highway_lyrics_3.clear();
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_3);
```

아무것도 뜨지 않습니다.

```
(이미지)
```

## ⑧ insert() 

여기서부터 쓸만한 함수가 나오기 시작합니다. 문자열 수정의 대표적인 함수입니다. 어느 위치에 내가 원하는 문자열을 끼우고 싶을 때 사용하는 함수입니다.

원리는 간단할 겁니다. 위치에 대한 배열의 인덱스를 입력 받으면 인자로 들어온 문자열 크기만큼 메모리를 늘려주고, 기존의 문자열을 끼울 문자열의 크기만큼 뒤에 붙여두고 끼울 문자열을 그 사이에 끼워 넣으면 됩니다. 문장이 길어 장황하니 정리하면

* (1) 배열의 인덱스와 넣고 싶은 문자열을 인자로 받아서,  *
* (2) 받은 문자열 크기만큼 메모리 할당 먼저 해 주고 기존의 문자열을 그 크기만큼 뒤로 밀기, *
* (3) 사이에 문자열을 넣어 주면 끝. *

해 봅시다. 우선 문자열을 입력받는 함수입니다. `const char*` 인자이므로 문자열이 될 테고, 뒤의 `const int` 형식은 들어갈 위치를 나타내는 인덱스 값입니다. 390번 줄과 391번 줄은 간단한 예외 처리이고 우리가 위에서 작성한 (1)~(3)은 394번 줄부터 시작됩니다. 


* (1) 배열의 인덱스와 넣고 싶은 문자열을 인자로 받아서, (388번 줄) *
* (2) 받은 문자열 크기만큼 메모리 할당 먼저 해 주고 기존의 문자열을 그 크기만큼 뒤로 밀기, (394, 396번 줄) *
* (3) 사이에 문자열을 넣어 주면 끝. (398번 줄) *

```cpp
const bool _jcode::JString::insert(const char* argStr, const int argIdx) noexcept {

	if (preventAccess()) return false;
	if (argIdx < 0 || argIdx > Length) return false;
	
	// Reallocation
	Buffer = (char*)realloc(Buffer, sizeof(char) * (Length + std::strlen(argStr) + 1));

	std::memcpy(&Buffer[argIdx + std::strlen(argStr)], &Buffer[argIdx], sizeof(char) * (Length - argIdx + 1));
	
	std::memcpy(&Buffer[argIdx], argStr, sizeof(char) * std::strlen(argStr));

	update(); // update variable 'Length'!!

	return true;
};
```

문자열의 크기가 달라졌으니 `update()`를 꼭 호출하여 `Length` 멤버 값을 최신화 해야 될겁니다. 

하나 더 만들어봅시다. `insert()`의 인자가 항상 `char*` 형 배열이라고는 할 수 없습니다. 동일한 JString형을 다른 문자열 사이에 넣고 싶을 때가 있습니다. 오버라이딩(overriding) 해 봅시다. 
  
`insert()` 함수를 하나 더 만들고 첫 번째 인자만 `const JString&`으로 수정합니다. `const`는 입력받을 문자열이 수정될 일이 없을뿐더러 수정이 되면 안되므로 `const` 지정자를 붙이는게 좋고, JString이 가진 문자열의 크기를 알 수 없으니 복사하여 인자로 받는 것 보단 레퍼런스로 받는 것이 더 좋을 겁니다. 우리는 `insert()` 함수를 위에 아주 정성들여 만들어 두었으니 그걸 사용하면 더 간단해집니다. 코드는 아래와 같습니다.

```cpp
const bool _jcode::JString::insert(const JString& argJStr, const int argIdx) noexcept {
	return insert(argJStr.data(), argIdx);
}

```

노래 가사를 망치기 싫지만 "INSERTED"라는 문자열을 끼워 넣어 봅시다.

```cpp
life_is_a_highway_lyrics_1.insert("INSERTED", 3);
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_1);

life_is_a_highway_lyrics_1.insert(life_is_a_highway_lyrics_2, 4);
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_1);

```

```
(이미지)
```

정상적으로 동작합니다. 맘에 듭니다.

## ⑨ erase()

함수명 근대로 erase 하는 함수입니다. 인덱스를 나타내는 인자를 두 개 받고, 그 사이의 문자열을 삭제하면 됩니다. `insert()`와는 다르게 그냥 뒤의 메모리를 복사하여 당겨 붙인 후 메모리만 realloc 해주면 됩니다. 과정은 다음과 같습니다. 

```cpp
const bool _jcode::JString::erase(const int argStrIdx, const int argEndIdx) noexcept {

	if (preventAccess()) return false;

	// Filter
	if (argStrIdx > argEndIdx) return false;
	if (argStrIdx < 0 || argEndIdx < 0) return false;

	std::memcpy(&Buffer[argStrIdx + 1], &Buffer[argEndIdx + 1], sizeof(char) * (Length - argEndIdx + 1));
	Buffer = (char*)realloc(Buffer, sizeof(char) * (Length - (argEndIdx - argStrIdx)) + 1);

	update(); // update variable 'Length'!!

	return true;
}

```

414번 줄부터 418번 줄 까지는 예외 처리입니다. 인덱스 값이 잘못 된 경우는 메모리 할당을 할 수 없으니 예외로 둡니다. 420번 줄에서 볼 수 있듯이 해당 길이만큼 내용물을 복사하고 메모리 크기를 바로잡는 과정입니다. 어차피 메모리를 크기가 작아지는 쪽으로 재할당 하는 경우에는 그 뒤의 값은 버려집니다. realloc 레퍼런스에서도 그렇게 말하고 있습니다.

```
이미지
```

테스트 해 봅시다. 

```cpp
life_is_a_highway_lyrics_3.erase(0, 10);
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_3);
```

```
이미지
```

정상적으로 지워지는 것을 확인할 수 있습니다. 


## ⑩ push_back(), pop_back()

어디서 많이 본 함수입니다. `std::vector<>`를 사용할 때 많이 쓰는 함수입니다. 사용 방식은 동일하게 구현해보도록 하겠습니다.   `pop_back()`의 경우 `std::string`에서는 아무것도 리턴하지 않지만 여기서는 가장 뒤의 글자를 리턴하도록 해 봅시다. 

구현입니다. 

```cpp
const bool _jcode::JString::push_back(const char argc) noexcept {
	
	if (preventAccess()) {
		
		Buffer = (char*)malloc(sizeof(char) * 2);
		
		Buffer[0] = argc;
		Buffer[1] = '\0';

		update(); // update variable 'Length'!!

		return true;

	} else {
		
		Buffer = (char*)realloc(Buffer, sizeof(char) * (Length + 2));
		
		Buffer[Length + 1] = Buffer[Length];
		Buffer[Length] = argc;

		update(); // update variable 'Length'!!

		return true;
	};
};


const char _jcode::JString::pop_back() noexcept {  
	
	if (preventAccess())
		return -0x01; // will be regarded as error code.

	else {

		const char tbufc = Buffer[Length - 1];

		Buffer = (char*)realloc(Buffer, sizeof(char) * (Length));
		Buffer[Length - 1] = '\0';

		update(); // update variable 'Length'!!

		return tbufc;
	}

};
```

먼저 `push_back()`의 경우는 고려해야 할 수가 두 가지입니다. (1)동적 할당이 되어있지 않은 경우, 배열 크기를 2로 잡고 첫 번째 Buffer[0]에는 인자를 집어넣고 두 번째 Buffer[1]에는 문자 종료를 알리는 ‘`\0`’을 넣어주면 됩니다. (2)동적 할당이 되어있는 경우 사이즈를 1크게 잡고 먼저 ‘`\0`’을 넣어준 후 인자를 바로 앞에 넣어주면 될 겁니다. 어렵지 않습니다.

pop_back()의 경우 또한 두 가지인데 먼저 (1)동적 할당이 되어있지 않은 경우는 `pop` 할 내용이 없으므로 예외 코드로 `-0x01`을 리턴합니다. (2)동적 할당이 되어있는 경우에는 크기를 먼저 줄이고 마지막에만 종료 문자를 넣어줍니다. 어차피 크기를 하나 줄이는 경우 기존에 있던 ‘\0’만 지워질 테니 배열 마지막에만 ‘\0’ 문자를 대입하면 됩니다. 이 또한 그렇게 어렵지 않습니다. 

테스트 해 봅시다. ‘A'를 push 하고 pop 해봅시다.

```cpp
life_is_a_highway_lyrics_1.push_back('A');
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_1);

life_is_a_highway_lyrics_1.pop_back();
PRINT_NORMAL_MSG(life_is_a_highway_lyrics_1);
```

그러면 결과는 아래와 같습니다.

```
이미지
```

테스트, 성공적.

## ⑪ replace()

배열의 어느 한 인덱스부터 지정한 인덱스까지 문자열을 치환하는 함수입니다. 작동 이해에 대한 어려움은 없으니 바로 구현해봅시다.

```cpp
const bool _jcode::JString::replace(const char* argStr, const int& argFrom, const int& argTil) noexcept {
	
	if (preventAccess()) return false;

	const int TargetStrLen = std::strlen(argStr);

	if (argTil == 0) {
		
		for (int idx = argFrom; idx != (argFrom + TargetStrLen); idx++) {


			Buffer[idx] = argStr[idx - argFrom];
		}	

		return true;

	} else {
		
		if ((argFrom - argTil != TargetStrLen))
			return false; // size mismatch.

		for (int idx = argFrom; idx != argTil; idx++)
			Buffer[idx] = argStr[idx];

		return true;
	}
};
```

루프를 돌면서 하나하나씩 대입하는 과정입니다. 바로 테스트 해 봅시다. 

```cpp
life_life_is_a_highway_lyrics_3.replace("_SAMPLE_", 8);
PRINT_NORMAL_MSG(life_life_is_a_highway_lyrics_3);
```

```
이미지
```

정상작동 합니다!!



> 필자는 공군 작전정보통신단 체계개발실에서 복무('17~'19)하였습니다. 이 포스트는 작전정보통신단 병사 **프로그래밍 동아리(LINK)** 에서의 활동을 바탕으로 작성한 내용입니다.

[다음 포스트: JString Design[4]](https://sjoon-oh.github.io/archivers/Jstring_Design_4)
