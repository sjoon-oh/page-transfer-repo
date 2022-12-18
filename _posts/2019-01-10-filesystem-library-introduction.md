---
layout: post
title:  "C++17 Filesystem Library Introduction"
date:   2019-01-10 09:00:00
categories: C/C++
permalink: /archivers/filesystem_library_introduction
---

# 간단한 filesystem(C++17) 맛보기

## 좋은 코드?

작년 중반 즈음, 아주 재미난 이야기를 듣습니다. 어느 개발 팀장 중 하나가 코드 평가 항목 중 하나는 ‘라인 수’라는 발언을 했다는 소식. 한마디로, 라인 수가 많으면 ‘좋은’ 또는 ‘잘 짜여진 코드’ 이므로 우수한 것 아니냐 라는 이야기입니다. 예.

<!--more-->

개발실 병사라고 다 개발하는 사람만 있는 것도 아니고, 그건 영외자도 마찬가지이지만, 어쨌든 여기 개발실에 있다보면 별 황당한 소리를 다 들을 수 있습니다. 코드 라인 수가 많으면 좋은 코드라니. 그렇다면 긴 문장은 좋은 글인가라는 의문을 품게 합니다. 
 
```
더군다나 
이렇게
글을 써도 
라인 수는
늘어나기 
마련인데
```

----------------------------------------------------------------------
## 내 코드를 세어 보자!

뭐, 좋습니다. 마침 잠깐 짬이 난 김에 라인 수를 카운트하는 간단한 프로그램을 만들게 됩니다. 당시 맡고 있던 체계가 있었는데 내가 얼마나 작성했는지 그거라도 어디 한번 세어 보자라는 생각이었습니다. 제로-베이스부터 시작한 프로젝트 였거든요.

자바를 주로 쓰는 사람은 알겠지만, 규모가 조금씩 불어나면 불어날수록, 기능을 분리하면 분리할수록 패키지 수는 늘어나고 그에 따른 클래스도 늘어납니다. 게다가 자바는 C++과 다르게 한 파일에 클래스 하나만 정의됩니다. 또 고맙게도 웹 프로젝트라 CSS, JSP, 자바스크립트 파일 등 클라이언트에 보여질 소스들은 각기 다른 폴더에 나뉘어져 들어가 있을 겁니다.

당시는 급하게 만든 나머지 파일 하나하나를 읽어 들여 라인을 세는 방식이었습니다. 예를 들면, 

![line_counter_folder](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-00.jpg)

path.txt라는 파일에 경로를 모두 적어두고, 그 경로를 읽고, 파일을 열고, 변수 하나를 선언하여 카운팅합니다.


![path](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-01.jpg)

뭐, 대략적인 소스는 아래와 같습니다.

```cpp
int main() {

	long long Accumulator_ = 0;
	
	std::string _T; // Temporary
	std::vector<std::string> pathList_;
	
	std::ifstream openFile_("path.txt"); // Write down all paths of files. Those can be found in properties menu.
	std::ofstream resultFile_("result.csv");
	
	_jcode::LineCounter counter_("");

	if(openFile_) {
	
		while(std::getline(openFile_, _T))
		pathList_.push_back(_T);
	}
	
	// 생략
}
```

오래 전에 작성한 코드라 기억이 가물가물 합니다. 일단 path.txt를 열어둡니다. 그리고 라인을 하나씩 읽어서 [주소][파일명]을 문자열 형식으로 `std::vector<string>` 형 변수 pathList_에 추가하고 있네요.

```
// Interface
long long _jcode::LineCounter::countLines() {

	std::ifstream openFile_(FileName_);
	// std::ifstream openFile_(FileName_.c_str());

	std::string _T;

	if (openFile_) 
		while (std::getline(openFile_, _T))
			Lines_++;

	return Lines_;
};
```

