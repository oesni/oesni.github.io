---
title:  "python 동적 클래스 생성과 Metaclass - 3"
excerpt: "metaclass활용한 클래스 작성"

categories:
  - Python
tags:
  - Python
date: 2022-08-18 21:38 +0900
last_modified_at: 2022-08-18 21:38 +0900
published: true
---

## python 동적 클래스 생성과 Metaclass (3)



### Enum과 EnumMeta

Enum은 EnumMeta라는 metaclass를 통해 구현되어있습니다. 첫번째 글 [[python 동적 클래스 생성과 Metaclass]] 에서 살펴봤던 예제를 다시 보겠습니다.



```python
from enum import Enum

Color = Enum('Color', 'RED, GREEN, BLUE')

color_red = Color.RED
color_green = Color(2)
color_blue = Color["BLUE"]

print(f"type of Color is {type(Color)}")
print(f"type of color_red is {type(color_red)}")

```



_Result:_

> Color.BLUE <br>
> 1 <br>
> GREEN <br>
> type of Color is <class 'enum.EnumMeta'> <br>
> type of color\_red is <enum 'Color'>



`Enum`에서 제공하는 function api를 사용해서 `Color` class를 만들었고, `Color` class에 접근해서 해당하는 instance를 만들 수 있었습니다. 그런데 Enum을 호출해서 생성한 `Color`  의 타입이 `EnumMeta` 입니다. 

```python
>>> type(Enum) is type(Color)
True
```

`Enum`과 `Color` 의 타입이 동일하네요. 몇가지 특이점을 발견할 수 있습니다.

1. `Enum` 을 call했는데 `EnumMeta` 의 instance를 반환했다. ( 😳 ) 

2. `Enum` 과 `Color` 는 타입이 같으니까, 두 class를 call했을경우 같은 함수가 호출된다. ( `EnumClass.__call__` )




해당 내용이 어떻게 구현되어있는지 `EnumMeta` 의 `__call__` 을 살펴보겠습니다.

```python
class EnumMeta(type):
    # ...
    def __call__(cls, value, names=None, *, module=None, qualname=None, type=None, start=1):
        """
        Either returns an existing member, or creates a new enum class.

        This method is used both when an enum class is given a value to match
        to an enumeration member (i.e. Color(3)) and for the functional API
        (i.e. Color = Enum('Color', names='RED GREEN BLUE')).

        When used for the functional API:

        `value` will be the name of the new class.

        `names` should be either a string of white-space/comma delimited names
        (values will start at `start`), or an iterator/mapping of name, value pairs.

        `module` should be set to the module this class is being created in;
        if it is not set, an attempt to find that module will be made, but if
        it fails the class will not be picklable.

        `qualname` should be set to the actual location this class can be found
        at in its module; by default it is set to the global scope.  If this is
        not correct, unpickling will fail in some circumstances.

        `type`, if set, will be mixed in as the first base class.
        """
        if names is None:  # simple value lookup
            return cls.__new__(cls, value)
        # otherwise, functional API: we're creating a new Enum type
        return cls._create_(
                value,
                names,
                module=module,
                qualname=qualname,
                type=type,
                start=start,
                )

    def _create_(cls, class_name, names, *, module=None, qualname=None, type=None, start=1):
        """
        Convenience method to create a new Enum class.

        `names` can be:

        * A string containing member names, separated either with spaces or
          commas.  Values are incremented by 1 from `start`.
        * An iterable of member names.  Values are incremented by 1 from `start`.
        * An iterable of (member name, value) pairs.
        * A mapping of member name -> value pairs.
        """
        metacls = cls.__class__
        bases = (cls, ) if type is None else (type, cls)
        _, first_enum = cls._get_mixins_(cls, bases)
        classdict = metacls.__prepare__(class_name, bases)

        # special processing needed for names?
        if isinstance(names, str):
            names = names.replace(',', ' ').split()
        if isinstance(names, (tuple, list)) and names and isinstance(names[0], str):
            original_names, names = names, []
            last_values = []
            for count, name in enumerate(original_names):
                value = first_enum._generate_next_value_(name, start, count, last_values[:])
                last_values.append(value)
                names.append((name, value))

        # Here, names is either an iterable of (name, value) or a mapping.
        for item in names:
            if isinstance(item, str):
                member_name, member_value = item, names[item]
            else:
                member_name, member_value = item
            classdict[member_name] = member_value
        enum_class = metacls.__new__(metacls, class_name, bases, classdict)
```



