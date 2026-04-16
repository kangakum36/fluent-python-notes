# Chapter 3 — Dictionaries and Sets

## Generic Mapping Types

- `Mapping` and `MutableMapping` ABCs formalize the interfaces of `dict` and similar types. They document and formalize the minimal interfaces for mappings.
- For all mapping types, keys must be hashable.
- **Hashable** means the object has a hash value which never changes during its lifetime and can be compared to other objects. Hashable objects which compare equal must have the same hash value.
- Immutable types are all hashable. `frozenset` is hashable. `tuple` is hashable if all items are.
- Can use literal syntax or the `dict` constructor with kwargs to build dicts.

## Dict Comprehensions

- A dictcomp builds a `dict` instance by producing a `key:value` pair from any iterable.

## Overview of Common Mapping Methods

- `clear` — remove items
- `__contains__` — contains check
- `copy` — shallow copy, optimized for dicts specifically
- `__copy__` — support for `copy.copy`; use when you don't know the type of the object being copied
- `default_factory` — this is the callable invoked by the `__missing__` method to set missing values in `defaultdict`. Basically the logic is:

  ```python
  def __missing__(self, key):
      if default_factory is None:
          raise KeyError(key)
      self[key] = value = self.default_factory()
      return value
  ```

  Note that `__missing__` is only called by the `__getitem__` method if a key is missing and the class implements `__missing__`. It is **not** called by `in`/`__contains__`. This means that for `defaultdict`s, contains checks or `d.get` without first trying to access the item will return `False`. This is a deliberate design choice: `get` and `in` are supposed to be non-mutating methods.

- `fromkeys` — new mapping from keys in iterable
- `get(k, [default])` — get item with key `k`, return `default` or `None` if missing
- `__getitem__` — invoked by `d[k]`

  **An important point about `__getitem__`:** If using `d[k]`, the Python interpreter will insert bytecode that causes `dict.__getitem__` to get called. So even if the user did `d.__getitem__ = lambda ...`, the lambda will not be called. What actually happens is that `d[k]` compiles to `BINARY_SUBSCR` bytecode, which the interpreter executes by calling `PyObject_GetItem(d, k)` in C, which then dispatches to the type's subscript slot. This is called **implicit special method lookup**. The same thing happens for other types (e.g. `list`).

- `items()` — view over items
- `__iter__()` — iterator over keys
- `keys()` — view over keys
- `__len__()` — number of items
- `__missing__()` — already covered
- `move_to_end()` — moves to beginning or end; only supported by `OrderedDict`, even though `dict` is ordered by default since Python 3.7
- `pop(k, [default])` — remove and return value at `k` or `default`
- `popitem()` — remove and return arbitrary item
- `__reversed__()` — iterator for keys last to first inserted
- `setdefault(k, [default])` — if `k` in `d`, return `d[k]`; else set `d[k] = default` and return
- `__setitem__(k, v)`
- `update(m, [**kargs])` — update `d` with items from mapping or iterable of `(k, v)` pairs
- `values()` — get view over values

## Handling Missing Keys with `setdefault`

Instead of doing:

```python
v = d.get(k, default)
v.append/update/whatever(...)
d[k] = v
```

Do:

```python
d.setdefault(k, default).append/update/whatever(...)
```

This is simpler and avoids extra lookups.

## Mappings with Flexible Key Lookup

- Sometimes convenient to have mappings return a made-up value when a missing key is searched.
- `defaultdict` is configured to create items on demand by having a `__missing__` method implemented which is called by `__getitem__`.
- The constructor of `defaultdict` takes a type and uses that type's constructor as `default_factory`.
- Can also customize the missing method to, for example, convert int arguments to `getitem` to strings if the lookup is equivalent (i.e. the intent is for `d[13]` to be equal to `d['13']`). The missing method would be:

  ```python
  def __missing__(self, key):
      if isinstance(key, str):
          raise KeyError(key)
      return self[str(key)]
  ```

