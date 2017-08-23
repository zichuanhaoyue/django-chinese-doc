=====================
Model field reference
=====================

.. module:: django.db.models.fields
   :synopsis: Built-in field types.

.. currentmodule:: django.db.models

This document contains all the API references of :class:`Field` including the
`field options`_ and `field types`_ Django offers.

.. seealso::

    If the built-in fields don't do the trick, you can try `django-localflavor
    <https://github.com/django/django-localflavor>`_ (`documentation
    <https://django-localflavor.readthedocs.io/>`_), which contains assorted
    pieces of code that are useful for particular countries and cultures.

    Also, you can easily :doc:`write your own custom model fields
    </howto/custom-model-fields>`.

.. note::

    Technically, these models are defined in :mod:`django.db.models.fields`, but
    for convenience they're imported into :mod:`django.db.models`; the standard
    convention is to use ``from django.db import models`` and refer to fields as
    ``models.<Foo>Field``.

.. _common-model-field-options:

Field options
=============

The following arguments are available to all field types. All are optional.

``null``
--------

.. attribute:: Field.null

If ``True``, Django will store empty values as ``NULL`` in the database. Default
is ``False``.

Avoid using :attr:`~Field.null` on string-based fields such as
:class:`CharField` and :class:`TextField` because empty string values will
always be stored as empty strings, not as ``NULL``. If a string-based field has
``null=True``, that means it has two possible values for "no data": ``NULL``,
and the empty string. In most cases, it's redundant to have two possible values
for "no data;" the Django convention is to use the empty string, not ``NULL``.

For both string-based and non-string-based fields, you will also need to
set ``blank=True`` if you wish to permit empty values in forms, as the
:attr:`~Field.null` parameter only affects database storage
(see :attr:`~Field.blank`).

.. note::

    When using the Oracle database backend, the value ``NULL`` will be stored to
    denote the empty string regardless of this attribute.

If you want to accept :attr:`~Field.null` values with :class:`BooleanField`,
use :class:`NullBooleanField` instead.

``blank``
---------

.. attribute:: Field.blank

If ``True``, the field is allowed to be blank. Default is ``False``.

Note that this is different than :attr:`~Field.null`. :attr:`~Field.null` is
purely database-related, whereas :attr:`~Field.blank` is validation-related. If
a field has ``blank=True``, form validation will allow entry of an empty value.
If a field has ``blank=False``, the field will be required.

.. _field-choices:

``choices``
-----------

.. attribute:: Field.choices

An iterable (e.g., a list or tuple) consisting itself of iterables of exactly
two items (e.g. ``[(A, B), (A, B) ...]``) to use as choices for this field. If
this is given, the default form widget will be a select box with these choices
instead of the standard text field.

The first element in each tuple is the actual value to be set on the model,
and the second element is the human-readable name. For example::

    YEAR_IN_SCHOOL_CHOICES = (
        ('FR', 'Freshman'),
        ('SO', 'Sophomore'),
        ('JR', 'Junior'),
        ('SR', 'Senior'),
    )

Generally, it's best to define choices inside a model class, and to
define a suitably-named constant for each value::

    from django.db import models

    class Student(models.Model):
        FRESHMAN = 'FR'
        SOPHOMORE = 'SO'
        JUNIOR = 'JR'
        SENIOR = 'SR'
        YEAR_IN_SCHOOL_CHOICES = (
            (FRESHMAN, 'Freshman'),
            (SOPHOMORE, 'Sophomore'),
            (JUNIOR, 'Junior'),
            (SENIOR, 'Senior'),
        )
        year_in_school = models.CharField(
            max_length=2,
            choices=YEAR_IN_SCHOOL_CHOICES,
            default=FRESHMAN,
        )

        def is_upperclass(self):
            return self.year_in_school in (self.JUNIOR, self.SENIOR)

Though you can define a choices list outside of a model class and then
refer to it, defining the choices and names for each choice inside the
model class keeps all of that information with the class that uses it,
and makes the choices easy to reference (e.g, ``Student.SOPHOMORE``
will work anywhere that the ``Student`` model has been imported).

You can also collect your available choices into named groups that can
be used for organizational purposes::

    MEDIA_CHOICES = (
        ('Audio', (
                ('vinyl', 'Vinyl'),
                ('cd', 'CD'),
            )
        ),
        ('Video', (
                ('vhs', 'VHS Tape'),
                ('dvd', 'DVD'),
            )
        ),
        ('unknown', 'Unknown'),
    )

The first element in each tuple is the name to apply to the group. The
second element is an iterable of 2-tuples, with each 2-tuple containing
a value and a human-readable name for an option. Grouped options may be
combined with ungrouped options within a single list (such as the
`unknown` option in this example).

For each model field that has :attr:`~Field.choices` set, Django will add a
method to retrieve the human-readable name for the field's current value. See
:meth:`~django.db.models.Model.get_FOO_display` in the database API
documentation.

Note that choices can be any iterable object -- not necessarily a list or tuple.
This lets you construct choices dynamically. But if you find yourself hacking
:attr:`~Field.choices` to be dynamic, you're probably better off using a proper
database table with a :class:`ForeignKey`. :attr:`~Field.choices` is meant for
static data that doesn't change much, if ever.

Unless :attr:`blank=False<Field.blank>` is set on the field along with a
:attr:`~Field.default` then a label containing ``"---------"`` will be rendered
with the select box. To override this behavior, add a tuple to ``choices``
containing ``None``; e.g. ``(None, 'Your String For Display')``.
Alternatively, you can use an empty string instead of ``None`` where this makes
sense - such as on a :class:`~django.db.models.CharField`.

``db_column``
-------------

.. attribute:: Field.db_column

The name of the database column to use for this field. If this isn't given,
Django will use the field's name.

If your database column name is an SQL reserved word, or contains
characters that aren't allowed in Python variable names -- notably, the
hyphen -- that's OK. Django quotes column and table names behind the
scenes.

``db_index``
------------

.. attribute:: Field.db_index

If ``True``, a database index will be created for this field.

``db_tablespace``
-----------------

.. attribute:: Field.db_tablespace

The name of the :doc:`database tablespace </topics/db/tablespaces>` to use for
this field's index, if this field is indexed. The default is the project's
:setting:`DEFAULT_INDEX_TABLESPACE` setting, if set, or the
:attr:`~Options.db_tablespace` of the model, if any. If the backend doesn't
support tablespaces for indexes, this option is ignored.

``default``
-----------

.. attribute:: Field.default

The default value for the field. This can be a value or a callable object. If
callable it will be called every time a new object is created.

The default can't be a mutable object (model instance, ``list``, ``set``, etc.),
as a reference to the same instance of that object would be used as the default
value in all new model instances. Instead, wrap the desired default in a
callable. For example, if you want to specify a default ``dict`` for
:class:`~django.contrib.postgres.fields.JSONField`, use a function::

    def contact_default():
        return {"email": "to1@example.com"}

    contact_info = JSONField("ContactInfo", default=contact_default)

``lambda``\s can't be used for field options like ``default`` because they
can't be :ref:`serialized by migrations <migration-serializing>`. See that
documentation for other caveats.

For fields like :class:`ForeignKey` that map to model instances, defaults
should be the value of the field they reference (``pk`` unless
:attr:`~ForeignKey.to_field` is set) instead of model instances.

The default value is used when new model instances are created and a value
isn't provided for the field. When the field is a primary key, the default is
also used when the field is set to ``None``.

``editable``
------------

.. attribute:: Field.editable

If ``False``, the field will not be displayed in the admin or any other
:class:`~django.forms.ModelForm`. They are also skipped during :ref:`model
validation <validating-objects>`. Default is ``True``.

``error_messages``
------------------

.. attribute:: Field.error_messages

The ``error_messages`` argument lets you override the default messages that the
field will raise. Pass in a dictionary with keys matching the error messages you
want to override.

Error message keys include ``null``, ``blank``, ``invalid``, ``invalid_choice``,
``unique``, and ``unique_for_date``. Additional error message keys are
specified for each field in the `Field types`_ section below.

``help_text``
-------------

.. attribute:: Field.help_text

Extra "help" text to be displayed with the form widget. It's useful for
documentation even if your field isn't used on a form.

