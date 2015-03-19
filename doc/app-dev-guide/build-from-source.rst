.. highlight:: shell-session

Building Hibari from Source
===========================

This section describes the basic recipes to build the following items:

- Hibari Release Package
- Hibari Documentation
- Erlang/OTP System

Required Third Party Software
-----------------------------

Before getting started, review this checklist of tools and
software. Please install and set up as needed.

Mandatory Items (Required for Building Hibari)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The following software is required in order to download Hibari and
build a release package:

- Git -- http://git-scm.com/

  * Must be version 1.5.4 or newer.

    * 1.7.3.4 is the version most recently tested for Hibari.

  * If you haven't yet done so, please configure your email address
    and name for Git::

      $ git config --global user.email "you@example.com"
      $ git config --global user.name "Your Name"

  * If you haven't yet done so, you must sign up for a GitHub
    account -- https://github.com/

    * Anonymous read-only access using the GIT protocol is default.
    * Team members with read-write access: be sure to add your SSH
      public key under your GitHub account.

- Python -- http://www.python.org

  * Required by Repo
  * Must be version 2.4 or newer

    * 2.7 is the version most recently tested for Hibari.

    .. caution::
       Python 3.x might be too new.

- Repo -- http://source.android.com/source/git-repo.html

  * Install as follows::

      $ mkdir -p ~/bin
      $ curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
      $ chmod a+x ~/bin/repo

  * The downloading and packaging process also uses Rebar
    (https://github.com/basho/rebar/wiki) but this tool is included in
    the Hibari Git repositories so you do not need to install it
    separately.

- OpenSSL -- http://www.openssl.org/

  * Required for Erlang's crypto module.

- Erlang/OTP -- http://www.erlang.org/

  * Must be version R16B01 or newer.

    * 17.4 is the version most recently tested for Hibari.

  * For information on building Erlang/OTP from source, see
    <<ErlangOTP>> in this document.


Optional Items (Required for Building Hibari's Documentation)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The following software is required only if you want to build Hibari's
documentation from source. Note that an online version of the
documentation is available at http://hibari.github.com/hibari-doc/.

- AsciiDoc -- http://www.methods.co.nz/asciidoc/index.html

  * Must be version 8.6.1 or newer

    * 8.6.4 is the version most recently tested for Hibari

  * Plus the following support tools:

    * ImageMagick -- http://www.imagemagick.org/
    * Graphviz -- http://www.graphviz.org/
    * Mscgen -- http://www.mcternan.me.uk/mscgen/
    * Dia -- http://projects.gnome.org/dia/

- dblatex -- http://dblatex.sourceforge.net/

  * Optional for building a PDF version of Hibari's documentation.

- w3m -- http://w3m.sourceforge.net/

  * Optional for building a text version of Hibari's documentation.


Downloading Hibari
------------------

Follow these steps to download the Hibari repositories from GitHub.

#. Create a working directory and retrieve the Hibari manifest files::

     $ mkdir working-directory
     $ cd working-directory
     $ repo init -u git://github.com/hibari/manifests.git -m hibari-default.xml

   .. note::
      Your "Git" identity is needed during the repo init step.  Please
      enter the name and email of your GitHub account if you have one.
      Team members having read-write access should use ``repo init -u
      git@github.com:hibari/manifests.git -m hibari-default-rw.xml``.

   .. tip::
      If you want to checkout the latest development version of Hibari,
      please append `` -b dev`` to the repo init command.

#. Download Hibari's Git repositories::

     $ Repo sync

   After the repo sync, your working directory has the following structure::

     <working-directory>
      |- hibari/
        |- .git/
        |- .gitignore
        |- Makefile
        |- dialyze-ignore-warnings.txt
        |- dialyze-nospec-ignore-warnings.txt
        |- lib/                             <1>
          |- <application_name>/
            |- .git/
            |- .gitignore
            |- ebin/
            |- include/
              |- *.hrl
            |- priv/
            |- rebar.config
            |- src/
              |- <application_name>.app.src
              |- *.erl
            |- test/
              |- eunit/
                |- *.erl
              |- eqc/
                |- *.erl
          :
        |- rebar
        |- rebar.config
        |- rel/                             <2>
          |- files/
            |- app.config
            |- erl
            |- hibari
            |- hibari-admin
            |- nodetool
            |- nodetool-admin
            |- vm.args
          |- hibari/
            :
            |- releases/
              |- <release_vsn>/
                :
              :
            :
          |- reltool.config
      |- hibari-doc/                        <3>
        :
      |- manifests/                         <4>
        :
      |- patches/                           <5>
        :
      |- rebar/                             <6>
        :
      |- .repo/
        :

<1> Applications
<2> Releases
<3> Documentation
<4> Manifests
<5> Patches
<6> Rebar


Building the Hibari Release Package
-----------------------------------

Follow these steps to build a Hibari release package.

#. Building *basic recipe*::

     $ cd working-directory/hibari
     $ make

.. tip::
   If the response is "make: erl: Command not found", please make
   sure Erlang/OTP is installed and "otp-installing-directory-name/bin"
   is added to your $PATH environment.

#. Release packaging *basic recipe*::

     $ cd working-directory/hibari
     $ make package

.. note::
   A release package tarball "hibari-X.Y.Z-dev-ARCH-WORDSIZE.tgz"
   and md5sum file "hibari-X.Y.Z-dev-ARCH-WORDSIZE-md5sum.txt" is written
   into your working-directory. You can then use these files to perform a
   single-node or multi-node Hibari installation as described in
   <<getting-started>>.

[[HibariAsciiDoc]]

Building Hibari's Documentation
-------------------------------

Follow these steps to build Hibari's documentation.

#. Building Hibari's "Guides" *basic recipe*::

     $ cd working-directory/hibari-doc/src/hibari
     $ make clean -OR- make realclean
     $ make

#. Building Hibari's "Website" *basic recipe*::

     $ cd working-directory/hibari-doc/src/hibari/website
     $ make clean -OR- make realclean
     $ make

.. note::
   HTML documentation is written in the "./public_html" directory.

Hibari's documentation is authored using AsciiDoc and a few auxillary
tools:

- ImageMagick
- dblatex
- Dia
- Graphviz
- Mscgen
- w3m

Hibari's documentation is generated with AsciiDoc and a manually
modified version of the a2x tool.  A fake lang-ja.conf file can be
easily created by making a symlink to the lang-en.conf file.

.. code-block:: python

   diff -r -u 8.6.4-orig/bin/a2x.py 8.6.4/bin/a2x.py
   --- 8.6.4-orig/bin/a2x.py	2011-04-24 00:50:26.000000000 +0900
   +++ 8.6.4/bin/a2x.py	2011-04-24 00:35:55.000000000 +0900
   @@ -156,7 +156,10 @@
     def shell_copy(src, dst):
       verbose('copying "%s" to "%s"' % (src,dst))
         if not OPTIONS.dry_run:
   -        shutil.copy(src, dst)
   +        try:
   +            shutil.copy(src, dst)
   +        except shutil.Error:
   +            return

     def shell_rm(path):
         if not os.path.exists(path):
    Only in 8.6.4/etc/asciidoc: lang-ja.conf


[[ErlangOTP]]

Building and Installing Erlang/OTP
----------------------------------

Follow these steps to download and build Erlang/OTP from source, and
to install the system. These steps provide a basic recipe; not all
options are addressed.

.. note::
   Please make sure to have the 'openssl-devel' package installed
   on your system before configuring and building Erlang/OTP.

#. Download the source code for your Erlang/OTP system::

     $ cd working-directory
     $ wget http://www.erlang.org/download/otp_src_R14B01.tar.gz

#. Untar the source code for your Erlang/OTP system::

     $ tar -xzf otp_src_R14B01.tar.gz

#. Configure Erlang/OTP::

     $ cd working-directory/otp_src_R14B01
     $ ./configure --prefix=otp-installing-directory-name

#. Build Erlang/OTP::

     $ make

#. Install Erlang/OTP::

     $ sudo make install

.. caution::
   Please make sure "otp-installing-directory-name/bin" is added
   to your $PATH environment.
