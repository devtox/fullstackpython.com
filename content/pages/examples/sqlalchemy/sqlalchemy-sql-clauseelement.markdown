title: sqlalchemy.sql ClauseElement code examples
category: page
slug: sqlalchemy-sql-clauseelement-examples
sortorder: 500031065
toc: False
sidebartitle: sqlalchemy.sql ClauseElement
meta: Python example code for the ClauseElement class from the sqlalchemy.sql module of the SQLAlchemy project.


ClauseElement is a class within the sqlalchemy.sql module of the SQLAlchemy project.


## Example 1 from GINO
[GINO](https://github.com/fantix/gino)
([project documentation](https://python-gino.readthedocs.io/en/latest/)
and
[PyPI package information](https://pypi.org/project/gino/))
is an [object-relational mapper (ORM)](/object-relational-mappers-orms.html)
built on SQLAlchemy that is non-blocking and therefore designed to work properly
with asynchronously-run code, for example, an application written with
[asyncio](https://docs.python.org/3/library/asyncio.html).

GINO is open sourced under the [BSD License](https://github.com/python-gino/gino/blob/master/LICENSE).

[**GINO / src/gino / crud.py**](https://github.com/python-gino/gino/blob/master/src/gino/./crud.py)

```python
# crud.py
import inspect
import itertools
import weakref

import sqlalchemy as sa
~~from sqlalchemy.sql import ClauseElement

from . import json_support
from .declarative import Model, InvertDict
from .exceptions import NoSuchRowError
from .loader import AliasLoader, ModelLoader

DEFAULT = object()


class _Create:
    def __get__(self, instance, owner):
        if instance is None:
            return owner._create_without_instance
        else:
            return instance._create


class _Query:
    def __get__(self, instance, owner):
        owner._check_abstract()
        q = sa.select([owner.__table__])
        if instance is not None:
            q = q.where(instance.lookup())
        return q.execution_options(model=weakref.ref(owner))


## ... source file abbreviated to get to ClauseElement examples ...


            owner._check_abstract()
            q = owner.__table__.delete()
            return q.execution_options(model=weakref.ref(owner))
        else:
            return instance._delete


class UpdateRequest:

    def __init__(self, instance: "CRUDModel"):
        self._instance = instance
        self._values = {}
        self._props = {}
        self._literal = True
        self._locator = None
        if instance.__table__ is not None:
            try:
                self._locator = instance.lookup()
            except LookupError:
                pass

    def _set(self, key, value):
        self._values[key] = value

    def _set_prop(self, prop, value):
~~        if isinstance(value, ClauseElement):
            self._literal = False
        self._props[prop] = value

    async def apply(self, bind=None, timeout=DEFAULT):
        if self._locator is None:
            raise TypeError(
                "Model {} has no table, primary key or custom lookup()".format(
                    self._instance.__class__.__name__
                )
            )
        cls = type(self._instance)
        values = self._values.copy()

        json_updates = {}
        for prop, value in self._props.items():
            value = prop.save(self._instance, value)
            updates = json_updates.setdefault(prop.prop_name, {})
            if self._literal:
                updates[prop.name] = value
            else:
                if isinstance(value, int):
                    value = sa.cast(value, sa.BigInteger)
~~                elif not isinstance(value, ClauseElement):
                    value = sa.cast(value, sa.Unicode)
                updates[sa.cast(prop.name, sa.Unicode)] = value
        for prop_name, updates in json_updates.items():
            prop = getattr(cls, prop_name)
            from .dialects.asyncpg import JSONB

            if isinstance(prop.type, JSONB):
                if self._literal:
                    values[prop_name] = prop.concat(updates)
                else:
                    values[prop_name] = prop.concat(
                        sa.func.jsonb_build_object(*itertools.chain(*updates.items()))
                    )
            else:
                raise TypeError(
                    "{} is not supported to update json "
                    "properties in Gino. Please consider using "
                    "JSONB.".format(prop.type)
                )

        opts = dict(return_model=False)
        if timeout is not DEFAULT:
            opts["timeout"] = timeout
        clause = (


## ... source file abbreviated to get to ClauseElement examples ...


        )
        if bind is None:
            bind = cls.__metadata__.bind
        row = await bind.first(clause)
        if not row:
            raise NoSuchRowError()
        for k, v in row.items():
            self._instance.__values__[self._instance._column_name_map.invert_get(k)] = v
        for prop in self._props:
            prop.reload(self._instance)
        return self

    def update(self, **values):

        cls = type(self._instance)
        for key, value in values.items():
            prop = cls.__dict__.get(key)
            if isinstance(prop, json_support.JSONProperty):
                value_from = "__profile__"
                method = self._set_prop
                k = prop
            else:
                value_from = "__values__"
                method = self._set
                k = key
~~            if not isinstance(value, ClauseElement):
                setattr(self._instance, key, value)
                value = getattr(self._instance, value_from)[key]
            method(k, value)
        return self


class Alias:

    def __init__(self, model, *args, **kwargs):
        model._check_abstract()
        self.model = model
        self.alias = model.__table__.alias(*args, **kwargs)

    def __getattr__(self, item):
        rv = getattr(
            self.alias.columns,
            item,
            getattr(self.alias, item, getattr(self.model, item, DEFAULT)),
        )
        if rv is DEFAULT:
            raise AttributeError
        return rv

    def __iter__(self):


## ... source file continues with no further ClauseElement examples...

```