Note that this value is *not* HTML-escaped in automatically-generated
forms. This lets you include HTML in :attr:`~Field.help_text` if you so
desire. For example::

    help_text="Please use the following format: <em>YYYY-MM-DD</em>."

Alternatively you can use plain text and
``django.utils.html.escape()`` to escape any HTML special characters. Ensure
that you escape any help text that may come from untrusted users to avoid a
cross-site scripting attack.

``primary_key``
---------------

.. attribute:: Field.primary_key

If ``True``, this field is the primary key for the model.

If you don't specify ``primary_key=True`` for any field in your model, Django
will automatically add an :class:`AutoField` to hold the primary key, so you
don't need to set ``primary_key=True`` on any of your fields unless you want to
override the default primary-key behavior. For more, see
:ref:`automatic-primary-key-fields`.

``primary_key=True`` implies :attr:`null=False <Field.null>` and
:attr:`unique=True <Field.unique>`. Only one primary key is allowed on an
object.

The primary key field is read-only. If you change the value of the primary
key on an existing object and then save it, a new object will be created
alongside the old one.

``unique``
----------

.. attribute:: Field.unique

If ``True``, this field must be unique throughout the table.

This is enforced at the database level and by model validation. If
you try to save a model with a duplicate value in a :attr:`~Field.unique`
field, a :exc:`django.db.IntegrityError` will be raised by the model's
:meth:`~django.db.models.Model.save` method.

This option is valid on all field types except :class:`ManyToManyField`,
:class:`OneToOneField`, and :class:`FileField`.

Note that when ``unique`` is ``True``, you don't need to specify
:attr:`~Field.db_index`, because ``unique`` implies the creation of an index.

``unique_for_date``
-------------------

.. attribute:: Field.unique_for_date

Set this to the name of a :class:`DateField` or :class:`DateTimeField` to
require that this field be unique for the value of the date field.

For example, if you have a field ``title`` that has
``unique_for_date="pub_date"``, then Django wouldn't allow the entry of two
records with the same ``title`` and ``pub_date``.

Note that if you set this to point to a :class:`DateTimeField`, only the date
portion of the field will be considered. Besides, when :setting:`USE_TZ` is
``True``, the check will be performed in the :ref:`current time zone
<default-current-time-zone>` at the time the object gets saved.

This is enforced by :meth:`Model.validate_unique()` during model validation
but not at the database level. If any :attr:`~Field.unique_for_date` constraint
involves fields that are not part of a :class:`~django.forms.ModelForm` (for
example, if one of the fields is listed in ``exclude`` or has
:attr:`editable=False<Field.editable>`), :meth:`Model.validate_unique()` will
skip validation for that particular constraint.

``unique_for_month``
--------------------

.. attribute:: Field.unique_for_month

Like :attr:`~Field.unique_for_date`, but requires the field to be unique with
respect to the month.

``unique_for_year``
-------------------

.. attribute:: Field.unique_for_year

Like :attr:`~Field.unique_for_date` and :attr:`~Field.unique_for_month`.

``verbose_name``
-------------------

.. attribute:: Field.verbose_name

A human-readable name for the field. If the verbose name isn't given, Django
will automatically create it using the field's attribute name, converting
underscores to spaces. See :ref:`Verbose field names <verbose-field-names>`.

``validators``
-------------------

.. attribute:: Field.validators

A list of validators to run for this field. See the :doc:`validators
documentation </ref/validators>` for more information.

Registering and fetching lookups
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Field`` implements the :ref:`lookup registration API <lookup-registration-api>`.
The API can be used to customize which lookups are available for a field class, and
how lookups are fetched from a field.

.. _model-field-types:

Field types
===========

.. currentmodule:: django.db.models

``AutoField``
-------------

.. class:: AutoField(**options)

An :class:`IntegerField` that automatically increments
according to available IDs. You usually won't need to use this directly; a
primary key field will automatically be added to your model if you don't specify
otherwise. See :ref:`automatic-primary-key-fields`.

``BigAutoField``
----------------

.. class:: BigAutoField(**options)

.. versionadded:: 1.10

A 64-bit integer, much like an :class:`AutoField` except that it is
guaranteed to fit numbers from ``1`` to ``9223372036854775807``.

``BigIntegerField``
-------------------

.. class:: BigIntegerField(**options)

A 64-bit integer, much like an :class:`IntegerField` except that it is
guaranteed to fit numbers from ``-9223372036854775808`` to
``9223372036854775807``. The default form widget for this field is a
:class:`~django.forms.TextInput`.

``BinaryField``
-------------------

.. class:: BinaryField(**options)

A field to store raw binary data. It only supports ``bytes`` assignment. Be
aware that this field has limited functionality. For example, it is not possible
to filter a queryset on a ``BinaryField`` value. It is also not possible to
include a ``BinaryField`` in a :class:`~django.forms.ModelForm`.

.. admonition:: Abusing ``BinaryField``

    Although you might think about storing files in the database, consider that
    it is bad design in 99% of the cases. This field is *not* a replacement for
    proper :doc:`static files </howto/static-files/index>` handling.

``BooleanField``
----------------

.. class:: BooleanField(**options)

A true/false field.

The default form widget for this field is a
:class:`~django.forms.CheckboxInput`.

If you need to accept :attr:`~Field.null` values then use
:class:`NullBooleanField` instead.

The default value of ``BooleanField`` is ``None`` when :attr:`Field.default`
isn't defined.

``CharField``
-------------

.. class:: CharField(max_length=None, **options)

A string field, for small- to large-sized strings.

For large amounts of text, use :class:`~django.db.models.TextField`.

The default form widget for this field is a :class:`~django.forms.TextInput`.

:class:`CharField` has one extra required argument:

.. attribute:: CharField.max_length

    The maximum length (in characters) of the field. The max_length is enforced
    at the database level and in Django's validation.

.. note::

    If you are writing an application that must be portable to multiple
    database backends, you should be aware that there are restrictions on
    ``max_length`` for some backends. Refer to the :doc:`database backend
    notes </ref/databases>` for details.

.. admonition:: MySQL users

    If you are using this field with MySQLdb 1.2.2 and the ``utf8_bin``
    collation (which is *not* the default), there are some issues to be aware
    of. Refer to the :ref:`MySQL database notes <mysql-collation>` for
    details.

``CommaSeparatedIntegerField``
------------------------------

.. class:: CommaSeparatedIntegerField(max_length=None, **options)

.. deprecated:: 1.9

    This field is deprecated in favor of :class:`~django.db.models.CharField`
    with ``validators=[``\ :func:`validate_comma_separated_integer_list
    <django.core.validators.validate_comma_separated_integer_list>`\ ``]``.

A field of integers separated by commas. As in :class:`CharField`, the
:attr:`~CharField.max_length` argument is required and the note about database
portability mentioned there should be heeded.

``DateField``
-------------

.. class:: DateField(auto_now=False, auto_now_add=False, **options)

A date, represented in Python by a ``datetime.date`` instance. Has a few extra,
optional arguments:

.. attribute:: DateField.auto_now

    Automatically set the field to now every time the object is saved. Useful
    for "last-modified" timestamps. Note that the current date is *always*
    used; it's not just a default value that you can override.

    The field is only automatically updated when calling :meth:`Model.save()
    <django.db.models.Model.save>`. The field isn't updated when making updates
    to other fields in other ways such as :meth:`QuerySet.update()
    <django.db.models.query.QuerySet.update>`, though you can specify a custom
    value for the field in an update like that.

.. attribute:: DateField.auto_now_add

    Automatically set the field to now when the object is first created. Useful
    for creation of timestamps. Note that the current date is *always* used;
    it's not just a default value that you can override. So even if you
    set a value for this field when creating the object, it will be ignored.
    If you want to be able to modify this field, set the following instead of
    ``auto_now_add=True``:

    * For :class:`DateField`: ``default=date.today`` - from
      :meth:`datetime.date.today`
    * For :class:`DateTimeField`: ``default=timezone.now`` - from
      :func:`django.utils.timezone.now`

The default form widget for this field is a
:class:`~django.forms.TextInput`. The admin adds a JavaScript calendar,
and a shortcut for "Today". Includes an additional ``invalid_date`` error
message key.

The options ``auto_now_add``, ``auto_now``, and ``default`` are mutually exclusive.
Any combination of these options will result in an error.

