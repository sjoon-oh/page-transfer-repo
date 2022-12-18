---
title:  "VerilogHDL Tutorial(1)"
date:   2020-04-18 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - Electronic Engineering
tags:
   - Electronic Engineering
toc: true
toc_sticky: true
---

# Verilog
## Verilog 규칙

Verilog 어휘 규칙은 다음을 따릅니다. 

1. 여백(white space)
- 빈칸(space), 탭(tab), 줄바꿈.
- 어휘 토큰들을 분리하기 위해 사용되는 경우를 제외하고는 무시됩니다.
- 공백(blank)과 탭은 문자열에서 의미 있게 취급됩니다.

2. 주석(comment)
- HDL 소스코드의 설명을 위해 사용되며, 컴파일과정에서 무시됩니다.
- 단일 라인 주석문; // 로 시작되어 해당 라인의 끝까지 유효합니다.
- 블록 주석문; /* ~ */ 로 표시됩니다. 단, 블록 주석문은 내포(nested)될 수 없습니다.

<!--more-->

3. 연산자(operator)
- 단항 연산자, 2항 연산자, 3항 연산자.

4. 식별자(identifier)
- 객체에 고유의 이름을 지정하기 위해 사용됩니다.
- 대소문자를 구별하여 인식합니다.
- 가독성을 위해 밑줄 사용 가능
- 단순 식별자: 일련의 문자, 숫자, 기호 $, 밑줄 등으로 구성됩니다. 이때 첫번째 문자는 숫자나 기호 $ 사용 불가하며 문자 또는 밑줄만 사용합니다.
- 확장 식별자(escaped identifier): \ (back slash)로 시작되며, 여백(빈칸, 탭, 줄바꿈) 등으로 끝납니다.  프린트 가능한 ASCII 문자들을 식별자에 포함시키는 수단을 제공합니다.

5. 키워드(keyword)
- Verilog 구성 요소를 정의하기 위해 미리 정의된 식별자입니다.
- 확장문자가 포함된 키워드는 키워드로 인식되지 않습니다.

예를 들어 유효한 식별자는 아래와 같습니다.

```verilog
shiftreg_a
busa_index
error_condition
merge_ab
_bus3
n$657
```

```verilog
// 확장 식별자
\busa+index
\-clock
\***error-condition***
\net1/\net2
\{a,b}
\a*(b+c)
```


## 수 표현(Number Representation)

### 정수형

정수형은 10진수, 16진수, 8진수, 2진수를 표현할 수 있습니다. 이 때 사용되는 형식은 아래 규칙을 따릅니다.

> [size_constant]'<base_format> <number_value>

size_constant는 값의 비트 크기를 나타내는 상수입니다. 0이 아닌 unsigned 10진수가 사용되고 생략 가능합니다. unsigned 수가 지정되지 않는다면 32비트로 표현됩니다. 또한 상위 비트가 x(unknown) 또는 z(high-impedance)인 unsized unsigned 상수는 그 상수가 사용되는 수식의 비트 크기만큼 확장됩니다. 

'base_format 자리는 밑수(base)를 지정하는 문자(d, D, h, H, o, O, b,
B)가 오게 됩니다. 

이때 몇 가지 규칙은 아래와 같습니다.

- 비트 크기와 밑수를 갖지 않는 단순 10진수는 signed 정수로 취급됩니다.
- 부호 지정자 없이 밑수만 지정되면 unsigned 정수로 취급됩니다.
- 밑수 지정자와 부호 지정자 s가 함께 사용되면 signed 정수로 취급됩니다. 이때 부호 지정자 s는 비트 패턴에는 영향을 미치지 않으며, 비트 패턴의
해석에만 영향을 미칩니다.
- 음수는 2의 보수(2’s complementary) 형식으로 표현됩니다.
- 지정된 비트 크기보다 unsigned 수의 크기가 작은 경우는 MSB 왼쪽에 0이 삽입되며, MSB가 x 또는 z이면, x 또는 z가 왼쪽에 삽입됩니다.
- 값에 물음표(?)가 사용되면 z로 취급합니다.
- 첫번째 문자를 제외하고는 밑줄(underscore)이 사용될 수 있습니다.

| Number | # of bits | Base    | Stored           |
|--------|-----------|---------|------------------|
| 10     | 32        | Decimal | 00....01010      |
| 2'b10  | 2         | Binary  | 10               |
| 3'd5   | 3         | Decimal | 101              |
| 8'o5   | 8         | Octal   | 00000101         |
| 3'b5   | Invalid   |         |                  |
| 12'hx  | 12        | Hex     | xxxxxxxxxxxx     |
| 'bz    | Unsized   | Binary  | zz...zzz(32bits) |