```cpp
for (auto itor_ = pathList_.begin(); itor_ != pathList_.end(); itor_++) {
	
		counter_.setFileName(*itor_);

		counter_.countLines();

		if (!counter_.getFileName().substr(0, 2).compare("//")) // User comments in path.txt.
			continue;

		if (counter_.getFileName().compare("")) {
			resultFile_ << counter_.getFileName() << "\t" << counter_.getLines() << std::endl;
			std::cout << counter_.getFileName() << ": " << counter_.getLines() << std::endl;
		}

		Accumulator_ += counter_.getLines();

		counter_.resetCounter();
	}
```

라인 하나씩 카운트하고 그냥 출력해줍니다. csv 형식으로 저장도 합니다. 저걸 돌려보면 

![result](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-02.jpg)


뭐, 이런식으로 저장은 될 겁니다. 

그런데, 파일 개수가 딱 봐도 너무 많습니다. 그리고 진행 중인 프로젝트인데 수시로 파일 명이 바뀌고, 추가되고, 삭제되고 하지요. 이건 딱 이정도로만 사용하는 게 적당한 것 같습니다. 지금 보니 너무 급히 짜서 어디 재사용할 수도 없는 코드네요.

반 년정도가 지난 지금, 다른 이유로 갑자기 라인 카운터가 필요해졌는데, 그 사이 프로젝트 버전(폴더)라던지 파일 개수라던지, 이름이라던지 너무 많이 바뀌었습니다. path.txt에 작성할 수도 있겠지만 그건 너무 노가다 작업입니다. 게다가 그러기 위해서는 모든 폴더, 모든 파일의 경로와 이름을 작성해야 합니다. 경로나 이름의 철자가 하나라도 틀리면 파일은 읽을 수 없으니 체크도 해 줘야 합니다.

----------------------------------------------------------------------

아무리 생각해도 이건 아닌 것 같습니다. 때 마침 새로운 컴파일러를 얻었으니 이걸 한번 사용해보도록 하겠습니다. 프로그램 목적은 아래와 같이 짜 보고 싶습니다.

1. 폴더 명을 지정했을 때, **하위 폴더에 있는 모든 파일**을 보고 싶다.
2. 모든 파일을 읽을 필요는 없다. 라이브러리 파일 등은 제외해야 하니까. **내가 작성한 소스만 골라** 라인 개수를 세고 싶다.

작년의 조잡한 라인 카운터를 제작할 당시는 1번 이유 때문에 포기했습니다. 일단 컴파일러 버전이 낮아 C++11도 제대로 지원하지 않았기 때문입니다. 첨부한 이미지는 Visual Studio(이하 VS) 2013 버전이지만 일부 라이브러리 문제 때문에 VS Platform Tool은 2010 버전을 사용했었습니다. 즉, 그나마 있던 2013도 에디터로만 이용될 뿐이었습니다. 

기존에 가지고 있던 컴파일러로 1번 관련 작업을 하기 위해서는 WinAPI를 알아야 했습니다. 그 외에 딱히 떠오르는 방법이 없었습니다. WinAPI를 공부하면 되지 않느냐? 작은 집을 짓는데 콘크리트 기둥부터 세우는 격입니다. 시간 대비 결과가 뻔했습니다. 필요한 것만 뽑아 쓰면 되지 않느냐? 당시 맡고 있던 프로젝트를 빨리 진행해야 했기에 잠시 짬 날때 할 수 있는 일이 아닙니다.

C++17에는 공식적으로 `filesystem` 라이브러리가 추가되었습니다. 이 라이브러리는 boost 라이브러리에 있던 것인데, C++17에 와서야 공식적으로 포함되었습니다. 그때 가지고 있던 컴파일러는 C++11 일부까지만 지원했기 때문에 음, 불가능. 

![vs2017](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-03.jpg)

음, 새로운 컴파일러입니다. VS2017 Professional입니다. 얼마 전 우연히 얻게 되었습니다. 그러면 `filesystem` 라이브러리가 있고, 맡고 있는 프로젝트도 얼추 마무리 단계에 들어갔으니 여유 있을 때 조잡한 라인 카운터를 한번 개선해보도록 하겠습니다.

