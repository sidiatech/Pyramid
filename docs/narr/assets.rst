.. index::
   single: assets

.. _assets_chapter:

Assets
======

An :term:`asset` is any file contained within a Python :term:`package` which
is *not* a Python source code file.  For example, each of the following is an
asset:

- a :term:`Chameleon` template file contained within a Python package.

- a GIF image file contained within a Python package.

- a CSS file contained within a Python package.

- a JavaScript source file contained within a Python package.

- A directory within a package that does not have an ``__init__.py``
  in it (if it possessed an ``__init__.py`` it would *be* a package).

The use of assets is quite common in most web development projects.  For
example, when you create a :app:`Pyramid` application using one of the
available "paster" templates, as described in :ref:`creating_a_project`, the
directory representing the application contains a Python :term:`package`.
Within that Python package, there are directories full of files which are
assets.  For example, there is a ``templates`` directory which contains
``.pt`` files, and a ``static`` directory which contains ``.css``, ``.js``,
and ``.gif`` files.

.. _understanding_assets:

Understanding Assets
--------------------

Let's imagine you've created a :app:`Pyramid` application that uses a
:term:`Chameleon` ZPT template via the
:func:`pyramid.chameleon_zpt.render_template_to_response` API.  For example,
the application might address the asset named ``templates/some_template.pt``
using that API within a ``views.py`` file inside a ``myapp`` package:

.. ignore-next-block
.. code-block:: python
   :linenos:

   from pyramid.chameleon_zpt import render_template_to_response
   render_template_to_response('templates/some_template.pt')

"Under the hood", when this API is called, :app:`Pyramid` attempts
to make sense out of the string ``templates/some_template.pt``
provided by the developer.  To do so, it first finds the "current"
package.  The "current" package is the Python package in which the
``views.py`` module which contains this code lives.  This would be the
``myapp`` package, according to our example so far.  By resolving the
current package, :app:`Pyramid` has enough information to locate
the actual template file.  These are the elements it needs:

- The *package name* (``myapp``)

- The *asset name* (``templates/some_template.pt``)

:app:`Pyramid` uses the :term:`pkg_resources` API to resolve the package name
and asset name to an absolute (operating-system-specific) file name.  It
eventually passes this resolved absolute filesystem path to the Chameleon
templating engine, which then uses it to load, parse, and execute the
template file.

Package names often contain dots.  For example, ``pyramid`` is a package.
Asset names usually look a lot like relative UNIX file paths.

.. index::
   pair: overriding; assets

.. _overriding_assets_section:

Overriding Assets
-----------------

It can often be useful to override specific assets from "outside" a given
:app:`Pyramid` application.  For example, you may wish to reuse an existing
:app:`Pyramid` application more or less unchanged.  However, some specific
template file owned by the application might have inappropriate HTML, or some
static asset (such as a logo file or some CSS file) might not be appropriate.
You *could* just fork the application entirely, but it's often more
convenient to just override the assets that are inappropriate and reuse the
application "as is".  This is particularly true when you reuse some "core"
application over and over again for some set of customers (such as a CMS
application, or some bug tracking application), and you want to make
arbitrary visual modifications to a particular application deployment without
forking the underlying code.

To this end, :app:`Pyramid` contains a feature that makes it possible to
"override" one asset with one or more other assets.  In support of this
feature, a :term:`Configurator` API exists named
:meth:`pyramid.config.Configurator.override_asset`.  This API allows you to
*override* the following kinds of assets defined in any Python package:

- Individual :term:`Chameleon` templates.

- A directory containing multiple Chameleon templates.

- Individual static files served up by an instance of the
  ``pyramid.view.static`` helper class.

- A directory of static files served up by an instance of the
  ``pyramid.view.static`` helper class.

- Any other asset (or set of assets) addressed by code that uses the
  setuptools :term:`pkg_resources` API.

.. note:: The :term:`ZCML` directive named ``asset`` serves the same purpose
   as the :meth:`pyramid.config.Configurator.override_asset` method.

.. index::
   single: override_asset

.. _override_asset:

The ``override_asset`` API
~~~~~~~~~~~~~~~~~~~~~~~~~~

An individual call to :meth:`pyramid.config.Configurator.override_asset`
can override a single asset.  For example:

.. ignore-next-block
.. code-block:: python
   :linenos:

   config.override_asset(
            to_override='some.package:templates/mytemplate.pt',
            override_with='another.package:othertemplates/anothertemplate.pt')

The string value passed to both ``to_override`` and ``override_with`` sent to
the ``override_asset`` API is called an :term:`asset specification`.  The
colon separator in a specification separates the *package name* from the
*asset name*.  The colon and the following asset name are optional.  If they
are not specified, the override attempts to resolve every lookup into a
package from the directory of another package.  For example:

.. ignore-next-block
.. code-block:: python
   :linenos:

   config.override_asset(to_override='some.package',
                         override_with='another.package')

Individual subdirectories within a package can also be overridden:

.. ignore-next-block
.. code-block:: python
   :linenos:

   config.override_asset(to_override='some.package:templates/',
                         override_with='another.package:othertemplates/')


If you wish to override a directory with another directory, you *must*
make sure to attach the slash to the end of both the ``to_override``
specification and the ``override_with`` specification.  If you fail to
attach a slash to the end of a specification that points to a directory,
you will get unexpected results.

You cannot override a directory specification with a file specification, and
vice versa: a startup error will occur if you try.  You cannot override an
asset with itself: a startup error will occur if you try.

Only individual *package* assets may be overridden.  Overrides will not
traverse through subpackages within an overridden package.  This means that
if you want to override assets for both ``some.package:templates``, and
``some.package.views:templates``, you will need to register two overrides.

The package name in a specification may start with a dot, meaning that
the package is relative to the package in which the configuration
construction file resides (or the ``package`` argument to the
:class:`pyramid.config.Configurator` class construction).
For example:

.. ignore-next-block
.. code-block:: python
   :linenos:

   config.override_asset(to_override='.subpackage:templates/',
                         override_with='another.package:templates/')

Multiple calls to ``override_asset`` which name a shared ``to_override`` but
a different ``override_with`` specification can be "stacked" to form a search
path.  The first asset that exists in the search path will be used; if no
asset exists in the override path, the original asset is used.

Asset overrides can actually override assets other than templates and static
files.  Any software which uses the
:func:`pkg_resources.get_resource_filename`,
:func:`pkg_resources.get_resource_stream` or
:func:`pkg_resources.get_resource_string` APIs will obtain an overridden file
when an override is used.
