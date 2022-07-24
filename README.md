<img src="logo.png" align="left" style="width:128px; margin-right: 20px;" />

# py-automapper

**Version**
1.0.2

**Author**
anikolaienko

**Copyright**
anikolaienko

**License**
The MIT License (MIT)

**Last updated**
24 Jul 2022

**Package Download**
https://pypi.python.org/pypi/py-automapper

**Build Status**
TODO

---

Table of Contents:
- [py-automapper](#py-automapper)
- [Versions](#versions)
- [About](#about)
- [Usage](#usage)
  - [Different field names](#different-field-names)
  - [Overwrite field value in mapping](#overwrite-field-value-in-mapping)
  - [Extensions](#extensions)

# Versions
Check [CHANGELOG.md](/CHANGELOG.md)

# About

**Python auto mapper** is useful for multilayer architecture which requires constant mapping between objects from separate layers (data layer, presentation layer, etc).

Inspired by: [object-mapper](https://github.com/marazt/object-mapper)

The major advantage of py-automapper is its extensibility, that allows it to map practically any type, discover custom class fields and customize mapping rules. Read more in [documentation](https://anikolaienko.github.io/py-automapper).

# Usage
Install package:
```bash
pip install py-automapper
```

Let's say we have domain model `UserInfo` and its API representation `PublicUserInfo` with `age` field missing:
```python
class UserInfo:
    def __init__(self, name: str, age: int, profession: str):
        self.name = name
        self.age = age
        self.profession = profession

class PublicUserInfo:
    def __init__(self, name: str, profession: str):
        self.name = name
        self.profession = profession

user_info = UserInfo("John Malkovich", 35, "engineer")
```
To create `PublicUserInfo` object:
```python
from automapper import mapper

public_user_info = mapper.to(PublicUserInfo).map(user_info)

print(vars(public_user_info))
# {'name': 'John Malkovich', 'profession': 'engineer'}
```
You can register which class should map to which first:
```python
# Register
mapper.add(UserInfo, PublicUserInfo)

public_user_info = mapper.map(user_info)

print(vars(public_user_info))
# {'name': 'John Malkovich', 'profession': 'engineer'}
```

## Different field names
If your target class field name is different from source class.
```python
class PublicUserInfo:
    def __init__(self, full_name: str, profession: str):
        self.full_name = full_name       # UserInfo has `name` instead
        self.profession = profession
```
Simple map:
```python
public_user_info = mapper.to(PublicUserInfo).map(user_info, fields_mapping={
    "full_name": user_info.name
})
```
Preregister and map. Source field should start with class name followed by period sign and field name:
```python
mapper.add(UserInfo, PublicUserInfo, fields_mapping={"full_name": "UserInfo.name"})
public_user_info = mapper.map(user_info)

print(vars(public_user_info))
# {'full_name': 'John Malkovich', 'profession': 'engineer'}
```

## Overwrite field value in mapping
Very easy if you want to field just have different value, you provide a new value:
```python
public_user_info = mapper.to(PublicUserInfo).map(user_info, fields_mapping={
    "full_name": "John Cusack"
})

print(vars(public_user_info))
# {'full_name': 'John Cusack', 'profession': 'engineer'}
```

## Extensions
`py-automapper` has few predefined extensions for mapping to classes for frameworks:
* [FastAPI](https://github.com/tiangolo/fastapi) and [Pydantic](https://github.com/samuelcolvin/pydantic)
* [TortoiseORM](https://github.com/tortoise/tortoise-orm)

When you first time import `mapper` from `automapper` it checks default extensions and if modules are found for these extensions, then they will be automatically loaded for default `mapper` object.

What does extension do? To know what fields in Target class are available for mapping `py-automapper` need to extract the list of these fields. There is no generic way to do that for all Python objects. For this purpose `py-automapper` uses extensions.

List of default extensions can be found in [/automapper/extensions](/automapper/extensions) folder. You can take a look how it's done for a class with `__init__` method or for Pydantic or TortoiseORM models.

You can create your own extension and register in `mapper`:
```python
from automapper import mapper

class TargetClass:
    def __init__(self, **kwargs):
        self.name = kwargs["name"]
        self.age = kwargs["age"]
    
    @staticmethod
    def get_fields(cls):
        return ["name", "age"]

source_obj = {"name": "Andrii", "age": 30}

try:
    # Map object
    target_obj = mapper.to(TargetClass).map(source_obj)
except Exception as e:
    print(f"Exception: {repr(e)}")
    # Output:
    # Exception: KeyError('name')

    # mapper could not find list of fields from BaseClass
    # let's register extension for class BaseClass and all inherited ones
    mapper.add_spec(TargetClass, TargetClass.get_fields)
    target_obj = mapper.to(TargetClass).map(source_obj)

    print(f"Name: {target_obj.name}; Age: {target_obj.age}")
```

You can also create your own clean Mapper without any extensions and define extension for very specific classes, e.g. if class accepts `kwargs` parameter in `__init__` method and you want to copy only specific fields. Next example is a bit complex but probably rarely will be needed:
```python
from typing import Type, TypeVar

from automapper import Mapper

# Create your own Mapper object without any predefined extensions
mapper = Mapper()

class TargetClass:
    def __init__(self, **kwargs):
        self.data = kwargs.copy()

    @classmethod
    def fields(cls):
        return ["name", "age", "profession"]

source_obj = {"name": "Andrii", "age": 30, "profession": None}

try:
    target_obj = mapper.to(TargetClass).map(source_obj)
except Exception as e:
    print(f"Exception: {repr(e)}")
    # Output:
    # Exception: MappingError("No spec function is added for base class of <class 'type'>")

# Instead of using base class, we define spec for all classes that have `fields` property
T = TypeVar("T")

def class_has_fields_property(target_cls: Type[T]) -> bool:
    return callable(getattr(target_cls, "fields", None))
    
mapper.add_spec(class_has_fields_property, lambda t: getattr(t, "fields")())

target_obj = mapper.to(TargetClass).map(source_obj)
print(f"Name: {target_obj.data['name']}; Age: {target_obj.data['age']}; Profession: {target_obj.data['profession']}")
# Output:
# Name: Andrii; Age: 30; Profession: None

# Skip `None` value
target_obj = mapper.to(TargetClass).map(source_obj, skip_none_values=True)
print(f"Name: {target_obj.data['name']}; Age: {target_obj.data['age']}; Has profession: {hasattr(target_obj, 'profession')}")
# Output:
# Name: Andrii; Age: 30; Has profession: False
```
