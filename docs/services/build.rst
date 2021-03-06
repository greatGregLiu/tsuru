.. Copyright 2016 tsuru authors. All rights reserved.
   Use of this source code is governed by a BSD-style
   license that can be found in the LICENSE file.

+++++++++++++++++++++
Building your service
+++++++++++++++++++++

.. _`service manifest`: `Creating a service manifest`_

Overview
========

This document is a hands-on guide to turning your existing cloud service into a
tsuru service.

In order to create a service you need to implement a provisioning API for your
service, which tsuru will call using `HTTP protocol
<http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Request_methods>`_
when a customer creates a new instance or binds a service instance with an app.

You will also need to create a YAML document that will serve as the service
manifest. We provide a command-line tool to help you to create this manifest
and manage your service.

Install crane
=============

`crane` is a command-line interface (CLI) used by service manager to manage
services. Before start, make sure that you have `crane` installed: https://tsuru-crane.readthedocs.io/en/latest/#installing

Creating your service API
=========================

To create your service API, you can use any programming language or framework.
In this tutorial we will use `Flask <http://flask.pocoo.org>`_.

Authentication
==============

tsuru uses basic authentication for authenticating the services, for more
details, check the :ref:`service API workflow
<service_api_flow_authentication>`.

Using Flask, you can manage basic authentication using a decorator described in
this Flask snippet: http://flask.pocoo.org/snippets/8/.

Prerequisites
-------------

First, let's ensure that Python and pip are already installed:

.. highlight:: bash

::

    $ python --version
    Python 2.7.2

    $ pip
    Usage: pip COMMAND [OPTIONS]

    pip: error: You must give a command (use "pip help" to see a list of commands)

For more information about how to install python you can see the `Python
download documentation <http://python.org/download/>`_ and about how to install
pip you can see the `pip installation instructions
<http://www.pip-installer.org/en/latest/installing.html>`_.

Now, with python and pip installed, you can use pip to install Flask:

.. highlight:: bash

::

    $ pip install flask

Now that Flask is installed, it's time to create a file called api.py and add
the code needed to create a minimal Flask application:

.. highlight:: python

::

    from flask import Flask
    app = Flask(__name__)

    @app.route("/")
    def hello():
        return "Hello World!"

    if __name__ == "__main__":
        app.run()

For run this app you can do:

.. highlight:: bash

::

    $ python api.py
     * Running on http://127.0.0.1:5000/

If you open your web browser and access the url http://127.0.0.1:5000/ you will
see the message "Hello World!".

Then, you need to implement the resources of a tsuru service API, as described
in the :doc:`tsuru service API workflow </services/api>`.

Listing available plans
-----------------------

tsuru will get the list of available plans by issuing a GET request in the
``/resources/plans`` URL. Let's create the view that will handle this kind
of request:


.. highlight:: python

::

    import json


    @app.route("/resources/plans", methods=["GET"])
    def plans():
        plans = [{"name": "small", "description": "small instance"},
                 {"name": "medium", "description": "medium instance"},
                 {"name": "big", "description": "big instance"},
                 {"name": "giant", "description": "giant instance"}]
        return json.dumps(plans)

Creating new instances
----------------------

For new instances tsuru sends a POST to /resources with the parameters needed
for creating an instance. If the service instance is successfully created, your
API should return 201 in status code.

Let's create the view for this action:

.. highlight:: python

::

    from flask import request


    @app.route("/resources", methods=["POST"])
    def add_instance():
        name = request.form.get("name")
        plan = request.form.get("plan")
        team = request.form.get("team")
        # use the given parameters to create the instance
        return "", 201

Binding instances to apps
-------------------------

In the bind action, tsuru calls your service via POST on
``/resources/<service-instance-name>/bind-app`` with the parameters needed for
binding an app into a service instance.

If the bind operation succeeds, the API should return 201 as status code with
the variables to be exported in the app environment on body in JSON format.

As an example, let's create a view that returns a json with a fake variable
called "SOMEVAR" to be injected in the app environment:

.. highlight:: python

::

    import json

    from flask import request


    @app.route("/resources/<name>/bind-app", methods=["POST"])
    def bind_app(name):
        app_host = request.form.get("app-host")
        # use name and app_host to bind the service instance and the #
        application
        envs = {"SOMEVAR": "somevalue"}
        return json.dumps(envs), 201

Unbinding instances from apps
-----------------------------

In the unbind action, tsuru issues a ``DELETE`` request to the URL
``/resources/<service-instance-name>/bind-app``.