.. note::
    As currently implemented, setting ``auto_now`` or ``auto_now_add`` to
    ``True`` will cause the field to have ``editable=False`` and ``blank=True``
    set.

.. note::
    The ``auto_now`` and ``auto_now_add`` options will always use the date in
    the :ref:`default timezone <default-current-time-zone>` at the moment of
    creation or update. If you need something different, you may want to
    consider simply using your own callable default or overriding ``save()``
    instead of using ``auto_now`` or ``auto_now_add``; or using a
    ``DateTimeField`` instead of a ``DateField`` and deciding how to handle the
    conversion from datetime to date at display time.

``DateTimeField``
-----------------

.. class:: DateTimeField(auto_now=False, auto_now_add=False, **options)

A date and time, represented in Python by a ``datetime.datetime`` instance.
Takes the same extra arguments as :class:`DateField`.

The default form widget for this field is a single
:class:`~django.forms.TextInput`. The admin uses two separate
:class:`~django.forms.TextInput` widgets with JavaScript shortcuts.

``DecimalField``
----------------

.. class:: DecimalField(max_digits=None, decimal_places=None, **options)

A fixed-precision decimal number, represented in Python by a
:class:`~decimal.Decimal` instance. Has two **required** arguments:

.. attribute:: DecimalField.max_digits

    The maximum number of digits allowed in the number. Note that this number
    must be greater than or equal to ``decimal_places``.

.. attribute:: DecimalField.decimal_places

    The number of decimal places to store with the number.

For example, to store numbers up to ``999`` with a resolution of 2 decimal
places, you'd use::

    models.DecimalField(..., max_digits=5, decimal_places=2)

And to store numbers up to approximately one billion with a resolution of 10
decimal places::

    models.DecimalField(..., max_digits=19, decimal_places=10)

The default form widget for this field is a :class:`~django.forms.NumberInput`
when :attr:`~django.forms.Field.localize` is ``False`` or
:class:`~django.forms.TextInput` otherwise.

.. note::

    For more information about the differences between the
    :class:`FloatField` and :class:`DecimalField` classes, please
    see :ref:`FloatField vs. DecimalField <floatfield_vs_decimalfield>`.

``DurationField``
-----------------

.. class:: DurationField(**options)

A field for storing periods of time - modeled in Python by
:class:`~python:datetime.timedelta`. When used on PostgreSQL, the data type
used is an ``interval`` and on Oracle the data type is ``INTERVAL DAY(9) TO
SECOND(6)``. Otherwise a ``bigint`` of microseconds is used.

.. note::

    Arithmetic with ``DurationField`` works in most cases. However on all
    databases other than PostgreSQL, comparing the value of a ``DurationField``
    to arithmetic on ``DateTimeField`` instances will not work as expected.

``EmailField``
--------------

.. class:: EmailField(max_length=254, **options)

A :class:`CharField` that checks that the value is a valid email address. It
uses :class:`~django.core.validators.EmailValidator` to validate the input.

``FileField``
-------------

.. class:: FileField(upload_to=None, max_length=100, **options)

A file-upload field.

.. note::
    The ``primary_key`` and ``unique`` arguments are not supported, and will
    raise a ``TypeError`` if used.

Has two optional arguments:

.. attribute:: FileField.upload_to

    This attribute provides a way of setting the upload directory and file name,
    and can be set in two ways. In both cases, the value is passed to the
    :meth:`Storage.save() <django.core.files.storage.Storage.save>` method.

    If you specify a string value, it may contain :func:`~time.strftime`
    formatting, which will be replaced by the date/time of the file upload (so
    that uploaded files don't fill up the given directory). For example::

        class MyModel(models.Model):
            # file will be uploaded to MEDIA_ROOT/uploads
            upload = models.FileField(upload_to='uploads/')
            # or...
            # file will be saved to MEDIA_ROOT/uploads/2015/01/30
            upload = models.FileField(upload_to='uploads/%Y/%m/%d/')

    If you are using the default
    :class:`~django.core.files.storage.FileSystemStorage`, the string value
    will be appended to your :setting:`MEDIA_ROOT` path to form the location on
    the local filesystem where uploaded files will be stored. If you are using
    a different storage, check that storage's documentation to see how it
    handles ``upload_to``.

    ``upload_to`` may also be a callable, such as a function. This will be
    called to obtain the upload path, including the filename. This callable must
    accept two arguments and return a Unix-style path (with forward slashes)
    to be passed along to the storage system. The two arguments are:

    ======================  ===============================================
    Argument                Description
    ======================  ===============================================
    ``instance``            An instance of the model where the
                            ``FileField`` is defined. More specifically,
                            this is the particular instance where the
                            current file is being attached.

                            In most cases, this object will not have been
                            saved to the database yet, so if it uses the
                            default ``AutoField``, *it might not yet have a
                            value for its primary key field*.

    ``filename``            The filename that was originally given to the
                            file. This may or may not be taken into account
                            when determining the final destination path.
    ======================  ===============================================

    For example::

        def user_directory_path(instance, filename):
            # file will be uploaded to MEDIA_ROOT/user_<id>/<filename>
            return 'user_{0}/{1}'.format(instance.user.id, filename)

        class MyModel(models.Model):
            upload = models.FileField(upload_to=user_directory_path)

.. attribute:: FileField.storage

    A storage object, which handles the storage and retrieval of your
    files. See :doc:`/topics/files` for details on how to provide this object.

The default form widget for this field is a
:class:`~django.forms.ClearableFileInput`.

Using a :class:`FileField` or an :class:`ImageField` (see below) in a model
takes a few steps:

1. In your settings file, you'll need to define :setting:`MEDIA_ROOT` as the
   full path to a directory where you'd like Django to store uploaded files.
   (For performance, these files are not stored in the database.) Define
   :setting:`MEDIA_URL` as the base public URL of that directory. Make sure
   that this directory is writable by the Web server's user account.

2. Add the :class:`FileField` or :class:`ImageField` to your model, defining
   the :attr:`~FileField.upload_to` option to specify a subdirectory of
   :setting:`MEDIA_ROOT` to use for uploaded files.

3. All that will be stored in your database is a path to the file
   (relative to :setting:`MEDIA_ROOT`). You'll most likely want to use the
   convenience :attr:`~django.db.models.fields.files.FieldFile.url` attribute
   provided by Django. For example, if your :class:`ImageField` is called
   ``mug_shot``, you can get the absolute path to your image in a template with
   ``{{ object.mug_shot.url }}``.

For example, say your :setting:`MEDIA_ROOT` is set to ``'/home/media'``, and
:attr:`~FileField.upload_to` is set to ``'photos/%Y/%m/%d'``. The ``'%Y/%m/%d'``
part of :attr:`~FileField.upload_to` is :func:`~time.strftime` formatting;
``'%Y'`` is the four-digit year, ``'%m'`` is the two-digit month and ``'%d'`` is
the two-digit day. If you upload a file on Jan. 15, 2007, it will be saved in
the directory ``/home/media/photos/2007/01/15``.

If you wanted to retrieve the uploaded file's on-disk filename, or the file's
size, you could use the :attr:`~django.core.files.File.name` and
:attr:`~django.core.files.File.size` attributes respectively; for more
information on the available attributes and methods, see the
:class:`~django.core.files.File` class reference and the :doc:`/topics/files`
topic guide.

.. note::
    The file is saved as part of saving the model in the database, so the actual
    file name used on disk cannot be relied on until after the model has been
    saved.

The uploaded file's relative URL can be obtained using the
:attr:`~django.db.models.fields.files.FieldFile.url` attribute. Internally,
this calls the :meth:`~django.core.files.storage.Storage.url` method of the
underlying :class:`~django.core.files.storage.Storage` class.

.. _file-upload-security:

Note that whenever you deal with uploaded files, you should pay close attention
to where you're uploading them and what type of files they are, to avoid
security holes. *Validate all uploaded files* so that you're sure the files are
what you think they are. For example, if you blindly let somebody upload files,
without validation, to a directory that's within your Web server's document
root, then somebody could upload a CGI or PHP script and execute that script by
visiting its URL on your site. Don't allow that.

Also note that even an uploaded HTML file, since it can be executed by the
browser (though not by the server), can pose security threats that are
equivalent to XSS or CSRF attacks.

:class:`FileField` instances are created in your database as ``varchar``
columns with a default max length of 100 characters. As with other fields, you
can change the maximum length using the :attr:`~CharField.max_length` argument.

``FileField`` and ``FieldFile``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. currentmodule:: django.db.models.fields.files

.. class:: FieldFile

When you access a :class:`~django.db.models.FileField` on a model, you are
given an instance of :class:`FieldFile` as a proxy for accessing the underlying
file.

The API of :class:`FieldFile` mirrors that of :class:`~django.core.files.File`,
with one key difference: *The object wrapped by the class is not necessarily a
wrapper around Python's built-in file object.* Instead, it is a wrapper around
the result of the :attr:`Storage.open()<django.core.files.storage.Storage.open>`
method, which may be a :class:`~django.core.files.File` object, or it may be a
custom storage's implementation of the :class:`~django.core.files.File` API.

In addition to the API inherited from
:class:`~django.core.files.File` such as :meth:`~django.core.files.File.read`
and :meth:`~django.core.files.File.write`, :class:`FieldFile` includes several
methods that can be used to interact with the underlying file:

.. warning::

    Two methods of this class, :meth:`~FieldFile.save` and
    :meth:`~FieldFile.delete`, default to saving the model object of the
    associated ``FieldFile`` in the database.

.. attribute:: FieldFile.name

The name of the file including the relative path from the root of the
:class:`~django.core.files.storage.Storage` of the associated
:class:`~django.db.models.FileField`.

.. attribute:: FieldFile.size

The result of the underlying :attr:`Storage.size()
<django.core.files.storage.Storage.size>` method.

.. attribute:: FieldFile.url

A read-only property to access the file's relative URL by calling the
:meth:`~django.core.files.storage.Storage.url` method of the underlying
:class:`~django.core.files.storage.Storage` class.

.. method:: FieldFile.open(mode='rb')

Opens or reopens the file associated with this instance in the specified
``mode``. Unlike the standard Python ``open()`` method, it doesn't return a
file descriptor.

Since the underlying file is opened implicitly when accessing it, it may be
unnecessary to call this method except to reset the pointer to the underlying
file or to change the ``mode``.

.. method:: FieldFile.close()

Behaves like the standard Python ``file.close()`` method and closes the file
associated with this instance.

.. method:: FieldFile.save(name, content, save=True)

This method takes a filename and file contents and passes them to the storage
class for the field, then associates the stored file with the model field.
If you want to manually associate file data with
:class:`~django.db.models.FileField` instances on your model, the ``save()``
method is used to persist that file data.

Takes two required arguments: ``name`` which is the name of the file, and
``content`` which is an object containing the file's contents.  The
optional ``save`` argument controls whether or not the model instance is
saved after the file associated with this field has been altered. Defaults to
``True``.

Note that the ``content`` argument should be an instance of
:class:`django.core.files.File`, not Python's built-in file object.
You can construct a :class:`~django.core.files.File` from an existing
Python file object like this::

    from django.core.files import File
    # Open an existing file using Python's built-in open()
    f = open('/path/to/hello.world')
    myfile = File(f)

Or you can construct one from a Python string like this::

    from django.core.files.base import ContentFile
    myfile = ContentFile("hello world")

For more information, see :doc:`/topics/files`.

.. method:: FieldFile.delete(save=True)

Deletes the file associated with this instance and clears all attributes on
the field. Note: This method will close the file if it happens to be open when
``delete()`` is called.

The optional ``save`` argument controls whether or not the model instance is
saved after the file associated with this field has been deleted. Defaults to
``True``.

Note that when a model is deleted, related files are not deleted. If you need
to cleanup orphaned files, you'll need to handle it yourself (for instance,
with a custom management command that can be run manually or scheduled to run
periodically via e.g. cron).

