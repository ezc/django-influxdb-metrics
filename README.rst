Django InfluxDB Metrics
=======================

A reusable Django app that sends metrics about your project to InfluxDB.

Installation
------------

To get the latest stable release from PyPi

.. code-block:: bash

    pip install django-influxdb-metrics

To get the latest commit from GitHub

.. code-block:: bash

    pip install -e git+git://github.com/bitmazk/django-influxdb-metrics.git#egg=influxdb_metrix

Add ``influxdb_metrix`` to your ``INSTALLED_APPS``

.. code-block:: python

    INSTALLED_APPS = (
        ...,
        'influxdb_metrix',
    )

Settings
--------

You need to set the following settings::

    INFLUXDB_HOST = 'your.influxdbhost.com'
    INFLUXDB_PORT = '8086'
    INFLUXDB_USER = 'youruser'
    INFLUXDB_PASSWORD = 'yourpassword'
    INFLUXDB_DATABASE = 'yourdatabase'

    # Optional:
    INFLUXDB_SERIES_PREFIX = 'yourservername.'
    INFLUXDB_SERIES_POSTFIX = '.whatever'


Usage
-----

The app comes with several management commands which you should schedule via
crontab.


influxdb_get_memory_usage
+++++++++++++++++++++++++

Collects the total memory of your user, plus the memory and name of the largest
process.

You can run it like this::

    ./manage.py influxdb_get_memory_usage
    ./manage.py influxdb_get_memory_usage username

If you don't provide a username, the current user that runs the script will be
used.

You could schedule it like this::

    * * * * * cd /path/to/project/ && /path/to/venv/bin/python /path/to/project/manage.py influxdb_get_memory_usage username > $HOME/mylogs/cron/influxdb-get-memory-usage.log 2>&1

The series created in your influxdb will be named
``<prefix>server.memory.usage<postfix>`` and will have the following columns:

* ``value``: The total memory usage in bytes
* ``largest_process``: Memory usage of the largest process in bytes
* ``largest_process_name``: String representing the largest process name


influxdb_get_disk_usage
+++++++++++++++++++++++

Collects the total disk usage for the given path.

You can run it like this::

    ./manage.py influxdb_get_disk_usage $HOME

You should give an absolute path to the folder which you want to measure. On a
shared hosting environment this would probably be your home folder.

You could schedule it like this::

    * * * * * cd /path/to/project/ && /path/to/venv/bin/python /path/to/project/manage.py influxdb_get_disk_usage $HOME > $HOME/mylogs/cron/influxdb-get-disk-usage.log 2>&1

The series created in your influxdb will be named
``<prefix>server.disk.usage<postfix>`` and will have the following columns:

* ``value``: The total memory usage in bytes


influxdb_get_database_size
++++++++++++++++++++++++++

Collects the total disk usage for the given database.

You can run it like this::

    ./manage.py influxdb_get_database_size db_role db_name

You shoudl provide role and name for the database you want to measure. Make
sure that you have a ``.pgpass`` file in place so that you don't need to enter
a password for this user.

You could schedule it like this::

    * * * * * cd /path/to/project/ && /path/to/venv/bin/python /path/to/project/manage.py influxdb_get_database_size $HOME > $HOME/mylogs/cron/influxdb-get-database-size.log 2>&1

The series created in your influxdb will be named
`<prefix>server.postgresql.size<postfix>` and will have the following columns:

* ``value``: The total database size in bytes


InfluxDBEmailBackend
++++++++++++++++++++

If you would like to track tne number of emails sent, you can set your
`EMAIL_BACKEND`::

    EMAIL_BACKEND = 'influxdb_metrics.email.InfluxDBEmailBackend'

When the setting is set, metrics will be sent every time you run ``.manage.py
send_mail``.

The series created in your influxdb will be named
``<prefix>django.email.sent<postfix>`` and will have the following columns:

* ``value``: The number of emails sent


InfluxDBRequestMiddleware
+++++++++++++++++++++++++

If you would like to track the number and speed of all requests, you can add
the ``InfluxDBRequestMiddleware`` at the end of your ``MIDDLEWARE_CLASSES``::

    MIDDLEWARE_CLASSES = [
        ...
        'influxdb_metrics.middleware.InfluxDBRequestMiddleware',
    ]

The series created in your influxdb will be named
``<prefix>django.request<postfix>`` and will have the following columns:

* ``value``: The request time in milliseconds.
* ``is_ajax``: `1` if it was an AJAX request, otherwise `0`
* ``method``: The request method (`GET` or `POST`)
* ``module``: The python module that handled the request
* ``view``: The view class or function that handled the request
* ``referer``: The full URL from `request.META['HTTP_REFERER']`
* ``referer_tld``: The top level domain of the referer. It tries to be smart and
  regards ``google.co.uk`` as a top level domain (instead of ``co.uk``)

If you have a highly frequented site, this table could get big really quick.
You should make sure to create a shard with a low retention time for this
series (i.e. 7d) and add a continuous query to downsample the data into
hourly/daily averages. When doing that, you will obviously lose the detailed
information like ``referer`` and ``referer_tld`` but it might make sense to
create a second continuous query to count and downsample at least the
``referer_tld`` values.

NOTE: I don't know what impact this has on overall request time or how much
stress this would put on the influxdb server if you get thousands of requests.
It would probably wise to consider something like statsd to aggregate the
requests first and then send them to influxdb in bulk.


Tracking User Count
+++++++++++++++++++

This app's ``models.py`` contains a ``post_save`` and a ``post_delete`` handler
which will detect when a user is created or deleted.

The series created in your influxdb will be named
``<prefix>django.user.count<postfix>`` and will have the following columns:

* ``value``: The total number of users in the database


Tracking User Logins
++++++++++++++++++++

This app's ``models.py`` contains a handler for the ``user_logged_in`` signal.

The series created in your influxdb will be named
``<prefix>django.user.logins<postfix>`` and will have the following columns:

* ``value``: 1


Tracking Failed User Logins
+++++++++++++++++++++++++++

This app's ``models.py`` contains a handler for the ``user_logged_failed``
signal.

The series created in your influxdb will be named
``<prefix>django.user.logins.failed<postfix>`` and will have the following
columns:

* ``value``: 1


Contribute
----------

If you want to contribute to this project, please perform the following steps

.. code-block:: bash

    # Fork this repository
    # Clone your fork
    mkvirtualenv -p python2.7 django-influxdb-metrics
    make develop

    git co -b feature_branch master
    # Implement your feature and tests
    git add . && git commit
    git push -u origin feature_branch
    # Send us a pull request for your feature branch