### 실수형(Real)
IEEE Std. 754-1985(IEEE standard for double-precision
floating-point number)

> value.value: decimal notation

> baseEexponent: scientific notation (the E is not case sensitive)

실수형 수의 표현 예시는 아래와 같습니다.

```verilog
0.5
679.00123
3e4
0.5e-0
5.6E-2
87E-4
93.432_26e-5 
```


## 문자열

문자열은 이중 인용 부호(“ ”)로 표현합니다. 문자열은 단일 라인에 존재해야 하며, 여러 라인에 걸친 문자열은 사용 불가능합니다. 문자열은 8비트 ASCII 값으로 표현되는 unsigned 정수형 상수로 취급되고 문자열 변수는 reg형의 변수입니다. 또한 문자열 내의 문자 수에 8을 곱한 크기의 비트 폭을 가집니다.

다음은 string을 저장 및 출력하는 코드 입니다.

```verilog
reg [8*18:1] str1;
initial begin
	str1 = "Hello Verilog HDL!";
end
```

```verilog
module str_test;
reg [8*10:1] str1;

initial begin
	str1 = “Hello”;
	$display(“%s is stored as %h", str1, str1);
	str1 = {str1, " !!!"};
	$display(“%s is stored as %h", str1, str1);
end
endmodule
```



## Module

Verilog 에서 module은 설계의 기본 단위가 됩니다. 큰 틀에서 아래와 같은 구조를 따릅니다. 



```verilog
module module_name (port_list);
	port 선언;
	reg 선언;
	wire 선언;
	parameter 선언;

	하위 모듈 호출
	always, initial 문
	function, task 정의문
	assign 문
	function, task 호출문

endmodule
```

모듈 내의 연결 방법은 다음과 같이 암시적 내부 연결(Implicit Internal Connection)과 명시적 내부 연결(Explicit Internal Connection)이 있습니다. 

```verilog
module module_name (port_name, port_name, ... );
	module_items
endmodule
```

```verilog
module module_name (
	.port_name (signal_name ),
	.port_name (signal_name ), ... );
	
	module_items
endmodule
```

다음의 예제를 보면 직관적으로 이해할 수 있습니다.

```verilog
module ex1 ( a1, b1, out1);
	input [3:0] a1, b1;
	output out1;
	assign out1 = ( a1 >= b1 ); // continuous assignment
endmodule
```

```verilog
// 2 input MUX with 2 bit widths
module ex2 (input wire [1:0] i0, i1, 
	input wire sel, 
	output wire [1:0] out2);

	wire t0, t1;
	assign out2 = {t1, t0}; // concatenation
	assign t1 = sel ? i1[1] : i0[1];
	assign t0 = sel ? i1[0] : i0[0];
endmodule
```

```verilog
module exp_port1 (.a(n1), .b(n2) );
// n1, n2는 모듈 내부에서 선언
// a, b는 포트연결로 정의
module exp_port2 (.a({b,c}), d, .f(g, h[2]));
// b, c, d, g, h[2]는 모듈 내부에서 선언
// a, d, f는 포트연결로 정의
module exp_port3 ({a, b}, .c(d))
// a, b, d는 모듈 내부에서 선언
// {a, b}, c는 포트연결로 정의
```

## Port

포트는 아래와 같은 형식으로 선언하며 방향은 다음과 같습니다. 

> `port_direction data_type signed [port_size] port_name, port_name, .. ;`

- input : 스칼라(scalar)나 벡터(vector)의 입력 포트 선언
- output : 스칼라(scalar)나 벡터(vector)의 출력 포트 선언
- inout : 스칼라(scalar)나 벡터(vector)의 양방향 포트 선언

예를 들어, 

```verilog
input a1, a2, en; // 3개의 스칼라 1 비트 포트
input signed [7:0] a, b; // 2개의 8비트 signed 값을 갖는 포트
output reg signed [16:0] res; // 데이터형과 signed 속성을 갖는 포트
output reg [11:0] cnt1; // little endian 표기방식
inout [0:15] data_bus; // big endian 표기방식
input [15:12] addr; // msb:lsb는 정수 값
parameter BW = 32;
input [BW-1:0] addr; // 상수 표현식 사용 가능
parameter SIZE = 4096;
input [log2(SIZE)-1:0] addr; // 상수 함수를 선언에서 호출가능
```