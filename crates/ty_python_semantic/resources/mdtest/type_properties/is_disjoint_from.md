# Disjointness relation

Two types `S` and `T` are disjoint if their intersection `S & T` is empty (equivalent to `Never`).
This means that it is known that no possible runtime object inhabits both types simultaneously.

## Basic builtin types

```py
from typing_extensions import Literal, LiteralString, Any
from ty_extensions import Intersection, Not, TypeOf, is_disjoint_from, static_assert

static_assert(is_disjoint_from(bool, str))
static_assert(not is_disjoint_from(bool, bool))
static_assert(not is_disjoint_from(bool, int))
static_assert(not is_disjoint_from(bool, object))

static_assert(not is_disjoint_from(Any, bool))
static_assert(not is_disjoint_from(Any, Any))
static_assert(not is_disjoint_from(Any, Not[Any]))

static_assert(not is_disjoint_from(LiteralString, LiteralString))
static_assert(not is_disjoint_from(str, LiteralString))
static_assert(not is_disjoint_from(str, type))
static_assert(not is_disjoint_from(str, type[Any]))
```

## Class hierarchies

```py
from ty_extensions import is_disjoint_from, static_assert, Intersection, is_subtype_of
from typing import final

class A: ...
class B1(A): ...
class B2(A): ...

# B1 and B2 are subclasses of A, so they are not disjoint from A:
static_assert(not is_disjoint_from(A, B1))
static_assert(not is_disjoint_from(A, B2))

# The two subclasses B1 and B2 are also not disjoint ...
static_assert(not is_disjoint_from(B1, B2))

# ... because they could share a common subclass ...
class C(B1, B2): ...

# ... which lies in their intersection:
static_assert(is_subtype_of(C, Intersection[B1, B2]))

# However, if a class is marked final, it can not be subclassed ...
@final
class FinalSubclass(A): ...

static_assert(not is_disjoint_from(FinalSubclass, A))

# ... which makes it disjoint from B1, B2:
static_assert(is_disjoint_from(B1, FinalSubclass))
static_assert(is_disjoint_from(B2, FinalSubclass))

# Instance types can also be disjoint if they have disjoint metaclasses.
# No possible subclass of `Meta1` and `Meta2` could exist, therefore
# no possible subclass of `UsesMeta1` and `UsesMeta2` can exist:
class Meta1(type): ...
class UsesMeta1(metaclass=Meta1): ...

@final
class Meta2(type): ...

class UsesMeta2(metaclass=Meta2): ...

static_assert(is_disjoint_from(UsesMeta1, UsesMeta2))
```

## Tuple types

```py
from typing_extensions import Literal, Never
from ty_extensions import TypeOf, is_disjoint_from, static_assert

static_assert(is_disjoint_from(tuple[()], TypeOf[object]))
static_assert(is_disjoint_from(tuple[()], TypeOf[Literal]))

static_assert(is_disjoint_from(tuple[None], None))
static_assert(is_disjoint_from(tuple[None], Literal[b"a"]))
static_assert(is_disjoint_from(tuple[None], Literal["a"]))
static_assert(is_disjoint_from(tuple[None], Literal[1]))
static_assert(is_disjoint_from(tuple[None], Literal[True]))

static_assert(is_disjoint_from(tuple[Literal[1]], tuple[Literal[2]]))
static_assert(is_disjoint_from(tuple[Literal[1], Literal[2]], tuple[Literal[1]]))
static_assert(is_disjoint_from(tuple[Literal[1], Literal[2]], tuple[Literal[1], Literal[3]]))

static_assert(not is_disjoint_from(tuple[Literal[1], Literal[2]], tuple[Literal[1], int]))
static_assert(not is_disjoint_from(tuple[Literal[1], Literal[2]], tuple[int, ...]))

# TODO: should pass
static_assert(is_disjoint_from(tuple[int, int], tuple[None, ...]))  # error: [static-assert-error]
```

## Unions

```py
from typing_extensions import Literal
from ty_extensions import Intersection, is_disjoint_from, static_assert

static_assert(is_disjoint_from(Literal[1, 2], Literal[3]))
static_assert(is_disjoint_from(Literal[1, 2], Literal[3, 4]))

static_assert(not is_disjoint_from(Literal[1, 2], Literal[2]))
static_assert(not is_disjoint_from(Literal[1, 2], Literal[2, 3]))
```