## Variations of `dict`

- **`collections.OrderedDict`** — maintains keys in insertion order, allowing iteration over items in predictable order.
- **`ChainMap`** — holds a list of mappings that can be searched as one. Lookup is performed on each mapping in order, succeeds if the key is found in any of them.
- **`collections.Counter`** — holds an integer count for each key. Updating an existing key adds to its count. Implements various useful operators and methods like `+`, `-`, and `most_common`.
- **`collections.UserDict`** — pure Python implementation of a mapping that works like a standard `dict`, designed to be subclassed.

## Subclassing `UserDict`

- Easier than subclassing `dict` because the built-in implementation has shortcuts that we're forced to override, but `UserDict` we can inherit from with no problems.

## Immutable Mappings

- The `types` module provides a wrapper class called `MappingProxyType` which returns a `mappingproxy` instance that is a read-only but dynamic view of the original mapping. Useful for exposing private mappings which the user should not change.

## Set Theory

- A collection of unique objects; `frozenset` is the hashable version.
- Set types implement essential set operations: union, intersection, and difference.
- Empty set is `set()`, but otherwise use `{1, 2, 3}` as it will run faster since Python doesn't have to fetch a constructor or build a list. To process the literal there is a special `BUILD_SET` bytecode.
- Set comprehensions work like dict comprehensions.
- Sets have `and`, reverse `and`, `intersection`, `&=`, same for `or`, difference, and symmetric difference. They also have `contains`, subset, proper subset, superset, proper superset. Finally, they have `add`/`clear`/`copy`/`discard(e)`/`__iter__`/`__len__`/`pop()`/`remove(e)`. `discard` fails silently while `remove` raises an error.

## Dict and Set Under the Hood

- They are obviously faster than lists for lookups.
- Sparse array.
- Keeps 1/3 of the buckets empty.
- Objects that are similar but not equal should have hash values that differ widely.
- Python calls `hash(key)` to obtain a hash value and uses least significant bits as an offset to look up a bucket.
- If the bucket is empty, raises `KeyError`. If the keys match, it is found.
- If the keys don't match, hash collision. To resolve, the algorithm takes different bits in the hash, massages them, and uses the result as an offset to look up a different bucket. The collision resolution process is repeated until an empty bucket is found.
- The strategy actually used is described in `dictobject.c`, and is basically to do `j = ((5*j) + 1) mod 2**i` for some initial `j` in the range, repeating `2**i` times, which is guaranteed to generate each int in the range once. However, there is also a perturb constant which is used to shift the hash value and get all bits into play. The code is:

  ```c
  perturb >>= PERTURB_SHIFT;
  j = (5*j) + 1 + perturb;
  // use j % 2**i as the next table index
  ```

- The `5 * j + 1` pseudo-scrambles, which is better when all the bits are in play.
- There are a lot of considerations in the CPython codebase. I also found some comments about better cache affinity if selecting consecutive slots as candidates, but apparently this causes more collisions which ends up hurting performance.
- If on insertion Python determines the hash table is too crowded, it can resize.

## Practical Consequences of How `dict` Works

- Keys must be hashable, meaning they support a stable hash, support equality, and `a == b => hash(a) == hash(b)`.
- Dicts have significant memory overhead since hash tables must be sparse to work.
- If handling a large quantity of records, tuples could be better than dicts for this reason.
- Can use the `__slots__` class attribute for user-defined types to change storage of instance attrs from dict to tuple.
- Key search is very fast — basically dicts trade space for time.
- Key ordering depends on insertion order, because if 2 keys collide then which slot each occupies depends on which was inserted first.
- Adding items to a dict may change order of existing keys. This is because of resizing. In Python 3.7+, insertion order is preserved, but CPython will still raise a runtime error on size changes during iteration. This is because resize during iteration could invalidate internal iterator state, and it's unclear whether the iterator should handle the new item(s).

## Practical Consequences on Sets

- Same as dict.