----------------------------------------------------------------------

## Hello filesystem

일단 filesystem 라이브러리를 봅시다. 

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-04.jpg)

공식 레퍼런스에도 있네요? 한번 들어가 봅시다. 

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-05.jpg)

filesystem 헤더 안에 정의되고 네임스페이스는 filesystem 이랍니다. 바로 추가합시다.

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-06.jpg)

일단 에러는 없습니다. 네임스페이스 filesystem 안에 정의되어 있다 하니 fs로 간단하게 쓰게끔 using으로 선언해둡시다. 그러면, 

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-07.jpg)

에러가 뜹니다!! 이상합니다. 분명히 filesystem 네임스페이스 안에 정의되어 있다고 공식 레퍼런스에도 써 있건만. 어쩔 수 없으니 헤더를 열어 봅시다. 

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-08.jpg)

엥? C++17인데 `experimental` 이라고?? 믿을 수가 없습니다. 샘플로 다른 문서도 열어봅시다. C++17에서 지원하는 것 라이브러리 다른 것들도 열어 보죠 뭐. <variant>, <any>, <optional>을 추가 해보면,

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-09.jpg)

filesystem만 문제가 있습니다. 윈도우라 표준을 따르지 않는다, 실험적이므로 알아서 잘 써라 이런 건가 봅니다. 어쨌든 저는 요 놈이 필요하니 쓰려면 어쩔 수 없습니다. 

```cpp
#include <filesystem> // experimental since C++17, VS2017

// experimental? since C++17
namespace fs = std::experimental::filesystem;
```

까라면 까야 됩니다. 이 놈을 fs라 선언해두고 씁시다. 다시 돌아가서, 

1. 폴더 명을 지정했을 때, 하위 폴더에 있는 모든 파일을 보고 싶다. 

이 짓을 하고 싶습니다. 파일 시스템 표준 라이브러리도 있겠다, 한번 해 봅시다. 


## 본격적으로 디자인 해 보자!

우선, 이 놈의 목적을 위한 첫 번째 단계는, **어느 특정 폴더(그 하위의 무수히 많은 폴더 포함) 안에 있는 파일을 읽어 와야 한다는 것입니다.** 어차피 파일을 열어서 라인 수를 세고, 닫고, 또 다시 열고 세고, 닫고 하는 데에는 [절대경로][파일명]이 필요하니 문자열로 받아 `std::vector<std::string>` 이나 `std::list<std::string>`에 담아 두는 것으로 합시다. 
  
그러려면 최상위 루트 폴더, 즉 프로젝트 폴더 명을 지정 할 수 있어야 겠지요. 간단히 `std::string` 인스턴스에 받아 둡니다. 

```cpp
std::string DirectoryName = "";
std::string FileExtensions = "";
```

제가 생각해도 직관적인 변수 명입니다. 설명은 생략하겠습니다. 

```cpp

#include <iostream>

#ifndef CONSOLE_OUT_SERIES
#define CONSOLE_OUT_SERIES
#endif

#define COUT std::cout
#define CIN std::cin
#define ENDL std::endl

#ifdef CONSOLE_OUT_SERIES

#define CONSOLE_OUT(MSG) \
   do { COUT << MSG << ENDL; } while (0)
#define CONSOLE_OUT_SYSTEM(SYS_MSG) \
   do { COUT << "SYSTEM: " << SYS_MSG << ENDL; } while (0)
#define CONSOLE_OUT_MESSAGE(MSG) \
   do { COUT << "MESSAGE: " << MSG << ENDL; } while (0)

#endif

// 생략

namespace _jcode {

   class ExtensionFinder final {
      // This code is tested in Windows 7 32-bit system.
   private:
      std::string RootDirName; // Must be initialized.
      std::list<std::string> DirUnderRootList;

      std::vector<std::string> TargetFileExtension; // used when finding a specific extension of files.
      
      // Exception catcher
      // class!!
      class ExtensionFinderException final {
      private:
         std::string ExceptionMessage;

      public:
         // ctor
         // ExtensionFinderException() = default;
         ExtensionFinderException(const char*);
         ExtensionFinderException(std::string&);

         // dtor
         virtual ~ExtensionFinderException() = default;

         // interface
         inline const std::string isError() {
```

