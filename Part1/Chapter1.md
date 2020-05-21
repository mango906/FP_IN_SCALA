# 챕터 1. 함수형 프로그래밍이란 무엇인가?

함수형 프로그래밍이란 프로그램을 **순수 함수** 들로만 구축한다는 것이다.

**순수 함수**란 **부수 효과** 없는 함수를 지칭한다.

간단하게 설명하면 어떤 함수에 동일한 인자를 주었을 때, **항상 같은 값을 리턴하는 함수**이다.

부수 효과는 다음과 같은 예가 있다.

* 변수를 수정한다.
* 자료구조를 제자리에서 수정한다.
* 객체의 필드를 설정한다.
* 예외를 던지거나 오류를 내면서 실행을 중단하.
* 콘솔에 출력하거나 사용자의 입력을 읽어들인다.
* 파일에 기록하거나 파일에서 읽어들인다.
* 화면에 그린다
* .... 기타 등등

[순수 함수란?](https://jeong-pro.tistory.com/23)

## 1.1 FP의 이점: 간단한 예제

### 1.1.1 부수 효과가 있는 프로그램

```scala
class Cafe{
  def buyCoffee(cc: CreditCard): Coffee = {
    val cup = new Coffee()	// 세미콜론은 없어도 된다. 한 블록 안의 문장들은 새 줄로 구분된다.
    cc.charge(cup.price)
    cup	// 별도의 리턴 구문은 필요없다. 마지막 줄이 자동으로 리턴된다.
  }
}
```

위 코드에서 `cc.chage(cup.price)` 가 부수 효과의 예이다. `cc.charge` 라는 신용카드 청구에는 신용카드 사와 접촉해서 거래를 승인하고, 대금을 청구하고, 만약 이가 성공한다면 참조를 위해 거래 기록을 영구적으로 기록하는 등의 작업이 필요할 것이다. 하지만 이 함수 자체는 단지 하나의 `Coffee` 객체를 돌려줄 뿐이고, 그 외의 동작은 모두 **부수적으로** (on the side) 일어난다.

위 함수에 대한 테스트 코드를 작성한다고 해보자. 테스트를 위해 신용카드 회사와 연결해서 카드 이용 대금을 청구할 수 없기 때문에 테스트 작성이 어렵다. (검사성(testability)가 부족하다.)

그렇다면 실제로 설계를 변경해 볼 필요가 있다. 실제 대금 결제를 위해 신용카드 회사와 연동하는 방법에 관한 지식을 `CreditCard` 에 집어넣는 것은 좋지않다. 또 결제에 관한 정보를 내부 시스템에 영속적으로 기록하는 방법에 대한 지식도 집어넣지 말아야 할 것이다. 

`CreditCard` 에서 이러한 부분을 삭제하고,  `Payments` 객체를 `buyCoffee` 에 전달한다면 코드의 모듈성과 검사성을 좀 더 높일 수 있다.

```scala
class Cafe{
  def buyCoffee(cc: CreditCard, p: Payments): Coffee = {
    val cup = new Coffee()
    p.charge(cc, cup.price)
    cup
  }
}
```

`p.charge(cc, cup.price)` 를 호출 할 때 여전히 부수 효과가 발생하지만, 적어도 검사성은 조금 높아졌다. `Payments` 를 하나의 인터페이스로 만들고 그 인터페이스의 Mock을 작성하면 검사를 수월하게 진행할 수 있다. 하지만 이 또한 이상적인 방법은 아니다. 

[모의(Mock) 이란?](http://www.incodom.kr/Mock)

예를 들어 `buyCoffee` 호출 이후에 어떤 내부 값이 있을 떄, 검사 과정에서 `p.charge` 를 통해 값이 적절히 변경되었는지 확인해야 한다.

검사 외에도 이와 같은 방법은 `buyCoffee` 함수를 재사용하기 어렵다는 문제가 있다. 예를 들어 커피를 열두 잔 주문한다고 하자. 그렇다면 loop를 돌려 `buyCoffee` 함수를 열두 번 호출하면 될것이다.  그렇다면 신용카드에는 열두 번의 청구가 발생할 것이고, 12번의 카드 수수료가 발생할 것이다. 이러한 문제는 특별한 논리(logic)를 갖춘 `buyCoffees` 라는 새로운 함수를 만들면 될 것이다. 지금 예제의 `buyCoffee` 함수는 아주 간단하므로 새 함수를 만드는 일이 어렵지 않지만, 그렇지 않을 경우 코드의 재사용성과 합성(composition) 능력에 해가 될 수 있다.

### 1.1.2 함수적 해법: 부수 효과의 제거

이에 대한 함수적 해법은 부수 효과들을 제거하고 `buyCoffee` 가 `Coffee` 뿐만 아니라 **청구건을 하나의 값으로 돌려주게** 하는 것이다. 

```scala
class Cafe{
  def buyCoffee(cc: CreditCard) : (Coffee, Charge) = {
    val cup = new Coffee()
    (cup, Charge(cc, cup.price))
  }
}

case class Charge(cc: CreditCard, amount: Double){
  def combine(other: Charge): Charge =
  	if(cc == other.cc)
  		Charge(cc, vamount + other.amount)
  	else 
  		throw new Exception("Can't combine charges to diffrent cards")
}
```

이로써 `buyCoffee` 함수는 `Coffee` 뿐만 아니라 `Charge` 도 리턴한다. 이제 `buyCoffee` 함수를 재사용해 `buyCoffees` 함수를 만들 수 있다.

```scala
class Cafe{
  def buyCoffee(cc: CreditCard): (Coffee, Charge) = ...
  
  def buyCoffess(cc: CreditCard, n: Int): (List[Coffee], Charge) = {
    val purchases = List[(Coffee, Charge)] = List.fill(n)(buyCoffee(cc))
    val (coffees, charges) =
    	purchase.unzip(coffees, charges.reduce((c1, c2) => c1.combine(c2))) 
  }
}
```

이제 `buyCoffee` 를 재사용하여 `buyCoffees` 를 정의하였으며, 두 함수 모두 `Payments` 인터페이스의 Mock을 정의하지 않고도 손쉽게 검사할 수 있다. 실제로 `Cafe` 는 이제 `Charge` 의 대금이 어떻게 처리되는지 전혀 알지 못한다.

`Charge` 를 입급(first-class) 값으로 만들면, 청구건들을 다루는 업무 논리(business logic)를 좀 더 쉽게 조립할 수 있다. 예를 들어 커피숍에서 몇 시간 일하면서 커피를 여러 번 주문했다고 할 때, 같은 카드에 대한 청구건들을 하나의 `List[Charge]` 로 취합하는 다음과 같은 함수를 작성할 수 있다.

```scala
def coalesce(charges: List[Charge]): List[Charge] = 
	charges.groupBy(_.cc).values.map(_.reduce(_ combine _)).toList
```

## 1.2 (순수)함수란 구체적으로 무엇인가?

입력 형식이 A이고 출력 형식이 B인 함수 `f: A=>B` 에서 b는 오직 a의 값에 의해서만 결정되고, 내부 또는 외부 공정의 상태 변경은 `f(a)` 의 결과를 계산하는데 아무런 영향을 주지 않는다. 즉 실제 함수는 주어진 입력으로 뭔가를 계산하는 것 외에는 프로그램의 실행에 어떠한 영향도 미치지 않는다. 이를 보고 함수에 부수효과가 없다고 말한다. 그런 함수를 좀 더 명시적으로 **순수함수** 라고 부른다.

순수함수의 간단한 예로는 더하기 함수가 있다. 더하기 함수는 정수 값 두개를 입력받고 정수 값 하나를 돌려준다. 주어진 임의의 두 정수에 대해 이 함수는 **항상 같은 값을 돌려주고, 그 외의 일은 전혀 일어나지 않는다.** 

순수 함수의 이러한 개념을 **참조 투명성**(referential transparency, RT) 이라는 개념을 이용해서 공식화할 수 있다. 참조 투명성은 함수가 아니라 **표현식**(expression)의 한 속성이다. 스칼라 해석기에 입력했을 때 답이 나오는 것이면 모두 표현식이다. 예를 들어 `2 + 3` 은 하나의 표현식이다. `2 + 3` 을 값 `5` 로 바꾸어도 프로그램의 의미는 전혀 바뀌지 않는다.

이것이 표현식의 참조 투명성이다. 즉, 임의의 프로그램에서 만일 어떤 표현식을 그 평가 결과로 바꾸어도 프로그램의 의미가 변하지 않는다면 그 표현식은 참조에 투명한 것이다.

