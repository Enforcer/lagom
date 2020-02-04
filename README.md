# Lagom - Dependency injection container
[![Build Status](https://travis-ci.org/meadsteve/lagom.svg?branch=master)](https://travis-ci.org/meadsteve/lagom)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/meadsteve/lagom/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/meadsteve/lagom/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/meadsteve/lagom/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/meadsteve/lagom/?branch=master)
![PyPI](https://img.shields.io/pypi/v/lagom.svg?style=plastic)
## Usage
Everything in Lagom is based on types. To create an object
you pass the type to the container:
```python
container = Container()
some_thing = container[SomeClass]
```

### Defining a singleton
```python
container[SomeExpensiveToCreateClass] = SomeExpensiveToCreateClass("up", "left")
```
alternatively if you want to defer construction until it's needed:
```python
container[SomeExpensiveToCreateClass] = Singleton(SomeExpensiveToCreateClass)
```


### Defining a type that gets recreated every time
```python
container[SomeClass] = lambda: SomeClass("down", "spiral")
```
if the type needs things from the container the lambda can take a
single argument which is the container:
```python
container[SomeClass] = lambda c: SomeClass(c[SomeOtherDep], "spinning")
```

if your construction logic is longer than would fit in a lambda a
function can also be bound to the container:
```python
@dependency_definition(container)
def my_constructor() -> MyComplexDep:
    # Really long
    # stuff goes here
    return MyComplexDep(some_number=5)
```

### Alias a concrete instance to an ABC
```python
container[SomeAbc] = ConcreteClass
```

### Partially bind a function
Apply a function decorator to any function.
```python
@bind_to_container(container)
def handle_some_request(request: typing.Dict, game: Game):
    # do something to the game
    pass
```

This function can now be called omitting any arguments 
that the container knows how to build.
```python
# we can now call the following. the game argument will automagically
# come from the container
handle_some_request(request={"roll_dice": 5})
```


### Invocation level caching
Suppose you have a function and you want all the dependencies
to share an instance of an object then you can define invocation
level shared dependencies.

```python

class ProfileLoader:
    def __init__(self, loader: DataLoader):
        pass

class AvatarLoader:
    def __init__(self, loader: DataLoader):
        pass

@bind_to_container(container, shared=[DataLoader])
def handle_some_request(request: typing.Dict, profile: ProfileLoader, user_avatar: AvatarLoader):
    # do something to the game
    pass
```

now each invocation of handle_some_request will get the same instance
of loader so this class can cache values for the invocation lifetime.

### Alternative to decorator
The above example can also be used without a decorator if you want
to keep the pure unaltered function available for testing.

```python
def handle_some_request(request: typing.Dict, game: Game):
    pass

# This new function can be bound to a route or used wherever
# need
func_with_injection = container.partial(handle_some_request)
```
## Full Example

### App setup
```python
from abc import ABC
from dataclasses import dataclass

from lagom import Container

#--------------------------------------------------------------
# Here is an example of some classes your application may be built from


@dataclass
class DiceApiUrl:
    url: str


class RateLimitingConfig:
    pass


class DiceClient(ABC):
    pass


class HttpDiceClient(DiceClient):

    def __init__(self, url: DiceApiUrl, limiting: RateLimitingConfig):
        pass


class Game:
    def __init__(self, dice_roller: DiceClient):
        pass

#--------------------------------------------------------------
# Next we setup some definitions

container = Container()
# We need a specific url
container[DiceApiUrl] = DiceApiUrl("https://roll.diceapi.com")
# Wherever our code wants a DiceClient we get the http one
container[DiceClient] = HttpDiceClient

#--------------------------------------------------------------
# Now the container can build the game object

game = container[Game]

```

### Modifying the container instead of patching in tests
Taking the container from above we can now swap out
the dice client to a test double/fake. When we get an
instance of the `Game` class it will have the new
fake dice client injected in.

```python
def container_fixture():
    from my_app.prod_container import container
    return container.clone() # Cloning enables overwriting deps

def test_something(container_fixture: Container):
    container_fixture[DiceClient] = FakeDice(always_roll=6)
    game_to_test = container_fixture[Game]
    # TODO: act & assert on something
```

## Design Goals
* The API should expose sensible typing (for use in pycharm/mypy)
* Everything should be done by type. No reliance on names.
* All domain code should remain unmodified. No special decorators.
* Usage of the container should make code testable without monkey patching.
* Make use of modern python features (3.7 at the time of creation)