## Intersections

```py
from typing_extensions import Literal, final, Any
from ty_extensions import Intersection, is_disjoint_from, static_assert, Not

@final
class P: ...

@final
class Q: ...

@final
class R: ...

# For three pairwise disjoint classes ...
static_assert(is_disjoint_from(P, Q))
static_assert(is_disjoint_from(P, R))
static_assert(is_disjoint_from(Q, R))

# ... their intersections are also disjoint:
static_assert(is_disjoint_from(Intersection[P, Q], R))
static_assert(is_disjoint_from(Intersection[P, R], Q))
static_assert(is_disjoint_from(Intersection[Q, R], P))

# On the other hand, for non-disjoint classes ...
class X: ...
class Y: ...
class Z: ...

static_assert(not is_disjoint_from(X, Y))
static_assert(not is_disjoint_from(X, Z))
static_assert(not is_disjoint_from(Y, Z))

# ... their intersections are also not disjoint:
static_assert(not is_disjoint_from(Intersection[X, Y], Z))
static_assert(not is_disjoint_from(Intersection[X, Z], Y))
static_assert(not is_disjoint_from(Intersection[Y, Z], X))

# If one side has a positive fully-static element and the other side has a negative of that element, they are disjoint
static_assert(is_disjoint_from(int, Not[int]))
static_assert(is_disjoint_from(Intersection[X, Y, Not[Z]], Intersection[X, Z]))
static_assert(is_disjoint_from(Intersection[X, Not[Literal[1]]], Literal[1]))

class Parent: ...
class Child(Parent): ...

static_assert(not is_disjoint_from(Parent, Child))
static_assert(not is_disjoint_from(Parent, Not[Child]))
static_assert(not is_disjoint_from(Not[Parent], Not[Child]))
static_assert(is_disjoint_from(Not[Parent], Child))
static_assert(is_disjoint_from(Intersection[X, Not[Parent]], Child))
static_assert(is_disjoint_from(Intersection[X, Not[Parent]], Intersection[X, Child]))

static_assert(not is_disjoint_from(Intersection[Any, X], Intersection[Any, Not[Y]]))
static_assert(not is_disjoint_from(Intersection[Any, Not[Y]], Intersection[Any, X]))

static_assert(is_disjoint_from(Intersection[int, Any], Not[int]))
static_assert(is_disjoint_from(Not[int], Intersection[int, Any]))
```

## Special types

### `Never`

`Never` is disjoint from every type, including itself.

```py
from typing_extensions import Never
from ty_extensions import is_disjoint_from, static_assert

static_assert(is_disjoint_from(Never, Never))
static_assert(is_disjoint_from(Never, None))
static_assert(is_disjoint_from(Never, int))
static_assert(is_disjoint_from(Never, object))
```

### `None`

```py
from typing_extensions import Literal, LiteralString
from ty_extensions import is_disjoint_from, static_assert, Intersection, Not

static_assert(is_disjoint_from(None, Literal[True]))
static_assert(is_disjoint_from(None, Literal[1]))
static_assert(is_disjoint_from(None, Literal["test"]))
static_assert(is_disjoint_from(None, Literal[b"test"]))
static_assert(is_disjoint_from(None, LiteralString))
static_assert(is_disjoint_from(None, int))
static_assert(is_disjoint_from(None, type[object]))

static_assert(not is_disjoint_from(None, None))
static_assert(not is_disjoint_from(None, int | None))
static_assert(not is_disjoint_from(None, object))

static_assert(is_disjoint_from(Intersection[int, Not[str]], None))
static_assert(is_disjoint_from(None, Intersection[int, Not[str]]))
```

### Literals

