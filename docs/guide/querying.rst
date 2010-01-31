=====================
Querying the database
=====================
:class:`~mongoengine.Document` classes have an :attr:`objects` attribute, which
is used for accessing the objects in the database associated with the class.
The :attr:`objects` attribute is actually a
:class:`~mongoengine.queryset.QuerySetManager`, which creates and returns a new
a new :class:`~mongoengine.queryset.QuerySet` object on access. The
:class:`~mongoengine.queryset.QuerySet` object may may be iterated over to
fetch documents from the database::

    # Prints out the names of all the users in the database
    for user in User.objects:
        print user.name

Filtering queries
=================
The query may be filtered by calling the
:class:`~mongoengine.queryset.QuerySet` object with field lookup keyword 
arguments. The keys in the keyword arguments correspond to fields on the
:class:`~mongoengine.Document` you are querying::

    # This will return a QuerySet that will only iterate over users whose
    # 'country' field is set to 'uk'
    uk_users = User.objects(country='uk')

Fields on embedded documents may also be referred to using field lookup syntax
by using a double-underscore in place of the dot in object attribute access
syntax::
    
    # This will return a QuerySet that will only iterate over pages that have
    # been written by a user whose 'country' field is set to 'uk'
    uk_pages = Page.objects(author__country='uk')

Querying lists
--------------
On most fields, this syntax will look up documents where the field specified
matches the given value exactly, but when the field refers to a
:class:`~mongoengine.ListField`, a single item may be provided, in which case
lists that contain that item will be matched::

    class Page(Document):
        tags = ListField(StringField())

    # This will match all pages that have the word 'coding' as an item in the
    # 'tags' list
    Page.objects(tags='coding')

Query operators
===============
Operators other than equality may also be used in queries; just attach the
operator name to a key with a double-underscore::
    
    # Only find users whose age is 18 or less
    young_users = Users.objects(age__lte=18)

Available operators are as follows:

* ``neq`` -- not equal to
* ``lt`` -- less than
* ``lte`` -- less than or equal to
* ``gt`` -- greater than
* ``gte`` -- greater than or equal to
* ``in`` -- value is in list (a list of values should be provided)
* ``nin`` -- value is not in list (a list of values should be provided)
* ``mod`` -- ``value % x == y``, where ``x`` and ``y`` are two provided values
* ``all`` -- every item in array is in list of values provided
* ``size`` -- the size of the array is 
* ``exists`` -- value for field exists

Limiting and skipping results
=============================
Just as with traditional ORMs, you may limit the number of results returned, or
skip a number or results in you query.
:meth:`~mongoengine.queryset.QuerySet.limit` and
:meth:`~mongoengine.queryset.QuerySet.skip` and methods are available on
:class:`~mongoengine.queryset.QuerySet` objects, but the prefered syntax for
achieving this is using array-slicing syntax::

    # Only the first 5 people
    users = User.objects[:5]

    # All except for the first 5 people
    users = User.objects[5:]

    # 5 users, starting from the 10th user found
    users = User.objects[10:15]

You may also index the query to retrieve a single result. If an item at that
index does not exists, an :class:`IndexError` will be raised. A shortcut for
retrieving the first result and returning :attr:`None` if no result exists is
provided (:meth:`~mongoengine.queryset.QuerySet.first`)::
    
    >>> # Make sure there are no users
    >>> User.drop_collection()
    >>> User.objects[0]
    IndexError: list index out of range
    >>> User.objects.first() == None
    True
    >>> User(name='Test User').save()
    >>> User.objects[0] == User.objects.first()
    True

Retrieving unique results
-------------------------
To retrieve a result that should be unique in the collection, use
:meth:`~mongoengine.queryset.QuerySet.get`. This will raise
:class:`~mongoengine.queryset.DoesNotExist` if no document matches the query,
and :class:`~mongoengine.queryset.MultipleObjectsReturned` if more than one
document matched the query.

A variation of this method exists, 
:meth:`~mongoengine.queryset.Queryset.get_or_create`, that will create a new
document with the query arguments if no documents match the query. An 
additional keyword argument, :attr:`defaults` may be provided, which will be
used as default values for the new document, in the case that it should need
to be created::

    >>> a = User.objects.get_or_create(name='User A', defaults={'age': 30})
    >>> b = User.objects.get_or_create(name='User A', defaults={'age': 40})
    >>> a.name == b.name and a.age == b.age
    True

Default Document queries
========================
By default, the objects :attr:`~mongoengine.Document.objects` attribute on a
document returns a :class:`~mongoengine.queryset.QuerySet` that doesn't filter
the collection -- it returns all objects. This may be changed by defining a
method on a document that modifies a queryset. The method should accept two
arguments -- :attr:`doc_cls` and :attr:`queryset`. The first argument is the
:class:`~mongoengine.Document` class that the method is defined on (in this
sense, the method is more like a :func:`classmethod` than a regular method),
and the second argument is the initial queryset. The method needs to be
decorated with :func:`~mongoengine.queryset.queryset_manager` in order for it
to be recognised. ::

    class BlogPost(Document):
        title = StringField()
        date = DateTimeField()

        @queryset_manager
        def objects(doc_cls, queryset):
            # This may actually also be done by defining a default ordering for
            # the document, but this illustrates the use of manager methods
            return queryset.order_by('-date')

