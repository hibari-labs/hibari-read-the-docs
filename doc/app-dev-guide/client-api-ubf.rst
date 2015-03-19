Client API: UBF
===============

link:http://github.com/ubf/ubf[The UBF protocol] is a
formally-specified family of protocols that are supported by a large
number of client languages.  This section attempts to describe the
layers of the UBF protocol stack, how to use the UBF client in Erlang
and other languages, and how to use that client to access a Hibari
storage cluster.

The Hibari source distribution includes UBF/EBF protocol support for the
following languages:

- Erlang, see xref:using-ubf-erlang-client[]
- Java, see xref:using-ubf-java-client[]
- Python, see xref:using-ubf-python-client[]

[[hibari-server-impl-of-ubf-proto-stack]]

The Hibari Server's Implementation of the UBF Protocol Stack
------------------------------------------------------------

UBF(A): Bottom Layer, transport and session protocol layer
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. highlight:: erlang

This layer plays the same basic role as many other serialized data
transport protocols that use TCP for host-to-host transport, such as
link:http://en.wikipedia.org/wiki/Open_Network_Computing_Remote_Procedure_Call[ONC-RPC],
link:http://en.wikipedia.org/wiki/IIOP[CORBA IIOP],
link:http://en.wikipedia.org/wiki/Protocol_buffers[Protocol Buffers],
and link:http://en.wikipedia.org/wiki/Thrift_(protocol)[Thrift].

Hibari servers support several of these session protocols on top
of a TCP/IP transport protocol.  The choice of session protocol is
a matter of convenience and/or support for the application
developer. Hibari should be as easy for an app developer to use
Ruby and JSON-RPC as it is to use Python and Thrift or EBF.

- UBF(A), Joe Armstrong's original session layer protocol
- EBF, the Erlang Binary Format.  The session layer protocol is a
  thin, efficient that uses the Erlang BIFs `term_to_binary()` and
  `binary_to_term()` to serialize Erlang data terms.  This protocol
  is very closely related to the link:http://bert-rpc.org/[BERT protocol].
- JSON over TCP, also called JSF (the JavaScript
  Format).  Erlang terms are encoded as
  link:http://en.wikipedia.org/wiki/JSON[JSON terms]
  and transmitted directly over a TCP transport.  This
  protocol is not in common use but is easy to implement in the UBF
  server framework.
- HTTP, the link:http://en.wikipedia.org/wiki/HTTP[Hypertext
  Transfer Protocol].  This protocol is used to support Hibari's
  link:http://en.wikipedia.org/wiki/JSON-RPC[JSON-RPC] server.
- link:http://en.wikipedia.org/wiki/Thrift_(protocol)[Thrift].
  Similar to EBF, except that Thrift's binary encoding is used for
  the wire protocol instead of UBF(A) or Erlang's native wire
  formats.
- link:http://en.wikipedia.org/wiki/Protocol_buffers[Protocol Buffers].
  Similar to EBF, except that Google's Protocol Buffers binary
  encoding is used for the wire protocol instead of UBF(A) or
  Erlang's native wire formats.
  *Hibari support is experimental (i.e. not yet implemented).*
- link:http://hadoop.apache.org/avro/docs/current/[Avro].
  Similar to EBF, except that Avro's binary encoding is used for the
  wire protocol instead of UBF(A) or Erlang's native wire formats.
  *Hibari support is experimental (i.e. not yet implemented).*

UBF(B): Middle Layer, the "contract"
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

UBF(B) is a programming language for describing types in UBF(A)
and protocols between clients and servers. UBF(B) is roughly
equivalent to to Verified XML, XML-schemas, SOAP and WDSL.

This layer enforces a protocol "contract", a formal specification of
all data sent by the client and by the server.  Any data that does not
precisely conform to the protocol is rejected by the contract checker
(which is embedded in the server).  If the client wishes, it may also
use the contract checker to validate data sent by the server, though
this not commonly done.

UBF(C): Top Layer, the UBF Metaprotocol
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The metaprotocol is used at the beginning of a UBF session to select
one of the UBF(B) contracts that the TCP listener is capable of
offering.  At the moment, Hibari servers support only the "gdss"
contract, but other contracts may be added in the future.

[[ubf-representation-of-strings]]

UBF representation of strings vs. binaries
------------------------------------------

The Erlang language does not have a data type specifically for
strings.  Instead, strings are typically represented as lists of
integers (ASCII byte values) and/or binaries.