```py
from typing_extensions import Literal, LiteralString
from ty_extensions import Intersection, Not, TypeOf, is_disjoint_from, static_assert, AlwaysFalsy, AlwaysTruthy

static_assert(is_disjoint_from(Literal[True], Literal[False]))
static_assert(is_disjoint_from(Literal[True], Literal[1]))
static_assert(is_disjoint_from(Literal[False], Literal[0]))

static_assert(is_disjoint_from(Literal[1], Literal[2]))

static_assert(is_disjoint_from(Literal["a"], Literal["b"]))

static_assert(is_disjoint_from(Literal[b"a"], LiteralString))
static_assert(is_disjoint_from(Literal[b"a"], Literal[b"b"]))
static_assert(is_disjoint_from(Literal[b"a"], Literal["a"]))

static_assert(is_disjoint_from(type[object], TypeOf[Literal]))
static_assert(is_disjoint_from(type[str], LiteralString))

static_assert(not is_disjoint_from(Literal[True], Literal[True]))
static_assert(not is_disjoint_from(Literal[False], Literal[False]))
static_assert(not is_disjoint_from(Literal[True], bool))
static_assert(not is_disjoint_from(Literal[True], int))

static_assert(not is_disjoint_from(Literal[1], Literal[1]))

static_assert(not is_disjoint_from(Literal["a"], Literal["a"]))
static_assert(not is_disjoint_from(Literal["a"], LiteralString))
static_assert(not is_disjoint_from(Literal["a"], str))

# TODO: No errors
# error: [static-assert-error]
static_assert(is_disjoint_from(AlwaysFalsy, Intersection[LiteralString, Not[Literal[""]]]))
# error: [static-assert-error]
static_assert(is_disjoint_from(Intersection[Not[Literal[True]], Not[Literal[False]]], bool))
# error: [static-assert-error]
static_assert(is_disjoint_from(Intersection[AlwaysFalsy, Not[Literal[False]]], bool))
# error: [static-assert-error]
static_assert(is_disjoint_from(Intersection[AlwaysTruthy, Not[Literal[True]]], bool))

# TODO: No errors
# The condition `is_disjoint(T, Not[T])` must still be satisfied after the following transformations:
# `LiteralString & AlwaysTruthy` -> `LiteralString & ~Literal[""]`
# error: [static-assert-error]
static_assert(is_disjoint_from(Intersection[LiteralString, AlwaysTruthy], Not[LiteralString] | AlwaysFalsy))
# `LiteralString & ~AlwaysFalsy`  -> `LiteralString & ~Literal[""]`
# error: [static-assert-error]
static_assert(is_disjoint_from(Intersection[LiteralString, Not[AlwaysFalsy]], Not[LiteralString] | AlwaysFalsy))
```

### Class, module and function literals

```toml
[environment]
python-version = "3.12"
```

```py
from types import ModuleType, FunctionType
from ty_extensions import TypeOf, is_disjoint_from, static_assert

class A: ...
class B: ...

type LiteralA = TypeOf[A]
type LiteralB = TypeOf[B]

# Class literals for different classes are always disjoint.
# They are singleton types that only contain the class object itself.
static_assert(is_disjoint_from(LiteralA, LiteralB))

# The class A is a subclass of A, so A is not disjoint from type[A]:
static_assert(not is_disjoint_from(LiteralA, type[A]))

# The class A is disjoint from type[B] because it's not a subclass of B:
static_assert(is_disjoint_from(LiteralA, type[B]))

# However, type[A] is not disjoint from type[B], as there could be
# classes that inherit from both A and B:
static_assert(not is_disjoint_from(type[A], type[B]))

import random
import math

static_assert(is_disjoint_from(TypeOf[random], TypeOf[math]))
static_assert(not is_disjoint_from(TypeOf[random], ModuleType))
static_assert(not is_disjoint_from(TypeOf[random], object))

def f(): ...
def g(): ...

static_assert(is_disjoint_from(TypeOf[f], TypeOf[g]))
static_assert(not is_disjoint_from(TypeOf[f], FunctionType))
static_assert(not is_disjoint_from(TypeOf[f], object))
```

### `AlwaysTruthy` and `AlwaysFalsy`

```py
from ty_extensions import AlwaysFalsy, AlwaysTruthy, is_disjoint_from, static_assert
from typing import Literal

static_assert(is_disjoint_from(None, AlwaysTruthy))
static_assert(not is_disjoint_from(None, AlwaysFalsy))

static_assert(is_disjoint_from(AlwaysFalsy, AlwaysTruthy))
static_assert(not is_disjoint_from(str, AlwaysFalsy))
static_assert(not is_disjoint_from(str, AlwaysTruthy))

static_assert(is_disjoint_from(Literal[1, 2], AlwaysFalsy))
static_assert(not is_disjoint_from(Literal[0, 1], AlwaysTruthy))
```