.. currentmodule:: django.db.models

``FilePathField``
-----------------

.. class:: FilePathField(path=None, match=None, recursive=False, max_length=100, **options)

A :class:`CharField` whose choices are limited to the filenames in a certain
directory on the filesystem. Has three special arguments, of which the first is
**required**:

.. attribute:: FilePathField.path

    Required. The absolute filesystem path to a directory from which this
    :class:`FilePathField` should get its choices. Example: ``"/home/images"``.

.. attribute:: FilePathField.match

    Optional. A regular expression, as a string, that :class:`FilePathField`
    will use to filter filenames. Note that the regex will be applied to the
    base filename, not the full path. Example: ``"foo.*\.txt$"``, which will
    match a file called ``foo23.txt`` but not ``bar.txt`` or ``foo23.png``.

.. attribute:: FilePathField.recursive

    Optional. Either ``True`` or ``False``. Default is ``False``. Specifies
    whether all subdirectories of :attr:`~FilePathField.path` should be included

.. attribute:: FilePathField.allow_files

    Optional.  Either ``True`` or ``False``.  Default is ``True``.  Specifies
    whether files in the specified location should be included.  Either this or
    :attr:`~FilePathField.allow_folders` must be ``True``.

.. attribute:: FilePathField.allow_folders

    Optional.  Either ``True`` or ``False``.  Default is ``False``.  Specifies
    whether folders in the specified location should be included.  Either this
    or :attr:`~FilePathField.allow_files` must be ``True``.

Of course, these arguments can be used together.

The one potential gotcha is that :attr:`~FilePathField.match` applies to the
base filename, not the full path. So, this example::

    FilePathField(path="/home/images", match="foo.*", recursive=True)

...will match ``/home/images/foo.png`` but not ``/home/images/foo/bar.png``
because the :attr:`~FilePathField.match` applies to the base filename
(``foo.png`` and ``bar.png``).

:class:`FilePathField` instances are created in your database as ``varchar``
columns with a default max length of 100 characters. As with other fields, you
can change the maximum length using the :attr:`~CharField.max_length` argument.

``FloatField``
--------------

.. class:: FloatField(**options)

A floating-point number represented in Python by a ``float`` instance.

The default form widget for this field is a :class:`~django.forms.NumberInput`
when :attr:`~django.forms.Field.localize` is ``False`` or
:class:`~django.forms.TextInput` otherwise.

.. _floatfield_vs_decimalfield:

.. admonition:: ``FloatField`` vs. ``DecimalField``

    The :class:`FloatField` class is sometimes mixed up with the
    :class:`DecimalField` class. Although they both represent real numbers, they
    represent those numbers differently. ``FloatField`` uses Python's ``float``
    type internally, while ``DecimalField`` uses Python's ``Decimal`` type. For
    information on the difference between the two, see Python's documentation
    for the :mod:`decimal` module.

``ImageField``
--------------

.. class:: ImageField(upload_to=None, height_field=None, width_field=None, max_length=100, **options)

Inherits all attributes and methods from :class:`FileField`, but also
validates that the uploaded object is a valid image.

In addition to the special attributes that are available for :class:`FileField`,
an :class:`ImageField` also has ``height`` and ``width`` attributes.

To facilitate querying on those attributes, :class:`ImageField` has two extra
optional arguments:

.. attribute:: ImageField.height_field

    Name of a model field which will be auto-populated with the height of the
    image each time the model instance is saved.

.. attribute:: ImageField.width_field

    Name of a model field which will be auto-populated with the width of the
    image each time the model instance is saved.

Requires the `Pillow`_ library.

.. _Pillow: https://pillow.readthedocs.io/en/latest/

:class:`ImageField` instances are created in your database as ``varchar``
columns with a default max length of 100 characters. As with other fields, you
can change the maximum length using the :attr:`~CharField.max_length` argument.

The default form widget for this field is a
:class:`~django.forms.ClearableFileInput`.

``IntegerField``
----------------

.. class:: IntegerField(**options)

An integer. Values from ``-2147483648`` to ``2147483647`` are safe in all
databases supported by Django. The default form widget for this field is a
:class:`~django.forms.NumberInput` when :attr:`~django.forms.Field.localize`
is ``False`` or :class:`~django.forms.TextInput` otherwise.

``GenericIPAddressField``
-------------------------

.. class:: GenericIPAddressField(protocol='both', unpack_ipv4=False, **options)

An IPv4 or IPv6 address, in string format (e.g. ``192.0.2.30`` or
``2a02:42fe::4``). The default form widget for this field is a
:class:`~django.forms.TextInput`.

