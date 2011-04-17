####################
Reusable Django Apps
####################

This document helps you create reusable Django applications.

References:

* http://ericholscher.com/projects/django-conventions/app/
* http://www.b-list.org/weblog/2007/mar/27/reusable-django-apps/

Start an application
====================

First of all, you can use Paster to create the app skeleton.
http://packages.python.org/DjangoPluggableApp/

But maybe it would be better to use distutils2 from now on.

Reusable and pluggable Models
=============================

Declare your models in settings
-------------------------------

* Declare base model as abstract
* Inherit from base model to provide the default model
* Declare used model in settings
* Use get_model() to load model from settings
* Your application uses the model returned by get_model()

.. code-block:: python

  # __init__.py
  from library.settings import load_settings
  
  apply_settings()

  # settings.py
  LIBRARY_BOOK_MODEL = 'library.default_models.Book'
  
  def apply_settings():
      """Updates django.conf.settings with application settings."""
      pass

  # utils.py
  
  def get_model(module_name):
      """Imports the model module_name."""
      mod = __import__(module_name)
      return mod

  # models.py
  from django.db import models
  
  class BookBase(models.Model):
      title = models.CharField(max_length=100)
      
      class Meta:
          abstract = True

  # default_models.py
  # Is it loaded with syncdb? Definitly no.
  class Book(BookBase):
      pass

  # urls.py
  from django.conf import settings
  
  from library.utils import get_model
  
  Book = get_model(settings.LIBRARY_BOOK_MODEL)
  
  urlpatterns = ('',
    url('^books/$', ListView.as_view(model=Book)), name='book_list'),
  )

Add methods to managers
-----------------------

Put your managers in `APP/managers.py`

Extend QuerySet and EmptyQuerySet to be able to chain querysets like this::

  Book.objects.filter(title__contains='foo').is_published()

See http://djangosnippets.org/snippets/2117/
or use the following pattern:

.. code-block:: python

  from django.db import models
  from django.db.models.query import QuerySet, EmptyQuerySet
  
  
  class BookEmptyQuerySet(EmptyQuerySet):
      """Specific EmptyQuerySet for Book model"""
      def is_published(self, *args, **kwargs):
          """Always returns BookEmptyQuerySet."""
          return self
  
  
  class BookQuerySet(QuerySet):
      """Specific QuerySet for Book model"""
      def is_published(self, is_published=True, *args, **kwargs):
          """Return books which are on loan if is_is_published is False."""
          clone = self.filter(is_on_loan=not is_published, *args, **kwargs)
          return clone
      
      def none(self):
          """
          Returns an empty QuerySet.
          """
          return self._clone(klass=BookEmptyQuerySet)
  
  
  class BookManager(models.Manager):
      """Specific manager for Book model"""
      def get_empty_query_set(self):
          return BookEmptyQuerySet(self.model, using=self._db)
      
      def get_query_set(self):
          return BookQuerySet(self.model, using=self._db)
      
      def is_published(self, *args, **kwargs):
          """Proxy to queryset"""
          return self.get_query_set().is_published(*args, **kwargs)

Importing inside the reusable app
=================================

The best solution for standalone apps is to write it with the assumption that they’ll live directly on the Python path, and so can use absolute imports stemming off the app name; e.g., `APP/views.py` should be able to assume that from APP import models will work. This also makes life easier for the end user of the app, because it’s quite a bit simpler to use a single installed copy of the app in multiple projects.

Forms
=====

Put your forms in `django-APP/APP/forms.py`

Reusable Templates
==================

Put your templates in `django-APP/APP/templates/APP/template.html`

Placed your templatetags in `APP/templatetags/APP_tags.py`
Name your templatetags with the name of the APP they belongs to.

.. code-block:: python

    {% load APP_tags %}
    
There have also been a couple proposals for changing how Django looks for tag libraries, so that it’d become possible to do things like `{% load myapp.foo %}`, and that might be something worth looking into since it’d make this much easier.

Use blocks or snippets (?) for portions of content
--------------------------------------------------

As an example, book_list.html includes snippets/book_overview.html where the latter displays book title, summary...
=> Someone who wants to display the overview a custom template of his own can include the snippet.
=> Someone who wants to override the snippet for a project can do it in the project's templates directory (beware of the order of settings.TEMPLATE_LOADERS).

Locales
=======

Put your translations in `APP/locale/{{language_code}}/django.po`

Middleware
==========