`ExtensionFinder`라는 클래스가 있습니다. 제가 작성한겁니다. 여담이지만 간단한 프로그램이라도 저는 래핑 작업을 좋아합니다. 둘러 싸는 작업입니다. 이유는 묻지 마세요. 

`ExtensonFinder`를 보면 매크로로 뭐라뭐라 정의해 두었고 `_jcode` 라는 네임스페이스에 정의해 두었습니다. 딱 봐도 직관적입니다. `RootDirName`은 최상위 폴더 명, `DirUnderRootList`는 최상위 폴더 아래에 있는 폴더들의 리스트 일테고, `TargetFileExtension`은 조금 이따가 설명하도록 하겠습니다.


## 폴더 순회

하위 폴더에 있는 파일을 보려면 최상위 폴더 아래에 있는 폴더 들 또한 순회해야 합니다. 여기서부터 `filesystem` 라이브러리가 도와 줄 겁니다. 순회하는 방법은 아래와 같습니다. 

```cpp
void _jcode::ExtensionFinder::showConsoleRootFolderFileList() const {

   try {

      for (auto& file_name : fs::directory_iterator(RootDirName)) {

         if (fs::is_regular_file(file_name))
            COUT << "\t(REG)FILE: " << file_name << ENDL;

         else if (fs::is_other(file_name))
            COUT << "\t(OTH)FILE: " << file_name << ENDL;

         else if (fs::is_directory(file_name))
            ; // should be filtered at showConsoleRootFolderList() function.

         else
            CONSOLE_OUT_SYSTEM("Unknown case detected.");
      }
   
   } catch (std::exception& argException) {
      CONSOLE_OUT_SYSTEM(argException.what());
   };

};
```

`showConsoleRootFolderFileList` 함수는 단순히 출력만 하는 디버깅용 함수입니다. 여기서 집중해야 할 부분은 붉게 표시한 부분입니다. 레퍼런스를 보면, 

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-11.jpg)

`std::filesystem::directory_entry` 요소를 순회하는 반복자입니다. 그렇지만 하위 폴더에 있는 놈들은 순회하지 않는다고 합니다. 예를 들어, 

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-12.jpg)

그림과 같은 폴더가 있다고 합시다. E:\프로젝트\Line_Counter_ 라는 경로를 제시하면, `directory_iterator`는 E:\프로젝트\Line_Counter_ 바로 밑의 폴더를 훑습니다. 이를 출력하면 하위 폴더의 요소들이 폴더인지, 일반 파일인지 등을 구분하지 않고 말입니다. 분기 없이 단순히 

```cpp
for(auto& file_name : fs::directory_iterator(RootDirName)) {
	COUT << file_name << ENDL;
}
```

라고 적은 후 컴파일 하고 실행 시켰다면, 

```
E:\프로젝트\Line_Counter_\Debug
E:\프로젝트\Line_Counter_\Line_Counter_
E:\프로젝트\Line_Counter_\Release
E:\프로젝트\Line_Counter_\Line_Counter_.sdf
E:\프로젝트\Line_Counter_\Line_Counter_.sln
E:\프로젝트\Line_Counter_\Line_Counter_.v12.suo
```

가 출력됩니다.


## 파일 구분

저는 파일을 구분해야만 합니다. 폴더는 읽을 수도 없을뿐더러 하위 폴더 경로는 확실히 구분을 해 두어야 순회가 가능하니까요. 이를 위해 라이브러리는 다음과 같은 함수를 제공합니다. 


![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-13.jpg)

File type을 알 수 있는 is_ 시리즈가 있습니다. 저는 이 놈이 폴더인지, 일반 파일인지 체크를 해야 하니, 


![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-14.jpg)

