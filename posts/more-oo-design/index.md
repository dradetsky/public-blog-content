+++
title = "More Good OO Design"
date = 2025-01-06T12:16:23-08:00
draft = true
+++

Another case of code doctoring. We have the following:

```python
# in file_a.py
@dataclass
class Project:
    project_name: str
    version: Optional[Union[SemverString, float]]
    project_root: str
    # many more fields...

# in file_b.py
@dataclass
class Profile(HasCredentials):
    profile_name: str
    target_name: str
    threads: int
    # many more fields...

# in file_c.py
@dataclass
class RuntimeConfig(Project, Profile):
    profile_name: str

    @classmethod
    def from_parts(
        cls,
        project: Project,
        profile: Profile,
    ) -> "RuntimeConfig":

        return cls(
            project_name=project.project_name,
            version=project.version,
            project_root=project.project_root,
            # many more params...
            profile_name=profile.profile_name,
            target_name=profile.target_name,
            threads=profile.threads,
        )
```

In the real code, we passed like 50 params into the ctor inside of `from_parts`,
but I'm simplifying.

In this code, `RuntimeConfig` and `Profile`/`Project` are tightly coupled; If
you change the fields of `Profile` you (probably) have to change at least one
method of `RuntimeConfig`. This isn't ideal.

Instead, we could write:

```python
@dataclass
class RuntimeConfig:
    profile: Profile
    project: Project

    @classmethod
    def from_parts(
        cls,
        project: Project,
        profile: Profile,
    ) -> "RuntimeConfig":

        return cls(
            project=project,
            profile=profile,
        )
```

Now the fields of `Profile` and the ctor of `RuntimeConfig` are no longer
coupled. However, there is now a functional difference. Whereas with the above
code, I could write `runtime_conf_inst.version`, I would now have to write
`runtime_conf_inst.project.version` for the same effect.

Depending on what exactly you're doing, this might actually be a benefit, not a
problem. It might be preferable to do the reference this way. However, in this
case there will probably be code which relies on `runtime_conf_inst.version`
so we need to be compatible[^change].

In the abstract what we want to do is:

- For every field `some_field` which is a member of `Profile` but not `Project`,
  we want to define a property `RuntimeConfig.some_field =>
  RuntimeConfig.profile.some_field`.

- And vice-versa for `Project`.

- If there are any fields `com_field` which appear on both, we have to somehow
  define `RuntimeConfig.com_field` to return whatever value it's supposed to.
  For example, by manually specifying the target field.

This could probably be accomplished via a class decorator. To begin with, we
could define:

```python
from dataclasses import (
    dataclass,
    fields,
)
import operator
from functools import reduce

@dataclass
class A:
    x: int
    y: int

@dataclass
class B:
    z: int
    w: int

@dataclass
class C:
    x: int
    u: int
    v: int


def field_names(klass):
    return [x.name for x in fields(klass)]

def unique_fields(klasses):
    r = {}
    for k in klasses:
        non_k = set(klasses) - set([k])
        non_k_fields_seq = [field_names(nk) for nk in non_k]
        non_k_fields = set(reduce(operator.add, non_k_fields_seq, []))
        k_fields = set(field_names(k))
        uniq_fs = k_fields - non_k_fields
        r[k] = uniq_fs
    return r
```

Now `unique_fields` will yield `A => [x], B => [w, z], C => [u, v]`. We would now
like to write something like:

```python
@delegates_to([A, B, C])
@dataclass
class D:
    a: A
    b: B
    c: C
```

Whose result would be the equivalent of adding several definitions like:

```python
class D:
    @property
    def x(self):
        return self.a.x

    @property
    def w(self):
        return self.b.w
    
    # ...
```

This is left as an exercise for the reader, but it should be possible to do in
terms of `unique_fields`.

[^change]: We could also just say "No, fix the other code", and that might be
    the right thing to do, but for purpose of argument we'll assume that we have
    to support the previous API.