### Instance types versus `type[T]` types

An instance type is disjoint from a `type[T]` type if the instance type is `@final` and the class of
the instance type is not a subclass of `T`'s metaclass.

```py
from typing import final
from ty_extensions import is_disjoint_from, static_assert

@final
class Foo: ...

static_assert(is_disjoint_from(Foo, type[int]))
static_assert(is_disjoint_from(type[object], Foo))
static_assert(is_disjoint_from(type[dict], Foo))

# Instance types can be disjoint from `type[]` types
# even if the instance type is a subtype of `type`

@final
class Meta1(type): ...

class UsesMeta1(metaclass=Meta1): ...

static_assert(not is_disjoint_from(Meta1, type[UsesMeta1]))

class Meta2(type): ...
class UsesMeta2(metaclass=Meta2): ...

static_assert(not is_disjoint_from(Meta2, type[UsesMeta2]))
static_assert(is_disjoint_from(Meta1, type[UsesMeta2]))
```

### `type[T]` versus `type[S]`

By the same token, `type[T]` is disjoint from `type[S]` if `T` is `@final`, `S` is `@final`, or the
metaclass of `T` is disjoint from the metaclass of `S`.

```py
from typing import final
from ty_extensions import static_assert, is_disjoint_from

@final
class Meta1(type): ...

class Meta2(type): ...

static_assert(is_disjoint_from(type[Meta1], type[Meta2]))

class UsesMeta1(metaclass=Meta1): ...
class UsesMeta2(metaclass=Meta2): ...

static_assert(is_disjoint_from(type[UsesMeta1], type[UsesMeta2]))
```

### `property`

```py
from ty_extensions import is_disjoint_from, static_assert, TypeOf
from typing import final

class C:
    @property
    def prop(self) -> int:
        return 1

reveal_type(C.prop)  # revealed: property

@final
class D:
    pass

static_assert(not is_disjoint_from(int, TypeOf[C.prop]))
static_assert(not is_disjoint_from(TypeOf[C.prop], int))
static_assert(is_disjoint_from(TypeOf[C.prop], D))
static_assert(is_disjoint_from(D, TypeOf[C.prop]))
```

## Callables

No two callable types are disjoint because there exists a non-empty callable type
`(*args: object, **kwargs: object) -> Never` that is a subtype of all fully static callable types.
As such, for any two callable types, it is possible to conceive of a runtime callable object that
would inhabit both types simultaneously.

```py
from ty_extensions import CallableTypeOf, is_disjoint_from, static_assert
from typing_extensions import Callable, Literal, Never

def mixed(a: int, /, b: str, *args: int, c: int = 2, **kwargs: int) -> None: ...

static_assert(not is_disjoint_from(Callable[[], Never], CallableTypeOf[mixed]))
static_assert(not is_disjoint_from(Callable[[int, str], float], CallableTypeOf[mixed]))

# Using gradual form
static_assert(not is_disjoint_from(Callable[..., None], Callable[[], None]))
static_assert(not is_disjoint_from(Callable[..., None], Callable[..., None]))
static_assert(not is_disjoint_from(Callable[..., None], Callable[[Literal[1]], None]))

# Using `Never`
static_assert(not is_disjoint_from(Callable[[], Never], Callable[[], Never]))
static_assert(not is_disjoint_from(Callable[[Never], str], Callable[[Never], int]))
```

A callable type is disjoint from all literal types.

```py
from ty_extensions import CallableTypeOf, is_disjoint_from, static_assert
from typing_extensions import Callable, Literal, Never

static_assert(is_disjoint_from(Callable[[], None], Literal[""]))
static_assert(is_disjoint_from(Callable[[], None], Literal[b""]))
static_assert(is_disjoint_from(Callable[[], None], Literal[1]))
static_assert(is_disjoint_from(Callable[[], None], Literal[True]))
```

A callable type is disjoint from nominal instance types where the classes are final and whose
`__call__` is not callable.

