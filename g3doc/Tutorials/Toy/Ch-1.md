# 1장: Toy 튜토리얼 안내

[TOC]

이 튜토리얼은 MLIR 기반의 기본 toy 언어 구현을 통해 실행됩니다. 이 튜토리얼의 목표는 MLIR의 개념; 특히, [통용어(dialect)](../../LangRef.md#dialects)가 어떻게 언어별 구성 및 변환을 쉽게 지원하면서 LLVM 또는 다른 코드 생성 인프라로 쉽게 이동하는 방법을 제공하는지에 대해 소개하는 것입니다.
이 튜토리얼은 [LLVM Kaleidoscope 튜토리얼](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html)의 모델을 기반으로 합니다.

이 튜토리얼은 MLIR을 복제하여 빌드했다고 가정합니다; 만약 아직 하지 않았다면 [MLIR 시작](https://github.com/tensorflow/mlir#getting-started-with-mlir)을 참고하세요.

## 목차

이 튜토리얼은 다음과 같이 7장으로 나누어져 있습니다:

-   [#1장](Ch-1.md): Toy 언어와 Toy의 AST(추상 구문 트리)의 정의에 대한 안내
-   [#2장](Ch-2.md): MLIR 통용어를 내보내기 위한 AST를 알아보고 기본 MLIR 개념을 소개합니다. MLIR에서 사용자 정의 작업에 시맨틱을 추가하는 방법을 보여줍니다.
-   [#3장](Ch-3.md): 패턴 재작성 시스템을 사용한 고수준 언어별 최적화
-   [#4장](Ch-4.md): 인터페이스를 사용한 일반 통용어 독립적 변환 작성 방법. 통용어의 특정 정보를 크기 유추 및 인라이닝과 같은 일반적인 변환에 연결하는 방법을 보여줍니다.
-   [#5장](Ch-5.md): 저수준 통용어로 부분적으로 저수준화하기. 고수준 언어로 된 각각의 용어를 최적화 목적으로 일반적인 아핀 지향적 통용어로 변환합니다.
-   [#6장](Ch-6.md): LLVM 및 코드 생성으로 저수준화. 코드 생성을 위해 LLVM IR을 대상으로 삼고 더 저수준화된 프레임워크를 자세히 설명합니다.
-   [#7장](Ch-7.md): Toy 심화 : 복합 타입에 대한 지원 추가. MLIR에 사용자 정의 타입을 추가하는 방법과 기존 파이프라인에 타입을 적합시키는 방법을 설명합니다.

## 언어

이 튜토리얼은 "Toy"라고 하는 Toy 언어로 설명됩니다.
Toy는 함수를 정의하고 수학 연산을 수행하며 결과를 인쇄할 수 있는 텐서 기반 언어입니다.

코드 생성을 간단하게 유지하기 위해 2 이하의 랭크 텐서로 제한되며 Toy의 유일한 데이터 타입은 64비트 부동 소수점 타입(일명 C언어에서 'double')입니다. 따라서 모든 값은 암시적으로 double 정도이며, `값`은 변경할 수 없고 (즉, 모든 작업이 새로 할당된 값을 리턴) 할당 해제가 자동으로 관리됩니다.

```Toy {.toy}
def main() {
  # 정적 변수값으로 초기화 된 <2, 3> 크기의 배열 a를 정의하세요. 배열 크기는 제공된 정적 리터럴에서 유추됩니다.
  var a = [[1, 2, 3], [4, 5, 6]];

  # b는 a와 동일한데, 정적 리터럴로 된 텐서는 암시적으로 변환됩니다: 새로운 변수들은 텐서를 재배열하는 방법입니다(요소 수가 일치해야 합니다)
  var b<2, 3> = [1, 2, 3, 4, 5, 6];

  # transpose()와 print()는 유일한 내장함수이며, 다음 구문은 b를 전치하고 결과를 출력하기 전 요소별 곱셈을 수행할 것입니다.
  print(transpose(a) * transpose(b));
}
```

타입 확인은 타입 유추를 통해 정적으로 수행됩니다; 언어는 필요시 텐서 크기를 지정하기 위한 타입 선언만 필요합니다.
함수는 일반적임: 매개 변수는 랭크가 없습니다(즉, 텐서라는 것을 알고 있지만 그 크기는 알 수 없습니다).
호출 사이트에서 새로 발견된 모든 시그니처에 특화되어 있습니다.
사용자 정의 함수를 추가하여 이전 예제를 다시 살펴보겠습니다.

```Toy {.toy}
# 알려지지 않은 크기의 인수에 대한 연산을 하는 사용자 정의 일반 함수
def multiply_transpose(a, b) {
  return transpose(a) * transpose(b);
}

def main() {
  # 리터럴 값으로 초기화된 변수 `a`를 <2, 3> 텐서로 정의.
  var a = [[1, 2, 3], [4, 5, 6]];
  var b<2, 3> = [1, 2, 3, 4, 5, 6];

  # 이 호출은 두 인수 모두에 대해 'multiply_transpose'를 <2, 3>으로 특화시키고 `c`의
  # 초기 리턴 타입 <3, 2>를 추론합니다.
  var c = multiply_transpose(a, b);

  # 두 인수에 대해 <2, 3>을 사용하여 'multiply_transpose'를 두 번째로 호출하면 이전에
  # 특화된 유추 버전을 재사용하고 <3, 2>를 리턴합니다.
  var d = multiply_transpose(b, a);

  # 두 차원에 대해 <3, 2> (<2, 3> 대신)를 새로 호출하면 또 다른
  # 'multiply_transpose' 특화가 일어납니다.
  var e = multiply_transpose(c, d);

  # 마지막으로 호환되지 않는 크기로 'multiply_transpose'를 호출하면 크기 유추 오류가 발생합니다.
  var f = multiply_transpose(transpose(a), c);
}
```

## The AST

위 코드의 AST는 매우 간단합니다. 여기에 덤프가 있습니다:

```
Module:
  Function
    Proto 'multiply_transpose' @test/ast.toy:5:1'
    Args: [a, b]
    Block {
      Return
        BinOp: * @test/ast.toy:6:25
          Call 'transpose' [ @test/ast.toy:6:10
            var: a @test/ast.toy:6:20
          ]
          Call 'transpose' [ @test/ast.toy:6:25
            var: b @test/ast.toy:6:35
          ]
    } // Block
  Function
    Proto 'main' @test/ast.toy:9:1'
    Args: []
    Block {
      VarDecl a<> @test/ast.toy:11:3
        Literal: <2, 3>[<3>[1.000000e+00, 2.000000e+00, 3.000000e+00], <3>[4.000000e+00, 5.000000e+00, 6.000000e+00]] @test/ast.toy:11:17
      VarDecl b<2, 3> @test/ast.toy:12:3
        Literal: <6>[1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00] @test/ast.toy:12:17
      VarDecl c<> @test/ast.toy:15:3
        Call 'multiply_transpose' [ @test/ast.toy:15:11
          var: a @test/ast.toy:15:30
          var: b @test/ast.toy:15:33
        ]
      VarDecl d<> @test/ast.toy:18:3
        Call 'multiply_transpose' [ @test/ast.toy:18:11
          var: b @test/ast.toy:18:30
          var: a @test/ast.toy:18:33
        ]
      VarDecl e<> @test/ast.toy:21:3
        Call 'multiply_transpose' [ @test/ast.toy:21:11
          var: b @test/ast.toy:21:30
          var: c @test/ast.toy:21:33
        ]
      VarDecl f<> @test/ast.toy:24:3
        Call 'multiply_transpose' [ @test/ast.toy:24:11
          Call 'transpose' [ @test/ast.toy:24:30
            var: a @test/ast.toy:24:40
          ]
          var: c @test/ast.toy:24:44
        ]
    } // Block
```

이 결과를 재현할 수 있으며 `examples/toy/Ch1/` 디렉토리의 예제를 실행할 수 있습니다;
`path/to/BUILD/bin/toyc-ch1 test/Examples/Toy/Ch1/ast.toy -emit=ast`
명령어를 실행하세요.
ex: `llvm-project/build/bin/toyc-ch1
llvm-project/llvm/projects/mlir/test/Examples/Toy/Ch1/ast.toy -emit=ast`

렉서(Lexer) 코드는 매우 간단합니다; 이는 모두 `examples/toy/Ch1/include/toy/Lexer.h` 헤더에 있습니다.
파서(Parser)는 `examples/toy/Ch1/include/toy/Parser.h`에서 찾을 수 있습니다; 이는 반복 강하 파서입니다.
이러한 렉서/파서에 익숙하지 않은 경우 [Kaleidoscope 튜토리얼](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl02.html)의 처음 두 장에서 자세히 설명하는 LLVM Kaleidoscope와 매우 유사합니다.

[다음 장](Ch-2.md)에서는 AST를 MLIR로 바꾸는 방법에 대해 설명할 것입니다.