The IPv6 address normalization follows :rfc:`4291#section-2.2` section 2.2,
including using the IPv4 format suggested in paragraph 3 of that section, like
``::ffff:192.0.2.0``. For example, ``2001:0::0:01`` would be normalized to
``2001::1``, and ``::ffff:0a0a:0a0a`` to ``::ffff:10.10.10.10``. All characters
are converted to lowercase.

.. attribute:: GenericIPAddressField.protocol

    Limits valid inputs to the specified protocol.
    Accepted values are ``'both'`` (default), ``'IPv4'``
    or ``'IPv6'``. Matching is case insensitive.

.. attribute:: GenericIPAddressField.unpack_ipv4

    Unpacks IPv4 mapped addresses like ``::ffff:192.0.2.1``.
    If this option is enabled that address would be unpacked to
    ``192.0.2.1``. Default is disabled. Can only be used
    when ``protocol`` is set to ``'both'``.

If you allow for blank values, you have to allow for null values since blank
values are stored as null.

``NullBooleanField``
--------------------

.. class:: NullBooleanField(**options)

Like a :class:`BooleanField`, but allows ``NULL`` as one of the options. Use
this instead of a :class:`BooleanField` with ``null=True``. The default form
widget for this field is a :class:`~django.forms.NullBooleanSelect`.

``PositiveIntegerField``
------------------------

.. class:: PositiveIntegerField(**options)

Like an :class:`IntegerField`, but must be either positive or zero (``0``).
Values from ``0`` to ``2147483647`` are safe in all databases supported by
Django. The value ``0`` is accepted for backward compatibility reasons.

``PositiveSmallIntegerField``
-----------------------------

.. class:: PositiveSmallIntegerField(**options)

Like a :class:`PositiveIntegerField`, but only allows values under a certain
(database-dependent) point. Values from ``0`` to ``32767`` are safe in all
databases supported by Django.

``SlugField``
-------------

.. class:: SlugField(max_length=50, **options)

:term:`Slug` is a newspaper term. A slug is a short label for something,
containing only letters, numbers, underscores or hyphens. They're generally used
in URLs.

Like a CharField, you can specify :attr:`~CharField.max_length` (read the note
about database portability and :attr:`~CharField.max_length` in that section,
too). If :attr:`~CharField.max_length` is not specified, Django will use a
default length of 50.

Implies setting :attr:`Field.db_index` to ``True``.

It is often useful to automatically prepopulate a SlugField based on the value
of some other value.  You can do this automatically in the admin using
:attr:`~django.contrib.admin.ModelAdmin.prepopulated_fields`.

.. attribute:: SlugField.allow_unicode

    .. versionadded:: 1.9

    If ``True``, the field accepts Unicode letters in addition to ASCII
    letters. Defaults to ``False``.

``SmallIntegerField``
---------------------

.. class:: SmallIntegerField(**options)

Like an :class:`IntegerField`, but only allows values under a certain
(database-dependent) point. Values from ``-32768`` to ``32767`` are safe in all
databases supported by Django.

``TextField``
-------------

.. class:: TextField(**options)

A large text field. The default form widget for this field is a
:class:`~django.forms.Textarea`.

If you specify a ``max_length`` attribute, it will be reflected in the
:class:`~django.forms.Textarea` widget of the auto-generated form field.
However it is not enforced at the model or database level. Use a
:class:`CharField` for that.

.. admonition:: MySQL users

    If you are using this field with MySQLdb 1.2.1p2 and the ``utf8_bin``
    collation (which is *not* the default), there are some issues to be aware
    of. Refer to the :ref:`MySQL database notes <mysql-collation>` for
    details.

``TimeField``
-------------

.. class:: TimeField(auto_now=False, auto_now_add=False, **options)

A time, represented in Python by a ``datetime.time`` instance. Accepts the same
auto-population options as :class:`DateField`.

The default form widget for this field is a :class:`~django.forms.TextInput`.
The admin adds some JavaScript shortcuts.

``URLField``
------------

.. class:: URLField(max_length=200, **options)

A :class:`CharField` for a URL.

The default form widget for this field is a :class:`~django.forms.TextInput`.

Like all :class:`CharField` subclasses, :class:`URLField` takes the optional
:attr:`~CharField.max_length` argument. If you don't specify
:attr:`~CharField.max_length`, a default of 200 is used.

``UUIDField``
-------------

.. class:: UUIDField(**options)

A field for storing universally unique identifiers. Uses Python's
:class:`~python:uuid.UUID` class. When used on PostgreSQL, this stores in a
``uuid`` datatype, otherwise in a ``char(32)``.

Universally unique identifiers are a good alternative to :class:`AutoField` for
:attr:`~Field.primary_key`. The database will not generate the UUID for you, so
it is recommended to use :attr:`~Field.default`::

    import uuid
    from django.db import models

    class MyUUIDModel(models.Model):
        id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
        # other fields

Note that a callable (with the parentheses omitted) is passed to ``default``,
not an instance of ``UUID``.

Relationship fields
===================

.. module:: django.db.models.fields.related
   :synopsis: Related field types

.. currentmodule:: django.db.models

Django also defines a set of fields that represent relations.

.. _ref-foreignkey:

``ForeignKey``
--------------

.. class:: ForeignKey(othermodel, on_delete, **options)

A many-to-one relationship. Requires a positional argument: the class to which
the model is related.

.. versionchanged:: 1.9

    ``on_delete`` can now be used as the second positional argument (previously
    it was typically only passed as a keyword argument). It will be a required
    argument in Django 2.0.

.. _recursive-relationships:

To create a recursive relationship -- an object that has a many-to-one
relationship with itself -- use ``models.ForeignKey('self',
on_delete=models.CASCADE)``.

.. _lazy-relationships:

If you need to create a relationship on a model that has not yet been defined,
you can use the name of the model, rather than the model object itself::

    from django.db import models

    class Car(models.Model):
        manufacturer = models.ForeignKey(
            'Manufacturer',
            on_delete=models.CASCADE,
        )
        # ...

    class Manufacturer(models.Model):
        # ...
        pass

Relationships defined this way on :ref:`abstract models
<abstract-base-classes>` are resolved when the model is subclassed as a
concrete model and are not relative to the abstract model's ``app_label``:

.. snippet::
    :filename: products/models.py

    from django.db import models

    class AbstractCar(models.Model):
        manufacturer = models.ForeignKey('Manufacturer', on_delete=models.CASCADE)

        class Meta:
            abstract = True

.. snippet::
    :filename: production/models.py

    from django.db import models
    from products.models import AbstractCar

    class Manufacturer(models.Model):
        pass

    class Car(AbstractCar):
        pass

    # Car.manufacturer will point to `production.Manufacturer` here.

To refer to models defined in another application, you can explicitly specify
a model with the full application label. For example, if the ``Manufacturer``
model above is defined in another application called ``production``, you'd
need to use::

    class Car(models.Model):
        manufacturer = models.ForeignKey(
            'production.Manufacturer',
            on_delete=models.CASCADE,
        )

This sort of reference can be useful when resolving circular import
dependencies between two applications.

A database index is automatically created on the ``ForeignKey``. You can
disable this by setting :attr:`~Field.db_index` to ``False``.  You may want to
avoid the overhead of an index if you are creating a foreign key for
consistency rather than joins, or if you will be creating an alternative index
like a partial or multiple column index.

Database Representation
~~~~~~~~~~~~~~~~~~~~~~~

Behind the scenes, Django appends ``"_id"`` to the field name to create its
database column name. In the above example, the database table for the ``Car``
model will have a ``manufacturer_id`` column. (You can change this explicitly by
specifying :attr:`~Field.db_column`) However, your code should never have to
deal with the database column name, unless you write custom SQL. You'll always
deal with the field names of your model object.

.. _foreign-key-arguments:

Arguments
~~~~~~~~~

:class:`ForeignKey` accepts other arguments that define the details of how the
relation works.

.. attribute:: ForeignKey.on_delete

    When an object referenced by a :class:`ForeignKey` is deleted, Django will
    emulate the behavior of the SQL constraint specified by the
    :attr:`on_delete` argument. For example, if you have a nullable
    :class:`ForeignKey` and you want it to be set null when the referenced
    object is deleted::

        user = models.ForeignKey(
            User,
            models.SET_NULL,
            blank=True,
            null=True,
        )

    .. deprecated:: 1.9

        :attr:`~ForeignKey.on_delete` will become a required argument in Django
        2.0. In older versions it defaults to ``CASCADE``.

