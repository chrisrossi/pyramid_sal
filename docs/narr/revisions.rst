Revisions
=========

.. note::

  This may be an optional feature for engines. Additionally, engines which do
  support revisions may have the ability to turn the feature off.

Each commit in a `SAL` database is assigned an integer ID which identifies
that revision of the database. Revision IDs are assigned serially, with higher
numbers coming later chronologically. During a transaction a log message and
user information can be set that is saved with the revision as part of the
revision log.

.. code-block:: python

   # Some business logic in your application
   ...

   # Set revision log data
   session.set_revision_user('Chris Rossi <chris@archimedeanco.com>')
   session.set_revision_note('Edited document %s' % document.title)


The revision log is available as the ``revision_log`` attribute of
``Session``.

.. code-block:: python

   # Print the 10 most recent transactions, most recent first
   recent = reversed(session.transaction_log[-10:])
   for log_entry in recent:
       print "%d %s %s" % (log_entry.revision, log_entry.user, log_entry.note)

When a persistent object is changed in any way during the course of a
transaction, a new copy is made and associated with the new revision.  The
current revision of a persistent object is available via its `__revision__`
attribute.

.. code-block:: python

   version = document.__revision__

A specific revision can be retrieved by passing the revision ID to
``Session.get()``.

.. code-block:: python

   uuid = ...
   revision_id = ...
   old_document = session.get(uuid, revision_id)

Under normal circumstances, when retrieving objects from the database, the most
recent revision is returned.  This behavior can be modified by telling the
session specifically to use a revision in the past.  This can be used to expose
a view of the database as it existed at some point in the past.

.. code-block:: python

   # View database at some point in the past
   revision_id = ...
   session.use_revision(revision)

The current (latest) revision can be retreived by calling
`Session.current_revision()`.

.. code-block:: python

   # View database in its current state
   revision_id = session.current_revision()
   session.use_revision(revision_id)

Revisions may be assigned tags which are string identifiers.
``Session.use_revision()`` can optionally accept one of these tags rather than
an integer revision ID.  Creating or updating a tag is a modification to the
database and is treated like any other modification--it must be committed to
take effect and the transaction in which it occurs will be assigned a new
revision ID.

.. code-block:: python

   PUBLISHED = 'published'
   # Publish the current revision
   revision_id = session.current_revision()
   session.set_revision_tag(revision_id, PUBLISHED)

   # Use the published revision
   session.use_revision(PUBLISHED)

Suppose the `SAL` database is being used as a backend for a web site content
management system. This feature can allow a workflow whereby users normally
view the site using the `PUBLISHED` tag but a content administrator can view
the site at the latest revision for making changes. When a content
administrator is satisfied with the latest revision of the site, the
administrator can make the current site visible to all users by updating the
`PUBLISHED` tag to the latest revision.

When using a revision other than the most recent, data in `SAL` is immutable.
This is a safety feature to avoid a user mistakenly revising an old version of
some content without seeing the most recent changes. An engine may raise an
exception either at the time that a persistent object is mutated (recommended)
or at commit time (acceptable).

.. code-block:: python

   uuid = ...
   session.use_revision(PUBLISHED)
   document = session.get(uuid)

   # Assuming that PUBLISHED != session.current_revision(), the following
   # statement will hopefully immediately raise an exception. An engine
   # implementation may opt to defer the exception until the transaction is
   # committed.
   document.body = document.body + u"\nNeeds more cowbell.\n"

The most recent revision, and only the most recent revision, can be rolled
back.

.. code-block:: python

   n_revisions = len(session.revision_log)
   session.roll_back()
   assert len(session.revision_log) == n_revisions - 1

   # Not that you'd want to do this, but you could use this to roll back
   # every revision stored in the database.
   for i in xrange(len(session.revision_log)):
       session.roll_back()

Frequently update databases may wish to limit the amount of storage devoted to
past revisions.  The ``Session.forget_revisions()`` method can be used to
forget revisions older than a certain amount of time.

.. code-block:: python

   one_week_ago = datetime.datetime.now() - datetime.timedelta(days=7)
   session.forget_revisions(one_week_ago)