A UBF contract makes a distinction between a string, list, and
binary. In the case of a string, UBF(A) encodes a string using the
notation ``{'#S', "Hello, world!"}`` to represent the string "Hello,
world!".

This string encoding is cumbersome to use for developers; in Erlang,
the ``ubf.hrl`` header file includes a macro ``?S("Hello, world!")``
as a slightly less ugly shortcut.  When using other languages, the
2-tuple and the atom ``'#S'``  would be created as any other 2-tuple
and atom.

Fortunately, there is only one case where the string type is
necessary: using the ``startSession`` metaprotocol command to start
using the Hibari data server contract.  An example will be shown
below.

[[using-ubf-in-any-language]]

Steps for Using a UBF-based Protocol in Any Language
----------------------------------------------------

The steps to use a UBF-based protocol are the same in any language.

1. Create a connection to the UBF server.

   * ... or the EBF server, or the JSON-RPC server, or the Thrift
     server, or the ....

2. Use the UBF metaprotocol to start using the ``gdss`` contract,
   i.e. the Hibari server contract.
3. Send one or more Hibari server queries and decode the respective
   server responses.
4. Close the connection to the UBF server.

[[the-hibari-ubf-protocol-contract]]

The Hibari UBF Protocol Contract
--------------------------------

The Hibari UBF Protocol contract can be found in the file
``ubf_gdss_plugin.con``.

.. note::
   See the Hibari source code for the most up-to-date version of
   this file. link:./misc-codes/ubf_gdss_plugin.con[This documentation has a copy
   of `ubf_gdss_plugin.con`], though it may be slightly out-of-date.

The names of the UBF types specified in the contract may differ
slightly from the names of the types used in this document's
xref:client-api-erlang[].  For example, the UBF contract calls the key
expiration time time ``exp_time()``, while the type system in this
document calls it ``expiry()``.  However, in all cases of slightly
different names, the fundamental data type that both names use is the
same: e.g. ``integer()`` for expiration time.

For each command, the UBF contract uses the following naming
conventions:

- ``CommandName_req()`` for the request sent from client -> server,
  e.g. ``set_req()`` for the ``set`` command.
- ``CommandName_res()`` for the response sent from server -> client,
  e.g. ``set_res()`` for the ``set`` response.

The general form of a UBF RPC call is a tuple.  The first element in
the tuple is the name of the command, and the following elements are
arguments for that command.  The response can be any Erlang term, but
the Hibari contract will only return the atom or tuple types.

The following is a mapping of UBF client request type to its Erlang
API function, in alphabetical order.:

- ``add_req()`` -> ``brick_simple:add()``, see xref:brick-simple-add[].
- ``delete_req()`` -> ``brick_simple:delete()``, see
  xref:brick-simple-delete[].
- ``do_req()`` -> ``brick_simple:do()``, see xref:brick-simple-do[].
- ``get_req()`` -> ``brick_simple:get()``, see xref:brick-simple-get[].
- ``get_many_req()`` -> ``brick_simple:get_many()``, see
  xref:brick-simple-get-many[].
- ``replace_req()`` -> ``brick_simple:replace()``, see
  xref:brick-simple-replace[].
- ``set_req()`` -> ``brick_simple:set()``, see xref:brick-simple-set[].
- ``rename_req()`` -> ``brick_simple:rename()``, see
  xref:brick-simple-rename[].

[[using-ubf-erlang-client]]

Using the UBF Client Library for Erlang
---------------------------------------

.. important::

   1. When using the Erlang shell for experimentation & prototyping, that
      shell must have the path to the Erlang UBF client
      library in its search path.  The easiest way to do this is to use
      the arguments ``-pz /path/to/ubf/library/ebin`` to your Erlang
      shell's ``erl`` command.

   2. When writing code, the statement ``-include("ubf.hrl").`` at the top
      of your source module to gain access to the ``?S()`` macro.  Due to
      limitations in the Erlang shell, macros cannot be used in the shell.

As outlined in xref:using-ubf-in-any-language[], the first step is to
create a connection to a Hibari server.  If the Hibari cluster has
multiple nodes, then it doesn't matter which one that you connect to:
all nodes can handle any UBF request and will route the query to the
proper brick.