Put your middleware in `APP/middleware.py`

Context Processors
==================

Put your context processors in `APP/context_processors.py`

Feeds
=====

Pour your feeds in `APP/feeds.py`

URLs
====

Use the permalink decorator when using `get_absolute_url()` in models.

.. coding-block:: python

    # urls.py
    (r'^people/(\d+)/$', 'people.views.details'),
   
    # models.py
    from django.db import models
    
    @models.permalink
    def get_absolute_url(self):
        return ('myapp.views.details', [str(self.id)])

When `get_absolute_urls` need to be hardcoded, this should be documentated so end-users will know that they need to use `ABSOLUTE_URL_OVERRIDES <http://docs.djangoproject.com/en/1.3/ref/settings/#absolute-url-overrides>`_.

Document your app
=================

Placed in a docs directory at the same level as the APP directory : django-APP/docs/

You should have a look at `Sphinx-Documentation <http://sphinx.pocoo.org/>`_

This tool helps you to make greats docs.

You can also link your DVCS with `ReadTheDocs <http://readthedocs.org/>`_ so they automatically generate an up-to-date web version of your doc.

Example: http://readthedocs.org/docs/django-floppyforms/en/latest/


Settings merge and default settings
===================================

.. code-block:: python

    # project/settings.py
    MY_APP_CONFIG = {
        'ENABLE_CHUCK_NORRIZ_MODE': True,
    }
    
    # APP/__init__.py
    from django.conf import settings

    app_settings = dict({
        'FOO': 42,
        'ENABLE_CHUCK_NORRIZ_MODE': False,
    }, **getattr(settings, 'MY_APP_CONFIG', {}))

Then to use it just do load it:

.. code-block:: python

    # foo/bar.py
    from APP import app_settings
    print app_settings.get('FOO') # 42
    print app_settings.get('ENABLE_CHUCK_NORRIZ_MODE') # True

http://blog.akei.com/post/4575980188/une-autre-facon-de-gerer-ses-settings-dapplication
un autre exemple : https://bitbucket.org/benoitbryon/django-formrenderingtools/src/e6930c651fa3/djc/formrenderingtools/settings.py, notamment utilisé dans les tests : https://bitbucket.org/benoitbryon/django-formrenderingtools/src/e6930c651fa3/djc/formrenderingtools/tests.py#cl-41

Test your application
=====================

Unified way to run the tests
----------------------------

* Buildout (djangorecipe)
* standalone script (the Pinax way) https://github.com/pinax/pinax/blob/master/tests/runner.py

    Pourquoi pas tout simplement manage.py test app ?
    (avec des settings spécifiques aux tests,
    genre ./manage.py test --settings=setting_test app ?)
    
Usually override `Django TestCase <http://docs.djangoproject.com/en/dev/topics/testing/#testcase>`_

Test the models
---------------

The your model creation and deletion.

.. code-block:: python

    """
    Testing the creation and deletion of Model
    
    >>> from APP.models import MyModel
    >>> obj = MyModel.objects.create(title='Toto')
    >>> MyModel.objects.all()
    [<MyModel: Toto>]
    >>> obj.delete()
    """

Put your fixtures in files prefixed with app name `APP/fixtures/APP_testdata.json`.
Avoid `initial_data.json`.

Test your views
--------------

Test the response status_code from the django client mock. For non-regression.

.. code-block:: python

    """
    Testing the views
    
    >>> from django.test import Client
    >>> from django.core.urlresolvers import reverse
    
    >>> client = Client()
    >>> response = client.get(reverse('view_name'))
    >>> response.status_code
    200
    """


Packaging for Pypi
==================

Package simple apps and write comments about it
Do it again

Disutils2 tutorial : http://distutils2.notmyidea.org/tutorial.html

.. code-block :: python

      classifiers  = ['Programming Language :: Python :: 2.5',
                      'Environment :: Web Environment',
                      'Framework :: Django',
                      ...]


How to package (with distutils2)
--------------------------------

1) Install distutils2 (or use Python 3.3)
2) Go on your app root
3) Launch "pysetup run mkcfg" and follow the wizard
4) Try to install it elsewhere

How to upload distribution
--------------------------

1) Run "pysetup run sdist" (create an archive of your app)
2) Run "pysetup run register" (to register the project on pypi)
3) Run "pysetup run upload" (to send archive to pypi)

Links
=====
Django Pluggable App : https://bitbucket.org/bearstech/djangopluggableapp
