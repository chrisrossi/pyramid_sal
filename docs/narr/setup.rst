Setup
=====

Most of the features of SAL are implemented by an engine which knows about a
specific storage layer, such as `ZODB` or `SQLAlchemy`.  A SAL database is
constructed by passing in an ``Engine`` instance.

.. code-block:: python

   from pyramid_sal import DB
   from pyramid_sal_zodb import Engine

   config = {'url': 'zeo://localhost:8020/'}
   db = DB(Engine(**config))

Actual arguments to engines will vary depending on the engine being used.  All
engines, by specification, are able to accept arbitrary keyword arguments.

If the engine to be used won't be known until runtime, the `DB` can accept an
``engine_name`` parameter which is the dotted name of the engine class to be
used.

.. code-block:: python

   from pyramid_sal import DB

   def db_factory(dotted_name, **config):  # e.g. 'pyramid_sal_zodb.Engine'
       return DB(engine_name=dotted_name, config=config)

To work with a database you must establish a connection.

.. code-block:: python

   session = db.session()

Sessions are not thread safe--a new session should be used for each thread.  An
engine may, behind the scenes, employ some sort of connection pooling.  This is
usually a feature of the underlying storage layer.

Sessions work with the `transaction <http://pypi.python.org/pypi/transaction/>`_
package, which allows using multiple dabases within a single transaction
framework.

.. code-block:: python

   import transaction
   transaction.commit()

If writing a WSGI application, use of a transaction managing middleware such as
`repoze.tm2 <http://pypi.python.org/pypi/repoze.tm2/>`_ is recommended.
