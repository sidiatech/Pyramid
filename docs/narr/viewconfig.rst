.. _view_config_chapter:

.. _view_configuration:

.. _view_lookup:

View Configuration
==================

.. index::
   single: view lookup

:term:`View lookup` is the :app:`Pyramid` subsystem responsible for finding and
invoking a :term:`view callable`.  :term:`View configuration` controls how
:term:`view lookup` operates in your application.  During any given request,
view configuration information is compared against request data by the view
lookup subsystem in order to find the "best" view callable for that request.

In earlier chapters, you have been exposed to a few simple view configuration
declarations without much explanation. In this chapter we will explore the
subject in detail.

.. index::
   pair: resource; mapping to view callable
   pair: URL pattern; mapping to view callable

Mapping a Resource or URL Pattern to a View Callable
----------------------------------------------------

A developer makes a :term:`view callable` available for use within a
:app:`Pyramid` application via :term:`view configuration`.  A view
configuration associates a view callable with a set of statements that
determine the set of circumstances which must be true for the view callable to
be invoked.

A view configuration statement is made about information present in the
:term:`context` resource (or exception) and the :term:`request`.

View configuration is performed in one of two ways:

- By running a :term:`scan` against application source code which has a
  :class:`pyramid.view.view_config` decorator attached to a Python object as
  per :ref:`mapping_views_using_a_decorator_section`.

- By using the :meth:`pyramid.config.Configurator.add_view` method as per
  :ref:`mapping_views_using_imperative_config_section`.

.. index::
   single: view configuration parameters

.. _view_configuration_parameters:

View Configuration Parameters
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All forms of view configuration accept the same general types of arguments.

Many arguments supplied during view configuration are :term:`view predicate`
arguments.  View predicate arguments used during view configuration are used to
narrow the set of circumstances in which :term:`view lookup` will find a
particular view callable.

:term:`View predicate` attributes are an important part of view configuration
that enables the :term:`view lookup` subsystem to find and invoke the
appropriate view.  The greater the number of predicate attributes possessed by
a view's configuration, the more specific the circumstances need to be before
the registered view callable will be invoked.  The fewer the number of
predicates which are supplied to a particular view configuration, the more
likely it is that the associated view callable will be invoked.  A view with
five predicates will always be found and evaluated before a view with two, for
example.

This does not mean however, that :app:`Pyramid` "stops looking" when it finds a
view registration with predicates that don't match.  If one set of view
predicates does not match, the "next most specific" view (if any) is consulted
for predicates, and so on, until a view is found, or no view can be matched up
with the request.  The first view with a set of predicates all of which match
the request environment will be invoked.

If no view can be found with predicates which allow it to be matched up with
the request, :app:`Pyramid` will return an error to the user's browser,
representing a "not found" (404) page.  See :ref:`changing_the_notfound_view`
for more information about changing the default :term:`Not Found View`.

Other view configuration arguments are non-predicate arguments.  These tend to
modify the response of the view callable or prevent the view callable from
being invoked due to an authorization policy.  The presence of non-predicate
arguments in a view configuration does not narrow the circumstances in which
the view callable will be invoked.

.. _nonpredicate_view_args:

Non-Predicate Arguments
+++++++++++++++++++++++

