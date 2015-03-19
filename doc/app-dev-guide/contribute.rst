.. highlight:: shell

Contributing to Hibari
======================

GitHub, Git, and Repo
---------------------

**to be added**

List the working directories for all of Hibari's "projects"::

  $ repo forall -c "pwd"

.. note::
   Each project has a corresponding Git repository and (default)
   revision.  Check the "manifests/hibari-default.xml" file for
   details.

Start a new topic (e.g. new-topic-name) branch::

  $ repo start new-topic-name `repo forall -c "pwd" | xargs echo`

Abandon an existing topic (e.g. topic-name) branch::

  $ repo abandon topic-name `repo forall -c "pwd" | xargs echo`

Track and checkout the master branch::

  $ repo forall -c "git branch --track master github/master"
  $ repo forall -c "git checkout master"

Track and checkout the dev (i.e. Development) branch::

  $ repo forall -c "git branch --track dev github/dev"
  $ repo forall -c "git checkout dev"

Code, Branch, and Version Management
------------------------------------

**to be added**

Documentation
-------------

**to be added**

Submitting Patches
------------------

**to be added**
