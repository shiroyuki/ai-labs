---
name: imagination-di
description: Write Python code that uses the Imagination dependency injection framework in standalone mode. Use when the user asks for Python code with Imagination, `imagination.decorator.service.registered`, or `imagination.standalone.use`, or wants services wired via constructor injection without manual instantiation.
---

# Writing Python with Imagination (Standalone Mode)

Imagination is a Java Beans-inspired dependency injection framework for Python
(https://github.com/shiroyuki/Imagination). This skill produces code that uses
**standalone mode** with the **decorator API**.

Docs: https://imagination.readthedocs.io/en/latest/getting-started/10-standalone_mode.html

## The core rule

**Generated code must NEVER instantiate a registered service directly.**

- Never write `Foo()`, `Foo(bar)` or `Foo(bar=Bar())` for a class decorated with `@registered`.
- Never construct a dependency to pass into another service's constructor.
- Declare dependencies as constructor parameters and let Imagination inject them.
- Retrieve a service only through `imagination.standalone.use`.

Why: Imagination builds services lazily and as singletons. Manual instantiation
creates a second copy outside the container, which defeats injection and breaks
the shared-state guarantee.

## The three building blocks

| Purpose | API |
|---|---|
| Register a service | `@registered(...)` from `imagination.decorator.service` |
| Inject dependencies | Constructor parameters, resolved from type hints |
| Retrieve a service | `use(ServiceClass)` from `imagination.standalone` |

Requires Imagination v3.4 or later, which is when `use` was introduced.
Install with `pip install imagination`.

## Workflow

1. **Identify the services.** Split the logic into classes with a single responsibility (adapters, repositories, domain services, clients).
2. **Decorate each class** with `@registered()`.
3. **Declare dependencies in `__init__`** with a type hint that is itself a registered class. The container resolves it from the hint automatically.
4. **Pass constant values** (strings, numbers, flags) with `Parameter(...)`. Pass environment-derived values with `EnvironmentVariable(...)`.
5. **Retrieve the entry-point service** with `use(...)` at the application boundary (`main()`, a request handler, a CLI command). Call methods on the returned object.
6. **Check the output:** no registered class is instantiated anywhere in the code.

## Patterns

### 1. Service with no dependencies

```python
from imagination.decorator.service import registered


@registered()
class DataAdapter:
    def fetch(self) -> list[dict]:
        ...
```

### 2. Service that depends on another service (type-hint injection)

Leave the injection to the type hint. Do not pass anything for `DataAdapter`.

```python
from imagination.decorator.service import registered


@registered()
class DataService:
    def __init__(self, adapter: DataAdapter):
        self._adapter = adapter

    def total(self) -> int:
        return len(self._adapter.fetch())
```

### 3. Service with primitive parameters

Use `Parameter` for primitive constructor arguments, in constructor order.
Arguments with a registered-class type hint can be omitted from the list.

```python
from imagination.decorator.config import Parameter
from imagination.decorator.service import registered


@registered([Parameter("Panda")])
class Greeter:
    def __init__(self, name: str, service: DataService):
        self._name = name
        self._service = service

    def greet(self) -> str:
        return f"Hello {self._name}, total={self._service.total()}"
```

### 4. Explicit service reference (only when the type hint is not enough)

```python
from imagination.decorator.config import Parameter, Service
from imagination.decorator.service import registered


@registered([Parameter("Panda"), Service(DataService)])
class Greeter:
    def __init__(self, name: str, service: DataService):
        ...
```

Prefer pattern 3. Use `Service(...)` only when the container cannot infer the
dependency from the hint.

### 5. Value from an environment variable

```python
from imagination.decorator.config import EnvironmentVariable
from imagination.decorator.service import registered


@registered([EnvironmentVariable("DATABASE_URL")])
class DatabaseClient:
    def __init__(self, url: str):
        self._url = url
```

### 6. Retrieving and using a service

```python
from imagination.standalone import use


def main() -> None:
    greeter = use(Greeter)   # container builds DataAdapter -> DataService -> Greeter
    print(greeter.greet())


if __name__ == "__main__":
    main()
```

`use(Class)` returns a typed instance, so IDEs offer completion. It is
preferred over `container.get(Class)`, which gives the same result without type
inference.

## Complete example

```python
from imagination.decorator.config import Parameter
from imagination.decorator.service import registered
from imagination.standalone import use


@registered()
class DataAdapter:
    def fetch(self) -> list[dict]:
        return [{"id": 1}, {"id": 2}]


@registered()
class DataService:
    def __init__(self, adapter: DataAdapter):
        self._adapter = adapter

    def total(self) -> int:
        return len(self._adapter.fetch())


@registered([Parameter("Panda")])
class Greeter:
    def __init__(self, name: str, service: DataService):
        self._name = name
        self._service = service

    def greet(self) -> str:
        return f"Hello {self._name}, total={self._service.total()}"


def main() -> None:
    print(use(Greeter).greet())


if __name__ == "__main__":
    main()
```

## Behavior to remember

- **Lazy:** decorating a class does not instantiate it. Instantiation happens on first `use(...)` or when another service needs it.
- **Singleton:** `use(Foo)` always returns the same instance, so do not keep per-call state on a service.
- **Registration happens at import time.** The module defining a service must be imported before `use(...)` is called. In multi-module projects, import the service modules in the entry point, or from a package `__init__.py`.
- **Entity-type services only.** The decorator API supports entity-type declarations only.
- **Standalone config sources** are read in this order: the `IMAGINATION_CONF_FILEPATH` environment variable, `imagination.xml`, `services.xml` (current directory), then decorators. Prefer decorators; do not generate XML unless asked.

## Anti-patterns (never generate these)

```python
# WRONG: manual instantiation of registered services
adapter = DataAdapter()
service = DataService(adapter)

# WRONG: building a dependency inside a constructor
class DataService:
    def __init__(self):
        self._adapter = DataAdapter()

# WRONG: creating the service directly in application code
greeter = Greeter("Panda", DataService(DataAdapter()))

# WRONG: using use() inside a service constructor to fetch dependencies
class DataService:
    def __init__(self):
        self._adapter = use(DataAdapter)   # declare it as a constructor parameter instead
```

## Pre-delivery checklist

- [ ] Every service class has `@registered(...)`.
- [ ] Every dependency is a constructor parameter with a type hint.
- [ ] No registered class is instantiated with `ClassName(...)` anywhere.
- [ ] Services are retrieved only with `use(...)`, at the application boundary.
- [ ] Constants use `Parameter(...)`; environment values use `EnvironmentVariable(...)`.
- [ ] Imports come from `imagination.decorator.service`, `imagination.decorator.config` and `imagination.standalone`.
- [ ] Modules that register services are imported before the first `use(...)`.
- [ ] Services hold no per-call mutable state, since they are singletons.