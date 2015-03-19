Client API: Thrift
==================

"TBF" is a link:https://github.com/apache/thrift[Thrift protocol]
defined by UBF contract xref:the-hibari-ubf-protocol-contract[].
This section attempts to describe the Hibari Thrift API which allows
users to access Hibari with Thrift clients in any Thrift supported
programming languages, and how to extend the API for application uses.

The Hibari Thrift API
---------------------

The Hibari Thrift API is defined as Hibari Service in
link:./misc-codes/hibari.thrift[hibari.thrift]. At the time this API
was developed, only Thrift 0.4.0 is available to us. This version is
our first attempt to adopt Thrift.  Some of the functions and options
are not yet supported.

.. important::
   The Hibari Thrift API only supports Thrift 0.4.0 or above.

.. code-block:: thrift

   service Hibari {

      /**
       * Check connection availability / keepalive
       */
      oneway void keepalive()

      /**
       * Hibari Server Info
       */
      string info()

      /**
       * Hibari Description
       */
      string description()

      /**
       * Hibari Contract
       */
      string contract()

      /**
       * Add
       */
      HibariResponse Add(1: Add request)
          throws (1:HibariException ouch)

      /**
       * Replace
       */
      HibariResponse Replace(1: Replace request)
          throws (1:HibariException ouch)

      /**
       * Set
       */
      HibariResponse Set(1: Set request)
          throws (1:HibariException ouch)

      /**
       * Rename
       */
      HibariResponse Rename(1: Rename request)
          throws (1:HibariException ouch)

      /**
       * Delete
       */
      HibariResponse Delete(1: Delete request)
          throws (1:HibariException ouch)

      /**
       * Get
       */
      HibariResponse Get(1: Get request)
          throws (1:HibariException ouch)
      }

For each primitive utility function, it has exactly one input
parameter.  The parameter is an object that has a name matching its
function. The object carries all mandatory and optional parameters to
Hibari. This object could also be used to implement micro-transactions
in the future.

Mapping UBF Contract Types to Thrift Types
------------------------------------------

You can find more details of the UBF / Thrift type conversion in
(link:https://github.com/ubf/ubf-thrift[UBF-Thrift]).

Mapping UBF Contract to Thrift Service
--------------------------------------

Mapping UBF types to thrift primitives is different from mapping UBF
contracts to service. Thrift mainly uses 2 different types to compose
a request (struct and field).

If you are using Thrift to generate client code, you probably don't
need to worry about how the request being constructed. Visit
link:http://wiki.apache.org/thrift/ThriftGeneration[Thrift Wiki] for
the instruction to install Thrift and to generate client code.  You
will also need link:./misc-codes/hibari.thrift[hibari.thrift] to get
started.

If you are interested in the UBF contract, the Hibari NTBF contract
can be found in the file of ``ntbf_gdss_plugin.con``.

Examples of using a Thrift client
---------------------------------

Once you get the generated code, connecting to Hibari is easy.  For
example, adding the key ``'fookey'`` to table ``tab1`` with a value of
``'Hello, world!'`` in the following 3 languages.

.. highlight:: erlang

In Erlang::

  -include("hibari_thrift.hrl").

  % init
  {ok, Client} = thrift_client:start_link("127.0.0.1", 7600, hibari_thrift),

  % create the input parameter object
  Request = #add{table=<<"tab1">>, key=<<"fookey">>, value=<<"Hello, world!">},

  % send request
  try
    HibariResponse = thrift_client:call(Client, 'Add', [Request]),
  catch
    HibariException ->
      HibariException
  end,

  ok = thrift_client:close(Client).


.. highlight:: java

In Java::

  import com.hibari.rpc.*;

  // init
  TTransport transport = new TSocket("127.0.0.1", 7600);
  TProtocol proto = new TBinaryProtocol(transport);
  Hibari.Client client = new Hibari.Client(proto);
  transport.open();

  // create the input parameter object
  Add request = new Add("tab1", ByteBuffer.wrap("fookey".getBytes()),
    ByteBuffer.wrap("Hello, world!".getBytes())))

  // send request
  try {
    HibariResponse response = client.Add(request);
  } catch (HibariException e) {
    // ...
  }

  transport.close();


.. highlight:: python

In python::

  from hibari import Hibari

  # init
  transport = TSocket.TSocket('localhost', 7600)
  transport.setTimeout(None)
  transport = TTransport.TBufferedTransport(transport)
  protocol = TBinaryProtocol.TBinaryProtocol(transport)
  client = Hibari.Client(protocol)
  transport.open()

  # create the input parameter object
  request = Add()
  request.table = "tab1"
  request.key = b"fookey"
  request.value = b"Hello, world!"

  # send request
  response = client.Add(request)

  transport.close()


Mapping TBF Contract Responses From Thrift Client
-------------------------------------------------

TBF only responses one of two generic types to all functions in Hibari
Thrift API, HibariResponse or HibariException. One could expect a
HibariResponse in an any successful cases.  Otherwise a
HibariException should be thrown.
