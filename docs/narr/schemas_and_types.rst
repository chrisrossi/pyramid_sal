Schemas and Types
=================

In order for SAL to store your data you must provide metadata describing the
type and structure of the data you want to store.  A schema describes a related
set of storable data.  A schema can be thought of in the abstract as
representing some aspect of an object, such as its being a person, or a
document, etc...

.. code-block:: python

   from pyramid_sal.metadata import Schema
   from pyramid_sal.metadata import BackReference
   from pyramid_sal.metadata import Boolean
   from pyramid_sal.metadata import Blob
   from pyramid_sal.metadata import Clob
   form pyramid_sal.metadata import Date
   from pyramid_sal.metadata import DateTime
   from pyramid_sal.metadata import Float
   from pyramid_sal.metadata import Reference
   from pyramid_sal.metadata import String

   PersonsName = Schema(
       first_name=String(),
       middle_name_or_initial=String(),
       last_name=String(),
       title=String(),
       suffix=String(),
   )

   Person = Schema(
       name=PersonsName(),
       phone_number=String(),
       birthday=Date(),
       attractiveness=Float(),
   )

   Document = Schema(
       title=String(),
       author=Reference(Person, backref=BackReference(
           'documents', order_by='created'))
       body=Clob(),
       created=DateTime()
   )

   File = Schema(
       filename=String(),
       created=DateTime(),
       mime_type=String(),
       data=Blob()
   )

As you can see, schemas can be nested.  The ``name`` field of the ``Person``
schema uses the ``PersonsName`` schema.  A simple one to many relationship is
established between ``Document`` and ``Person`` using the ``author`` field of
``Document``.  The ``backref`` specifier will cause instances providing
``Person`` to have an attribute, ``documents``, which will retrieve the
documents authored by that person in order of creation date.

Schemas can be extended, similar to the way a class might be extended.

.. code-block:: python

   FamousPerson = Person.extend(
       famous_for=String(),
       how_famous=Float(),
       how_popular=Float(),
       also_rich=Boolean()
   )

A type represents something which can be stored directly in the database.  A
type is an aggregate of one or more schemas.

.. code-block:: python

   from pyramid_sal.metadata import Type
   from pyramid_sal.metadata import Integer

   HumanResource = Schema(
       hours_per_week=Integer(),
       effectiveness=Float(),
       time_estimate_error=Float(),
       attractiveness=Float(),
   )

   EmployeeType = Type(Person, HumanResource)
   DocumentType = Type(Document)

A type may optionally be passed a ``cls`` parameter specifying a class to be
used for instances of the type.  If ``cls`` is not specified ``SAL`` will use
a generic class for instances of the type.

.. code-block:: python

   from my_app.models import File as FileModel

   FileType = Type(File, cls=FileModel)

A type uses a particular schema is said to implement that schema.

.. code-block:: python

   assert Employee.implementedBy(EmployeeType)

In order for `SAL` to be able to use a type, the type must be registered with
`SAL` at runtime.

.. code-block:: python

   from pyramid_sal import DB
   from my_app import db_factory
   from my_app.metadata import DocumentType
   from my_app.metadata import EmployeeType
   from my_app.metadata import FileType

   def init_app(sal_engine, sal_config):
       db = db_factory(engine_name=sal_engine, config=sal_config)
       assert isinstance(db, DB)
       db.add_type('Employee', EmployeeType)
       db.add_type('Document', DocumentType)
       db.add_type('File', FileType)

You can use a database session to initiate a database using your registered
metadata.

.. code-block:: python

   ...
       session = db.session()
       session.init_db()

Once registered a type may be retrieved from the runtime by name.

.. code-block:: python

   from my_app import current_session

   session = current_session()
   type = session.db.types['Employee']

A type instance is callable and can be used as a factory for creating an
instance of that type.  Keyword arguments may be passed in that correspond to
fields in a schema provided by that type.

.. code-block:: python

   import datetime

   employee = EmployeeType(birthday=datetime.date(1975, 7, 7))

An instance of a type that implements a schema is said to provide that schema.

.. code-block:: python

   assert Employee.providedBy(employee)

For an instance which provides a schema the fields on that schema may be
retrieved by simple attribute access.

.. code-block:: python

   birthday = employee.birthday
   today = datetime.date.today()
   if birthday.day = today.day and birthday.month = today.month:
       print "Today is %s's birthday!" % employee.name

Sometimes fields with the same name may be defined in more than one schema
provided by an instance.  For example, both ``Person`` and ``Employee`` provide
the field ``attractiveness`` and are implemented by ``EmployeeType``.  Since
these are distinct schemas, the fields have distinct values.  (Someone might
have crazy looking eyebrows but always provide 100% unit test coverage for
their code, making them more attractive as an employee than they might be just
in general.)

.. code-block:: python

   attractiveness = employee.attractiveness

The above code raises an exception because the ``attractiveness`` attribute is
ambiguous.  A schema, however, can be called with an instance as an argument.
This adapts the instance to the particular schema we're interested in, exposing
only the attributes provided by that schema, removing the ambiguity.

.. code-block:: python

   person = Person(employee)
   attractiveness = person.attractiveness

In order for an instance to be persisted it must be added to the database.
This can be done explicitly using the session.

.. code-block:: python

   session.add(employee)

Alternatively, an instance can be added implicitly if it is made to relate to
an already stored instance. In the following example, a new document is
created and given an author. Because (we assume) the author is an object that
is already in the database, the relationship suffices to add the document to
the database without an explicit call to ``Session.add()``.

.. code-block:: python

   def add_document(title, author, body):
       document = Document(title=title, author=author, body=body)
       document.created = datetime.datetime.now()
       return document

Each stored instance is given a globally unique identifier which is an instance
of ``uuid.UUID`` from the Python standard library.  An object can be retrieved
from the database using its uuid.

.. code-block:: python

   uuid = employee.__uuid__
   assert session[uuid] is employee

Use of globally unique idenitifiers is intended to ease the burden of moving
objects from on database to another by avoiding potential id conflicts while at
the same time maintaining an unambiguous identity for that object.

