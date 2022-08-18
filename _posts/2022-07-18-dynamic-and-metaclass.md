---
title:  "python 동적 클래스 생성과 Metaclass"
excerpt: "object와 type"

categories:
  - Python
tags:
  - Python
date: 2022-07-18 15:37 +0900
last_modified_at: 2022-08-18 21:02 +0900
published: true
---

## python 동적 클래스 생성과 Metaclass



> Python 3.9 기준으로 작성되었습니다.

회사에서 python으로 특정 도메인과 관련된 분석 프로그램을 개발하고 있습니다. 각 대상에 대한 속성값을 관리해야 하는데, 대상의 종류(타입)가 다양하고, 각 종류별로 다수의 속성값이 존재하고, 각 속성별로 적용 가능한 value가 정해져 있습니다.

쉽게 말하면 각 타입별로 Enum 그룹을 관리해야 합니다. Dictionary로 관리할 수도 있지만, 타입도 많고 관리해야할 속성의 수도 많고, 개발 단계에서 변경이 잦다보니(ㅠㅠ..), 변경된 속성을 제대로 반영하지 못하거나 처리해야할 값이 누락되는 등의 버그가 발생하기 쉽습니다. 또한, 함수의 파라미터로 넘겨줄 때 타입 체크를 하지 못하는 문제도 있습니다.



**TODO:** 

따라서, **각 대상 별 속성을 관리하기 위한 Class를 작성**하기로합니다. 요구 사항은 다음과 같습니다.

- 타입 체크 가능
- subscript operator 사용
- 속성 값 직접 접근 가능
- 코드에서 속성 리스트 직관적으로 확인 가능
- 속성 리스트 빠른 수정 가능
- 잘못된 속성에 접근하거나 값을 넣을 때 적절한 Exception 발생
- 동적 타입 생성 가능



_Enum_ 클래스의 경우 위에서 언급한 대부분의 기능을 제공하고 있습니다.

- 타입체크 가능
- 동적 타입 생성
- 에러 처리
- 직관적인 코드 
- ...



