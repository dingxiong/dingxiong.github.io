---
layout: post
title: Python -- Sqlalchemy
date: 2024-03-05 14:31 -0800
categories: [programming-language, python]
tags: [python, sqlalchemy]
---

Start learning sqlalchemy from
[here](https://docs.sqlalchemy.org/en/14/tutorial/index.html)

- concept:
  - engine
  - metadata: This provides abstract concepts of tabls, columns, constraints,
    foreign keys, etc. `create_all` api will create all tables.
  - expression builder: kind of abstraction of components of writing a sql
    sentence. It uses an expression pattern, and build lazy object like
    BinaryExpression, etc. Operators like `_and`, `_or`.
  - ORM: It can handle sort, group by, foreign keys, joins and nested queries.

## How ORM works

First, as stated in the official documents, there are two ORM mapping styles:
imperative mapping and declarative mapping. Since most of us use the latter in
real application, we only focus on declarative mapping.

Use below snippet to illustrate all following concepts.

```python
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import declarative_base
from sqlalchemy.orm import relationship

Base = declarative_base()

class User(Base):
    __tablename__ = "user_account"
    id = Column(Integer, primary_key=True)
    name = Column(String(30))
    fullname = Column(String)
    addresses = relationship(
        "Address", back_populates="user", cascade="all, delete-orphan"
    )
    def __repr__(self):
        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

class Address(Base):
    __tablename__ = "address"
    id = Column(Integer, primary_key=True)
    email_address = Column(String, nullable=False)
    user_id = Column(Integer, ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
    def __repr__(self):
        return f"Address(id={self.id!r}, email_address={self.email_address!r})"
```

### Base

The first thing you noticed about is `Base` class. It has meta class
[DeclarativeMeta](https://github.com/sqlalchemy/sqlalchemy/blob/ab42b0e3d98386c8a13edea3206ef43f018de3b6/lib/sqlalchemy/orm/decl_api.py#L55).
Not sure why it does not overwrite the `__new__` method, but `__init__`.

```python
def __init__(cls, classname, bases, dict_, **kw):
    # early-consume registry from the initial declarative base,
    # assign privately to not conflict with subclass attributes named
    # "registry"
    reg = getattr(cls, "_sa_registry", None)
    if reg is None:
        reg = dict_.get("registry", None)
        if not isinstance(reg, registry):
            raise exc.InvalidRequestError(
                "Declarative base class has no 'registry' attribute, "
                "or registry is not a sqlalchemy.orm.registry() object"
            )
        else:
            cls._sa_registry = reg

    if not cls.__dict__.get("__abstract__", False):
        _as_declarative(reg, cls, dict_)
    type.__init__(cls, classname, bases, dict_)
```

As, you can see, it add `_sa_registry` attribute to each subclass. The critical
part above is `_as_declarative`. This function creates a `ClassManager` for
each table, scan `__tablename__` and columns of this table to create a `Table`
object. Also, it generate a `__init__` method for the subclass.

One interesting thing about this auto generated `__init__` method is that
[IT IS SO HACKY!](https://github.com/sqlalchemy/sqlalchemy/blob/ab42b0e3d98386c8a13edea3206ef43f018de3b6/lib/sqlalchemy/orm/instrumentation.py#L620-L627).
It is created by a text template!!. Maybe, I can leverage this idea in future
work.

### Registry

Each model will carry a registry with it, or `_sa_registry`. For instance,
`User.registry` will return the registry.

Key fields of registry

- `_managers`: a map that stores the mapped `ClassManager`.
- `metadata`: same as the metadata field of each table class.

### ClassManager

Each table model has attribute `_sa_class_manager`.

### Metadata

All table class and registry holds the same metadata object. Usually, an
application has only one metadata.

### Attribute

When you do a `obj.attr = xxx` for a model, you actually hit a `__set__` method
inside the
[Attribute class](https://github.com/sqlalchemy/sqlalchemy/blob/ab42b0e3d98386c8a13edea3206ef43f018de3b6/lib/sqlalchemy/orm/attributes.py#L458).
And this action can do a lot of magic things such as add this `obj` to the
`session.identity_map._modified` set, which should not run at the same time
with `session.flush`.

## How `create_engine` works

Engine is just a bridge that connects a `pool` and `dialect`. The
`create_engine` function does a few things.

1. Initialize the
   [dialect and its dbapi](https://github.com/sqlalchemy/sqlalchemy/blob/ab42b0e3d98386c8a13edea3206ef43f018de3b6/lib/sqlalchemy/engine/create.py#L545).
   The connection string is pared by
   [url.py](https://github.com/sqlalchemy/sqlalchemy/blob/ab42b0e3d98386c8a13edea3206ef43f018de3b6/lib/sqlalchemy/engine/url.py#L661)
   and loaded by the dialect registry
   [Loading function](https://github.com/sqlalchemy/sqlalchemy/blob/ab42b0e3d98386c8a13edea3206ef43f018de3b6/lib/sqlalchemy/dialects/__init__.py#L22)

2. Create a pool.

3. register the `on_connect` and `first_connect` function with `connect` event.
   This is the place you can add some hooks when a new connection is created.

### Dialects

mysql+mysqldb: driver + dialect

## Session

`orm.session.Session` has `_new`, `_deleted`, `_dirty` and `identity_map` that
contain pending changes, so you can print out these members to debug. Changes
won't be sent to db unless you do `session.flush()`. And `sesion.commit()` does
flush automatically.

Note that `session.merge()` is different. It will flush the update statement to
db explicitly, so you do not need do flush again.

`AsyncSession` has a member `sync_session`, so async session is like a proxy.
You can check the `sync_session` status to know what this async session is
doing.

Scoped session is session factory attached to a registry. Each time, you get
the session in the registry if it exists, or create a new session if not. In
most cases, the registry is a thread local registry, then scoped session is
thread local. Note, here, we use `scoped session` and `scoped session factory`
interchangeably because `class scoped_session` is
[decorated by](https://github.com/sqlalchemy/sqlalchemy/blob/ab42b0e3d98386c8a13edea3206ef43f018de3b6/lib/sqlalchemy/orm/scoping.py#L75)
`@create_proxy_methods`, so methods of session are ported to scope session
factory.

Scoped session is the default session object created by Flask-sqlalchemy. The
scope here refers to request context. Each request gets a new session. See
[code](https://github.com/pallets-eco/flask-sqlalchemy/blob/da0e0df80cc368d95dc5a118ce857cead172aded/flask_sqlalchemy/__init__.py#L886-L900).

## HowTos

### How to get the underlying sqlite connection in sqlalchemy

For sync db

```
db.session.connection()._dbapi_connection.dbapi_connection
```

For async db

```
l =  asyncio.get_event_loop()
x = l.run_until_complete(async_db.session.connection())
x.sync_connection._dbapi_connection.dbapi_connection._connection._connection
```

## Alembic

Alembic is the database migration tool based on sqlalchemy.

`alembic downgrade -1` reverts current revision. How does alembic interpret
`-1`? The call stack is

```
command.downgrade
  -> script.base._downgrade_revs
    -> script.revision._collect_downgrade_revisions
      -> script.revision._parse_downgrade_target
```

It uses a regular expression to parse str with format `branch@symbol@relative`,
so `-1` corresponds to `branch = None, symbol = None, relative = -1`.
