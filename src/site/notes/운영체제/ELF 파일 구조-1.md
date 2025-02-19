---
{"dg-publish":true,"permalink":"/운영체제/ELF 파일 구조-1/","created":"2025-02-17T13:10:46.756+09:00"}
---

# 컴파일과 ELF 파일 구조
## 프로그램 컴파일 과정gcc hello.c
![](https://i.imgur.com/vPLWJXc.png)
![](https://i.imgur.com/uNDAh5L.png)
- Source 코드를 Compile하여 Object File을 만들고, Linker로 Executable File을 만든 뒤, 우리가 사용하는 하드웨어(Memory)에 적재시켜 Process화 한다.
### Linker
- 링킹 과정에서 라이브러리 파일과 오브젝트 파일에 있는 함수 정의와 전역 변수의 위치를 순차적으로 조사한 후, 이 정보를 테이블에 저장하고, 해당 주소들을 함수 호출 코드에 삽입합니다.
## Object Files
### Relocatable Object File(.o)
- Compile-time에 생성되는 바이너리 코드와 데이터로 이루어진 파일
- 다른 재배치 가능한 오브젝트 파일과 통합 가능
### Executable Object File : 실행파일
- 메모리에 직접 로딩되어 실행될 수 있는 바이너리 코드와 데이터로 이루어진 파일
### Shared Object File (.so)
- Load-time이나 Run-time에 동적으로 메모리에 로드되고 링크될 수 있는 타입의 재비치 가능한 오브젝트 파일
![](https://i.imgur.com/lc3hdrw.png)
## Object File Format
- COFF(Common Object FIle format) : System V 계열 초기 ==Unix==에서 사용
- PE(Portable Executable) : ==Windows== NT계열에서 사용하는 COFF의 변종
- ==ELF(Executable and Liking Format)== : ==Linux==, Solaris와 같은 대부분의 Unix에서 사용
  ![](https://i.imgur.com/Kpt19mY.png)
## Object FIle 구조 분석
### 파일 헤더에는 해당 파일이 실행되기 위한 모든 정보가 있다.
- 메모리에 어떻게 적재되는가? 
- 어디서부터 실행되어야 하는가? 
- 실행에 필요한 라이브러리(.so 또는 .dll)은 어떠한 것이 있는가? 
- 필요한 stack heap 메모리의 크기를 얼마로 할것인가?
### 오브젝트 파일(.o) 구조
![](https://i.imgur.com/LXw2Uo1.png)
### 실행 파일 구조
![](https://i.imgur.com/RSxzUPh.png)
- ==실행파일을 만들면== `.text`, `.rodata`, `.data`, `.strtab`, `.symtab`이 무조건 있다.
- `.bss`영역은 프로세스가 될 때 0으로 초기화 되기 때문에 실행파일 자체에는 실제 데이터로 저장되지 않는다.
- `strip`으로 `.strtab`, `.symtab` 섹션 삭제 가능하다.
	- ELF 파일 사이즈가 줄어들지만 디버깅 하기 어려워짐
- 동적 링킹을 한다면 `.interp`, `.dynsym`, `.dynstr`,  `.dynamic`영역이 추가로 생긴다.
---
# 심볼과 링킹
## Symbol과 Symbol Resolution
### 모든 오브젝트파일에는 심볼 테이블을 가지고 있다.
- ==Global== Symbol (다른 모듈에서 참조 가능) : 일반함수, 외부함수, 전역변수, 외부변수
	- strong symbol : 함수, 초기화된 전역변수
	- weak symbol : 초기화되지 않은 전역변수
- ==Local== Symbol (해당 모듈에서만 참조) : 정적함수, 정적변수, 파일명
	- 파일명 : 컴파일러가 오브젝트 파일을 생성할 때, 해당 오브젝트 파일이 어떤 소스 파일에서 컴파일되었는지 정보를 포함하기 위해 기록하는 심볼
### 심볼 해석(Symbol Resolution) 규칙
- 여러개의 ==strong== symbol은 허락되지 않는다.
- ==strong== symbol과 ==weak== symbol이 주어질 때, ==strong== symbol을 선택한다.
- 여러 개의 ==weak== symbol이 있을 경우 아무거나 선택한다.
### Symbol Resolution - 1
![](https://i.imgur.com/L6xKOUZ.png)
- 지역 변수는 해당 블록에서만 유효하고 다른 파일이나 함수에서 참조할 수 없으니 Symbol이 아니다.
- 함수 선언 != 함수 → 함수 선언은 Symbol이 아니다.
- Strong 심볼인 `add`가 두 파일에서 존재하므로 링킹을 하면 에러가 난다.
### Symbol Resolution - 2
![](https://i.imgur.com/4YNy4vE.png)
- `sum`에 `static`이 있어서 `add.c`에서만 접근 가능하다.(Local Symbol)
- `main.c'의 extern int sum;`는 아무것도 참조하지 못해 `undefined reference` 오류가 난다.
### 링킹(심볼테이블 병합 및 재구성)
![](https://i.imgur.com/syVdVAS.png)
## Static Linking vs Dynamic Linking
![](https://i.imgur.com/ue6x92V.png)
- Library
	- 이전에 작성한 코드의 집합
	- 헤더 파일에서 참조하는 함수를 구현할 수 있게 해주는 이진 코드의 데이터베이스
### static Linking
![|400](https://i.imgur.com/rXd9CfZ.png)
- 실행 가능한 파일을 만들 때 프로그램에서 사용하는 모든 라이브러리 모듈을 복사하는 방식
- ==Compile Time==에 프로그램마다 static libraries를 linking
	- file size ↑
		- 내 코드의 크기는 작더라도 큰 라이브러리가 들어가 있음(중복 발생)
	- 보안 좋음
	- Dynamic Linking보다 빠름
	- Compatibility issues X
### Dynamic Linking
![|400](https://i.imgur.com/QMOFBSJ.png)
- Compile TIme에 linking 관련 정보만 부여
- ==run time==에 shared libraries를 loading한 후 linking
	- 라이브러리를 올릴 때 보안 위협 우려(바꿔치기)
# Linking
- 실행파일에 정적 또는 동적으로 링크된 symbol과 symbol definition(symbol table)을 병합 및 재구성
---
# 출처
- 숭실대학교 이정현 교수님 시스템 보안
- [The Linking Process Exposed Static vs Dynamic Libraries](https://medium.com/@dkwok94/the-linking-process-exposed-static-vs-dynamic-libraries-977e92139b5f)
- [Simon (2018.01.24). Linker를 마무리 짓자 - ELF와 fromelf 까지!](https://blog.naver.com/poloamerica/221192585902)