#. Create a connection to the UBF server (on "localhost" TCP port 7581)::

     (asdf@bb3)54> {ok, P1, _} = ubf_client:connect("localhost", 7581, [{proto, ubf}], 5000).
     {ok,<0.139.0>,{'#S', "gdss_meta_server"}}

   The second step is to use the UBF metaprotocol to select the Hibari
   server, contract, called "gdss", for all further commands for this
   connection.

   .. tip::
      The Hibari server contract is "stateless".  All replies terms
      from the ``ubf_client:rpc/2`` function use the form
      ``{reply,ServerReply,UBF_StateName}``.  Because the Hibari
      server contract is stateless, the ``UBF_StateName`` will
      always be the atom ``none``.

#. Use the UBF metaprotocol to request the "gdss" contract::

     (asdf@bb3)55> ubf_client:rpc(P1, {startSession, {'#S', "gdss"}, []}).
     {reply,{ok,ok},none}

   Now that the UBF connection is set up, we can use it to set a key
   "foo".

#. Set the key "foo" in table `tab1` with the value "foo val", no
   expiration time, no flags, and a timeout of 5 seconds::

     (asdf@bb3)59> ubf_client:rpc(P1, {set, tab1, <<"foo">>, <<"foo val">>, 0, [], 5000}).
     {reply,ok,none}

   .. note::
      Note that the return value of both the ``set_req()`` (in the
      example above) and ``get_req()`` (in the example below) return the
      same types described in the xref:brick-simple-set[] and
      xref:brick-simple-get[], respectively.

      The only difference is that the ``ubf_client:rpc/2`` function wraps
      the server's reply in a 3-tuple: ``{reply,ServerReply,none}``.

#. Get the key "foo" in table `tab1`, timeout in 5 seconds::

     (asdf@bb3)66> ubf_client:rpc(P1, {get, tab1, <<"foo">>, [], 5000}).
     {reply,{ok,1273009092549799,<<"foo val">>},none}

   If the client sends a request that violates the contract, the server
   will tell you, as in this example.

#. Send a contract-violating request::

     (asdf@bb3)89> ubf_client:rpc(P1, {bbb, 3000}).
     {reply,{clientBrokeContract,{bbb,3000},[]},none}

   When you are done with the connection, it is polite to close the
   connection explicitly.  The server will quietly clean up its side
   of the connection if the client forgets to call or cannot call
   ``stop/1``.

#. Close the UBF connection::

     (asdf@bb3)92> ubf_client:stop(P1).
     ok

[[using-ubf-java-client]]

Using the UBF Client Library for Java
-------------------------------------

.. highlight:: shell

The source code for the UBF client library for Java is included in
the UBF source repository at
link:http://github.com/ubf/ubf[http://github.com/ubf/ubf], in
the ``priv/java`` subdirectory.

Compiling the UBF client library for Java
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Please update your UBF client library code to the "master" branch
   for a date after 10 May 2010, or use the Git tag "v1.14" or later.
   Versions of the library before 10 May 2010 and tag "v1.14" have
   several bugs that will prevent the UBF client from working
   correctly.
#. Change directory to the ``priv/java`` directory of the UBF client
   library source distribution.
#. Run ``make``.
#. (Optional) Copy the class files in the ``classes`` subdirectory to
   a suitable directory for your Java development environment.

Compiling the UBF client library test program HibariTest.java
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Change directory to the ``gdss-ubf-proto/priv/java`` subdirectory in
   the Hibari source distribution.
#. Edit the ``Makefile`` to change the ``UBF_CLASSES_DIR`` variable to
   point to the ``priv/java/classes`` subdirectory of the UBF package's
   source code (or the subdirectory where those classes have been
   formally installed on your system).
#. Run the following two ``make`` commands.  The second assumes that the
   Hibari server's UBF server is on the local machine, "localhost"::

     $ make HibariTest
     $ make run-HibariTest

#. If the Hibari server is not running on the local machine, then run
   ``make -n run-HibariTest`` to show the ``java`` command that is
   used to run the test program.  Cut-and-paste the command into your
   shell, then edit the last argument to specify the hostname of a
   Hibari server.

Examining the HibariTest.java test program
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. hightlight:: java
   :linenothreshold: 5

The ``main()`` function does three things:

#. Create a new UBF connection to a Hibari server (hostname/IP address
   is specified in the first command line argument) and requests the
   ``gdss`` contract via the UBF metaprotocol.
#. Run the small test cases in the ``test_hibari_basics()`` method.
#. Close the UBF session and exit.

.. [[the-ubf-hibaritest-main-method]]

The ubf.HibariTest.main() method
""""""""""""""""""""""""""""""""