The possible values for :attr:`~ForeignKey.on_delete` are found in
:mod:`django.db.models`:

* .. attribute:: CASCADE

    Cascade deletes. Django emulates the behavior of the SQL constraint ON
    DELETE CASCADE and also deletes the object containing the ForeignKey.

* .. attribute:: PROTECT

    Prevent deletion of the referenced object by raising
    :exc:`~django.db.models.ProtectedError`, a subclass of
    :exc:`django.db.IntegrityError`.

* .. attribute:: SET_NULL

    Set the :class:`ForeignKey` null; this is only possible if
    :attr:`~Field.null` is ``True``.

* .. attribute:: SET_DEFAULT

    Set the :class:`ForeignKey` to its default value; a default for the
    :class:`ForeignKey` must be set.

* .. function:: SET()

    Set the :class:`ForeignKey` to the value passed to
    :func:`~django.db.models.SET()`, or if a callable is passed in,
    the result of calling it. In most cases, passing a callable will be
    necessary to avoid executing queries at the time your models.py is
    imported::

        from django.conf import settings
        from django.contrib.auth import get_user_model
        from django.db import models

        def get_sentinel_user():
            return get_user_model().objects.get_or_create(username='deleted')[0]

        class MyModel(models.Model):
            user = models.ForeignKey(
                settings.AUTH_USER_MODEL,
                on_delete=models.SET(get_sentinel_user),
            )

* .. attribute:: DO_NOTHING

    Take no action. If your database backend enforces referential
    integrity, this will cause an :exc:`~django.db.IntegrityError` unless
    you manually add an SQL ``ON DELETE`` constraint to the database field.

.. attribute:: ForeignKey.limit_choices_to

    Sets a limit to the available choices for this field when this field is
    rendered using a ``ModelForm`` or the admin (by default, all objects
    in the queryset are available to choose). Either a dictionary, a
    :class:`~django.db.models.Q` object, or a callable returning a
    dictionary or :class:`~django.db.models.Q` object can be used.

    For example::

        staff_member = models.ForeignKey(
            User,
            on_delete=models.CASCADE,
            limit_choices_to={'is_staff': True},
        )

    causes the corresponding field on the ``ModelForm`` to list only ``Users``
    that have ``is_staff=True``. This may be helpful in the Django admin.

    The callable form can be helpful, for instance, when used in conjunction
    with the Python ``datetime`` module to limit selections by date range. For
    example::

        def limit_pub_date_choices():
            return {'pub_date__lte': datetime.date.utcnow()}

        limit_choices_to = limit_pub_date_choices

    If ``limit_choices_to`` is or returns a :class:`Q object
    <django.db.models.Q>`, which is useful for :ref:`complex queries
    <complex-lookups-with-q>`, then it will only have an effect on the choices
    available in the admin when the field is not listed in
    :attr:`~django.contrib.admin.ModelAdmin.raw_id_fields` in the
    ``ModelAdmin`` for the model.

    .. note::

        If a callable is used for ``limit_choices_to``, it will be invoked
        every time a new form is instantiated. It may also be invoked when a
        model is validated, for example by management commands or the admin.
        The admin constructs querysets to validate its form inputs in various
        edge cases multiple times, so there is a possibility your callable may
        be invoked several times.

.. attribute:: ForeignKey.related_name

    The name to use for the relation from the related object back to this one.
    It's also the default value for :attr:`related_query_name` (the name to use
    for the reverse filter name from the target model). See the :ref:`related
    objects documentation <backwards-related-objects>` for a full explanation
    and example. Note that you must set this value when defining relations on
    :ref:`abstract models <abstract-base-classes>`; and when you do so
    :ref:`some special syntax <abstract-related-name>` is available.

    If you'd prefer Django not to create a backwards relation, set
    ``related_name`` to ``'+'`` or end it with ``'+'``. For example, this will
    ensure that the ``User`` model won't have a backwards relation to this
    model::

        user = models.ForeignKey(
            User,
            on_delete=models.CASCADE,
            related_name='+',
        )

.. attribute:: ForeignKey.related_query_name

    The name to use for the reverse filter name from the target model. It
    defaults to the value of :attr:`related_name` or
    :attr:`~django.db.models.Options.default_related_name` if set, otherwise it
    defaults to the name of the model::

        # Declare the ForeignKey with related_query_name
        class Tag(models.Model):
            article = models.ForeignKey(
                Article,
                on_delete=models.CASCADE,
                related_name="tags",
                related_query_name="tag",
            )
            name = models.CharField(max_length=255)

        # That's now the name of the reverse filter
        Article.objects.filter(tag__name="important")

    Like :attr:`related_name`, ``related_query_name`` supports app label and
    class interpolation via :ref:`some special syntax <abstract-related-name>`.

.. attribute:: ForeignKey.to_field

    The field on the related object that the relation is to. By default, Django
    uses the primary key of the related object. If you reference a different
    field, that field must have ``unique=True``.

.. attribute:: ForeignKey.db_constraint

    Controls whether or not a constraint should be created in the database for
    this foreign key. The default is ``True``, and that's almost certainly what
    you want; setting this to ``False`` can be very bad for data integrity.
    That said, here are some scenarios where you might want to do this:

    * You have legacy data that is not valid.
    * You're sharding your database.

    If this is set to ``False``, accessing a related object that doesn't exist
    will raise its ``DoesNotExist`` exception.

.. attribute:: ForeignKey.swappable

    Controls the migration framework's reaction if this :class:`ForeignKey`
    is pointing at a swappable model. If it is ``True`` - the default -
    then if the :class:`ForeignKey` is pointing at a model which matches
    the current value of ``settings.AUTH_USER_MODEL`` (or another swappable
    model setting) the relationship will be stored in the migration using
    a reference to the setting, not to the model directly.

    You only want to override this to be ``False`` if you are sure your
    model should always point towards the swapped-in model - for example,
    if it is a profile model designed specifically for your custom user model.

    Setting it to ``False`` does not mean you can reference a swappable model
    even if it is swapped out - ``False`` just means that the migrations made
    with this ForeignKey will always reference the exact model you specify
    (so it will fail hard if the user tries to run with a User model you don't
    support, for example).

    If in doubt, leave it to its default of ``True``.

``ManyToManyField``
-------------------

.. class:: ManyToManyField(othermodel, **options)

A many-to-many relationship. Requires a positional argument: the class to
which the model is related, which works exactly the same as it does for
:class:`ForeignKey`, including :ref:`recursive <recursive-relationships>` and
:ref:`lazy <lazy-relationships>` relationships.

Related objects can be added, removed, or created with the field's
:class:`~django.db.models.fields.related.RelatedManager`.

Database Representation
~~~~~~~~~~~~~~~~~~~~~~~

Behind the scenes, Django creates an intermediary join table to represent the
many-to-many relationship. By default, this table name is generated using the
name of the many-to-many field and the name of the table for the model that
contains it. Since some databases don't support table names above a certain
length, these table names will be automatically truncated to 64 characters and a
uniqueness hash will be used. This means you might see table names like
``author_books_9cdf4``; this is perfectly normal.  You can manually provide the
name of the join table using the :attr:`~ManyToManyField.db_table` option.

.. _manytomany-arguments:

Arguments
~~~~~~~~~

:class:`ManyToManyField` accepts an extra set of arguments -- all optional --
that control how the relationship functions.

.. attribute:: ManyToManyField.related_name

    Same as :attr:`ForeignKey.related_name`.

.. attribute:: ManyToManyField.related_query_name

    Same as :attr:`ForeignKey.related_query_name`.

.. attribute:: ManyToManyField.limit_choices_to

    Same as :attr:`ForeignKey.limit_choices_to`.

    ``limit_choices_to`` has no effect when used on a ``ManyToManyField`` with a
    custom intermediate table specified using the
    :attr:`~ManyToManyField.through` parameter.

