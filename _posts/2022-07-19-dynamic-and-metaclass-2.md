---
title:  "python 동적 클래스 생성과 Metaclass - 2"
excerpt: "metaclass 활용하기"

categories:
  - Python
tags:
  - Python
date: 2022-07-19 04:12 +0900
last_modified_at: 2022-08-18 21:19 +0900
published: true
---

## python 동적 클래스 생성과 Metaclass (2)


### why metaclass?

이전 글에서 class가 metaclass의 instance라는것을 알아보았습니다. 그러면 metaclass는 왜 필요하고 어떤 일을 하는걸까요?



Python 공식 문서 [Metaclasses](https://docs.python.org/3.9/reference/datamodel.html?highlight=metaclass#metaclasses) 항목에 다음과 같이 나와있습니다.



> #### 3.3.3.1.  Metaclasses
>
> ...
>
>
> **The class creation process can be customized** by passing the `metaclass` keyword argument in the class definition line, or by inheriting from an existing class that included such an argument. In the following example, both `MyClass` and `MySubclass` are instances of `Meta`:
>
> ```python
> class Meta(type):
>     pass
>
> class MyClass(metaclass=Meta):
>     pass
>
> class MySubclass(MyClass):
>     pass
> ```
>
>
> Any other keyword arguments that are specified in the class definition are passed through to all metaclass operations described below.
>
> When a class definition is executed, the following steps occur:
>
> - MRO entries are resolved;
>
> - the appropriate metaclass is determined;
>
> - the class namespace is prepared;
>
> - the class body is executed;
>
> - the class object is created.
>

> #### 3.3.3.7. Uses for metaclasses
>
> The potential uses for metaclasses are boundless. Some ideas that have been explored include enum, logging, interface checking, automatic delegation, automatic property creation, proxies, frameworks, and automatic resource locking/synchronization.



문서에는 메타클래스를 활용하면 클래스 생성 과정을 커스텀할 수 있다고 나와있는데, 쉽게 말하자면 클래스의 동작 자체를 변경할 수 있습니다. enum과 같은 특수한(일반적인 클래스와 다르게 동작하는) 클래스를 만들 수도 있고, 객체를 생성할 때 자동으로 속성을 만들거나 함수를 만드는 식의 동작도 가능합니다. 



> **metaclass는 상속이 아닙니다.**
>
> 위의 예제에서, MySubClass는 MyClass를 상속받았습니다. 따라서 MySubClass는 MyClass의 sub-class이고, Meta의 instance입니다(MyClass의 metaclass를 속성을 상속 받아서). 주의할 점은, MySubClass가 Meta의 인스턴스이고, MySubClass의 instance는 Meta의 인스턴스가 아닙니다.
>
> 지금 까지의 내용을 되돌아 보면, type(과 type의 subclass)의 instance는 class입니다. MySubClass의 인스터는 class가 아니기 때문에, 직관적으로 생각해봐도 MySubClass의 instance는 Meta(및 type)의 인스턴스가 아닙니다.





### class 동적으로 생성하기



```python
class First:
    class_name = "First"

    def __init__(self, name):
        self.name = name

    def func1(self):
        print(f"First({self.name})")

    pass

class Second(First):
    def func2(self):
        print(f"Second({self.name})")

first = First("object1")
second = Second("object2")

print(First.class_name)
first.func1()
second.func1()
second.func2()

```

_Result:_

> First <br>
> First(object1) <br>
> First(object2) <br>
> Second(object2)

위 코드는 일반적인 방법으로 클래스를 생성하는 코드입니다. First와 First를 상속받은 Second를 만들었습니다. 이제 type으로 직접 클래스를 만들어보겠습니다. python 문서 [Built-in Functions](https://docs.python.org/3.9/library/functions.html?highlight=type#type)에 `type()` 에 관한 내용이 있습니다.



> _class_ `type`(_object_)[¶](https://docs.python.org/3.9/library/functions.html?highlight=type#type "Permalink to this definition")
> _class_ `type`(_name_, _bases_, _dict_, _\*\*kwds_)
>
> With one argument, return the type of an _object_. The return value is a type object and generally the same object as returned by [`object.__class__`](https://docs.python.org/3.9/library/stdtypes.html#instance.__class__ "instance.__class__").
>
> The [`isinstance()`](https://docs.python.org/3.9/library/functions.html?highlight=type#isinstance "isinstance") built-in function is recommended for testing the type of an object, because it takes subclasses into account.
>
> With three arguments, return a new type object. This is essentially a dynamic form of the [`class`](https://docs.python.org/3.9/reference/compound_stmts.html#class) statement. The _name_ string is the class name and becomes the [`__name__`](https://docs.python.org/3.9/library/stdtypes.html#definition.__name__ "definition.__name__") attribute. The _bases_ tuple contains the base classes and becomes the [`__bases__`](https://docs.python.org/3.9/library/stdtypes.html#class.__bases__ "class.__bases__") attribute; if empty, [`object`](https://docs.python.org/3.9/library/functions.html?highlight=type#object "object"), the ultimate base of all classes, is added. The _dict_ dictionary contains attribute and method definitions for the class body; it may be copied or wrapped before becoming the [`__dict__`](https://docs.python.org/3.9/library/stdtypes.html#object.__dict__ "object.__dict__") attribute. The following two statements create identical [`type`](https://docs.python.org/3.9/library/functions.html?highlight=type#type "type") objects:
>
>
>
> ```python
> >>> class X:
> ...     a = 1
> ...
> >>> X = type('X', (), dict(a=1))
>
> ```
>
>
>
> See also [Type Objects](https://docs.python.org/3.9/library/stdtypes.html#bltin-type-objects).
>
> Keyword arguments provided to the three argument form are passed to the appropriate metaclass machinery (usually [`__init_subclass__()`](https://docs.python.org/3.9/reference/datamodel.html#object.__init_subclass__ "object.__init_subclass__")) in the same way that keywords in a class definition (besides _metaclass_) would.
>
> See also [Customizing class creation](https://docs.python.org/3.9/reference/datamodel.html#class-customization).
>
> _Changed in version 3.6:_ Subclasses of [`type`](https://docs.python.org/3.9/library/functions.html?highlight=type#type "type") which don’t override `type.__new__` may no longer use the one-argument form to get the type of an object.

오버로딩된 `type()` 은 파라미터에 따라 

1. _class_ `type`(_object_) → 파라미터로 받은 객체의 타입 리턴

2. _class_ `type`(_name_, _bases_, _dict_, _\*\*kwds_) → 새로운 `type` 객체(클래스) 리턴


으로 동작합니다.

1번은 우리가 객체의 타입을 알아보기 위해 사용한 함수고, 2번은 type의 인스턴스(Class)를 생성하기 위해 사용합니다. 각각 파라미터를 살펴보면 다음과 같습니다.

- `name` (str): 생성할 클래스의 이름 → 클래스의 `__name__` attribute

- `bases` (tuple): 상속받을 클래스 → 클래스의 `__bases__` attribute

- `dict` (dict): 클래스의 attribute와 method 정의 → 클래스의 `__dict__`  attribute

- `**kwds` : 클래스 정의 시 넘겨주는 keyword argument와 동일하게 동작



> **주의** : dict 파라미터에서 넘겨주는 변수들은 모두 생성되는 클래스의 class variable이 됩니다. instance variable은 `__init__()` 함수 내부에서 정의합니다.

> 주의2: 생성한 class의 이름은 `name` 파라미터에 의해 결정됩니다. 클래스 객체를 저장하는 변수의 이름과는 관련이 없습니다.





아래 예제는 위에서 정의한 First, Second와 동일한 attribute, method를 가지는 MyFirst, MySecond를 생성합니다.

```python
def my__init__(self, name):
    self.name = name

def my_func1(self):
    print(f"MyFirst({self.name})")

MyFirst = type(
    "MyFirst",
    (),
    {
        "__init__": my__init__,
        "func1": my_func1,
        "class_name": "MyFirst"
    }
)

my_first = MyFirst("object3")

print(MyFirst.class_name)
my_first.func1()

MySecond = type(
    "MySecond",
    (MyFirst,),
    {
        "func2": lambda self: print(f"MySecond({self.name})")
    }
)

my_second = MySecond("object4")
my_second.func1()
my_second.func2()

```

_Result:_ 

> MyFirst <br>
> MyFirst(object3) <br>
> MyFirst(object4) <br>
> MySecond(object4)

`dict` 파라미터에 넘겨준 dictionary는 해당 클래스(type object)의 속성(attribute)이 됩니다.  아래 이미지는 vscode의 watch에서 네 클래스를 확인한 이미지인데, 각 클래스의 attribute를 확인할 수 있습니다. 

<img width="503" alt="image" src="https://user-images.githubusercontent.com/19154301/185394613-31837a65-7f31-46d0-928c-2e5e57ad7b35.png">

MyFirst와 MySecond도 우리가 의도했던 attribute를 모두 가지고 있네요. 그리고, 부모로부터 attribute를 상속받은 경우, 실제로 같은 객체를 공유합니다. 위 이미지에서 `First.func1` 과 `Second.func1` , `MyFirst.func1` 과 `MySecond.func1` 은  각각 같은 객체임을 확인할 수 있습니다. `class_name` 변수도 마찬가지입니다.



```python
>>> MyFirst.class_name ="NewName"
>>> print(MySecond.class_name)
NewName
```



MyClass의 인스턴스인 my\_class는 어떨까요? `MyClass`가 가지고 있는 `func1`, `class_name` attribute는 `MyClass`의 인스턴스인 `my_class`도 모두 가지고 있습니다. 하지만 각 attribute 객체의 타입에 따라 인스턴스가 생성될때 일어나는 동작이 다릅니다.

- class variable

클래스가 가지고 있는  class variable은 클래스와 해당 클래스의 인스턴스가 동일한 객체를 공유합니다.

```python
>>> MyFirst.class_name is my_first.class_name is my_second.class_name
True

```



- method

<img width="702" alt="image 2" src="https://user-images.githubusercontent.com/19154301/185395072-8d18480d-aac5-42c3-ae45-4084f9a25976.png">

클래스가 가지고 있는 `function` 객체는 인스턴스가 생성될 때 _bound method_ (`method`)객체가 됩니다. 객체의 함수를 call하면 자동으로 첫 번째 파라미터로 해당 객체가 들어가는데(일반적으로 사용하는 self 파라미터, classmethod나 staticmethod가 아닌경우), 해당 함수가 특정 객체에 bound된 _bound method_라서 그렇습니다. 

- class method & static method

class method나 static method는 어떻게 만들 수 있을까요? built-in 함수 [classmethod()](https://docs.python.org/3.9/library/functions.html#classmethod) 및 [staticmethod()](https://docs.python.org/3.9/library/functions.html#staticmethod) 함수를 사용하면 됩니다. 

```python
MyClass = type(
    "MyClass",
    (),
    {
        "my_class_method": classmethod(lambda cls: print(f"my_class_method called {id(cls)}")),
        "my_static_method": staticmethod(lambda: print("my_static_method called"))
    }
)
MyClass.my_class_method()

my_object = MyClass()
my_object.my_class_method()
my_object.my_static_method()

```

_result:_

> my\_class\_method called 5292481504 <br>
> my\_class\_method called 5292481504 <br>
> my\_static\_method called



<img width="660" alt="image 3" src="https://user-images.githubusercontent.com/19154301/185395159-5c101891-cfde-4156-9733-48a55f49f8ad.png">

class method는 해당 클래스 객체에 bound된 bound method입니다. 실행 결과를 보면 두 함수 `MyClass.my_class_method()` 와 `my_object.my_class_method()` 가 동일한 객체(`MyClass`)에 bound되어 있지만, 두 함수가 동일한 객체는 아닙니다. static method인 my\_static\_method는 `MyClass` 와 `my_obejct` 가 동일한 객체를 공유합니다.



### metaclass 활용하기

이제 지금까지 살펴본 내용으로, 실제 metaclass를 활용해보겠습니다. 인스턴스 변수를 생성하면 자동으로 getter와 setter 함수를 만들어주는 class를 만들어 보겠습니다. 

아래 코드는 getter의 이름은 `get_<variable_name>` , setter의 이름은 `set_<variable_name>` 로 자동으로 getter와 setter를 만들어주는 클래스 코드입니다.



```python
from typing import Any

class AutoGetSet(type):
    def __call__(self, *args: Any, **kwds: Any) -> Any:
        instance = super().__call__(*args, **kwds)
        variables = list(instance.__dict__.keys())

        for k in variables:
            setattr(instance, 'get_'+k, lambda k=k: getattr(instance, k))
            setattr(instance, 'set_'+k, lambda value, k=k: setattr(instance, k, value))

        return instance

class MyClass(metaclass = AutoGetSet):
    def __init__(self, id, name) -> None:
        self.id = id
        self.name = name
        pass

    def __str__(self):
        return f"{self.name}({self.id})"

m = MyClass(id=5, name="myname")
print(m.get_id(), m.get_name())

m.set_id(10)
m.set_name("my new name")
print(m.get_id(), m.get_name())

```

_Result:_

> _5 myname_ <br>
> _10 my new name_

지난 글에서 class는 call했을 경우 해당 타입의 instance를 리턴해주는 callable 객체라고 설명했습니다. class를 생성할때 metaclass 파라미터를 설정하면, 해당 클래스는 metaclass의 instance가 됩니다. 따라서, MyClass를 call하면 AutoGetSet의 `__call__` 이 호출됩니다. metaclass( `AutoGetSet` )의 `__call__` 에서 인스턴스를 생성한 뒤, 각 instance variable마다 getter와 setter를 만들어주는 방식으로 해당 내용을 구현했습니다. 



> Metaclass 와 `__call__`
> 객체를 call 했을때 실행되는 `__call__` 함수는 해당 객체의 클래스에 정의되어있습니다. 객체 m을 call하면 MyClass의 `__call__` 이 실행되겠죠.
> 마찬가지로 MyClass를 call 했을 경우 `MyClass`의 class인 `AutoGetSet`의 `__call__`이 실행된다는 점을 꼭 알아야합니다. 
> MyClass에서 정의한  `__call__`이 실행되는게 아닙니다!



> **lambda & variable**
> lambda 내부에서 (lambda)외부에 정의된 변수에 접근하면 어떤 일이 발생할까요?
> 언어마다 lambda가 작동하는 방식이 다른데, 어떤 언어는 lambda가 생성되는 시점의 value가 사용되기도 하고, reference가 사용되기도 합니다. 하지만, 파이썬은 둘 다 아닙니다. 실제 변수의 reference가 아닌 name을 통해 값을 불러옵니다.
> 정확히 말하면, labmda가 호출될때 해당 lambda가 생성된 scope에서 name을 통해 해당 변수에 접근합니다.
>
> ```python
> def get_lambda():
>     a = 6
>     my_lambda = lambda x: a + x
>     a = 7
>     return my_lambda
>
> a = 10
> func = get_lambda()
> print(func(5))
>
> my_lambda2 = lambda: print(y)
> y = 10
> my_lambda2()
>
> ```
>
> 위 코드를 실행하면 결과가 어떻게 될까요? my\_lambda2의 경우 정의되지 않은 변수를 사용했는데 에러가 발생하지는 않을까요? 결론부터 말하면, 실행 결과는
>
> ```
> 12
> 10
> ```
>
> 입니다.
>
> python lambda의 경우 lambda가 정의될때 특정 변수에 접근하지 않습니다. 그냥 `y` 라는 변수에 접근해야한다는 내용만 저장하는거죠. 그래서 lambda가 생성된 후에도, 외부에서 변수 값을 변경하면(심지어 해당 변수에 다른 타입의 객체를 저장하면!!) 람다의 내용도 변경될 수 있습니다. 특히, 위 `AutoGetSet` 처럼 반복문에서 lambda를 생성하는 경우 항상 주의해야 합니다. 



### 마무리 

이번 글에서는 metaclass를 이용해 자동으로 getter와 setter함수를 만들어주는 class를 만들어보았습니다.

다음 글에서는 Enum이 어떤 방식으로 구현되어있는지 살펴보도록 하겠습니다~



reference

[https://peps.python.org/pep-3115/](https://peps.python.org/pep-3115/) <br>
[https://docs.python.org/3.9/reference/datamodel.html?highlight=metaclass#metaclasses](https://docs.python.org/3.9/reference/datamodel.html?highlight=metaclass#metaclasses)