``permission``
  The name of a :term:`permission` that the user must possess in order to
  invoke the :term:`view callable`.  See :ref:`view_security_section` for more
  information about view security and permissions.

  If ``permission`` is not supplied, no permission is registered for this view
  (it's accessible by any caller).

``attr``
  The view machinery defaults to using the ``__call__`` method of the
  :term:`view callable` (or the function itself, if the view callable is a
  function) to obtain a response.  The ``attr`` value allows you to vary the
  method attribute used to obtain the response.  For example, if your view was
  a class, and the class has a method named ``index`` and you wanted to use
  this method instead of the class's ``__call__`` method to return the
  response, you'd say ``attr="index"`` in the view configuration for the view.
  This is most useful when the view definition is a class.

  If ``attr`` is not supplied, ``None`` is used (implying the function itself
  if the view is a function, or the ``__call__`` callable attribute if the view
  is a class).

``renderer``
  Denotes the :term:`renderer` implementation which will be used to construct a
  :term:`response` from the associated view callable's return value.

  .. seealso:: See also :ref:`renderers_chapter`.

  This is either a single string term (e.g., ``json``) or a string implying a
  path or :term:`asset specification` (e.g., ``templates/views.pt``) naming a
  :term:`renderer` implementation.  If the ``renderer`` value does not contain
  a dot (``.``), the specified string will be used to look up a renderer
  implementation, and that renderer implementation will be used to construct a
  response from the view return value.  If the ``renderer`` value contains a
  dot (``.``), the specified term will be treated as a path, and the filename
  extension of the last element in the path will be used to look up the
  renderer implementation, which will be passed the full path.

  When the renderer is a path???although a path is usually just a simple relative
  pathname (e.g., ``templates/foo.pt``, implying that a template named "foo.pt"
  is in the "templates" directory relative to the directory of the current
  :term:`package`)???the path can be absolute, starting with a slash on Unix or a
  drive letter prefix on Windows.  The path can alternatively be a :term:`asset
  specification` in the form ``some.dotted.package_name:relative/path``, making
  it possible to address template assets which live in a separate package.

  The ``renderer`` attribute is optional.  If it is not defined, the "null"
  renderer is assumed (no rendering is performed and the value is passed back
  to the upstream :app:`Pyramid` machinery unchanged).  Note that if the view
  callable itself returns a :term:`response` (see :ref:`the_response`), the
  specified renderer implementation is never called.

``http_cache``
  When you supply an ``http_cache`` value to a view configuration, the
  ``Expires`` and ``Cache-Control`` headers of a response generated by the
  associated view callable are modified.  The value for ``http_cache`` may be
  one of the following:

  - A nonzero integer.  If it's a nonzero integer, it's treated as a number of
    seconds.  This number of seconds will be used to compute the ``Expires``
    header and the ``Cache-Control: max-age`` parameter of responses to
    requests which call this view.  For example: ``http_cache=3600`` instructs
    the requesting browser to 'cache this response for an hour, please'.

  - A ``datetime.timedelta`` instance.  If it's a ``datetime.timedelta``
    instance, it will be converted into a number of seconds, and that number of
    seconds will be used to compute the ``Expires`` header and the
    ``Cache-Control: max-age`` parameter of responses to requests which call
    this view.  For example: ``http_cache=datetime.timedelta(days=1)``
    instructs the requesting browser to 'cache this response for a day,
    please'.

  - Zero (``0``).  If the value is zero, the ``Cache-Control`` and ``Expires``
    headers present in all responses from this view will be composed such that
    client browser cache (and any intermediate caches) are instructed to never
    cache the response.

  - A two-tuple.  If it's a two-tuple (e.g., ``http_cache=(1,
    {'public':True})``), the first value in the tuple may be a nonzero integer
    or a ``datetime.timedelta`` instance. In either case this value will be
    used as the number of seconds to cache the response.  The second value in
    the tuple must be a dictionary.  The values present in the dictionary will
    be used as input to the ``Cache-Control`` response header.  For example:
    ``http_cache=(3600, {'public':True})`` means 'cache for an hour, and add
    ``public`` to the Cache-Control header of the response'.  All keys and
    values supported by the ``webob.cachecontrol.CacheControl`` interface may
    be added to the dictionary.  Supplying ``{'public':True}`` is equivalent to
    calling ``response.cache_control.public = True``.

  Providing a non-tuple value as ``http_cache`` is equivalent to calling
  ``response.cache_expires(value)`` within your view's body.

  Providing a two-tuple value as ``http_cache`` is equivalent to calling
  ``response.cache_expires(value[0], **value[1])`` within your view's body.

  If you wish to avoid influencing the ``Expires`` header, and instead wish to
  only influence ``Cache-Control`` headers, pass a tuple as ``http_cache`` with
  the first element of ``None``, i.e., ``(None, {'public':True})``.


``require_csrf``

  CSRF checks will affect any request method that is not defined as a "safe"
  method by RFC2616. In practice this means that GET, HEAD, OPTIONS, and TRACE
  methods will pass untouched and all others methods will require CSRF. This
  option is used in combination with the ``pyramid.require_default_csrf``
  setting to control which request parameters are checked for CSRF tokens.

  This feature requires a configured :term:`session factory`.

  If this option is set to ``True`` then CSRF checks will be enabled for POST
  requests to this view. The required token will be whatever was specified by
  the ``pyramid.require_default_csrf`` setting, or will fallback to
  ``csrf_token``.

  If this option is set to a string then CSRF checks will be enabled and it
  will be used as the required token regardless of the
  ``pyramid.require_default_csrf`` setting.

  If this option is set to ``False`` then CSRF checks will be disabled
  regardless of the ``pyramid.require_default_csrf`` setting.

  In addition, if this option is set to ``True`` or a string then CSRF origin
  checking will be enabled.

  See :ref:`auto_csrf_checking` for more information.

  .. versionadded:: 1.7

``wrapper``
  The :term:`view name` of a different :term:`view configuration` which will
  receive the response body of this view as the ``request.wrapped_body``
  attribute of its own :term:`request`, and the :term:`response` returned by
  this view as the ``request.wrapped_response`` attribute of its own request.
  Using a wrapper makes it possible to "chain" views together to form a
  composite response.  The response of the outermost wrapper view will be
  returned to the user.  The wrapper view will be found as any view is found.
  See :ref:`view_lookup`.  The "best" wrapper view will be found based on the
  lookup ordering. "Under the hood" this wrapper view is looked up via
  ``pyramid.view.render_view_to_response(context, request,
  'wrapper_viewname')``. The context and request of a wrapper view is the same
  context and request of the inner view.

  If ``wrapper`` is not supplied, no wrapper view is used.