.. attribute:: ManyToManyField.symmetrical

    Only used in the definition of ManyToManyFields on self. Consider the
    following model::

        from django.db import models

        class Person(models.Model):
            friends = models.ManyToManyField("self")

    When Django processes this model, it identifies that it has a
    :class:`ManyToManyField` on itself, and as a result, it doesn't add a
    ``person_set`` attribute to the ``Person`` class. Instead, the
    :class:`ManyToManyField` is assumed to be symmetrical -- that is, if I am
    your friend, then you are my friend.

    If you do not want symmetry in many-to-many relationships with ``self``, set
    :attr:`~ManyToManyField.symmetrical` to ``False``. This will force Django to
    add the descriptor for the reverse relationship, allowing
    :class:`ManyToManyField` relationships to be non-symmetrical.

.. attribute:: ManyToManyField.through

    Django will automatically generate a table to manage many-to-many
    relationships. However, if you want to manually specify the intermediary
    table, you can use the :attr:`~ManyToManyField.through` option to specify
    the Django model that represents the intermediate table that you want to
    use.

    The most common use for this option is when you want to associate
    :ref:`extra data with a many-to-many relationship
    <intermediary-manytomany>`.

    If you don't specify an explicit ``through`` model, there is still an
    implicit ``through`` model class you can use to directly access the table
    created to hold the association. It has three fields to link the models.

    If the source and target models differ, the following fields are
    generated:

    * ``id``: the primary key of the relation.
    * ``<containing_model>_id``: the ``id`` of the model that declares the
      ``ManyToManyField``.
    * ``<other_model>_id``: the ``id`` of the model that the
      ``ManyToManyField`` points to.

    If the ``ManyToManyField`` points from and to the same model, the following
    fields are generated:

    * ``id``: the primary key of the relation.
    * ``from_<model>_id``: the ``id`` of the instance which points at the
      model (i.e. the source instance).
    * ``to_<model>_id``: the ``id`` of the instance to which the relationship
      points (i.e. the target model instance).

    This class can be used to query associated records for a given model
    instance like a normal model.

.. attribute:: ManyToManyField.through_fields

    Only used when a custom intermediary model is specified. Django will
    normally determine which fields of the intermediary model to use in order
    to establish a many-to-many relationship automatically. However,
    consider the following models::

        from django.db import models

        class Person(models.Model):
            name = models.CharField(max_length=50)

        class Group(models.Model):
            name = models.CharField(max_length=128)
            members = models.ManyToManyField(
                Person,
                through='Membership',
                through_fields=('group', 'person'),
            )

        class Membership(models.Model):
            group = models.ForeignKey(Group, on_delete=models.CASCADE)
            person = models.ForeignKey(Person, on_delete=models.CASCADE)
            inviter = models.ForeignKey(
                Person,
                on_delete=models.CASCADE,
                related_name="membership_invites",
            )
            invite_reason = models.CharField(max_length=64)

    ``Membership`` has *two* foreign keys to ``Person`` (``person`` and
    ``inviter``), which makes the relationship ambiguous and Django can't know
    which one to use. In this case, you must explicitly specify which
    foreign keys Django should use using ``through_fields``, as in the example
    above.

    ``through_fields`` accepts a 2-tuple ``('field1', 'field2')``, where
    ``field1`` is the name of the foreign key to the model the
    :class:`ManyToManyField` is defined on (``group`` in this case), and
    ``field2`` the name of the foreign key to the target model (``person``
    in this case).

    When you have more than one foreign key on an intermediary model to any
    (or even both) of the models participating in a many-to-many relationship,
    you *must* specify ``through_fields``. This also applies to
    :ref:`recursive relationships <recursive-relationships>`
    when an intermediary model is used and there are more than two
    foreign keys to the model, or you want to explicitly specify which two
    Django should use.

    Recursive relationships using an intermediary model are always defined as
    non-symmetrical -- that is, with :attr:`symmetrical=False <ManyToManyField.symmetrical>`
    -- therefore, there is the concept of a "source" and a "target". In that
    case ``'field1'`` will be treated as the "source" of the relationship and
    ``'field2'`` as the "target".

.. attribute:: ManyToManyField.db_table

    The name of the table to create for storing the many-to-many data. If this
    is not provided, Django will assume a default name based upon the names of:
    the table for the model defining the relationship and the name of the field
    itself.

.. attribute:: ManyToManyField.db_constraint

    Controls whether or not constraints should be created in the database for
    the foreign keys in the intermediary table. The default is ``True``, and
    that's almost certainly what you want; setting this to ``False`` can be
    very bad for data integrity. That said, here are some scenarios where you
    might want to do this:

    * You have legacy data that is not valid.
    * You're sharding your database.

    It is an error to pass both ``db_constraint`` and ``through``.

.. attribute:: ManyToManyField.swappable

    Controls the migration framework's reaction if this :class:`ManyToManyField`
    is pointing at a swappable model. If it is ``True`` - the default -
    then if the :class:`ManyToManyField` is pointing at a model which matches
    the current value of ``settings.AUTH_USER_MODEL`` (or another swappable
    model setting) the relationship will be stored in the migration using
    a reference to the setting, not to the model directly.

    You only want to override this to be ``False`` if you are sure your
    model should always point towards the swapped-in model - for example,
    if it is a profile model designed specifically for your custom user model.

    If in doubt, leave it to its default of ``True``.

:class:`ManyToManyField` does not support :attr:`~Field.validators`.

:attr:`~Field.null` has no effect since there is no way to require a
relationship at the database level.

``OneToOneField``
-----------------

.. class:: OneToOneField(othermodel, on_delete, parent_link=False, **options)

A one-to-one relationship. Conceptually, this is similar to a
:class:`ForeignKey` with :attr:`unique=True <Field.unique>`, but the
"reverse" side of the relation will directly return a single object.

.. versionchanged:: 1.9

    ``on_delete`` can now be used as the second positional argument (previously
    it was typically only passed as a keyword argument). It will be a required
    argument in Django 2.0.

This is most useful as the primary key of a model which "extends"
another model in some way; :ref:`multi-table-inheritance` is
implemented by adding an implicit one-to-one relation from the child
model to the parent model, for example.

One positional argument is required: the class to which the model will be
related. This works exactly the same as it does for :class:`ForeignKey`,
including all the options regarding :ref:`recursive <recursive-relationships>`
and :ref:`lazy <lazy-relationships>` relationships.

If you do not specify the :attr:`~ForeignKey.related_name` argument for
the ``OneToOneField``, Django will use the lower-case name of the current model
as default value.

With the following example::

    from django.conf import settings
    from django.db import models

    class MySpecialUser(models.Model):
        user = models.OneToOneField(
            settings.AUTH_USER_MODEL,
            on_delete=models.CASCADE,
        )
        supervisor = models.OneToOneField(
            settings.AUTH_USER_MODEL,
            on_delete=models.CASCADE,
            related_name='supervisor_of',
        )

your resulting ``User`` model will have the following attributes::

    >>> user = User.objects.get(pk=1)
    >>> hasattr(user, 'myspecialuser')
    True
    >>> hasattr(user, 'supervisor_of')
    True

A ``DoesNotExist`` exception is raised when accessing the reverse relationship
if an entry in the related table doesn't exist. For example, if a user doesn't
have a supervisor designated by ``MySpecialUser``::

    >>> user.supervisor_of
    Traceback (most recent call last):
        ...
    DoesNotExist: User matching query does not exist.

.. _onetoone-arguments:

Additionally, ``OneToOneField`` accepts all of the extra arguments
accepted by :class:`ForeignKey`, plus one extra argument:

.. attribute:: OneToOneField.parent_link

    When ``True`` and used in a model which inherits from another
    :term:`concrete model`, indicates that this field should be used as the
    link back to the parent class, rather than the extra
    ``OneToOneField`` which would normally be implicitly created by
    subclassing.

See :doc:`One-to-one relationships </topics/db/examples/one_to_one>` for usage
examples of ``OneToOneField``.

Field API reference
===================

