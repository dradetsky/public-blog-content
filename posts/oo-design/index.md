+++
title = "Good OO Design"
date = 2025-01-05T22:23:03-08:00
draft = true
+++

Sometimes job ads used to say "Knowledge of good OO design" or similar, which
seemed to mean something like: Have you skimmed the GoF book? This is obviously
not very interesting. How many people can actually tell you what the difference
between good and bad OO design?

I think Martin Fowler (or someone like that) said that the part of OO that
actually delivers value is the part where you use polymorphic dispatch instead
of if/then statements. In other words, all that Circle/Ellipse nonsense is aside
the point. What you want is to choose a data model where you get to do as much
polymorphic dispatch as you can.

Anyway, I saw some code in the wild that looked a bit[^sh] like this:

```python
# in file_0.py
class NodeType(StrEnum):
    Model = "model"
    Snapshot = "snapshot"
    Seed = "seed"
    Source = "source"

# in file_1.py
class UnrenderedConfig(ConfigSource):
    def get_config_dict(self, resource_type: NodeType, unrendered) -> Dict[str, Any]:
        if resource_type == NodeType.Seed:
            model_configs = unrendered.get("seeds")
        elif resource_type == NodeType.Snapshot:
            model_configs = unrendered.get("snapshots")
        elif resource_type == NodeType.Source:
            model_configs = unrendered.get("sources")
        else:
            model_configs = unrendered.get("models")
        if model_configs is None:
            return {}
        else:
            return model_configs
```

This is contrary to Fowler's advice because we're doing a manual type dispatch.

Now in spite of the fact that hardcoded strings like the above appeared in the
code in question, `NodeType` also defined `pluralize`[^sh2]:

```python
class NodeType(StrEnum):
    def pluralize(self) -> str:
        if self is self.Analysis:
            return "analyses"
        return f"{self}s"
```

So one approach would be to add:

```python
class NodeType(StrEnum):
    def model_config_name(self) -> str:
        if self in set([NodeType.Seed,
                        NodeType.Snapshot,
                        NodeType.Source]):
            return self.pluralize()
        else:
            return NodeType.Model.pluralize()
```

I'm not totally sure I understand _why_ we're doing this, but it at least allows
us to preserve existing behavior. Now we write:

```python
class UnrenderedConfig(ConfigSource):
    def get_config_dict(self, resource_type: NodeType, unrendered) -> Dict[str, Any]:
        model_configs = unrendered.get(resource_type.model_config_name())
        if model_configs is None:
            return {}
        else:
            return model_configs
``` 

In some cases, we might argue that although this is shorter, `model_config_name`
is the sort of thing that's the responsibility of the `UnrenderedConfig` class
and thus shouldn't be defined on `NodeType`[^gfmd]. In that case, an alternative
might be this:

```python
class UnrenderedConfig(ConfigSource):
    def model_config_name(self, resource_type):
        nts = set([NodeType.Seed,
                   NodeType.Snapshot,
                   NodeType.Source])
        if resource_type in nts:
            return resource_type.pluralize()
        else:
            return NodeType.Model.pluralize()
``` 

In GFOO-land, we might instead write something like[^lithp]:

```lisp
(defmethod model-config-name ((rt NodeType))
  (if (member? rt '(NodeType::Seed NodeType::Snapshot ...))
      (pluralize rt)
      (pluralize NodeType::Model)))
      
(defmethod get-config-dict ((uc UnrenderedConfig) (rt NodeType))
  (let* (n (model-config-name rt)
         model-configs (get-config uc n))
     (if (null? model-configs)
         (dict)
         model-configs)))
```

This way, even though `model-config-name` is dispatched exclusively on
`NodeType`, we can still make the relevant code part of the `UnrenderedConfig`
module (or overload `model-config-name` for the other classes in that module,
like `RenderedConfig`).

[^sh]: I shortened it for simplicity

[^sh2]: Again shortened.

[^gfmd]: Which is why ideally you want to work in a language that supports
    generic methods and multiple dispatch, but we can't have everything.

[^lithp]: This is (probably) not correct Common Lisp or Scheme code; I don't
    want to have to look up method names.