``decorator``
  A :term:`dotted Python name` to a function (or the function itself) which
  will be used to decorate the registered :term:`view callable`.  The decorator
  function will be called with the view callable as a single argument.  The
  view callable it is passed will accept ``(context, request)``.  The decorator
  must return a replacement view callable which also accepts ``(context,
  request)``. The ``decorator`` may also be an iterable of decorators, in which
  case they will be applied one after the other to the view, in reverse order.
  For example::

    @view_config(..., decorator=(decorator2, decorator1))
    def myview(request):
      ...

  Is similar to decorating the view callable directly::

    @view_config(...)
    @decorator2
    @decorator1
    def myview(request):
      ...

  An important distinction is that each decorator will receive a response
  object implementing :class:`pyramid.interfaces.IResponse` instead of the
  raw value returned from the view callable. All decorators in the chain must
  return a response object or raise an exception:

  .. code-block:: python

      def log_timer(wrapped):
          def wrapper(context, request):
              start = time.time()
              response = wrapped(context, request)
              duration = time.time() - start
              response.headers['X-View-Time'] = '%.3f' % (duration,)
              log.info('view took %.3f seconds', duration)
              return response
          return wrapper

``mapper``
  A Python object or :term:`dotted Python name` which refers to a :term:`view
  mapper`, or ``None``.  By default it is ``None``, which indicates that the
  view should use the default view mapper.  This plug-point is useful for
  Pyramid extension developers, but it's not very useful for "civilians" who
  are just developing stock Pyramid applications. Pay no attention to the man
  behind the curtain.

``accept``
  A :term:`media type` that will be matched against the ``Accept`` HTTP request header.
  If this value is specified, it must be a specific media type such as ``text/html`` or ``text/html;level=1``.
  If the media type is acceptable by the ``Accept`` header of the request, or if the ``Accept`` header isn't set at all in the request, this predicate will match.
  If this does not match the ``Accept`` header of the request, view matching continues.

  If ``accept`` is not specified, the ``HTTP_ACCEPT`` HTTP header is not taken into consideration when deciding whether or not to invoke the associated view callable.

  The ``accept`` argument is technically not a predicate and does not support wrapping with :func:`pyramid.config.not_`.

  See :ref:`accept_content_negotiation` for more information.

  .. versionchanged:: 1.10

      Specifying a media range is deprecated and will be removed in :app:`Pyramid` 2.0.
      Use explicit media types to avoid any ambiguities in content negotiation.

  .. versionchanged:: 2.0

      Removed support for media ranges.

``exception_only``

  When this value is ``True``, the ``context`` argument must be a subclass of
  ``Exception``. This flag indicates that only an :term:`exception view` should
  be created, and that this view should not match if the traversal
  :term:`context` matches the ``context`` argument. If the ``context`` is a
  subclass of ``Exception`` and this value is ``False`` (the default), then a
  view will be registered to match the traversal :term:`context` as well.

  .. versionadded:: 1.8

.. _predicate_view_args:

Predicate Arguments
+++++++++++++++++++

These arguments modify view lookup behavior. In general the more predicate
arguments that are supplied, the more specific and narrower the usage of the
configured view.

``name``
  The :term:`view name` required to match this view callable.  A ``name``
  argument is typically only used when your application uses :term:`traversal`.
  Read :ref:`traversal_chapter` to understand the concept of a view name.

  If ``name`` is not supplied, the empty string is used (implying the default
  view).

``context``
  An object representing a Python class of which the :term:`context` resource
  must be an instance *or* the :term:`interface` that the :term:`context`
  resource must provide in order for this view to be found and called.  This
  predicate is true when the :term:`context` resource is an instance of the
  represented class or if the :term:`context` resource provides the represented
  interface; it is otherwise false.

  It is possible to pass an exception class as the context if your context may
  subclass an exception. In this case *two* views will be registered. One
  will match normal incoming requests, and the other will match as an
  :term:`exception view` which only occurs when an exception is raised during
  the normal request processing pipeline.

  If ``context`` is not supplied, the value ``None``, which matches any
  resource, is used.

``route_name``
  If ``route_name`` is supplied, the view callable will be invoked only when
  the named route has matched.

  This value must match the ``name`` of a :term:`route configuration`
  declaration (see :ref:`urldispatch_chapter`) that must match before this view
  will be called.  Note that the ``route`` configuration referred to by
  ``route_name`` will usually have a ``*traverse`` token in the value of its
  ``pattern``, representing a part of the path that will be used by
  :term:`traversal` against the result of the route's :term:`root factory`.

  If ``route_name`` is not supplied, the view callable will only have a chance
  of being invoked if no other route was matched. This is when the
  request/context pair found via :term:`resource location` does not indicate it
  matched any configured route.

``request_type``
  This value should be an :term:`interface` that the :term:`request` must
  provide in order for this view to be found and called.

  If ``request_type`` is not supplied, the value ``None`` is used, implying any
  request type.

  *This is an advanced feature, not often used by "civilians"*.

``request_method``
  This value can be either a string (such as ``"GET"``, ``"POST"``,
  ``"PUT"``, ``"DELETE"``, ``"HEAD"``, or ``"OPTIONS"``) representing an HTTP
  ``REQUEST_METHOD`` or a tuple containing one or more of these strings.  A
  view declaration with this argument ensures that the view will only be called
  when the ``method`` attribute of the request (i.e., the ``REQUEST_METHOD`` of
  the WSGI environment) matches a supplied value.

  .. versionchanged:: 1.4
    The use of ``"GET"`` also implies that the view will respond to ``"HEAD"``.

  If ``request_method`` is not supplied, the view will be invoked regardless of
  the ``REQUEST_METHOD`` of the :term:`WSGI` environment.