.. class:: Field

    ``Field`` is an abstract class that represents a database table column.
    Django uses fields to create the database table (:meth:`db_type`), to map
    Python types to database (:meth:`get_prep_value`) and vice-versa
    (:meth:`from_db_value`).

    A field is thus a fundamental piece in different Django APIs, notably,
    :class:`models <django.db.models.Model>` and :class:`querysets
    <django.db.models.query.QuerySet>`.

    In models, a field is instantiated as a class attribute and represents a
    particular table column, see :doc:`/topics/db/models`. It has attributes
    such as :attr:`null` and :attr:`unique`, and methods that Django uses to
    map the field value to database-specific values.

    A ``Field`` is a subclass of
    :class:`~django.db.models.lookups.RegisterLookupMixin` and thus both
    :class:`~django.db.models.Transform` and
    :class:`~django.db.models.Lookup` can be registered on it to be used
    in ``QuerySet``\s (e.g. ``field_name__exact="foo"``). All :ref:`built-in
    lookups <field-lookups>` are registered by default.

    All of Django's built-in fields, such as :class:`CharField`, are particular
    implementations of ``Field``. If you need a custom field, you can either
    subclass any of the built-in fields or write a ``Field`` from scratch. In
    either case, see :doc:`/howto/custom-model-fields`.

    .. attribute:: description

        A verbose description of the field, e.g. for the
        :mod:`django.contrib.admindocs` application.

        The description can be of the form::

            description = _("String (up to %(max_length)s)")

        where the arguments are interpolated from the field's ``__dict__``.

    To map a ``Field`` to a database-specific type, Django exposes several
    methods:

    .. method:: get_internal_type()

        Returns a string naming this field for backend specific purposes.
        By default, it returns the class name.

        See :ref:`emulating-built-in-field-types` for usage in custom fields.

    .. method:: db_type(connection)

        Returns the database column data type for the :class:`Field`, taking
        into account the ``connection``.

        See :ref:`custom-database-types` for usage in custom fields.

    .. method:: rel_db_type(connection)

        .. versionadded:: 1.10

        Returns the database column data type for fields such as ``ForeignKey``
        and ``OneToOneField`` that point to the :class:`Field`, taking
        into account the ``connection``.

        See :ref:`custom-database-types` for usage in custom fields.

    There are three main situations where Django needs to interact with the
    database backend and fields:

    * when it queries the database (Python value -> database backend value)
    * when it loads data from the database (database backend value -> Python
      value)
    * when it saves to the database (Python value -> database backend value)

    When querying, :meth:`get_db_prep_value` and :meth:`get_prep_value` are used:

    .. method:: get_prep_value(value)

        ``value`` is the current value of the model's attribute, and the method
        should return data in a format that has been prepared for use as a
        parameter in a query.

        See :ref:`converting-python-objects-to-query-values` for usage.

    .. method:: get_db_prep_value(value, connection, prepared=False)

        Converts ``value`` to a backend-specific value. By default it returns
        ``value`` if ``prepared=True`` and :meth:`~Field.get_prep_value` if is
        ``False``.

        See :ref:`converting-query-values-to-database-values` for usage.

    When loading data, :meth:`from_db_value` is used:

    .. method:: from_db_value(value, expression, connection, context)

        Converts a value as returned by the database to a Python object. It is
        the reverse of :meth:`get_prep_value`.

        This method is not used for most built-in fields as the database
        backend already returns the correct Python type, or the backend itself
        does the conversion.

        See :ref:`converting-values-to-python-objects` for usage.

        .. note::

            For performance reasons, ``from_db_value`` is not implemented as a
            no-op on fields which do not require it (all Django fields).
            Consequently you may not call ``super`` in your definition.

    When saving, :meth:`pre_save` and :meth:`get_db_prep_save` are used:

    .. method:: get_db_prep_save(value, connection)

        Same as the :meth:`get_db_prep_value`, but called when the field value
        must be *saved* to the database. By default returns
        :meth:`get_db_prep_value`.

    .. method:: pre_save(model_instance, add)

        Method called prior to :meth:`get_db_prep_save` to prepare the value
        before being saved (e.g. for :attr:`DateField.auto_now`).

        ``model_instance`` is the instance this field belongs to and ``add``
        is whether the instance is being saved to the database for the first
        time.

        It should return the value of the appropriate attribute from
        ``model_instance`` for this field. The attribute name is in
        ``self.attname`` (this is set up by :class:`~django.db.models.Field`).

        See :ref:`preprocessing-values-before-saving` for usage.

    Fields often receive their values as a different type, either from
    serialization or from forms.

    .. method:: to_python(value)

        Converts the value into the correct Python object. It acts as the
        reverse of :meth:`value_to_string`, and is also called in
        :meth:`~django.db.models.Model.clean`.

        See :ref:`converting-values-to-python-objects` for usage.

    Besides saving to the database, the field also needs to know how to
    serialize its value:

    .. method:: value_to_string(obj)

        Converts ``obj`` to a string. Used to serialize the value of the field.

        See :ref:`converting-model-field-to-serialization` for usage.

    When using :class:`model forms <django.forms.ModelForm>`, the ``Field``
    needs to know which form field it should be represented by:

    .. method:: formfield(form_class=None, choices_form_class=None, **kwargs)

        Returns the default :class:`django.forms.Field` of this field for
        :class:`~django.forms.ModelForm`.

        By default, if both ``form_class`` and ``choices_form_class`` are
        ``None``, it uses :class:`~django.forms.CharField`. If the field has
        :attr:`~django.db.models.Field.choices` and ``choices_form_class``
        isn't specified, it uses :class:`~django.forms.TypedChoiceField`.

        See :ref:`specifying-form-field-for-model-field` for usage.

    .. method:: deconstruct()

        Returns a 4-tuple with enough information to recreate the field:

        1. The name of the field on the model.
        2. The import path of the field (e.g. ``"django.db.models.IntegerField"``).
           This should be the most portable version, so less specific may be better.
        3. A list of positional arguments.
        4. A dict of keyword arguments.

        This method must be added to fields prior to 1.7 to migrate its data
        using :doc:`/topics/migrations`.

.. _model-field-attributes:

=========================
Field attribute reference
=========================

Every ``Field`` instance contains several attributes that allow
introspecting its behavior. Use these attributes instead of ``isinstance``
checks when you need to write code that depends on a field's functionality.
These attributes can be used together with the :ref:`Model._meta API
<model-meta-field-api>` to narrow down a search for specific field types.
Custom model fields should implement these flags.

Attributes for fields
=====================

.. attribute:: Field.auto_created

     Boolean flag that indicates if the field was automatically created, such
     as the ``OneToOneField`` used by model inheritance.

.. attribute:: Field.concrete

    Boolean flag that indicates if the field has a database column associated
    with it.

.. attribute:: Field.hidden

    Boolean flag that indicates if a field is used to back another non-hidden
    field's functionality (e.g. the ``content_type`` and ``object_id`` fields
    that make up a ``GenericForeignKey``). The ``hidden`` flag is used to
    distinguish what constitutes the public subset of fields on the model from
    all the fields on the model.

    .. note::

        :meth:`Options.get_fields()
        <django.db.models.options.Options.get_fields()>`
        excludes hidden fields by default. Pass in ``include_hidden=True`` to
        return hidden fields in the results.

.. attribute:: Field.is_relation

    Boolean flag that indicates if a field contains references to one or
    more other models for its functionality (e.g. ``ForeignKey``,
    ``ManyToManyField``, ``OneToOneField``, etc.).

.. attribute:: Field.model

    Returns the model on which the field is defined. If a field is defined on
    a superclass of a model, ``model`` will refer to the superclass, not the
    class of the instance.

Attributes for fields with relations
====================================

These attributes are used to query for the cardinality and other details of a
relation. These attribute are present on all fields; however, they will only
have boolean values (rather than ``None``) if the field is a relation type
(:attr:`Field.is_relation=True <Field.is_relation>`).

.. attribute:: Field.many_to_many

    Boolean flag that is ``True`` if the field has a many-to-many relation;
    ``False`` otherwise. The only field included with Django where this is
    ``True`` is ``ManyToManyField``.

.. attribute:: Field.many_to_one

    Boolean flag that is ``True`` if the field has a many-to-one relation, such
    as a ``ForeignKey``; ``False`` otherwise.

.. attribute:: Field.one_to_many

    Boolean flag that is ``True`` if the field has a one-to-many relation, such
    as a ``GenericRelation`` or the reverse of a ``ForeignKey``; ``False``
    otherwise.

.. attribute:: Field.one_to_one

    Boolean flag that is ``True`` if the field has a one-to-one relation, such
    as a ``OneToOneField``; ``False`` otherwise.

.. attribute:: Field.related_model

    Points to the model the field relates to. For example, ``Author`` in
    ``ForeignKey(Author, on_delete=models.CASCADE)``. The ``related_model`` for
    a ``GenericForeignKey`` is always ``None``.