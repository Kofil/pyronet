# PyroNet #
PyroNet is a fast, minimalistic, lowlevel abstraction layer over the high performance Java NIO API. The aim of the API is not to provide new functionality, but to reduce the complexity of interfacing with non-blocking I/O. The PyroNet API takes care of the boilerplate code, while not forcing any protocol on the developer. The API is event driven and uses a single NIO Selector per PyroSelector. Single-thread access to PyroSelector(s) is enforced by the API. When facing hundreds or even thousands of connections, use multiple PyroSelectors or a PyroSelector pool. You will receive event notifications through a listener interface.

## Audience ##
The code in this project is targeted at Java programmers that (need to) squeeze out the last drop of performance out of their network code, and that see little value in code snippets, documentation or tutorials.
Have fun with the SVN repository, there are runnable test cases to get you started. The API is, just like NIO, not for the faint of heart, and if you're not out for a lowlevel API, I'd highly recommend you to take a look at [KryoNet](http://code.google.com/p/kryonet/) for an excellent highlevel networking API based on NIO.

## Core classes ##
As said, the core classes do not force any protocol on the developer. You can write HTTP servers, FTP servers, BitTorrent clients: you name it. The overall structure is very much like that of Java NIO.

[PyroSelector](http://code.google.com/p/pyronet/source/browse/trunk/%20pyronet/jawnae/pyronet/PyroSelector.java)<br>
-> <a href='http://code.google.com/p/pyronet/source/browse/trunk/%20pyronet/jawnae/pyronet/events/PyroSelectorListener.java'>PyroSelectorListener</a>

<a href='http://code.google.com/p/pyronet/source/browse/trunk/%20pyronet/jawnae/pyronet/PyroServer.java'>PyroServer</a><br>
-> <a href='http://code.google.com/p/pyronet/source/browse/trunk/%20pyronet/jawnae/pyronet/events/PyroServerListener.java'>PyroServerListener</a>

<a href='http://code.google.com/p/pyronet/source/browse/trunk/%20pyronet/jawnae/pyronet/PyroClient.java'>PyroClient</a><br>
-> <a href='http://code.google.com/p/pyronet/source/browse/trunk/%20pyronet/jawnae/pyronet/events/PyroClientListener.java'>PyroClientListener</a>




<h2>Cautionary note</h2>
Please keep in mind that the nature of non-blocking code is usually far more complex than that of blocking code. The API is event-driven, which means we're dealing with callbacks (or listeners, if you will) a lot.<br>
<br>
<br>
<h2>How to setup a TCP server</h2>
<pre><code>// create a selector
PyroSelector selector = new PyroSelector();

// listen on a port
PyroServer server1 = selector.listen(PORT1);
PyroServer server2 = selector.listen(new InetSocketAddress(HOST, PORT2));

PyroServerListener serverListener = new PyroServerListener()
{
   @Override
   public void acceptedClient(PyroClient client)
   {
      System.out.println("accepted-client: " + client);
   }
};

server1.addListener(serverListener);
server2.addListener(serverListener);

while (true)
{
   selector.select();
}
</code></pre>

<h2>Connecting to an HTTP server</h2>
<pre><code>public class RawHttpClient
{
   public static final String HOST = "www.google.com";
   public static final int    PORT = 80;

   public static void main(String[] args) throws IOException
   {
      PyroSelector selector = new PyroSelector();

      InetSocketAddress bind = new InetSocketAddress(HOST, PORT);
      System.out.println("connecting...");
      PyroClient client = selector.connect(bind);

      PyroClientListener listener = new PyroClientAdapter()
      {
         @Override
         public void connectedClient(PyroClient client)
         {
            System.out.println("&lt;connected&gt;");

            // create HTTP request
            StringBuilder request = new StringBuilder();
            request.append("GET / HTTP/1.1\r\n");
            request.append("Host: " + HOST + "\r\n");
            request.append("Connection: close\r\n");
            request.append("\r\n");

            byte[] data = request.toString().getBytes();
            client.write(ByteBuffer.wrap(data));
         }

         @Override
         public void receivedData(PyroClient client, ByteBuffer data)
         {
            while (data.hasRemaining())
               System.out.print((char) data.get());
            System.out.flush();
         }

         @Override
         public void disconnectedClient(PyroClient client)
         {
            System.out.println("&lt;disconnected&gt;");
         }
      };

      client.addListener(listener);

      while (true)
      {
         selector.select();
      }
   }
}
</code></pre>

<h2>Examples</h2>
The examples are provided in the <a href='http://code.google.com/p/pyronet/downloads/list'>JARs</a> and can be used as reference or unit test, and are available in the<br>
<a href='http://code.google.com/p/pyronet/source/browse/#svn/trunk/%20pyronet/test/jawnae/pyronet'>SVN Repository</a>

<h2>Example: Echo Server</h2>
<pre><code>public class RawEchoServer
{
   public static final String HOST = "127.0.0.1";
   public static final int    PORT = 8421;

   public static void main(String[] args) throws IOException
   {
      PyroSelector selector = new PyroSelector();
      PyroServer server = selector.listen(new InetSocketAddress(HOST, PORT));
      System.out.println("listening: " + server);

      server.addListener(new PyroServerListener()
      {
         @Override
         public void acceptedClient(PyroClient client)
         {
            System.out.println("accepted-client: " + client);

            echoBytesForTwoSeconds(client);
         }
      });

      while (true)
      {
         selector.select(100);
      }
   }

   static void echoBytesForTwoSeconds(PyroClient client)
   {
      try
      {
         client.setTimeout(2 * 1000);
      }
      catch (IOException exc)
      {
         exc.printStackTrace();
         return;
      }

      client.addListener(new PyroClientAdapter()
      {
         @Override
         public void receivedData(PyroClient client, ByteBuffer buffer)
         {
            ByteBuffer echo = buffer.slice();

            // convert data to text
            byte[] data = new byte[buffer.remaining()];
            buffer.get(data);
            String text = new String(data);

            // dump to console
            System.out.println("received \"" + text + "\" from " + client);

            client.write(echo);
         }

         @Override
         public void disconnectedClient(PyroClient client)
         {
            System.out.println("disconnected");
         }
      });
   }
}
</code></pre>

<h2>Example: Echo Client</h2>
<pre><code>public class RawEchoClient extends PyroClientAdapter
{
   public static final String HOST = "127.0.0.1";
   public static final int    PORT = 8421;

   public static void main(String[] args) throws IOException
   {
      RawEchoClient handler = new RawEchoClient();
      PyroSelector selector = new PyroSelector();

      InetSocketAddress bind = new InetSocketAddress(HOST, PORT);
      System.out.println("connecting...");
      PyroClient client = selector.connect(bind);
      client.addListener(handler);

      while (true)
      {
         // perform network I/O

         selector.select();
      }
   }

   @Override
   public void connectedClient(final PyroClient client)
   {
      System.out.println("connected: " + client);

      final String message = "hello there!";

      System.out.println("client: yelling \"" + message + "\" to the server");

      // send "hello there!"
      client.write(ByteBuffer.wrap(message.getBytes()));

      client.addListener(new PyroClientAdapter()
      {
         @Override
         public void receivedData(PyroClient client, ByteBuffer buffer)
         {
            // convert data to text
            byte[] data = new byte[buffer.remaining()];
            buffer.get(data);
            String text = new String(data);

            System.out.println("server echoed: \"" + text + "\"");
         }

         @Override
         public void droppedClient(PyroClient client, IOException cause)
         {
            System.out.println("lost connection");
         }

         @Override
         public void disconnectedClient(PyroClient client)
         {
            System.out.println("disconnected");
         }
      });
   }
}
</code></pre>