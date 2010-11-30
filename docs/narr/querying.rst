Indexing & Querying
===================

Indexing
--------

Content stored in a `SAL` repository may be indexed for fast searching and
retrieval. Indexes are specified at runtime at the same time that metadata is
provided to the database.

.. code-block:: python

   from pyramid_sal import DB
   from pyramid_sal.indexes import FieldIndex
   from pyramid_sal.indexes import TextIndex

   from my_app import db_factory
   from my_app.metadata import Document
   from my_app.metadata import DocumentType

   def init_app(sal_engine, sal_config):
       db = db_factory(engine_name=sal_engine, config=sal_config)
       assert isinstance(db, DB)
       db.add_type('Document', DocumentType)
       db.add_index(Document.body, TextIndex)

In the above example, the first argument to ``add_index`` is a schema field
to index.  The second argument is an index type.  An arbitrary index can also
be created by passing in a name for the index instead of a schema field.  In
this case, a `discriminator` callable must also be passed in.  The callable
accepts an object to to be indexed and returns a value to be used for the
index.

.. code-block:: python

   ...
       def get_title_or_name(obj, default):
           value = getattr(obj, 'title', default)
           if value is default:
               value = getattr(obj, 'name', default)
           return default

       db.add_index('title_or_name', FieldIndex, get_title_or_name)

Discriminators also accept a second argument, ``default``, which should be
returned if the object should not be indexed with a value.  In the above
example, if the object to be indexed lacks both a ``title`` and ``name`` field,
``default`` is returned, indicating the object should not be indexed for that
index.

Indexes may be retrieved by name or schema field from the database object.

.. code-block:: python

   title_or_name_index = db.indexes['title_or_name']
   text_index = db.indexes[Document.body]

Two indexes, ``type`` and ``path`` are always available.  The ``type`` path
allows an object to be looked up by type.  The ``path`` index allows an object
to be looked by its traversal path.  (See the traversal chapter.)

.. code-block:: python

   type_index = db.indexes['type']
   path_index = db.indexes['path']

Querying
--------

The `SAL` repository can be queried using the ``query`` method of the session.

.. code-block:: python

   last_xmas = datetime.datetime(2009, 12, 25)
   count, docs = session.query(
       (Document.created > last_xmas) & ('foo' in Document.body),
       order_by=Document.created, reverse=True, limit=10)

The first return value is the number of results returned and the second return
value is an iterator of the results.  No guarantee is made about the type of
the iterator, only that it is iterable.  It may very well be a generator.

The query method takes a single positional argument and several keyword
arguments. The positional argument is a query object. A query object can be
constructed using idiomatic Python simply by using a comparison operator with
a schema field or an index as a left value. (The `in` operator requires the
field or index to be on the right.)

.. code-block:: python

    query = Document.created > last_xmas

The comparison operators are overloaded for both schema field and index
classes, allowing comparison expressions in which one of those is involved as
the left hand value to yield query objects, like in the above example.  Query
objects additionally override the bitwise and (`&`) and the bitwise or (`|`)
operators, allowing queries to be combined.

.. code-block:: python

   query = (Document.created > last_xmas) & ('foo' in Document.body)

These expressions are shorthand and could also be constructed directly using
query classes.  The following expression is equivalent to the expression
above.

.. code-block:: python

   from pyramid_sal import query

   query = query.And(
       query.Gt(Document.created, last_xmas),
       query.Contains(Document.body, 'foo')
   )

There is also a facility for parsing a CQE (Content Query Expression).  The
following is also equivalent to the above two expressions.

.. code-block:: python

   from pyramid_sal.query import parse_query

   query = parse_query(
       "Document.created > last_xmas and 'foo' in Document.body",
       last_xmas=last_xmas
   )

The ``Session.query`` method can accept a CQE in place of a query object.

.. code-block:: python

   count, docs = session.query(
       "Document.created > last_xmas and 'foo' in Document.body",
       order_by=Document.created, reverse=True, limit=10,
       names=dict(last_xmas=last_xmas))

.. note::

   This chapter is low on detail in the interest of getting a basic outline of
   everything out there for review.  This problem has been thought about in
   quite a lot of detail--the expressions and parsing are already basically
   implemented on trunk of `repoze.catalog
   <http://svn.repoze.org/repoze.catalog/>`_ and would require mostly just some
   extra work to generalize.