`std::filesystem::is_directory()` 요 놈을 써 봅시다. 이 함수는 “Checks if the given file status or path corresponds to a directory.” 입니다. 디렉토리인지 아닌지 구분해 주는 놈입니다. 함수의 인자로 들어가는 `std::filesystem::path` 형은 `std::string`형와 변환이 가능합니다. 그러면 아까의 코드로 돌아가서, 


```cpp
if (fs::is_regular_file(file_name))
	COUT << "/t(REG)FILE: " << file_name << ENDL;
	
else if (fs::is_other(file_name))
	COUT << "/t(OTH)FILE: " << file_name << ENDL;

else if (fs::is_directory(file_name))
	; // should be filtered at showConsoleRootFolderFileList() function.
```

요로코롬 분기 해 주면 됩니다. 많은 옵션이 있지만, 파일과 폴더(디렉토리)만 구분되면 되기 때문에 일차적으로 걸러 줍니다. 

폴더와 파일을 구분하는 법을 알았습니다. 그러면 폴더 내에 하위 폴더, 그리고 파일들이 있을 테니, 일단 폴더들의 절대 경로만 우선적으로 뽑아내야 합니다. 무식한 이중 for문을 씁시다. 그러면, 
 
```cpp
void _jcode::ExtensionFinder::findUnderRootDirAddr() {

#ifdef CONSOLE_DEBUG_ON
CONSOLE_OUT_SYSTEM("ExtensonFinder::findUnderRootDirAddr()");
#endif

	DirUnderRootList.push_back(this->RootDirName);
		// Will start from here.
		// This variable contains itself. (root)
		
	for(auto& iterator : DirUnderRootList) {
		for(auto& dir_name : fs::directory_iterator(iterator)) {
		
			if(fs::is_directory(dir_name))
				DirUnderRootList.push_back(dir_name.path().string());

			else
				;
		}	
	}
}
```

`std::list` 형식의 `DirUnderRootList`에 **폴더의 절대경로**만 계속적으로 밀어 넣습니다. `std::filesystem::directory_iterator`는 `const path&` 형식을 반환합니다. 따라서 이는 일반 std::string형으로 변환되지 않으므로 `dir_name.path().string()`으로 변환하여 구겨 넣습니다. 

여기까지 제가 설정한 최상위 폴더 안에 있는 모든 폴더의 절대경로를 뽑았습니다. 여기서 끝나면 안됩니다. 하위 폴더의 모든 소스 파일을 열어 라인을 세는 것이 목적이므로 이번에는 각 폴더 안의 파일의 절대경로를 뽑습니다. 방식은 동일합니다. 단지 `is_regular_file()` 함수로 걸러주면 될 겁니다.


## 확장자 구분

하기 전에, 저는 소스파일만 보고 싶습니다. 소스 파일이라는게 다 좋은데, 형식이 많습니다. 자바스크립트 파일, C++ 소스, 헤더, CSS, JSP, 자바 등등등... 이 놈들을 개발자들은 확장자로 구분합니다. 즉, 선택적으로 확장자를 뽑아 그 파일만 열어 읽어주면 됩니다. 

간단합니다. 확장자를 문자열로 받고, 파싱. 파일의 절대경로를 뽑는 법은 알고 있으니 리스트의 확장자가 맡는지 비교해 주면 됩니다. 일단 비교구문입니다. 

```cpp
bool _jcode::ExtensionFinder::isTargetExtension(const std::string& argAddr)

// 추가 필요
```

너무너무 간단한 코드이니 설명은 생략합니다.

우리는 확장자의 목록을 한 줄로 입력받을 겁니다. 스페이스 바 기준으로 단어를 뽀개서 std::vector에다 넣읍시다. 간단합니다.