Enum은 [Functional API](https://docs.python.org/ko/3/library/enum.html#functional-api)를 제공하는 callable이라, 아래 코드와 같이 동적으로 타입 생성이 가능합니다. 또한, instance의 value를 직접 읽을 수 있고, 타입 체크도 가능합니다.

```python
from enum import Enum

Color = Enum('Color', 'RED, GREEN, BLUE')

color_red = Color.RED
color_green = Color(2)
color_blue = Color["BLUE"]

print(f"type of Color is {type(Color)}")
print(f"type of color_red is {type(color_red)}")

```



위 코드를 실행하면 아래와 같은 결과가 나옵니다.

> Color.BLUE <br>
1 <br>
GREEN <br>
type of Color is <class 'enum.EnumMeta'> <br>
type of color\_red is <enum 'Color'>



Functional API를 이용해서 Color 타입을 만들고,  3가지 방법으로 Color 인스턴스를 만들어 보았습니다. 참고로  Enum은 별도의 값을 지정해주지 않는 경우 첫번째 아이템부터 순서대로 1, 2, 3, ... 의 값이 부여됩니다. 그런데 Color의 타입은 Enum이 아니라 **EnumMeta**네요. Enum의 **[MetaClass](https://docs.python.org/3.9/reference/datamodel.html#metaclasses)**가 _EnumMeta_라서 그렇습니다.  Enum과 Enum메타의 상속관계는 어떻게 될까요?



_enum.py_

```python
# enum.py
# ...
class EnumMeta(type):
# ...
class Enum(metaclass=EnumMeta):
# ...

```



enum.py 코드를 살펴보면(vscode 등의 에디터 - 또는 IDE - 에서 Cmd 또는 Ctrl 키를 누른 상태로 Enum 클래스를 클릭하면 해당 코드가 보입니다.), EnumMeta는 type을 상속받고, Enum은 metaclass로 EnumMeta를 지정 해주었네요. type은 뭐고 metaclass는 무엇일까요?



### type과 metaclass



한마디로 정리하면, metaclass는 class의 class이고, type은 모든 class의 metaclass입니다. 그러면 class의 class라는건 무슨 뜻일까요??



Python은 모든게 객체(object)입니다. 우리가 쉽게 사용하는 변수도 객체고, 클래스도 객체고, 함수도 객체입니다. 그렇다면 이제 객체(objct)라는건 무슨 뜻일까요...???



![black-man-wait-what-meme](https://user-images.githubusercontent.com/19154301/185392033-c9066b90-9e18-4c01-8160-bd3a77359fdf.jpg){:width="50%" height="50%"}



파이썬은 객체지향프로그래밍 언어입니다. 언어마다 무엇이 객체이고, 인스턴스이고, 클래스인지 조금씩 다르게 정의하는데, 파이썬은 **모든것이 객체**로 이루어져 있고, 모든 객체는 타입을 가지고 있습니다. 그렇다면 타입을 가지고 있다는건 무슨 뜻일까요............????  타입을 가지고 있다는건, 해당 객체가 특정 클래스의 인스턴스라는 뜻입니다. 파이썬은 `type()` 함수를 사용해서 객체의 타입을 확인할 수 있습니다.



```python
integer = 10
type_of_integer = type(integer)
type_of_type_of_integer = type(type_of_integer)
type_of_type_of_type_of_integer = type(type_of_type_of_integer)

print(f"integer is instance of {type_of_integer}")
print(f"type_of_integer is instance of {type_of_type_of_integer}")
print(f"type_of_type_of_integer is instance of {type_of_type_of_type_of_integer}")

```



> 결과
>
> ```
> integer is instance of <class 'int'>
> type_of_integer is instance of <class 'type'>
> type_of_type_of_integer is instance of <class 'type'>
>
> ```



결과를 순서대로 살펴보면 다음과 같습니다.

> integer는 `int` 의 인스턴스입니다. <br>
> `int` 는 `type` 의 인스턴스입니다. <br>
> `type` 은 `type` 의 인스턴스입니다.

int는 class이자 타입이 `type` 인 객체이고, `type` 은 타입이 `type` 인 객체이자 class입니다. 왜 `type` 의 타입은 `type`  일까요 .........????? 위에서 언급한대로, `type` 은 모든 class의 metaclass이기 때문입니다. 결과적으로 클래스 `type` 은 `type` 의 인스턴스이고, `type` 의 타입은 `type` 입니다.

```
>>> type(type)
<class 'type'>
>>> isinstance(type, type)
True
```



#### Callable?

그런데 위 코드에서 객체의 타입을 확인하는 용도로 `type()` 함수를 사용했습니다. 그러면 `type` 은  함수일까요? class 일까요? 

결론만 이야기하면 Python의 경우 해당 객체의 타입(=함수인지, 클래스인지, ... )보다는 [호출할 수 있는지( = 객체에 () operator를 사용)](https://docs.python.org/3.9/reference/datamodel.html?highlight=__call__#object.__call__ "https://docs.python.org/3.9/reference/datamodel.html?highlight=__call__#object.__call__"), 어떤 파라미터를 넘겼을때 어떤 동작을 하는지가 중요합니다. 

호출할 수 있는 객체를 Callable이라고 부르고, `callable()` 함수로 확인할 수 있습니다.  참고로, callable의 타입은 `builtin_function_or_method` 입니다.

```
>>> callable(type)
True
>>> type(callable)
<class 'builtin_function_or_method'>
```


그러면 어떤 객체가 Callable인걸까요?????? 

정답은 `__call__()` 함수가 정의된 객체입니다. 따라서 호출 가능한 객체를 만드려면 `__call__()` 함수를 정의하면 됩니다.



[_https://docs.python.org/3.9/reference/datamodel.html?highlight=\_\_call\_\_#emulating-callable-objects_](https://docs.python.org/3.9/reference/datamodel.html?highlight=__call__#emulating-callable-objects)

> `object.``__call__`(_self_\[, _args..._\])
>
> Called when the instance is “called” as a function; if this method is defined, `x(arg1,arg2, ...)` roughly translates to `type(x).__call__(x, arg1, ...)`.



그런데 함수 말고도 `()` 연산자를 사용하는 객체가 있습니다. 바로 **class**입니다. 그렇다면 class도 callalbe일까요?

```
>>> class MyClass:
...     pass
...
>>> callable(MyClass)
True

```

네. MyClass도 callable입니다. MyClass를 호출하면 MyClass instance를 반환 해 줍니다. 이제 함수와 class가 비슷하게 보이지 않나요? class는 호출했을때 특정 타입의 객체를 반환해주는 callable object입니다. 여기서 주의할 점은, MyClass를 call했을 때 실행되는 `⁠__call__()` 함수는 MyClass 자신이 아닌, MyClass의 타입(클래스)에 의해 결정됩니다.



```
class MyClass:
    def __init__(self, name) -> None:
        self.name = name

    def __call__(self):
        print(f"My name is {self.name}")

my_instance1 = MyClass("instance1")
my_instance2 = MyClass("instance2")

my_instance1()
my_instance2()

```



결과:

> My name is instance1

> My name is instance2



위 코드에서는 명시적으로 4번 객체를 call했습니다. MyClass의 instance를 만들 때 MyClass를 두 번 호출했고, my\_instance1과 my\_instance2를 각각 한 번씩 호출했습니다. 위 예제에서 보는것처럼, MyClass에 정의된 `⁠__call__()` 은 해당 클래스의 인스턴스를 호출할 때 실행됩니다. 따라서, **MyClass를 호출할 때 실행되는 함수는 MyClass의 클래스인 `type` 에 정의되어 있습니다.**



지금까지 언급한 내용을 정리하면 다음과 같습니다.

- Python은 모든것이 객체
- 모든것이 객체이므로 변수, 함수, class도 객체
- 모든 객체는 class의 instance

- metalcass는 class의 class (= class는 metaclass의 instance - 주의할 점은, class는 metaclass를 상속받은게 아닙니다. 따라서 특정 class의 instance와 해당 class는 타입이 다릅니다.)
- type은 모든 class의 metaclass



### object와 class

`type` 이 모든 class의 _metaclass_라고 설명해드렸는데, 유사하게 `object` 는 모든 class의 superclass입니다. class를 정의할때 `obejct` 를 명시적으로 상속하지 않아도 자동적으로 상속됩니다. 따라서, 모든 파이썬 객체는 `object` 의 instance입니다. 위에서 설명드린 내용을 다 이해하셨다면, 아래 코드가 잘 이해되실거라 믿습니다 :). 

```
>>> import math
>>>
>>> class MyClass:
...     pass
...
>>> def function():
...     return
...
>>> issubclass(type, object)
True
>>>
>>> isinstance(type, type)
True
>>> isinstance(object, object)
True
>>> isinstance(type, object)
True
>>> isinstance(object, type)
True
>>>
>>> isinstance(MyClass, object)
True
>>> isinstance(function, object)
True
>>> isinstance(math, object)
True

```

참고로, 위에서 보여드린것처럼 모듈 또한 `object`  입니다.

```
>>> type(math)
<class 'module'>
>>> type(type(math))
<class 'type'>

```



### 결국 모든 것은 object

Python의 구성 요소는 결국 전부 객체입니다. 함수는 호출할 수 있는(callalbe) 객체이고, 클래스는 `type` 의 instance인 객체입니다. 함수와 클래스가 서로 완전히 다른것이 아니라는것을 이해하고 있는것이 중요합니다.



위 글의 내용을 요약하자면 다음과 같습니다.

- Python은 모든것이 객체

- **metalcass**는 class의 class(또는 type) → class는 metaclass의 instance

- 모든 **class**는 `type` 의 인스턴스
- 모든 **class**는 `object` 의 subclass → 모든 객체(object)는 `object` 의 instance



다음에는 MetaClass에 대해서 더 자세히 알아보겠습니다.