If the unbind operation succeeds, the API should return 200 as status code.
Let's create the view for this action:

.. highlight:: python

::

    @app.route("/resources/<name>/bind-app", methods=["DELETE"])
    def unbind_app(name):
        app_host = request.form.get("app-host")
        # use name and app-host to remove the bind
        return "", 200

Whitelisting units
------------------

When binding and unbindin application and service instances, tsuru will also
provide information about units that will have access to the service instance,
so the service API can handle any required whitelisting (writing ACL rules to a
network switch or authorizing access in a firewall, for example).

tsuru will send POST and DELETE requests to the route
``/resources/<name>/bind``, with the host of the app and the unit, so any
access control can be handled by the API:

.. highlight:: python

::

    @app.route("/resources/<name>/bind", methods=["POST", "DELETE"])
    def access_control(name):
        app_host = request.form.get("app-host")
        unit_host = request.form.get("unit-host")
        # use unit-host and app-host, according to the access control tool, and
        # the request method.
        return "", 201

Removing instances
------------------

In the remove action, tsuru issues a DELETE request to the URL
``/resources/<service_name>``.

If the service instance is successfully removed, the API should return 200 as
status code.

Let's create a view for this action:

.. highlight:: python

::

    @app.route("/resources/<name>", methods=["DELETE"])
    def remove_instance(name):
        # remove the instance named "name"
        return "", 200

Checking the status of an instance
----------------------------------

To check the status of an instance, tsuru issues a GET request to the URL
``/resources/<service_name>/status``. If the instance is ok, this URL should
return 204.

Let's create a view for this action:

.. highlight:: python

::

    @app.route("/resources/<name>/status", methods=["GET"])
    def status(name):
        # check the status of the instance named "name"
        return "", 204

The final code for our "fake API" developed in Flask is:

.. highlight:: python

::

    import json

    from flask import Flask, request

    app = Flask(__name__)


    @app.route("/resources/plans", methods=["GET"])
    def plans():
        plans = [{"name": "small", "description": "small instance"},
                 {"name": "medium", "description": "medium instance"},
                 {"name": "big", "description": "big instance"},
                 {"name": "giant", "description": "giant instance"}]
        return json.dumps(plans)


    @app.route("/resources", methods=["POST"])
    def add_instance():
        name = request.form.get("name")
        plan = request.form.get("plan")
        team = request.form.get("team")
        # use the given parameters to create the instance
        return "", 201


    @app.route("/resources/<name>/bind-app", methods=["POST"])
    def bind_app(name):
        app_host = request.form.get("app-host")
        # use name and app_host to bind the service instance and the #
        application
        envs = {"SOMEVAR": "somevalue"}
        return json.dumps(envs), 201


    @app.route("/resources/<name>/bind-app", methods=["DELETE"])
    def unbind_app(name):
        app_host = request.form.get("app-host")
        # use name and app-host to remove the bind
        return "", 200


    @app.route("/resources/<name>", methods=["DELETE"])
    def remove_instance(name):
        # remove the instance named "name"
        return "", 200


    @app.route("/resources/<name>/bind", methods=["POST", "DELETE"])
    def access_control(name):
        app_host = request.form.get("app-host")
        unit_host = request.form.get("unit-host")
        # use unit-host and app-host, according to the access control tool, and
        # the request method.
        return "", 201


    @app.route("/resources/<name>/status", methods=["GET"])
    def status(name):
        # check the status of the instance named "name"
        return "", 204

    if __name__ == "__main__":
        app.run()

.. _service_manifest:

Creating a service manifest
===========================

Using crane you can create a manifest template:

.. highlight:: bash

::

    $ crane template

This will create a manifest.yaml in your current path with this content:

.. highlight:: yaml

::

    id: servicename
    password: abc123
    endpoint:
        production: production-endpoint.com

The manifest.yaml is used by crane to defined the ID, the password and the
production endpoint of your service.

Change these information in the created manifest, and the `submit your
service`_:

.. highlight:: yaml

::

    id: servicename
    username: username_to_auth
    password: 1CWpoX2Zr46Jhc7u
    endpoint:
      production: production-endpoint.com
        test: test-endpoint.com:8080

_`submit your service`: `Submiting your service API`_

Submiting your service API
==========================

To submit your service, you can run:

.. highlight:: bash

::

    $ crane create manifest.yaml

For more details, check the :doc:`service API workflow </services/api>` and the
:doc:`crane usage guide </services/usage>`.