```cpp
void _jcode::ExtensionFinder::setFileTargetExtension(const char* argExtensionStr) {
   
   std::string Wrapper(argExtensionStr);
   std::istringstream iss(Wrapper); // temporary wrapper

   try {

      // Tokenizing, and inserting to list<string>
      std::copy(
         std::istream_iterator<std::string>(iss),
         std::istream_iterator<std::string>(),
         std::back_inserter(TargetFileExtension));

   } catch (std::exception argException) {
      
      CONSOLE_OUT_SYSTEM("Problem occurred when tokenizing.");
   }

};
```

여기서 `TargetFileExtension`은 우리가 선택할 확장자가 문자열로 들어갈 `std::vector<std::string>`형 멤버 변수입니다. `std::istringstream`으로 뽁뽁뽁 밀어주면서 분리시키면 됩니다. 

그러면 거의 다 되어 갑니다 이제 비교하면,

```cpp
// getFileAddressListExtensionOf
std::vector<std::vector<std::string>> _jcode::ExtensionFinder::getFileAddrListExtension() {
   
   std::vector<std::vector<std::string>> AddrList;
   int _x = 0x00;
   
   CONSOLE_OUT_SYSTEM("Filtered file lists.");

   for (auto& extensionItor : TargetFileExtension) { // first reiterates with extension list.

      AddrList.push_back(std::vector<std::string>()); // make some space!

      for (auto& folderItor : DirUnderRootList) {
         for (auto& item_name : fs::directory_iterator(folderItor)) {

            if (isTargetExtension(item_name.path().string(), extensionItor)
               && !fs::is_directory(item_name)
               ) {
            
               AddrList.at(_x).push_back(item_name.path().string());
               CONSOLE_OUT("\t" + item_name.path().string()); // shows what have been pushed.
            }
         }
      }

      _x++;
   };

   return AddrList;
};
```

아까 작성해 두었던 폴더 명의 리스트가 있었습니다. 그 리스트를 돌면서 선택한 확장자와 비교하고, `is_directory`로 폴더인지 구분하여 파일의 절대경로를 2차원 `vector` 형식으로 저장합니다. 물론, `directory_iterator()` 가 쓰였으니 당연히 `path().string()`으로 `push_back()` 해 주어야 합니다. 


## 라인 카운팅

제가 디자인한 `ExtensionFinder` 클래스의 최종 목표는 여기까지입니다. `getFileAddrListExtension` 함수가 메인이라고 할 수 있겠습니다. 파일의 절대경로를 2차원 vector 형식으로 저장했으니, 이 목록의 파일을 열어서, 읽어주면 끝이군요!! 읽는건 간단하니 설명 없이 코드만 보겠습니다.

```cpp
long long _jcode::CounterAdv::countListOf(const std::vector<std::vector<std::string>>& argAddrList) {

   std::ifstream Target_File;
   std::string _T;
   
   long long Lines = 0;
   
   for (auto& extension_itor : argAddrList) {
      for (auto& addr_itor : extension_itor) {

         Target_File.open(addr_itor); // open it up!

         if (Target_File)
            while (std::getline(Target_File, _T)) Lines++; // Counting..

         Target_File.close(); // must be closed.
      }
   };

   return Lines;
};
```

## 실행해보자!

요약 하자면, 초기 라인 카운터와는 다르게, path.txt를 직접 작성하는 역할을 `ExtensionFinder` 클래스가 담당하고 있는 구조입니다. 최상위 폴더만 지정하면 아래 폴더를 순회하며 읽어들입니다. `filesystem` 라이브러리도 쓸만 합니다.

실험해 봅시다. C:\Users\Administrator\Desktop\[ERITER]WRS_Sys\source\final\WRS_Sys 을 최상위 폴더로 지정하고, .xml 확장자만 뽑아 라인 카운팅 하면, 

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-15.jpg)

![capture](/assets/posts/2019-01-10-filesystem-library-introduction/2019-01-10-16.jpg)

음!! 잘 됩니다!


> 필자는 공군 작전정보통신단 체계개발실에서 복무('17~'19)하였습니다. 이 포스트는 작전정보통신단 병사 **프로그래밍 동아리(LINK)** 에서의 활동을 바탕으로 작성한 내용입니다.