```py
from ty_extensions import CallableTypeOf, is_disjoint_from, static_assert
from typing_extensions import Any, Callable, final

@final
class C: ...

static_assert(is_disjoint_from(bool, Callable[..., Any]))
static_assert(is_disjoint_from(C, Callable[..., Any]))
static_assert(is_disjoint_from(bool | C, Callable[..., Any]))

static_assert(is_disjoint_from(Callable[..., Any], bool))
static_assert(is_disjoint_from(Callable[..., Any], C))
static_assert(is_disjoint_from(Callable[..., Any], bool | C))

static_assert(not is_disjoint_from(str, Callable[..., Any]))
static_assert(not is_disjoint_from(bool | str, Callable[..., Any]))

static_assert(not is_disjoint_from(Callable[..., Any], str))
static_assert(not is_disjoint_from(Callable[..., Any], bool | str))

def bound_with_valid_type():
    @final
    class D:
        def __call__(self, *args: Any, **kwargs: Any) -> Any: ...

    static_assert(not is_disjoint_from(D, Callable[..., Any]))
    static_assert(not is_disjoint_from(Callable[..., Any], D))

def possibly_unbound_with_valid_type(flag: bool):
    @final
    class E:
        if flag:
            def __call__(self, *args: Any, **kwargs: Any) -> Any: ...

    static_assert(not is_disjoint_from(E, Callable[..., Any]))
    static_assert(not is_disjoint_from(Callable[..., Any], E))

def bound_with_invalid_type():
    @final
    class F:
        __call__: int = 1

    static_assert(is_disjoint_from(F, Callable[..., Any]))
    static_assert(is_disjoint_from(Callable[..., Any], F))

def possibly_unbound_with_invalid_type(flag: bool):
    @final
    class G:
        if flag:
            __call__: int = 1

    static_assert(is_disjoint_from(G, Callable[..., Any]))
    static_assert(is_disjoint_from(Callable[..., Any], G))
```

A callable type is disjoint from special form types, except for callable special forms.

```py
from ty_extensions import is_disjoint_from, static_assert, TypeOf
from typing_extensions import Any, Callable, TypedDict
from typing import Literal, Union, Optional, Final, Type, ChainMap, Counter, OrderedDict, DefaultDict, Deque

# Most special forms are disjoint from callable types because they are
# type constructors/annotations that are subscripted, not called.
static_assert(is_disjoint_from(Callable[..., Any], TypeOf[Literal]))
static_assert(is_disjoint_from(TypeOf[Literal], Callable[..., Any]))

static_assert(is_disjoint_from(Callable[[], None], TypeOf[Union]))
static_assert(is_disjoint_from(TypeOf[Union], Callable[[], None]))

static_assert(is_disjoint_from(Callable[[int], str], TypeOf[Optional]))
static_assert(is_disjoint_from(TypeOf[Optional], Callable[[int], str]))

static_assert(is_disjoint_from(Callable[..., Any], TypeOf[Type]))
static_assert(is_disjoint_from(TypeOf[Type], Callable[..., Any]))

static_assert(is_disjoint_from(Callable[..., Any], TypeOf[Final]))
static_assert(is_disjoint_from(TypeOf[Final], Callable[..., Any]))

static_assert(is_disjoint_from(Callable[..., Any], TypeOf[Callable]))
static_assert(is_disjoint_from(TypeOf[Callable], Callable[..., Any]))

# However, some special forms are callable (TypedDict and collection constructors)
static_assert(not is_disjoint_from(Callable[..., Any], TypeOf[TypedDict]))
static_assert(not is_disjoint_from(TypeOf[TypedDict], Callable[..., Any]))

static_assert(not is_disjoint_from(Callable[..., Any], TypeOf[ChainMap]))
static_assert(not is_disjoint_from(TypeOf[ChainMap], Callable[..., Any]))

static_assert(not is_disjoint_from(Callable[..., Any], TypeOf[Counter]))
static_assert(not is_disjoint_from(TypeOf[Counter], Callable[..., Any]))

static_assert(not is_disjoint_from(Callable[..., Any], TypeOf[DefaultDict]))
static_assert(not is_disjoint_from(TypeOf[DefaultDict], Callable[..., Any]))

static_assert(not is_disjoint_from(Callable[..., Any], TypeOf[Deque]))
static_assert(not is_disjoint_from(TypeOf[Deque], Callable[..., Any]))

static_assert(not is_disjoint_from(Callable[..., Any], TypeOf[OrderedDict]))
static_assert(not is_disjoint_from(TypeOf[OrderedDict], Callable[..., Any]))
```