You don't need to call your method :attr:`objects` -- you may define as many
custom manager methods as you like::

    class BlogPost(Document):
        title = StringField()
        published = BooleanField()

        @queryset_manager
        def live_posts(doc_cls, queryset):
            return queryset.order_by('-date')

    BlogPost(title='test1', published=False).save()
    BlogPost(title='test2', published=True).save()
    assert len(BlogPost.objects) == 2
    assert len(BlogPost.live_posts) == 1

Aggregation
===========
MongoDB provides some aggregation methods out of the box, but there are not as
many as you typically get with an RDBMS. MongoEngine provides a wrapper around
the built-in methods and provides some of its own, which are implemented as
Javascript code that is executed on the database server.

Counting results
----------------
Just as with limiting and skipping results, there is a method on
:class:`~mongoengine.queryset.QuerySet` objects -- 
:meth:`~mongoengine.queryset.QuerySet.count`, but there is also a more Pythonic
way of achieving this::

    num_users = len(User.objects)

Further aggregation
-------------------
You may sum over the values of a specific field on documents using
:meth:`~mongoengine.queryset.QuerySet.sum`::

    yearly_expense = Employee.objects.sum('salary')

.. note::
   If the field isn't present on a document, that document will be ignored from
   the sum.

To get the average (mean) of a field on a collection of documents, use
:meth:`~mongoengine.queryset.QuerySet.average`::

    mean_age = User.objects.average('age')

As MongoDB provides native lists, MongoEngine provides a helper method to get a
dictionary of the frequencies of items in lists across an entire collection --
:meth:`~mongoengine.queryset.QuerySet.item_frequencies`. An example of its use
would be generating "tag-clouds"::

    class Article(Document):
        tag = ListField(StringField())

    # After adding some tagged articles...
    tag_freqs = Article.objects.item_frequencies('tag', normalize=True)

    from operator import itemgetter
    top_tags = sorted(tag_freqs.items(), key=itemgetter(1), reverse=True)[:10]

Advanced queries
================
Sometimes calling a :class:`~mongoengine.queryset.QuerySet` object with keyword
arguments can't fully express the query you want to use -- for example if you
need to combine a number of constraints using *and* and *or*. This is made 
possible in MongoEngine through the :class:`~mongoengine.queryset.Q` class.
A :class:`~mongoengine.queryset.Q` object represents part of a query, and
can be initialised using the same keyword-argument syntax you use to query
documents. To build a complex query, you may combine 
:class:`~mongoengine.queryset.Q` objects using the ``&`` (and) and ``|`` (or)
operators. To use a :class:`~mongoengine.queryset.Q` object, pass it in as the
first positional argument to :attr:`Document.objects` when you filter it by
calling it with keyword arguments::

    # Get published posts
    Post.objects(Q(published=True) | Q(publish_date__lte=datetime.now()))

    # Get top posts
    Post.objects((Q(featured=True) & Q(hits__gte=1000)) | Q(hits__gte=5000))

.. warning::
   Only use these advanced queries if absolutely necessary as they will execute
   significantly slower than regular queries. This is because they are not
   natively supported by MongoDB -- they are compiled to Javascript and sent
   to the server for execution.

.. _guide-atomic-updates:

Atomic updates
==============
Documents may be updated atomically by using the
:meth:`~mongoengine.queryset.QuerySet.update_one` and
:meth:`~mongoengine.queryset.QuerySet.update` methods on a 
:meth:`~mongoengine.queryset.QuerySet`. There are several different "modifiers"
that you may use with these methods:

* ``set`` -- set a particular value
* ``unset`` -- delete a particular value (since MongoDB v1.3+)
* ``inc`` -- increment a value by a given amount
* ``dec`` -- decrement a value by a given amount
* ``push`` -- append a value to a list
* ``push_all`` -- append several values to a list
* ``pull`` -- remove a value from a list
* ``pull_all`` -- remove several values from a list

The syntax for atomic updates is similar to the querying syntax, but the 
modifier comes before the field, not after it::
    
    >>> post = BlogPost(title='Test', page_views=0, tags=['database'])
    >>> post.save()
    >>> BlogPost.objects(id=post.id).update_one(inc__page_views=1)
    >>> post.reload()  # the document has been changed, so we need to reload it
    >>> post.page_views
    1
    >>> BlogPost.objects(id=post.id).update_one(set__title='Example Post')
    >>> post.reload()
    >>> post.title
    'Example Post'
    >>> BlogPost.objects(id=post.id).update_one(push__tags='nosql')
    >>> post.reload()
    >>> post.tags
    ['database', 'nosql']