``request_param``
  This argument can be any string or a sequence of strings.  A view declaration
  with this argument ensures that the view will only be called when the
  :term:`request` has a key in the ``request.params`` dictionary (an HTTP
  ``GET`` or ``POST`` variable) that has a name which matches the supplied
  value.

  If any value supplied has an ``=`` sign in it, e.g.,
  ``request_param="foo=123"``, then the key (``foo``) must both exist in the
  ``request.params`` dictionary, *and* the value must match the right hand side
  of the expression (``123``) for the view to "match" the current request.

  If ``request_param`` is not supplied, the view will be invoked without
  consideration of keys and values in the ``request.params`` dictionary.

``match_param``
  This argument may be either a single string of the format "key=value" or a tuple
  containing one or more of these strings.

  This argument ensures that the view will only be called when the
  :term:`request` has key/value pairs in its :term:`matchdict` that equal those
  supplied in the predicate.  For example, ``match_param="action=edit"`` would
  require the ``action`` parameter in the :term:`matchdict` match the right
  hand side of the expression (``edit``) for the view to "match" the current
  request.

  If the ``match_param`` is a tuple, every key/value pair must match for the
  predicate to pass.

  If ``match_param`` is not supplied, the view will be invoked without
  consideration of the keys and values in ``request.matchdict``.

  .. versionadded:: 1.2

``containment``
  This value should be a reference to a Python class or :term:`interface` that
  a parent object in the context resource's :term:`lineage` must provide in
  order for this view to be found and called.  The resources in your resource
  tree must be "location-aware" to use this feature.

  If ``containment`` is not supplied, the interfaces and classes in the lineage
  are not considered when deciding whether or not to invoke the view callable.

  See :ref:`location_aware` for more information about location-awareness.

``xhr``
  This value should be either ``True`` or ``False``.  If this value is
  specified and is ``True``, the :term:`WSGI` environment must possess an
  ``HTTP_X_REQUESTED_WITH`` header (i.e., ``X-Requested-With``) that has the
  value ``XMLHttpRequest`` for the associated view callable to be found and
  called.  This is useful for detecting AJAX requests issued from jQuery,
  Prototype, and other Javascript libraries.

  If ``xhr`` is not specified, the ``HTTP_X_REQUESTED_WITH`` HTTP header is not
  taken into consideration when deciding whether or not to invoke the
  associated view callable.

``header``
  This param matches one or more HTTP header names or header name/value pairs.
  If specified, this param must be a string or a sequence of strings,
  each string being a header name or a ``headername:headervalue`` pair.

  - Each string specified as a bare header name without a value (for example
    ``If-Modified-Since``) will match a request if it contains an HTTP header
    with that same name.  The case of the name is not significant, and the
    header may have any value in the request.

  - Each string specified as a name/value pair (that is, if it contains a ``:``
    (colon), like ``User-Agent:Mozilla/.*``) will match a request only if it
    contains an HTTP header with the requested name (ignoring case, so
    ``User-Agent`` or ``user-agent`` would both match), *and* the value of the
    HTTP header matches the value requested (``Mozilla/.*`` in our example).
    The value portion is interpreted as a regular expression.

  The view will only be invoked if all strings are matching.

  If ``header`` is not specified, the composition, presence, or absence of HTTP
  headers is not taken into consideration when deciding whether or not to
  invoke the associated view callable.

``path_info``
  This value represents a regular expression pattern that will be tested
  against the ``PATH_INFO`` WSGI environment variable to decide whether or not
  to call the associated view callable.  If the regex matches, this predicate
  will be ``True``.

  If ``path_info`` is not specified, the WSGI ``PATH_INFO`` is not taken into
  consideration when deciding whether or not to invoke the associated view
  callable.

``physical_path``
  If specified, this value should be a string or a tuple representing the
  :term:`physical path` of the context found via traversal for this predicate
  to match as true.  For example, ``physical_path='/'``,
  ``physical_path='/a/b/c'``, or ``physical_path=('', 'a', 'b', 'c')``.  This
  is not a path prefix match or a regex, but a whole-path match.  It's useful
  when you want to always potentially show a view when some object is traversed
  to, but you can't be sure about what kind of object it will be, so you can't
  use the ``context`` predicate.  The individual path elements between slash
  characters or in tuple elements should be the Unicode representation of the
  name of the resource and should not be encoded in any way.

  .. versionadded:: 1.4a3

``is_authenticated``
  This value, if specified, must be either ``True`` or ``False``.
  If it is specified and ``True``, only a request from an authenticated user, as
  determined by the :term:`security policy` in use, will satisfy the predicate.
  If it is specified and ``False``, only a request from a user who is not
  authenticated will satisfy the predicate.

  .. versionadded:: 2.0

``effective_principals``
  If specified, this value should be a :term:`principal` identifier or a
  sequence of principal identifiers.  If the
  :meth:`pyramid.request.Request.effective_principals` method indicates that
  every principal named in the argument list is present in the current request,
  this predicate will return True; otherwise it will return False.  For
  example: ``effective_principals=pyramid.authorization.Authenticated`` or
  ``effective_principals=('fred', 'group:admins')``.

  .. versionadded:: 1.4a4

  .. deprecated:: 2.0
      Use ``is_authenticated`` or a custom predicate.

