# 2장: MLIR 내보내기 기본

[TOC]

이제 AST와 언어에 익숙해졌으니, MLIR이 Toy 컴파일을 도울 수 있는지 살펴봅시다.

## 소개: 다단계 중간 표현(MLIR)

LLVM([Kaleidoscope 튜토리얼](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html) 참조)과 같은 다른 컴파일러는 사전 정의된 고정 타입 세트(일반적으로 저수준/RISC와 같은) 명령어를 제공합니다.
LLVM IR을 내보내기 전에 언어별 타입 검사, 분석 또는 변환을 실행하는 것은 주어진 언어의 프론트엔드에 달려 있습니다.
예를 들어 Clang은 AST를 사용하여 정적 분석뿐만 아니라 AST 복제 및 재작성을 통한 C++ 템플릿 인스턴스화와 같은 변환도 수행합니다.
마지막으로 C/C++보다 고수준의 언어를 사용하는 언어는 LLVM IR을 생성하기 위해 AST부터 더 저수준화해야 할 수도 있습니다.

결과적으로 여러 프론트엔드가 분석이나 변환이 필요한 부분을 지원하기 위해 상당 부분의 인프라를 다시 구현하게 됩니다.
MLIR은 확장성을 위해 설계되어 이 문제를 다룹니다.
따라서 미리 정의된 명령어(MLIR 용어로 *작업(operations)*)나 타입이 거의 없습니다.

## MLIR 인터페이스

[언어 참조](../../LangRef.md)