.. code-block:: java

   public class HibariTest {

       public static void main(String[] args) throws Exception {
           Socket sock = null;
           UBFClient ubf = null;

           try {
               sock = new Socket(args[0], 7581);
               ubf = UBFClient.new_via_sock(new UBFString("gdss"), new UBFList(),
                       new FooHandler(), sock);
           } catch (Exception e) {
               System.out.println(e);
               System.exit(1);
           }

           test_hibari_basics(ubf);

           ubf.stopSession();
           System.out.println("Success, it works");
           System.exit(0);
       }
       /* ... */
    }


The ``test_hibari_basics()`` method performs the same basic UBF
operations as the Python EBF demonstration script described in
xref:using-ubf-python-client[].  Unlike the Python demo script, the
demo program does not use the Hibari ``do()`` command but rather then
single-operation commands like ``get(`` and ``set()``.

.. highlight:: java

#. Delete the key ``foo`` from table ``tab1``::

     public static void test_hibari_basics(UBFClient ubf)
             throws IOException, UBFException {
         // setup
         UBFObject res1 = ubf.rpc(
                UBF.tuple( new UBFAtom("delete"), new UBFAtom("tab1"),
                           new UBFBinary("foo"), new UBFList(),
                           new UBFInteger(4000)));
         System.out.println("Res 1:" + res1.toString());

#. Add the key `foo` to table `tab1`::

         // add - ok
         UBFObject res2 = ubf.rpc(
                 UBF.tuple( new UBFAtom("add"), atom_tab1,
                             new UBFBinary("foo"), new UBFBinary("bar"),
                             new UBFInteger(0), new UBFList(),
                             new UBFInteger(4000)));
         System.out.println("Res 2:" + res2.toString());
         if (! res2.equals(atom_ok))
             System.exit(1);

#. Add the key `foo` to table `tab1` again, this time expecting a failure::

         // add - ng
         UBFObject res3 = ubf.rpc(
                 UBF.tuple( new UBFAtom("add"), atom_tab1,
                            new UBFBinary("foo"), new UBFBinary("bar"),
                            new UBFInteger(0), new UBFList(),
                            new UBFInteger(4000)));
         System.out.println("Res 3:" + res3.toString());
         if (! ((UBFTuple)res3).value[0].equals(atom_key_exists))
             System.exit(1);

#. Get the key `foo` from table `tab1`::

         // get - ok
         UBFObject res4 = ubf.rpc(
                 UBF.tuple( new UBFAtom("get"), atom_tab1,
                            new UBFBinary("foo"), new UBFList(),
                            new UBFInteger(4000)));
         System.out.println("Res 4:" + res4.toString());
         if (! ((UBFTuple)res4).value[0].equals(atom_ok) ||
             ! ((UBFTuple)res4).value[2].equals("bar"))
             System.exit(1);

#. Set the key `foo` in table `tab1` to `bar bar`::

         // set - ok
         UBFObject res5 = ubf.rpc(
                 UBF.tuple( new UBFAtom("set"), atom_tab1,
                            new UBFBinary("foo"), new UBFBinary("bar bar"),
                            new UBFInteger(0), new UBFList(),
                            new UBFInteger(4000)));
         System.out.println("Res 5:" + res5.toString());
         if (! res5.equals(atom_ok))
             System.exit(1);

#. Get `foo` again and verify that the value is `bar bar`::

         // get - ok
         UBFObject res6 = ubf.rpc(
                 UBF.tuple( new UBFAtom("get"), atom_tab1,
                            new UBFBinary("foo"), new UBFList(),
                            new UBFInteger(4000)));
         System.out.println("Res 6:" + res6.toString());
         if (! ((UBFTuple)res6).value[0].equals(atom_ok) ||
             ! ((UBFTuple)res6).value[2].equals("bar bar"))
             System.exit(1);

The UBF event handler interface
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each ``UBFClient`` instance uses a separate thread to read data from
the server and do any of the following:

1. Signal to the other thread that a synchronous RPC response was
   received from the server.
2. Run a callback function when an ``event_out`` asynchronous event is
   received from the server.
3. The socket was closed unexpectedly.

In cases #2 and #3, a class that implements the ``UBFEventHandler``
interface is used to define the action to be taken in those cases.

The ``HibariTest.java`` contains a sample implementation of callback
functions for asynchronous events. A real application would probably
want to do something much more helpful than this example does.

.. code-block:: java

   public static class FooHandler implements UBFEventHandler {
       public FooHandler() {
       }
       public void handleEvent(UBFClient client, UBFObject event) {
           System.out.println("Hey, got an event: " + event.toString());
       }
       public void connectionClosed(UBFClient client) {
           System.out.println("Hey, connection closed, ignoring it\n");
       }
   }

.. tip::
   See xref:the-ubf-hibaritest-main-method[] for an example that uses
   this ``FooHandler`` class.

[[using-ubf-python-client]]

Using the EBF Client Library for Python
---------------------------------------

.. highlight:: shell

The source code for the EBF client library for Python is included in
the UBF source repository at
link:http://github.com/ubf/ubf[http://github.com/ubf/ubf], in
the ``priv/python`` subdirectory.

NOTE: Recall that the EBF protocol is very closely related to UBF.  The
only significant difference is the "layer 5" session protocol layer:
instead of using the UBF(A) protocol, the EBF (Erlang Binary Format)
protocol is used instead.  See
xref:hibari-server-impl-of-ubf-proto-stack[] for more details.

In addition, you will need the "py_interface" package, developed by
Tomas Abrahamsson and others. "py-interface" is distributed under the
link:http://www.fsf.org/licensing/education/licenses/lgpl.html[GNU
Library General Public License]. A git repository is hosted at
repo.or.cz. To clone it and build it, use::

  $ git clone git://repo.or.cz/py_interface.git
  $ cd py_interface
  $ autoconf
  $ ./configure
  $ make
  $ pwd

Use the output of the last command, ``pwd``, to remember the full
directory path to the "py-interface" library.  The example below
assumes that path is ``/tmp/py-interface``.

The ``pyebf.py`` file contains a small unit test that makes several
calls to the Hibari UBF contract's ``do_req()`` command. The results
of (almost) every command are verified using the ``assert`` function.

.. code-block:: python

   env PYTHONPATH=/path/to/py_interface python pyebf.py

.. highlight:: python

#. Connect to the Hibari server on "localhost" TCP port 7580 and use
   the UBF metaprotocol to switch to the ``gdss`` contract::

     ## login
     ebf.login('gdss', 'gdss_meta_server')

#. Delete the key ``'foo'`` from table ``tab1``::

     ## setup
     req0 = (Atom('do'), Atom('tab1'), [(Atom('delete'), 'foo', [])], [], 1000)
     res0 = ebf.rpc('gdss', req0)

#. Get the key ``'foo'`` from table ``tab1``::

     ## get - ng
     req1 = (Atom('do'), Atom('tab1'), [(Atom('get'), 'foo', [])], [], 1000)
     res1 = ebf.rpc('gdss', req1)
     assert res1[0] == 'key_not_exist'

#. Add the key ``'foo'`` to table ``tab1``.  The ``do_req()``
   interface requires managing the timestamp integers explicitly by
   the client; the timestamp ``1`` is used here::

     ## add - ok
     req2 = (Atom('do'), Atom('tab1'),
             [(Atom('add'), 'foo', 1, 'bar', 0, [])], [], 1000)
     res2 = ebf.rpc('gdss', req2)
     assert res2[0] == 'ok'

#. Add the key ``'foo'`` to table ``tab1``::

     ## add - ng
     req3 = (Atom('do'), Atom('tab1'),
             [(Atom('add'), 'foo', 1, 'bar', 0, [])], [], 1000)
     res3 = ebf.rpc('gdss', req3)
     assert res3[0][0] == 'key_exists'
     assert res3[0][1] == 1

#. Get the key ``'foo'`` from table ``tab1``, verifying that the
   timestamp is still ``1`` and value is still ``'bar'``::

     ## get - ok
     req4 = (Atom('do'), Atom('tab1'), [(Atom('get'), 'foo', [])], [], 1000)
     res4 = ebf.rpc('gdss', req4)
     assert res4[0][0] == 'ok'
     assert res4[0][1] == 1
     assert res4[0][2] == 'bar'

#. Set the key ``'foo'`` from table ``tab1``, using a new timestamp
   ``2``::

     ## set - ok
     req5 = (Atom('do'), Atom('tab1'),
             [(Atom('set'), 'foo', 2, 'baz', 0, [])], [], 1000)
     res5 = ebf.rpc('gdss', req5)
     assert res5[0] == 'ok'

#. Get the key ``'foo'`` from table ``tab1``, verifying both the new
   timestamp and new value::

     ## get - ok
     req6 = (Atom('do'), Atom('tab1'), [(Atom('get'), 'foo', [])], [], 1000)
     res6 = ebf.rpc('gdss', req6)
     assert res6[0][0] == 'ok'
     assert res6[0][1] == 2
     assert res6[0][2] == 'baz'