``custom_predicates``
  If ``custom_predicates`` is specified, it must be a sequence of references to
  custom predicate callables.   Custom predicates can be combined with
  predefined predicates as necessary.  Each custom predicate callable should
  accept two arguments, ``context`` and ``request``, and should return either
  ``True`` or ``False`` after doing arbitrary evaluation of the context
  resource and/or the request.  If all callables return ``True``, the
  associated view callable will be considered viable for a given request.
  This parameter is kept around for backward compatibility.

  .. deprecated:: 1.5
      See section below for new-style custom predicates.

``**predicates``
  Extra keyword parameters are used to invoke custom predicates, defined
  in your app or by third-party packages extending Pyramid and registered via
  :meth:`pyramid.config.Configurator.add_view_predicate`.  Use custom predicates
  when no set of predefined predicates do what you need.  See
  :ref:`view_and_route_predicates` for more information about custom
  predicates.

  .. versionadded:: 1.4a1

Inverting Predicate Values
++++++++++++++++++++++++++

You can invert the meaning of any predicate value by wrapping it in a call to
:class:`pyramid.config.not_`.

.. code-block:: python
    :linenos:

    from pyramid.config import not_

    config.add_view(
        'mypackage.views.my_view',
        route_name='ok',
        request_method=not_('POST')
        )

The above example will ensure that the view is called if the request method is
*not* ``POST``, at least if no other view is more specific.

This technique of wrapping a predicate value in ``not_`` can be used anywhere
predicate values are accepted:

- :meth:`pyramid.config.Configurator.add_view`

- :meth:`pyramid.view.view_config`

.. versionadded:: 1.5


.. index::
   single: view_config decorator

.. _mapping_views_using_a_decorator_section:

Adding View Configuration Using the ``@view_config`` Decorator
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. warning::

   Using this feature tends to slow down application startup slightly, as more
   work is performed at application startup to scan for view configuration
   declarations.  For maximum startup performance, use the view configuration
   method described in :ref:`mapping_views_using_imperative_config_section`
   instead.

The :class:`~pyramid.view.view_config` decorator can be used to associate
:term:`view configuration` information with a function, method, or class that
acts as a :app:`Pyramid` view callable.

Here's an example of the :class:`~pyramid.view.view_config` decorator that
lives within a :app:`Pyramid` application module ``views.py``:

.. code-block:: python
    :linenos:

    from resources import MyResource
    from pyramid.view import view_config
    from pyramid.response import Response

    @view_config(route_name='ok', request_method='POST', permission='read')
    def my_view(request):
        return Response('OK')

Using this decorator as above replaces the need to add this imperative
configuration stanza:

.. code-block:: python
    :linenos:

    config.add_view('mypackage.views.my_view', route_name='ok',
                    request_method='POST', permission='read')

All arguments to ``view_config`` may be omitted.  For example:

.. code-block:: python
    :linenos:

    from pyramid.response import Response
    from pyramid.view import view_config

    @view_config()
    def my_view(request):
        """ My view """
        return Response()

Such a registration as the one directly above implies that the view name will
be ``my_view``, registered with a ``context`` argument that matches any
resource type, using no permission, registered against requests with any
request method, request type, request param, route name, or containment.

The mere existence of a ``@view_config`` decorator doesn't suffice to perform
view configuration.  All that the decorator does is "annotate" the function
with your configuration declarations, it doesn't process them. To make
:app:`Pyramid` process your :class:`pyramid.view.view_config` declarations, you
*must* use the ``scan`` method of a :class:`pyramid.config.Configurator`:

.. code-block:: python
    :linenos:

    # config is assumed to be an instance of the
    # pyramid.config.Configurator class
    config.scan()

Please see :ref:`decorations_and_code_scanning` for detailed information about
what happens when code is scanned for configuration declarations resulting from
use of decorators like :class:`~pyramid.view.view_config`.

See :ref:`configuration_module` for additional API arguments to the
:meth:`~pyramid.config.Configurator.scan` method.  For example, the method
allows you to supply a ``package`` argument to better control exactly *which*
code will be scanned.

All arguments to the :class:`~pyramid.view.view_config` decorator mean
precisely the same thing as they would if they were passed as arguments to the
:meth:`pyramid.config.Configurator.add_view` method save for the ``view``
argument.  Usage of the :class:`~pyramid.view.view_config` decorator is a form
of :term:`declarative configuration`, while
:meth:`pyramid.config.Configurator.add_view` is a form of :term:`imperative
configuration`.  However, they both do the same thing.

.. index::
   single: view_config placement

.. _view_config_placement:

``@view_config`` Placement
++++++++++++++++++++++++++

A :class:`~pyramid.view.view_config` decorator can be placed in various points
in your application.

If your view callable is a function, it may be used as a function decorator:

.. code-block:: python
    :linenos:

    from pyramid.view import view_config
    from pyramid.response import Response

    @view_config(route_name='edit')
    def edit(request):
        return Response('edited!')

If your view callable is a class, the decorator can also be used as a class
decorator. All the arguments to the decorator are the same when applied against
a class as when they are applied against a function.  For example:

.. code-block:: python
    :linenos:

    from pyramid.response import Response
    from pyramid.view import view_config

    @view_config(route_name='hello')
    class MyView(object):
        def __init__(self, request):
            self.request = request

        def __call__(self):
            return Response('hello')

More than one :class:`~pyramid.view.view_config` decorator can be stacked on
top of any number of others.  Each decorator creates a separate view
registration.  For example:

.. code-block:: python
    :linenos:

    from pyramid.view import view_config
    from pyramid.response import Response

    @view_config(route_name='edit')
    @view_config(route_name='change')
    def edit(request):
        return Response('edited!')

This registers the same view under two different names.

The decorator can also be used against a method of a class:

.. code-block:: python
    :linenos:

    from pyramid.response import Response
    from pyramid.view import view_config

    class MyView(object):
        def __init__(self, request):
            self.request = request

        @view_config(route_name='hello')
        def amethod(self):
            return Response('hello')

When the decorator is used against a method of a class, a view is registered
for the *class*, so the class constructor must accept an argument list in one
of two forms: either a single argument, ``request``, or two arguments,
``context, request``.

The method which is decorated must return a :term:`response`.

Using the decorator against a particular method of a class is equivalent to
using the ``attr`` parameter in a decorator attached to the class itself. For
example, the above registration implied by the decorator being used against the
``amethod`` method could be written equivalently as follows:

.. code-block:: python
    :linenos:

    from pyramid.response import Response
    from pyramid.view import view_config

    @view_config(attr='amethod', route_name='hello')
    class MyView(object):
        def __init__(self, request):
            self.request = request

        def amethod(self):
            return Response('hello')


.. index::
   single: add_view

.. _mapping_views_using_imperative_config_section:

Adding View Configuration Using :meth:`~pyramid.config.Configurator.add_view`
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The :meth:`pyramid.config.Configurator.add_view` method within
:ref:`configuration_module` is used to configure a view "imperatively" (without
a :class:`~pyramid.view.view_config` decorator).  The arguments to this method
are very similar to the arguments that you provide to the
:class:`~pyramid.view.view_config` decorator.  For example:

.. code-block:: python
    :linenos:

    from pyramid.response import Response

    def hello_world(request):
        return Response('hello!')

    # config is assumed to be an instance of the
    # pyramid.config.Configurator class
    config.add_view(hello_world, route_name='hello')

The first argument, a :term:`view callable`, is the only required argument. It
must either be a Python object which is the view itself or a :term:`dotted
Python name` to such an object. In the above example, the ``view callable`` is
the ``hello_world`` function.

When you use only :meth:`~pyramid.config.Configurator.add_view` to add view
configurations, you don't need to issue a :term:`scan` in order for the view
configuration to take effect.

.. index::
   single: view_defaults class decorator

.. _view_defaults:

``@view_defaults`` Class Decorator
----------------------------------

.. versionadded:: 1.3

If you use a class as a view, you can use the
:class:`pyramid.view.view_defaults` class decorator on the class to provide
defaults to the view configuration information used by every ``@view_config``
decorator that decorates a method of that class.

For instance, if you've got a class that has methods that represent "REST
actions", all of which are mapped to the same route but different request
methods, instead of this:

.. code-block:: python
    :linenos:

    from pyramid.view import view_config
    from pyramid.response import Response

    class RESTView(object):
        def __init__(self, request):
            self.request = request

        @view_config(route_name='rest', request_method='GET')
        def get(self):
            return Response('get')

        @view_config(route_name='rest', request_method='POST')
        def post(self):
            return Response('post')

        @view_config(route_name='rest', request_method='DELETE')
        def delete(self):
            return Response('delete')

You can do this:

.. code-block:: python
    :linenos:

    from pyramid.view import view_defaults
    from pyramid.view import view_config
    from pyramid.response import Response

    @view_defaults(route_name='rest')
    class RESTView(object):
        def __init__(self, request):
            self.request = request

        @view_config(request_method='GET')
        def get(self):
            return Response('get')

        @view_config(request_method='POST')
        def post(self):
            return Response('post')

        @view_config(request_method='DELETE')
        def delete(self):
            return Response('delete')

In the above example, we were able to take the ``route_name='rest'`` argument
out of the call to each individual ``@view_config`` statement because we used a
``@view_defaults`` class decorator to provide the argument as a default to each
view method it possessed.

Arguments passed to ``@view_config`` will override any default passed to
``@view_defaults``.

The ``view_defaults`` class decorator can also provide defaults to the
:meth:`pyramid.config.Configurator.add_view` directive when a decorated class
is passed to that directive as its ``view`` argument.  For example, instead of
this:

.. code-block:: python
    :linenos:

    from pyramid.response import Response
    from pyramid.config import Configurator

    class RESTView(object):
        def __init__(self, request):
            self.request = request

        def get(self):
            return Response('get')

        def post(self):
            return Response('post')

        def delete(self):
            return Response('delete')

    def main(global_config, **settings):
        config = Configurator()
        config.add_route('rest', '/rest')
        config.add_view(
            RESTView, route_name='rest', attr='get', request_method='GET')
        config.add_view(
            RESTView, route_name='rest', attr='post', request_method='POST')
        config.add_view(
            RESTView, route_name='rest', attr='delete', request_method='DELETE')
        return config.make_wsgi_app()