MLIR은 완전히 확장 가능한 인프라로 설계되었습니다; 속성 고정 세트(예: 정적 메타 데이터), 작업 또는 타입이 없습니다.
MLIR은 [통용어(dialect)](../../LangRef.md#dialects) 개념으로 이러한 확장성을 지원합니다.
통용어는 유일한 `namespace`를 추상화하기 위한 그룹화 매커니즘을 제공합니다.

MLIR에서 추상화 및 계산의 핵심 단위인 [`작업`](../../LangRef.md#operations)은 여러 면에서 LLVM 명령어와 유사합니다.
작업에는 어플리케이션별 시맨틱이 있을 수 있으며 명령, 전역(함수와 같은), 모듈 등 LLVM의 모든 핵심 IR 구조를 나타내는 데 사용될 수 있습니다.

Toy의 `transpose` 작업을 위한 MLIR 어셈블리는 다음과 같습니다.

```mlir
%t_tensor = "toy.transpose"(%tensor) {inplace = true} : (tensor<2x3xf64>) -> tensor<3x2xf64> loc("example/file/path":12:1)
```

이 MLIR 작업의 구조를 분석해 봅시다.

-   `%t_tensor`

    *   이 작업으로 정의된 결과에 부여된 이름([충돌을 피하기 위해 %(sigil)](../../LangRef.md#identifiers-and-keywords)이 포함됨).
        작업은 정적 단일 할당(SSA)값으로 된 0개 이상의 결과를 정의할 수 있습니다 (Toy 문맥상 단일 결과 작업으로 제한함).
        이름은 구문 분석 중에 사용되지만 영구적이지 않습니다(예: 메모리 내 SSA값 표시에서 추적되지 않음).

-   `"toy.transpose"`

    *   작업의 이름. "`.`" 앞에 접두 통용어의 네임 스페이스가 있는 고유한 문자열이어야 합니다.
        이것은 Toy 통용어에서 `transpose` 작업으로 읽을 수 있습니다.

-   `(%tensor)`

    *   다른 연산에 의해 정의되거나 블록 인수를 참조하는 0개 이상의 SSA값인 입력 피연산자(또는 인수) 리스트.

-   `{ inplace = true }`

    *   항상 일정한 0개 이상의 특수 피연산자 속성 딕셔너리.
        상수 값이 true인 'inplace'라는 불리언 속성을 정의합니다.

-   `(tensor<2x3xf64) -> tensor<3x2xf64>`

    *   함수형의 작업 타입을 나타내며 괄호 안의 인수 타입과 화살표 뒤에는 반환값 타입을 입력합니다.

-   `loc("example/file/path":12:1)`

    *   이 작업이 시작된 소스 코드의 위치입니다.

여기에는 일반적인 작업 형태가 나와 있습니다.
위에 설명한 바와 같이, MLIR에서의 작업 세트는 확장 가능합니다.
이는 인프라가 작업 구조를 압축(opaque)하여 추론할 수 있어야 함을 뜻합니다.
이것은 작업의 구성을 여러 조각으로 축소시켜 수행됩니다.

-   작업 이름.
-   SSA 피연산자값 리스트.
-   [속성](../../LangRef.md#attributes) 리스트.
-   결과값 [타입](../../LangRef.md#type-system) 리스트.
-   디버깅용 [소스 위치](../../Diagnostics.md#source-locations).
-   (대부분 분기별) 후속 [블록] 리스트
-   (함수와 같은 구조용 작업)[영역](../../LangRef.md#regions) 리스트

MLIR에서 모든 작업에는 관련된 필수 소스 경로가 있습니다.
디버그 정보 경로가 메타 데이터이고 삭제될 수 있는 LLVM과는 달리, MLIR에서 위치는 핵심 요구 사항이며 API는 이 경로에 따라 달라집니다.
따라서 경로를 삭제하는 것은 실수가 아닌 명백한 선택입니다.

추가 설명: 변환이 작업을 다른 작업으로 대체하는 경우 새 작업에는 여전히 경로가 포함되어야 합니다.
이를 통해 해당 작업의 출처를 추적할 수 있습니다.

mlir-opt 도구(컴파일러 패스(pass) 테스트용 도구)는 기본적으로 출력에 경로를 포함하지 않습니다.
위치를 포함하려면 `-mlir-print-debuginfo` 플래그를 지정합니다.(더 많은 옵션을 보려면 `mlir-opt --help`를 실행하세요.)

### API 압축

MLIR은 완전히 확장 가능한 시스템으로 설계되었는데 인프라는 모든 핵심 구성요소(속성, 작업, 유형 등)를 압축 표현할 수 있는 기능이 있습니다.
이는 MLIR이 유효한 IR을 파싱, 표현 및 [왕복](../../Glossary.md#round-trip) 가능하게 합니다.
예를 들어, `.mlir`파일에 Toy 작업을 배치하고 아무 등록 없이 *mlir-opt*를 가지고 왕복할 수 있습니다.

```mlir
func @toy_func(%tensor: tensor<2x3xf64>) -> tensor<3x2xf64> {
  %t_tensor = "toy.transpose"(%tensor) { inplace = true } : (tensor<2x3xf64>) -> tensor<3x2xf64>
  return %t_tensor : tensor<3x2xf64>
}
```

등록되지 않은 속성, 작업 및 타입의 경우 MLIR은 일부 구조적 제약조건(SSA, 블록 중단 등)을 시행하지만 그렇지 않으면 완전히 압축합니다.
부트스트래핑 목적으로는 유용할 수 있지만 일반적으로 권장되지 않습니다.
압축 작업은 변환 및 분석에 의해 보수적으로 처리되어야 하며, 구성 및 조작(manipulate)이 훨씬 어렵습니다.

이를 조작(handling)하는 것은 Toy에 유효하지 않은 IR을 만들고 검증기를 거치지 않고 왕복하는 것을 보면 알 수 있습니다:

```mlir
// 실행: toyc %s -emit=mlir

func @main() {
  %0 = "toy.print"() : () -> tensor<2x3xf64>
}
```

여기에는 여러 가지 문제가 있는데, `toy.print` 작업은 종결문이 아니고 피연산자가 필요하며 값을 리턴해서는 안됩니다.
다음 장에서는 MLIR로 통용어 및 작업을 등록하고 검증기에 연결한 다음 더 나은 API를 추가하여 작업을 조작합니다.

## Toy 통용어 정의

MLIR을 효율적으로 인터페이스화하기 위해 새로운 Toy 통용어를 정의할 것입니다.
이 통용어는 Toy 언어의 의미를 올바르게 모델링할 뿐만 아니라 고수준 언어 분석이나 변환을 하는 쉬운 방법을 제공합니다.

```c++
/// 이 클래스는 Toy 통용어를 정의합니다.
/// 통용어는 mlir::Dialect에서 상속하고 사용자 정의 속성, 작업 및 타입을 (생성자에) 등록합니다.
/// 또한 가상 메서드를 통해 알려진 일반적인 동작을 재정의 할 수 있으며 다음 튜토리얼에서 설명합니다.
class ToyDialect : public mlir::Dialect {
 public:
  explicit ToyDialect(mlir::MLIRContext *ctx);
  /// 통용어 네임스페이스에 대한 유틸리티 접근자를 제공합니다.
  /// 이는 여러 유틸리티에서 사용됩니다.
  static llvm::StringRef getDialectNamespace() { return "toy"; }
};
```

이제 통용어를 전역 레지스트리에 등록할 수 있습니다.

```c++
  mlir::registerDialect<ToyDialect>();
```

이제 만들어진 새로운 `MLIRContext`는 Toy 통용어의 인스턴스를 포함하고 파싱 속성 및 타입과 같은 것들에 대한 특정 후크를 호출합니다.

## Toy의 작업 정의하기

이제 Toy 통용어가 생겼으므로 등록 작업을 시작할 수 있습니다.
이를 통해 나머지 시스템이 연결할 수 있는 시맨틱 정보를 제공할 수 있습니다.
`toy.constant` 작업 생성 과정을 알아봅시다.

```mlir
 %4 = "toy.constant"() {value = dense<1.0> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
```

이 작업은 피연산자를 갖지 않는 `value`라는 [밀집 요소](../../LangRef.md#dense-elements-attribute) 속성을 사용하고 단일 결과의 [TensorType](../../LangRef.md#tensor-type)를 리턴합니다 .
작업은 동작을 사용자화하기 위해 선택적 [*특성*](../../Traits.md)이 필요한 [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)의 `mlir::Op` 클래스를 상속합니다.
이러한 특성은 추가 접근자, 검증 등을 제공할 수 있습니다.

```c++
class ConstantOp : public mlir::Op<ConstantOp,
                     /// ConstantOp의 입력이 없습니다.
                     mlir::OpTrait::ZeroOperands,
                     /// ConstantOp는 단일 결과를 리턴합니다.
                     mlir::OpTrait::OneResult,
                     /// ConstantOp는 부작용이 없이 깔끔합니다.
                     mlir::OpTrait::HasNoSideEffect> {

 public:
  /// 기본 Op 클래스에서 생성자를 상속함.
  using Op::Op;

  /// 이 작업의 고유명을 제공하세요.
  /// MLIR은 이를 사용하여 작업을 등록하고 시스템 전체에서 작업을 고유하게 식별합니다.
  static llvm::StringRef getOperationName() { return "toy.constant"; }

  /// 속성에서 상수를 가져와서 상수값을 리턴합니다.
  mlir::DenseElementsAttr getValue();

  /// 작업은 정의한 특성 이상의 추가 검증을 제공할 수 있습니다.
  /// 여기서는 고정 작업의 명시적 불변이 유지되도록 합니다. 예를 들어 결과 타입은 TensorType이어야 합니다.
  LogicalResult verify();

  /// 입력 값 세트에서 이 작업을 빌드하기 위한 인터페이스를 제공합니다.
  /// 이 인터페이스는 빌더가 이 작업의 인스턴스를 쉽게 생성할 수 있게 합니다:
  ///   mlir::OpBuilder::create<ConstantOp>(...)
  /// 이 메소드는 MLIR이 작업을 생성하는 데 사용하는 주어진 `상태`를 채웁니다.
  /// 이 상태는 작업에 포함될 수 있는 모든 개별 요소들의 모음입니다.
  /// 주어진 리턴 타입과 `value` 속성으로 상수를 만듭니다.
  static void build(mlir::Builder *builder, mlir::OperationState &state,
                    mlir::Type result, mlir::DenseElementsAttr value);
  /// 상수를 만들고 주어진 'value'에서 타입을 재사용합니다.
  static void build(mlir::Builder *builder, mlir::OperationState &state,
                    mlir::DenseElementsAttr value);
  /// 주어진 '값'을 내보내어(broadcasting) 상수를 빌드합니다.
  static void build(mlir::Builder *builder, mlir::OperationState &state,
                    double value);
};
```

그리고 우리는 이 작업을 `ToyDialect` 생성자에 등록합니다:

```c++
ToyDialect::ToyDialect(mlir::MLIRContext *ctx)
    : mlir::Dialect(getDialectNamespace(), ctx) {
  addOperations<ConstantOp>();
}
```

### Op vs Operation: MLIR 작업 사용

이제 작업을 정의했으므로 접근하고 변환하려고 합니다.
MLIR에는 작업과 관련된 두 가지 주요 클래스인 `Operation`과 `Op`가 있습니다.
Operation은 작업의 실제 명시적 인스턴스이며 작업 인스턴스에 대한 일반 API를 나타냅니다.
`Op`는 `ConstantOp`와 같은 파생 작업의 기본 클래스이며 `Operation*`을 중심으로 스마트 포인터의 래퍼(wrapper) 역할을 합니다.
즉, Toy 작업을 정의할 때 `Operation` 클래스를 빌드하고 인터페이스화하기 위한 깨끗한 인터페이스를 제공합니다;
`ConstantOp`가 클래스 필드를 정의하지 않는 이유가 바로 이것입니다.
따라서 참조 또는 포인터 대신 값으로 이 클래스를 항상 전달합니다(*값으로 전달(passing by value)*은(는) 일반적인 관용구이며 속성, 타입 등에 비슷하게 적용됩니다).
LLVM의 캐스팅 인프라를 사용하여 toy 작업의 인스턴스를 항상 얻을 수 있습니다:

```c++
void processConstantOp(mlir::Operation *op) {
  ConstantOp op = llvm::dyn_cast<ConstantOp>(op);

  // 이 작업은 `ConstantOp`의 인스턴스가 아닙니다.
  if (!op)
    return;

  // 내부 작업 인스턴스를 다시 가져옵니다.
  mlir::Operation *internalOp = op.getOperation();
  assert(internalOp == op && "these operation instances are the same");
}
```

### 작업 정의 구체화(ODS, Operation Definition Specification) 프레임워크 사용

MLIR은 C++ 템플릿인 `mlir::Op`를 전문화할 뿐만 아니라 선언적 방식으로 작업 정의를 지원합니다.
이것은 [작업 정의 구체화](../../ OpDefinitions.md) 프레임워크를 통해 이루어집니다.
작업에 관한 사실은 TableGen 레코드에 간결하게 지정되며, 컴파일 시 해당 C++ 템플릿인 `mlir::Op` 전문화로 확장됩니다.
C++ API 표면 변경에 대한 단순성, 간결성 및 일반적인 안정성을 고려하면 MLIR에서 작업을 정의할 때 ODS 프레임워크를 사용하는 것이 좋습니다.

ConstantOp와 동등한 ODS를 정의하는 방법을 알아봅시다:

가장 먼저 해야 할 일은 C++에서 정의한 Toy 통용어에 대한 링크를 정의하는 것입니다.
이것은 정의할 모든 작업을 통용어에 연결하는 데 사용됩니다:

```tablegen
// 작업을 정의할 수 있도록 ODS 프레임워크에 'toy' 통용어에 대한 정의를 제공합니다.
def Toy_Dialect : Dialect {
  // 우리 통용어의 네임스페이스는 1-1에서 `ToyDialect::getDialectNamespace`에서 제공한 문자열에 해당한다.
  let name = "toy";

  // 통용어 클래스가 정의된 C++ 네임스페이스입니다.
  let cppNamespace = "toy";
}
```

이제 Toy 통용어로 향하는 링크를 정의했으므로 작업 정의를 시작할 수 있습니다.
ODS의 작업은 `Op` 클래스에서 상속하여 정의됩니다.
작업 정의를 단순화하기 위해 Toy 통용어에서 작업의 기본 클래스를 정의합니다.

```tablegen
// Toy 통용어 작업을 위한 기본 클래스.
// 이 작업은 OpBase.td의 기본 `Op` 클래스에서 상속되며 다음을 제공합니다:
//  * 작업의 부모 통용어.
//  * 작업의 니모닉(기계어를 언어로 바꿔준 것) 또는 통용어 접두어가 없는 이름.
//  * 작업의 특성 리스트.
class Toy_Op<string mnemonic, list<OpTrait> traits = []> :
    Op<Toy_Dialect, mnemonic, traits>;
```

모든 예비 작업이 정의되면 고정 작업을 정의 할 수 있습니다.

위의 기본 'Toy_Op' 클래스에서 상속하여 Toy 작업을 정의합니다.
여기 우리는 작업에 대한 니모닉과 특성 리스트를 제공합니다.
여기서 [니모닉](../../OpDefinitions.md#operation-name)은 통용어 접두사 없이
`ConstantOp::getOperationName`에 주어진 것과 일치합니다. `장난감 .`.
여기에서의 고정 작업은 'NoSideEffect'로 표시됩니다.
이것은 ODS 특성이며, `ConstantOp`: `mlir::OpTrait::HasNoSideEffect`를 정의할 때 제공하는 특성과 일대일 대응합니다.
C++ 정의에는 `ZeroOperands`와 `OneResult` 특성이 없습니다; 이것들은 우리가 나중에 정의하는 `arguments`와 `results` 필드에 따라 자동으로 추론됩니다.

```tablegen
def ConstantOp : Toy_Op<"constant", [NoSideEffect]> {
}
```

이 시점에서 TableGen에 의해 생성된 C++ 코드가 어떻게 보이는지 알고 싶을 것입니다.
`mlir-tblgen` 명령을 `gen-op-decls`나 `gen-op-defs` 동작과 함께 실행하면 됩니다.

```
${build_root}/bin/mlir-tblgen -gen-op-defs ${mlir_src_root}/examples/toy/Ch2/include/toy/Ops.td -I ${mlir_src_root}/include/
```

선택된 동작에 따라 `ConstantOp` 클래스 선언이나 구현을 출력합니다.
이 출력을 수작업으로 구현한 것과 비교하면 TableGen을 시작할 때 매우 유용합니다.

#### 인수 및 결과 정의하기

작업의 쉘이 정의되면 이제 작업에 [입력](../../OpDefinitions.md#operation-arguments) 및 [출력](../../OpDefinitions.md#operation-results)을 제공할 수 있습니다.
작업에 대한 입력, 혹은 인수는 SSA 피연산자 값에 대한 속성 또는 타입일 수 있습니다.
결과는 작업으로 생성된 값의 타입 세트에 해당합니다.

```tablegen
def ConstantOp : Toy_Op<"constant", [NoSideEffect]> {
  // 고정 작업은 속성을 유일한 입력으로 사용합니다.
  // `F64ElementsAttr`는 64비트 부동 소수점 ElementsAttr에 해당합니다.
  let arguments = (ins F64ElementsAttr:$value);

  // 고정 작업은 TensorType의 단일 값을 리턴합니다.
  // F64Tensor는 64비트 부동 소수점 TensorType에 해당합니다.
  let results = (outs F64Tensor);
}
```

인수 또는 결과에 이름을 제공하면(예를 들어 `$value`) ODS는 자동으로 일치하는 접근자(`DenseElementsAttr ConstantOp::value()`)를 생성합니다: .

#### 문서 추가

작업을 정의하고 나서 해야 할 것은 작업을 문서화하는 것입니다.
작업의 의미를 설명하기 위해 [`summary`와 `description`](../../OpDefinitions.md#operation-documentation) 필드를 제공할 수 있습니다.
이 정보는 통용어 사용자에게 유용하며 마크다운 문서를 자동 생성하는 데 사용될 수도 있습니다.

```tablegen
def ConstantOp : Toy_Op<"constant", [NoSideEffect]> {
  // 이 작업에 대한 요약 및 설명을 제공합니다.
  // 이것은 통용어 내 작업의 문서를 자동 생성하는 데 사용될 수 있습니다.
  let summary = "고정 작업";
  let description = [{
    고정 작업은 리터럴을 SSA 값으로 바꿉니다.
    데이터는 작업에 속성으로 추가됩니다. 예를 들어:

      %0 = "toy.constant"()
         { value = dense<[[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]> : tensor<2x3xf64> }
        : () -> tensor<2x3xf64>
  }];

  // 고정 작업은 속성을 유일한 입력으로 사용합니다.
  // `F64ElementsAttr`는 64비트 부동 소수점 ElementsAttr에 해당합니다.
  let arguments = (ins F64ElementsAttr:$value);

  // 일반 호출 작업은 단일값 TensorType을 리턴합니다.
  // F64Tensor는 64비트 부동 소수점 TensorType에 해당합니다.
  let results = (outs F64Tensor);
}
```

#### 작업 의미론 검증

이 시점에서 우리는 이미 대부분의 기존 C++ 작업의 정의를 다루었습니다.
다음으로 정의할 부분은 검증기입니다.
운좋게도, 명명된 접근자와 마찬가지로 ODS 프레임워크는 제시한 제약 조건에 따라 필요한 검증 로직을 자동으로 생성합니다.
즉, 리턴 타입의 구조라던가 `value` 입력 속성을 확인할 필요가 없습니다.
대부분의 경우 ODS 작업을 위해 추가 검증이 필요하지 않습니다.
추가 검증 로직을 추가하기 위해 작업이 [`verifier`] (../../ OpDefinitions.md # custom-verifier-code) 필드를 대체할 수 있습니다.
`verifier` 필드는 `ConstantOp::verify`의 일부로 실행될 C++ 코드 blob을 정의 할 수 있습니다.
이 blob은 다른 모든 작업이 이미 검증되었다고 추측할 수 있습니다.

```tablegen
def ConstantOp : Toy_Op<"constant", [NoSideEffect]> {
  // 이 작업에 대한 요약 및 설명을 제공합니다.
  // 이것은 통용어 내 작업의 문서를 자동 생성하는 데 사용될 수 있습니다.
  let summary = "고정 작업";
  let description = [{
    고정 작업은 리터럴을 SSA 값으로 바꿉니다.
    데이터는 작업에 속성으로 추가됩니다. 예를 들어:

      %0 = "toy.constant"()
         { value = dense<[[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]> : tensor<2x3xf64> }
        : () -> tensor<2x3xf64>
  }];

  // 고정 작업은 속성을 유일한 입력으로 사용합니다.
  // `F64ElementsAttr`는 64비트 부동 소수점 ElementsAttr에 해당합니다.
  let arguments = (ins F64ElementsAttr:$value);

  // 일반 호출 작업은 단일값 TensorType을 리턴합니다.
  // F64Tensor는 64비트 부동 소수점 TensorType에 해당합니다.
  let results = (outs F64Tensor);

  // 고정 작업에 추가 검증 로직을 추가하십시오.
  // 여기서 우리는 C++ 소스 파일에서 정적 `verify` 메소드를 호출합니다.
  // 이 코드 블록은 ConstantOp::verify 내부에서 실행되므로 현재 작업 인스턴스를 참조하는 데 `this`를 사용할 수 있습니다.
  let verifier = [{ return ::verify(*this); }];
}
```

#### `build` 메소드 추가하기

기존 C++ 예제에서 마지막으로 누락된 구성 요소는 `build` 메소드입니다.
ODS는 몇 가지 간단한 빌드 방법으로 자동 생성할 수 있으며 이 경우 첫 번째 빌드 방법으로 생성합니다.
나머지는 [`builders`] (../../ OpDefinitions.md # custom-builder-methods) 필드를 정의합니다.
이 필드는 C++ 매개변수 리스트에 해당하는 문자열과 인라인 구현을 구체화하는 데 사용될 수 있는 선택적 코드 블록으로 된 `OpBuilder` 객체 리스트를 가져옵니다.

```tablegen
def ConstantOp : Toy_Op<"constant", [NoSideEffect]> {
  // 이 작업에 대한 요약 및 설명을 제공합니다.
  // 이것은 통용어 내 작업의 문서를 자동 생성하는 데 사용될 수 있습니다.
  let summary = "고정 작업";
  let description = [{
    고정 작업은 리터럴을 SSA 값으로 바꿉니다.
    데이터는 작업에 속성으로 추가됩니다. 예를 들어:

      %0 = "toy.constant"()
         { value = dense<[[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]> : tensor<2x3xf64> }
        : () -> tensor<2x3xf64>
  }];

  // 고정 작업은 속성을 유일한 입력으로 사용합니다.
  // `F64ElementsAttr`는 64비트 부동 소수점 ElementsAttr에 해당합니다.
  let arguments = (ins F64ElementsAttr:$value);

  // 일반 호출 작업은 단일값 TensorType을 리턴합니다.
  // F64Tensor는 64비트 부동 소수점 TensorType에 해당합니다.
  let results = (outs F64Tensor);

  // 고정 작업에 추가 검증 로직을 추가하십시오.
  // 여기서 우리는 C++ 소스 파일에서 정적 `verify` 메소드를 호출합니다.
  // 이 코드 블록은 ConstantOp::verify 내부에서 실행되므로 현재 작업 인스턴스를 참조하는 데 `this`를 사용할 수 있습니다.
  let verifier = [{ return ::verify(*this); }];

  // 고정 작업에 대한 사용자 지정 빌드 방법을 추가합니다.
  // 이 메소드는 MLIR으로 작업을 생성하기 위한 `builder.create<ConstantOp>(...)`를 사용할 때 쓰이는 `state`를 만듭니다.
  let builders = [
    // 주어진 고정 텐서 값으로 상수를 만듭니다.
    OpBuilder<"Builder *builder, OperationState &result, "
              "DenseElementsAttr value", [{
      // 자동으로 생성된 `build` 메소드를 호출합니다.
      build(builder, result, value.getType(), value);
    }]>,

    // 주어진 고정 부동 소수점 값으로 상수를 만듭니다.
    // 이 빌더는 주어진 매개변수로 `ConstantOp::build`에 대한 선언을 만듭니다.
    OpBuilder<"Builder *builder, OperationState &result, double value">
  ];
}
```

위에서는 ODS 프레임워크에서 작업을 정의하기 위한 몇 가지 개념을 소개하지만,
지역변수, 가변성 피연산자 등 정의할 수 없었던 많은 것들이 있습니다.
자세한 것은 [전체 설명서](../../OpDefinitions.md)을 확인하세요.

## Toy 예제 완성하기

이제는 "Toy IR"을 생성할 수 있습니다. 이전 예제를 간단하게 한 버전입니다:

```.toy
# 알 수 없는 변수형의 인자에서 작동하는 사용자 정의 일반 함수.
def multiply_transpose(a, b) {
  return transpose(a) * transpose(b);
}

def main() {
  var a<2, 3> = [[1, 2, 3], [4, 5, 6]];
  var b<2, 3> = [1, 2, 3, 4, 5, 6];
  var c = multiply_transpose(a, b);
  var d = multiply_transpose(b, a);
  print(d);
}
```

다음과 같은 IR 결과가 나타납니다:

```mlir
module {
  func @multiply_transpose(%arg0: tensor<*xf64>, %arg1: tensor<*xf64>) -> tensor<*xf64> {
    %0 = "toy.transpose"(%arg0) : (tensor<*xf64>) -> tensor<*xf64> loc("test/codegen.toy":5:10)
    %1 = "toy.transpose"(%arg1) : (tensor<*xf64>) -> tensor<*xf64> loc("test/codegen.toy":5:25)
    %2 = "toy.mul"(%0, %1) : (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64> loc("test/codegen.toy":5:25)
    "toy.return"(%2) : (tensor<*xf64>) -> () loc("test/codegen.toy":5:3)
  } loc("test/codegen.toy":4:1)
  func @main() {
    %0 = "toy.constant"() {value = dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>} : () -> tensor<2x3xf64> loc("test/codegen.toy":9:17)
    %1 = "toy.reshape"(%0) : (tensor<2x3xf64>) -> tensor<2x3xf64> loc("test/codegen.toy":9:3)
    %2 = "toy.constant"() {value = dense<[1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00]> : tensor<6xf64>} : () -> tensor<6xf64> loc("test/codegen.toy":10:17)
    %3 = "toy.reshape"(%2) : (tensor<6xf64>) -> tensor<2x3xf64> loc("test/codegen.toy":10:3)
    %4 = "toy.generic_call"(%1, %3) {callee = @multiply_transpose} : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64> loc("test/codegen.toy":11:11)
    %5 = "toy.generic_call"(%3, %1) {callee = @multiply_transpose} : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64> loc("test/codegen.toy":12:11)
    "toy.print"(%5) : (tensor<*xf64>) -> () loc("test/codegen.toy":13:3)
    "toy.return"() : () -> () loc("test/codegen.toy":8:1)
  } loc("test/codegen.toy":8:1)
} loc("test/codegen.toy":0:0)
```

`llvm-project/build/bin/toyc-ch2 llvm-project/llvm/projects/mlir/test/Examples/Toy/Ch2/codegen.toy -emit=mlir -mlir-print-debuginfo`를 통해 `toyc-ch2`를 빌드해 직접 실행해볼 수 있습니다.
`llvm-project/build/bin/toyc-ch2 llvm-project/llvm/projects/mlir/test/Examples/Toy/Ch2/codegen.toy -emit=mlir -mlir-print-debuginfo 2> codegen.mlir` 이후 `llvm-project/build/bin/toyc-ch2 codegen.mlir -emit=mlir`으로 왕복도 확인할 수 있습니다.
또한 최종 정의 파일에서 `llvm-project/build/bin/mlir-tblgen`을 사용하고 생성된 C++ 코드를 연구해야 합니다.

이제 MLIR은 Toy 통용어 및 작업에 대해 알고 있습니다.
[다음 장](Ch-3.md)에서는 새로운 언어를 사용하여 Toy 언어에 대한 고수준 언어별 분석 및 변환을 구현할 것입니다.