코드를 살펴보면 `EnumMeta.__call__` 은 입력 파라미터에 따라 다른 타입의 객체를 리턴하게 구현되어있습니다. 

1. name이 None인 경우는 현재 class(`Enum`)의 인스턴스를 반환하고,  ( ex. `Color(2)` 으로 호출하는 경우)
2. `_create_` 를 호출하는 경우 metaclass ( `EnumMeta` )의 인스턴스를 반환하도록 구현되어 있습니다. (ex. `Enum('Color', 'RED, GREEN, BLUE')` 으로 호출하는 경우



파이썬의 유연성을 다시한번 확인할 수 있는 부분입니다! 같은 타입의 클래스(객체)가 서로 전혀 다른 클래스 처럼 보이기도 하고(동작하고), 하나의 클래스가 여러 타입의 인스턴스를 생성할 수도 있고, 객체가 생성된 후에 함수를 추가할 수도 있었습니다. 하지만 모두 코두의 불확실성을 증가시킬 수 있는 부분이니 명확한 설계가 필요해 보입니다.

위 구현 내용을 참고해서, [[python 동적 클래스 생성과 Metaclass]]에서 구현하려고 했던 클래스를 작성해보겠습니다.



> **method** vs **classmethod** vs **staticmethod** <br>
> method: 클래스에서 일반적으로 주로 사용하는 함수. 함수를 호출하면 해당 인스턴스가 첫 번째 파라미터로 들어갑니다. 클래스의 인스턴스를 통해 호출할 수 있습니다.<br>
> classmethod: 클래스 내부에서`@classmethod` decorator를 사용하여 정의할 수 있습니다. 함수를 호출하면 첫 번째 파라미터로 해당 클래스 객체가 들어갑니다. 인스턴스와 클래스를 통해 호출할 수 있습니다.<br>
> staticmethod: 클래스 내부에서 `@staticmethod` decorator를 사용하여 정의할 수 있습니다. 호출할때 자동으로 입력되는 파라미터가 없으며, 인스턴스와 클래스를 통해 호출할 수 있습니다.



> `__new__`  vs `__init__` <br>
> 일반적으로 클래스를 정의할 때 선언하는 `__init__` 함수는 객체를 초기화(**init**ialize) 할때 사용합니다. 주로 attribute를 생성하고, 값을 설정하는 역할을 합니다. `__init__` 함수의 경우 첫 번째 파라미터로 instance를 받습니다. 즉, `__init__` 이 호출되는 시점에서는 이미 object가 생성된 후 입니다.<br><br>
> `__new__` 는 실제로 instance(object)를 생성하는 static method입니다. `__new__` 함수는 객체를 생성하고 리턴합니다. instance생성 과정에서 `__new__` 가 인스턴스를 리턴하면, `__init__` 함수가 호출되고, 인스턴스를 리턴하지 않으면 `__init__` 이 호출되지 않습니다.<br> 공식 documentation에 의하면, immutable class( `int` , `str` , `tuple` )의 subclass가 인스턴스 생성 과정을 커스텀할 수 있게 하는게 주 목적이라고 합니다([참고](https://docs.python.org/3.9/reference/datamodel.html?highlight=__setattr__#object.__new__)). `__new__` 를 오버라이딩할 때, `super().__new__(cls, ...)` 호출해서 instance를 생성하면 됩니다.
> 글을 작성하면서 든 생각인데, `__new__` 함수를 오버라이딩해서 싱글톤 객체를 구현할 수 있지 않을까 싶네요.



### Attribute 클래스 구현

구현 요구 사항

구조

1. metaclass `Attributeclass`  및 `Attribute`  구현
2. Attribute를 호출해서 새로운 클래스 생성, 새로 생성된 클래스는 Attribute의 subclass이며, AttributeMeta의 instance임



기능

1. 클래스 생성 시  class 이름( `str` )과 속성 목록 (`dict`) 을 파라미터로 입력
2. instance 생성 시 파라미터는 `dict` 또는 `list` 가능
3. `dict` 로 instance를 생성하는 경우, 정의되지 않은 속성은 undefined value를 기본값으로 입력, `list` 로 instance생성 시 모든 속성에 대한 값을 입력해야 함. 잘못된 값이나 길이가 맞지 않을 경우 `ValueError`  발생

4. instance에서 속성에 이름으로 access 가능
5. dictionary-like로 구현 ( instance\[”key”\]로 속성에 접근 가능) 
6. key-value는 사전에 정의된 값만 사용 가능 → 잘못된 key 사용 시 `AttributeError` , 잘못된 value사용 시 `ValueError`  발생 ( `KeyError` 를 던져야하나 고민했지만 구현한 클래스 이름이 `Attribute` 라 `AttributeError` 를 사용했습니다 ㅎ)



```python
from typing import Any
from enum import Enum

class AttributeMeta(type):

    def __call__(cls, name=None, attr_dict=None):
        # create new Attribute type
        if type(attr_dict) is dict:
            metacls = cls.__class__
            bases = (Attribute,)
            attr_list = {k:Enum(k, '? '+' '.join(v)) for k,v in attr_dict.items()}
            classdict = {"_attrs":attr_list}

            attribute_class = metacls.__new__(metacls, name, bases, classdict)
            return attribute_class

        # create insatnce of Attribute
        instance = object.__new__(cls)
        attributes = cls._attrs
        for attr_name, enum in attributes.items():
            setattr(instance, attr_name, enum(1).name) #set as default value '?'

        # set attributes from attr_dict
        if type(name) is list:
            if len(name) != len(attributes):
                raise ValueError(f"input length({len(name)}) is not equl to {len(attributes)}")
            # assert len(name) == len(attributes)
            name = {k:v for k, v in zip(attributes.keys(), name)}

        if type(name) is dict:
            for k, v in name.items():
                if not k in attributes.keys():
                    raise AttributeError("Wrong attribute name '{0}' for {1}".format(k, cls.__name__))
                if not v in attributes[k]._member_names_:
                    raise ValueError("'{0}' is not assignable to {1}".format(k, v))
                setattr(instance, k, v)

        return instance


class Attribute(metaclass = AttributeMeta):

    def __getattribute__(self, name: str) -> Any:
        try:
            var = object.__getattribute__(self, name)
            return var
        except AttributeError:
            raise AttributeError("Wrong attribute name '{0}' for {1}".format(name, type(self).__name__))

    def __setattr__(self, name: str, value: Any) -> None:
        try:
            enum = self._attrs[name]
        except KeyError:
            raise AttributeError("Wrong attribute name '{0}' for {1}".format(name, type(self).__name__))

        if not value in enum._member_names_:
            raise ValueError("{0} is not assignable to {1}".format(value, name))
        else:
            super().__setattr__(name, value)

    def __getitem__(self, subscript):
        return getattr(self, subscript)

    def __setitem__(self, subscript, value):
        setattr(self, subscript, value)

    def __str__(self):
        to_str  = f"{type(self).__name__}({id(self)}\n"
        to_str += '\n'.join([f"{k}: {v}" for k, v in self.__dict__.items()])
        return to_str

if __name__ == "__main__":
    # class createion
    TestAttribute = Attribute(
        "TestAttribute",
        {
            'A1': ['y', 'n'],
            'A2': ['y', 'n']
        }
    )

    # intialize test
    attr = TestAttribute()
    attr2 = TestAttribute({
        'A1': 'n'
    })
    attr3 = TestAttribute(
        ['y', 'n']
    )

    # test get attribute
    assert(attr['A1'] is attr.A1)
    assert(attr['A2'] is attr.A2)
    assert(attr.A1 == '?')
    assert(attr.A2 == '?')
    assert(attr2.A1 == 'n')
    assert(attr2['A2'] == '?')
    assert(attr3.A1 == 'y')
    assert(attr3.A2 == 'n')

    # test set attribute
    attr.A1 = 'n'
    attr["A2"] = 'n'
    try:
        attr3 = TestAttribute({
            'A1': 'x'
        })
    except Exception as e:
        assert type(e) is ValueError

    try:
        attr3 = TestAttribute({
            'A3': 'y'
        })
    except Exception as e:
        assert type(e) is AttributeError

    try:
        attr3 = TestAttribute(['n'])
    except Exception as e:
        assert type(e) is ValueError

    try:
        attr3 = TestAttribute(['n', 'y', 'n'])
    except Exception as e:
        assert type(e) is ValueError

    try:
        attr3 = TestAttribute(['n', 'a'])
    except Exception as e:
        assert type(e) is ValueError

    try:
        attr.A1 = 'x'
    except Exception as e:
        assert type(e) is ValueError

    try:
        attr["A1"] = 'x'
    except Exception as e:
        assert type(e) is ValueError

    try:
        _ = attr.B
    except Exception as e:
        assert type(e) is AttributeError

    try:
        _ = attr["B"]
    except Exception as e:
        assert type(e) is AttributeError

    print("TEST PASSED!")

```



구현해야할 함수나 기능이 복잡하지 않아서 간단한 테스트 코드를 먼저 작성하고 TDD방식으로 코드를 작성했습니다.

`AttributeMeta`에 구현된 내용을 보면, 

- `Attribute` 를 호출 했을 때, `attr_dict` 파라미터가 `dict` 면 새로운 class를 생성합니다. 생성되는 class의 타입은 `AttributeMeta` 이며, `Attribute`의 subclass입니다.

    - `__new__` 함수를 호출해서 instance를 만든 후 리턴
    - `bases` 에 `Attribute`를 넣어 `Attribute` 를 상속

    - `attr_dict` 의 각 아이템은 enum형태로 만들어 생성하는 class의 class variable로 등록

        - enum 생성 시 undefined value를 뜻하는 ‘?’ 를 첫 번째 아이템으로 등록 

- `Attribute` (또는 subclass)를 호출 했을 때, 파라미터가 하나인 경우 instance를 생성합니다.


    - 우선 속성 목록을 기본값(’?’)로 초기화
    - 입력 파라미터 타입이 `list` 인 경우 `dict` 로 변경 후 

        - 속성 갯수와 입력된 리스트의 길이가 다를 경우 `ValueError`  에러 발생

    - 입력 파라미터 타입이 `dict` 인 경우 해당하는 속성에 값을 입력하고 instance 리턴

        - 속성 이름이나 값이 잘못된 경우 에러 발생 



`Attribute` 에 구현된 내용은,

- `__str__` 함수 구현

- `__setattr__` , `__getattr__`  구현 → 속성 이름과 값 검증. 클래스 생성 시 만든 enum을 검증에 사용 함.

- `__setitem__` , `__getitem__` 구현 → `__setattr__` 및 `__getattr__` 호출하도록 구현. 


입니다.



### 결론


구현한 내용을 보면 정해진 key와 value만 사용가능한 dictionary라고 봐도 될거같네요. <br>위 클래스의 구현 목적은 개발단계에서 관리 용이성(직관적 코드 작성, 빠른 변경 가능)과 safe-coding 이였는데, 두 목적다 만족할 것으로 보입니다. 속성을 관리하는 파이썬 파일을 따로 만들면 class생성 코드 자체를 설정 파일처럼 관리가 가능합니다.<br> 하지만, 모든 속성이 `str` 으로만 구현 되는 점, 다양하고 많은 속성을 관리하기 위해 작성한 class인데 자동 완성이 원활하게 작동하지 않는 다는 점 등 아직 개선할 사항이 있어보입니다.