To reduce the amount of repetition in the ``config.add_view`` statements, we
can move the ``route_name='rest'`` argument to a ``@view_defaults`` class
decorator on the ``RESTView`` class:

.. code-block:: python
    :linenos:

    from pyramid.view import view_defaults
    from pyramid.response import Response
    from pyramid.config import Configurator

    @view_defaults(route_name='rest')
    class RESTView(object):
        def __init__(self, request):
            self.request = request

        def get(self):
            return Response('get')

        def post(self):
            return Response('post')

        def delete(self):
            return Response('delete')

    def main(global_config, **settings):
        config = Configurator()
        config.add_route('rest', '/rest')
        config.add_view(RESTView, attr='get', request_method='GET')
        config.add_view(RESTView, attr='post', request_method='POST')
        config.add_view(RESTView, attr='delete', request_method='DELETE')
        return config.make_wsgi_app()

:class:`pyramid.view.view_defaults` accepts the same set of arguments that
:class:`pyramid.view.view_config` does, and they have the same meaning.  Each
argument passed to ``view_defaults`` provides a default for the view
configurations of methods of the class it's decorating.

Normal Python inheritance rules apply to defaults added via ``view_defaults``.
For example:

.. code-block:: python
    :linenos:

    @view_defaults(route_name='rest')
    class Foo(object):
        pass

    class Bar(Foo):
        pass

The ``Bar`` class above will inherit its view defaults from the arguments
passed to the ``view_defaults`` decorator of the ``Foo`` class.  To prevent
this from happening, use a ``view_defaults`` decorator without any arguments on
the subclass:

.. code-block:: python
    :linenos:

    @view_defaults(route_name='rest')
    class Foo(object):
        pass

    @view_defaults()
    class Bar(Foo):
        pass

The ``view_defaults`` decorator only works as a class decorator; using it
against a function or a method will produce nonsensical results.

.. index::
   single: view security
   pair: security; view

.. _view_security_section:

Configuring View Security
~~~~~~~~~~~~~~~~~~~~~~~~~

If an :term:`authorization policy` is active, any :term:`permission` attached
to a :term:`view configuration` found during view lookup will be verified. This
will ensure that the currently authenticated user possesses that permission
against the :term:`context` resource before the view function is actually
called.  Here's an example of specifying a permission in a view configuration
using :meth:`~pyramid.config.Configurator.add_view`:

.. code-block:: python
    :linenos:

    # config is an instance of pyramid.config.Configurator

    config.add_route('add', '/add.html', factory='mypackage.Blog')
    config.add_view('myproject.views.add_entry', route_name='add',
                    permission='add')

When an :term:`authorization policy` is enabled, this view will be protected
with the ``add`` permission.  The view will *not be called* if the user does
not possess the ``add`` permission relative to the current :term:`context`.
Instead the :term:`forbidden view` result will be returned to the client as per
:ref:`protecting_views`.

.. index::
   single: debugging not found errors
   single: not found error (debugging)

.. _debug_notfound_section:

:exc:`~pyramid.exceptions.NotFound` Errors
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It's useful to be able to debug :exc:`~pyramid.exceptions.NotFound` error
responses when they occur unexpectedly due to an application registry
misconfiguration.  To debug these errors, use the ``PYRAMID_DEBUG_NOTFOUND``
environment variable or the ``pyramid.debug_notfound`` configuration file
setting.  Details of why a view was not found will be printed to ``stderr``,
and the browser representation of the error will include the same information.
See :ref:`environment_chapter` for more information about how, and where to set
these values.

.. index::
   single: Accept
   single: Accept content negotiation

.. _accept_content_negotiation:

Accept Header Content Negotiation
---------------------------------

The ``accept`` argument to :meth:`pyramid.config.Configurator.add_view` can be used to control :term:`view lookup` by dispatching to different views based on the HTTP ``Accept`` request header.
Consider the example below in which there are three views configured.

.. code-block:: python

    from pyramid.httpexceptions import HTTPNotAcceptable
    from pyramid.view import view_config

    @view_config(accept='application/json', renderer='json')
    @view_config(accept='text/html', renderer='templates/hello.jinja2')
    def myview(request):
        return {
            'name': request.GET.get('name', 'bob'),
        }

    @view_config()
    def myview_unacceptable(request):
        raise HTTPNotAcceptable

Each view relies on the ``Accept`` header to trigger an appropriate response renderer.
The appropriate view is selected here when the client specifies headers such as ``Accept: text/*`` or ``Accept: application/json, text/html;q=0.9`` in which only one of the views matches or it's clear based on the preferences which one should win.
Similarly, if the client specifies a media type that no view is registered to handle, such as ``Accept: text/plain``, it will fall through to ``myview_unacceptable`` and raise ``406 Not Acceptable``.

There are a few cases in which the client may specify an ``Accept`` header such that it's not clear which view should win.
For example:

- ``Accept: */*``.
- More than one acceptable media type with the same quality.
- A missing ``Accept`` header.
- An invalid ``Accept`` header.

