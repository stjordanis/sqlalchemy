.. _queryguide_toplevel:

==================
ORM Querying Guide
==================

This section provides an overview of emitting queries with the
SQLAlchemy ORM using :term:`2.0 style` usage.

Readers of this section should be familiar with the SQLAlchemy overview
at :doc:`/tutorial`, and in particular most of the content here expands
upon the content at :ref:`tutorial_selecting_data`.


..  Setup code, not for display

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("sqlite+pysqlite:///:memory:", echo=True, future=True)
    >>> from sqlalchemy import MetaData, Table, Column, Integer, String
    >>> metadata = MetaData()
    >>> user_table = Table(
    ...     "user_account",
    ...     metadata,
    ...     Column('id', Integer, primary_key=True),
    ...     Column('name', String(30)),
    ...     Column('fullname', String)
    ... )
    >>> from sqlalchemy import ForeignKey
    >>> address_table = Table(
    ...     "address",
    ...     metadata,
    ...     Column('id', Integer, primary_key=True),
    ...     Column('user_id', None, ForeignKey('user_account.id')),
    ...     Column('email_address', String, nullable=False)
    ... )
    >>> metadata.create_all(engine)
    BEGIN (implicit)
    ...
    >>> from sqlalchemy.orm import declarative_base
    >>> Base = declarative_base()
    >>> from sqlalchemy.orm import relationship
    >>> class User(Base):
    ...     __tablename__ = 'user_account'
    ...
    ...     id = Column(Integer, primary_key=True)
    ...     name = Column(String(30))
    ...     fullname = Column(String)
    ...
    ...     addresses = relationship("Address", back_populates="user")
    ...
    ...     def __repr__(self):
    ...        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

    >>> class Address(Base):
    ...     __tablename__ = 'address'
    ...
    ...     id = Column(Integer, primary_key=True)
    ...     email_address = Column(String, nullable=False)
    ...     user_id = Column(Integer, ForeignKey('user_account.id'))
    ...
    ...     user = relationship("User", back_populates="addresses")
    ...
    ...     def __repr__(self):
    ...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
    >>> conn = engine.connect()
    >>> from sqlalchemy.orm import Session
    >>> session = Session(conn)
    >>> session.add_all([
    ... User(name="spongebob", fullname="Spongebob Squarepants", addresses=[
    ...    Address(email_address="spongebob@sqlalchemy.org")
    ... ]),
    ... User(name="sandy", fullname="Sandy Cheeks", addresses=[
    ...    Address(email_address="sandy@sqlalchemy.org"),
    ...     Address(email_address="squirrel@squirrelpower.org")
    ...     ]),
    ...     User(name="patrick", fullname="Patrick Star", addresses=[
    ...         Address(email_address="pat999@aol.com")
    ...     ]),
    ...     User(name="squidward", fullname="Squidward Tentacles", addresses=[
    ...         Address(email_address="stentcl@sqlalchemy.org")
    ...     ]),
    ...     User(name="ehkrabs", fullname="Eugene H. Krabs"),
    ... ])
    >>> session.commit()
    BEGIN ...
    >>> conn.begin()
    BEGIN ...


SELECT statements
=================

SELECT statements are produced by the :func:`_sql.select` function which
returns a :class:`_sql.Select` object::

    >>> from sqlalchemy import select
    >>> stmt = select(User).where(User.name == 'spongebob')

To invoke a :class:`_sql.Select` with the ORM, it is passed to
:meth:`_orm.Session.execute`:

.. sourcecode:: pycon+sql

    {sql}>>> result = session.execute(stmt)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('spongebob',){stop}
    >>> for user_obj in result.scalars():
    ...     print(f"{user_obj.name} {user_obj.fullname}")
    spongebob Spongebob Squarepants



.. _orm_queryguide_select_columns:

Selecting ORM Entities
---------------------------------

The :func:`_sql.select` construct accepts ORM entities, including mapped
classes as well as class-level attributes representing mapped columns, which
are converted into ORM-annotated :class:`_sql.FromClause` and
:class:`_sql.ColumnElement` elements at construction time.

A :class:`_sql.Select` object that contains ORM-annotated entities is normally
executed using a :class:`_orm.Session` object, and not a :class:`_future.Connection`
object, so that ORM-related features may take effect.

Below we select from the ``User`` entity, producing a :class:`_sql.Select`
that selects from the mapped :class:`_schema.Table` to which ``User`` is mapped:

.. sourcecode:: pycon+sql

    {sql}>>> result = session.execute(select(User).order_by(User.id))
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.id
    [...] (){stop}

When selecting from ORM entities, the entity itself is returned in the result
as a single column value; for example above, the :class:`_engine.Result`
returns :class:`_engine.Row` objects that have just a single column, that column
holding onto a ``User`` object::

    >>> result.fetchone()
    (User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)

When selecting a list of single-column ORM entities, it is typical to skip
the generation of :class:`_engine.Row` objects and instead receive
ORM entities directly, which is achieved using the :meth:`_engine.Result.scalars`
method::

    >>> result.scalars().all()
    [User(id=2, name='sandy', fullname='Sandy Cheeks'),
     User(id=3, name='patrick', fullname='Patrick Star'),
     User(id=4, name='squidward', fullname='Squidward Tentacles'),
     User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')]

ORM Entities are named in the result row based on their class name,
such as below where we SELECT from both ``User`` and ``Address`` at the
same time:

.. sourcecode:: pycon+sql

    >>> stmt = select(User, Address).join(User.addresses).order_by(User.id, Address.id)

    {sql}>>> for row in session.execute(stmt):
    ...    print(f"{row.User.name} {row.Address.email_address}")
    SELECT user_account.id, user_account.name, user_account.fullname,
    address.id AS id_1, address.email_address, address.user_id
    FROM user_account JOIN address ON user_account.id = address.user_id
    ORDER BY user_account.id, address.id
    [...] (){stop}
    spongebob spongebob@sqlalchemy.org
    sandy sandy@sqlalchemy.org
    sandy squirrel@squirrelpower.org
    patrick pat999@aol.com
    squidward stentcl@sqlalchemy.org


Selecting ORM Attributes
------------------------

The attributes on a mapped class, such as ``User.name`` and ``Address.email_address``,
have a similar behavior as that of the entity class itself such as ``User``
in that they are automatically converted into ORM-annotated Core objects
when passed to :func:`_sql.select`.   They may be used in the same way
as table columns are used:

.. sourcecode:: pycon+sql

    {sql}>>> result = session.execute(
    ...     select(User.name, Address.email_address).
    ...     join(User.addresses).
    ...     order_by(User.id, Address.id)
    ... )
    SELECT user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    ORDER BY user_account.id, address.id
    [...] (){stop}

ORM attributes, themselves known as :class:`_orm.InstrumentedAttribute`
objects, can be used in the same way as any :class:`_sql.ColumnElement`,
and are delivered in result rows just the same way, such as below
where we refer to their values by column name within each row::

    >>> for row in result:
    ...     print(f"{row.name}  {row.email_address}")
    spongebob  spongebob@sqlalchemy.org
    sandy  sandy@sqlalchemy.org
    sandy  squirrel@squirrelpower.org
    patrick  pat999@aol.com
    squidward  stentcl@sqlalchemy.org

Selecting ORM Bundles
---------------------

The :class:`_orm.Bundle` construct is an extensible ORM-only construct that
allows sets of column expressions to be grouped in result rows:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import Bundle
    >>> stmt = select(
    ...     Bundle("user", User.name, User.fullname),
    ...     Bundle("email", Address.email_address)
    ... ).join_from(User, Address)
    {sql}>>> for row in session.execute(stmt):
    ...     print(f"{row.user.name} {row.email.email_address}")
    SELECT user_account.name, user_account.fullname, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    [...] (){stop}
    spongebob spongebob@sqlalchemy.org
    sandy sandy@sqlalchemy.org
    sandy squirrel@squirrelpower.org
    patrick pat999@aol.com
    squidward stentcl@sqlalchemy.org


The :class:`_orm.Bundle` is potentially useful for creating lightweight
views as well as custom column groupings such as mappings.

.. seealso::

    :ref:`bundles` - in the ORM loading documentation.


Selecting ORM Aliases
---------------------

As discussed in the tutorial at :ref:`tutorial_using_aliases`, to create a
SQL alias of an ORM entity is achieved using the :func:`_orm.aliased`
construct against a mapped class:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import aliased
    >>> u1 = aliased(User)
    >>> print(select(u1).order_by(u1.id))
    SELECT user_account_1.id, user_account_1.name, user_account_1.fullname
    FROM user_account AS user_account_1 ORDER BY user_account_1.id

As is the case when using :meth:`_schema.Table.alias`, the SQL alias
is anonymously named.   For the case of selecting the entity from a row
with an explicit name, the :paramref:`_orm.aliased.name` parameter may be
passed as well:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import aliased
    >>> u1 = aliased(User, name="u1")
    >>> stmt = select(u1).order_by(u1.id)
    {sql}>>> row = session.execute(stmt).first()
    SELECT u1.id, u1.name, u1.fullname
    FROM user_account AS u1 ORDER BY u1.id
    [...] (){stop}
    >>> print(f"{row.u1.name}")
    spongebob

The :class:`_orm.aliased` construct is also central to making use of subqueries
with the ORM; the section :ref:`_orm_queryguide_subqueries` discusses this further.

.. _orm_queryguide_selecting_text:

Selecting from textual / separate statements
--------------------------------------------

The ORM supports loading of entities from SELECT statements that come from other
sources.  The typical use case is that of a textual SELECT statement, which
in SQLAlchemy is represented using the :func:`_sql.text` construct.   The
:func:`_sql.text` construct, once constructed, can be augmented with
information
about the ORM-mapped columns that the statement would load; this can then be
associated with the ORM entity itself so that ORM objects can be loaded based
on this statement.

Given a textual SQL statement we'd like to load from::

    >>> from sqlalchemy import text
    >>> textual_sql = text("SELECT id, name, fullname FROM user_account ORDER BY id")

We can add column information to the statement by using the
:meth:`_sql.TextClause.columns` method; when this method is invoked, the
:class:`_sql.TextClause` object is converted into a :class:`_sql.TextualSelect`
object, which takes on a role that is comparable to the :class:`_sql.Select`
construct.  The :meth:`_sql.TextClause.columns` method
is typically passed :class:`_schema.Column` objects or equivalent, and in this
case we can make use of the ORM-mapped attributes on the ``User`` class
directly::

    >>> textual_sql = textual_sql.columns(User.id, User.name, User.fullname)

We now have an ORM-configured SQL construct that as given, can load the "id",
"name" and "fullname" columns separately.   To use this SELECT statement as a
source of complete ``User`` entities instead, we can link these columns to a
regular ORM-enabled
:class:`_sql.Select` construct using the :meth:`_sql.Select.from_statement`
method:

.. sourcecode:: pycon+sql

    >>> # using from_statement()
    >>> orm_sql = select(User).from_statement(textual_sql)
    >>> for user_obj in session.execute(orm_sql).scalars():
    ...     print(user_obj)
    {opensql}SELECT id, name, fullname FROM user_account ORDER BY id
    [...] (){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')
    User(id=3, name='patrick', fullname='Patrick Star')
    User(id=4, name='squidward', fullname='Squidward Tentacles')
    User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')

The same :class:`_sql.TextualSelect` object can also be converted into
a subquery using the :meth:`_sql.TextualSelect.subquery` method,
and linked to the ``User`` entity to it using the :func:`_orm.aliased`
construct, in a similar manner as discussed below in :ref:`orm_queryguide_subqueries`:

.. sourcecode:: pycon+sql

    >>> # using aliased() to select from a subquery
    >>> orm_subquery = aliased(User, textual_sql.subquery())
    >>> stmt = select(orm_subquery)
    >>> for user_obj in session.execute(stmt).scalars():
    ...     print(user_obj)
    {opensql}SELECT anon_1.id, anon_1.name, anon_1.fullname
    FROM (SELECT id, name, fullname FROM user_account ORDER BY id) AS anon_1
    [...] (){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')
    User(id=3, name='patrick', fullname='Patrick Star')
    User(id=4, name='squidward', fullname='Squidward Tentacles')
    User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')

The difference between using the :class:`_sql.TextualSelect` directly with
:meth:`_sql.Select.from_statement` versus making use of :func:`_sql.aliased`
is that in the former case, no subuqery is produced in the resulting SQL.
This can in some scenarios be advantageous from a performance or complexity
perspective.


Joins
-----

ORM Join forms
^^^^^^^^^^^^^^

.. _orm_queryguide_subqueries:

Subqueries
----------

ORM Subquery forms (like from_Self equiv)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Correlated Subqueries
^^^^^^^^^^^^^^^^^^^^^

LATERAL Correlation
^^^^^^^^^^^^^^^^^^^

ORM Execution Options
=====================

.. _orm_queryguide_populate_existing:

Populate Existing
------------------

Autoflush
----------

Yield Per
----------

Loader Options
---------------

(a few lines, then link to relationship loading stuff)

ORM Bulk Update / Delete
========================