In these cases the preferred view is not clearly defined (see :rfc:`7231#section-5.3.2`) and :app:`Pyramid` will select one randomly.
This can be controlled by telling :app:`Pyramid` what the preferred relative ordering is between various media types by using :meth:`pyramid.config.Configurator.add_accept_view_order`.
For example:

.. code-block:: python

    from pyramid.config import Configurator

    def main(global_config, **settings):
        config = Configurator(settings=settings)
        config.add_accept_view_order('text/html')
        config.add_accept_view_order(
            'application/json',
            weighs_more_than='text/html',
        )
        config.scan()
        return config.make_wsgi_app()

Now, the ``application/json`` view should always be preferred in cases where the client wasn't clear.

.. index::
    single: default accept ordering

.. _default_accept_ordering:

Default Accept Ordering
~~~~~~~~~~~~~~~~~~~~~~~

:app:`Pyramid` will always sort multiple views with the same ``(name, context, route_name)`` first by the specificity of the ``accept`` offer.
For any set of media type offers with the same ``type/subtype``, the offers with params will weigh more than the bare ``type/subtype`` offer.
This means that ``text/plain;charset=utf8`` will always be offered before ``text/plain``.

By default, within a given ``type/subtype``, the order of offers is unspecified.
For example, ``text/plain;charset=utf8`` versus ``text/plain;charset=latin1`` are sorted randomly.
Similarly, between media types the order is also unspecified other than the defaults described below.
For example, ``image/jpeg`` versus ``image/png`` versus ``application/pdf``.
In these cases, the ordering may be controlled using :meth:`pyramid.config.Configurator.add_accept_view_order`.
For example, to sort ``text/plain`` higher than ``text/html`` and to prefer a ``charset=utf8`` versus a ``charset=latin-1`` within the ``text/plain`` media type:

.. code-block:: python

    config.add_accept_view_order('text/html')
    config.add_accept_view_order('text/plain;charset=latin-1')
    config.add_accept_view_order('text/plain', weighs_more_than='text/html')
    config.add_accept_view_order('text/plain;charset=utf8', weighs_more_than='text/plain;charset=latin-1')

It is an error to try and sort accept headers across levels of specificity.
You can only sort a ``type/subtype`` against another ``type/subtype``, not against a ``type/subtype;params``.
That ordering is a hard requirement.

By default, :app:`Pyramid` defines a very simple priority ordering for views that prefers human-readable responses over JSON:

- ``text/html``
- ``application/xhtml+xml``
- ``application/xml``
- ``text/xml``
- ``text/plain``
- ``application/json``

API clients tend to be able to specify their desired headers with more control than web browsers, and can specify the correct ``Accept`` value, if necessary.
Therefore, the motivation for this ordering is to optimize for readability.
Media types that are not listed above are ordered randomly during :term:`view lookup` between otherwise-similar views.
The defaults can be overridden using :meth:`pyramid.config.Configurator.add_accept_view_order` as described above.

.. index::
   single: HTTP caching

.. _influencing_http_caching:

Influencing HTTP Caching
------------------------

.. versionadded:: 1.1

When a non-``None`` ``http_cache`` argument is passed to a view configuration,
Pyramid will set ``Expires`` and ``Cache-Control`` response headers in the
resulting response, causing browsers to cache the response data for some time.
See ``http_cache`` in :ref:`nonpredicate_view_args` for the allowable values
and what they mean.

Sometimes it's undesirable to have these headers set as the result of returning
a response from a view, even though you'd like to decorate the view with a view
configuration decorator that has ``http_cache``.  Perhaps there's an
alternative branch in your view code that returns a response that should never
be cacheable, while the "normal" branch returns something that should always be
cacheable.  If this is the case, set the ``prevent_auto`` attribute of the
``response.cache_control`` object to a non-``False`` value.  For example, the
below view callable is configured with a ``@view_config`` decorator that
indicates any response from the view should be cached for 3600 seconds.
However, the view itself prevents caching from taking place unless there's a
``should_cache`` GET or POST variable:

.. code-block:: python

    from pyramid.view import view_config

    @view_config(http_cache=3600)
    def view(request):
        response = Response()
        if 'should_cache' not in request.params:
            response.cache_control.prevent_auto = True
        return response

Note that the ``http_cache`` machinery will overwrite or add to caching headers
you set within the view itself, unless you use ``prevent_auto``.

You can also turn off the effect of ``http_cache`` entirely for the duration of
a Pyramid application lifetime.  To do so, set the
``PYRAMID_PREVENT_HTTP_CACHE`` environment variable or the
``pyramid.prevent_http_cache`` configuration value setting to a true value. For
more information, see :ref:`preventing_http_caching`.

Note that setting ``pyramid.prevent_http_cache`` will have no effect on caching
headers that your application code itself sets.  It will only prevent caching
headers that would have been set by the Pyramid HTTP caching machinery invoked
as the result of the ``http_cache`` argument to view configuration.

.. index::
   pair: view configuration; debugging

.. _debugging_view_configuration:

Debugging View Configuration
----------------------------

See :ref:`displaying_matching_views` for information about how to display
each of the view callables that might match for a given URL.  This can be an
effective way to figure out why a particular view callable is being called
instead of the one you'd like to be called.
