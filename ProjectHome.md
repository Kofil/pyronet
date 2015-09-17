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
<pre><code>// create a selector<br>
PyroSelector selector = new PyroSelector();<br>
<br>
// listen on a port<br>
PyroServer server1 = selector.listen(PORT1);<br>
PyroServer server2 = selector.listen(new InetSocketAddress(HOST, PORT2));<br>
<br>
PyroServerListener serverListener = new PyroServerListener()<br>
{<br>
   @Override<br>
   public void acceptedClient(PyroClient client)<br>
   {<br>
      System.out.println("accepted-client: " + client);<br>
   }<br>
};<br>
<br>
server1.addListener(serverListener);<br>
server2.addListener(serverListener);<br>
<br>
while (true)<br>
{<br>
   selector.select();<br>
}<br>
</code></pre>

<h2>Connecting to an HTTP server</h2>
<pre><code>public class RawHttpClient<br>
{<br>
   public static final String HOST = "www.google.com";<br>
   public static final int    PORT = 80;<br>
<br>
   public static void main(String[] args) throws IOException<br>
   {<br>
      PyroSelector selector = new PyroSelector();<br>
<br>
      InetSocketAddress bind = new InetSocketAddress(HOST, PORT);<br>
      System.out.println("connecting...");<br>
      PyroClient client = selector.connect(bind);<br>
<br>
      PyroClientListener listener = new PyroClientAdapter()<br>
      {<br>
         @Override<br>
         public void connectedClient(PyroClient client)<br>
         {<br>
            System.out.println("&lt;connected&gt;");<br>
<br>
            // create HTTP request<br>
            StringBuilder request = new StringBuilder();<br>
            request.append("GET / HTTP/1.1\r\n");<br>
            request.append("Host: " + HOST + "\r\n");<br>
            request.append("Connection: close\r\n");<br>
            request.append("\r\n");<br>
<br>
            byte[] data = request.toString().getBytes();<br>
            client.write(ByteBuffer.wrap(data));<br>
         }<br>
<br>
         @Override<br>
         public void receivedData(PyroClient client, ByteBuffer data)<br>
         {<br>
            while (data.hasRemaining())<br>
               System.out.print((char) data.get());<br>
            System.out.flush();<br>
         }<br>
<br>
         @Override<br>
         public void disconnectedClient(PyroClient client)<br>
         {<br>
            System.out.println("&lt;disconnected&gt;");<br>
         }<br>
      };<br>
<br>
      client.addListener(listener);<br>
<br>
      while (true)<br>
      {<br>
         selector.select();<br>
      }<br>
   }<br>
}<br>
</code></pre>

<h2>Examples</h2>
The examples are provided in the <a href='http://code.google.com/p/pyronet/downloads/list'>JARs</a> and can be used as reference or unit test, and are available in the<br>
<a href='http://code.google.com/p/pyronet/source/browse/#svn/trunk/%20pyronet/test/jawnae/pyronet'>SVN Repository</a>

<h2>Example: Echo Server</h2>
<pre><code>public class RawEchoServer<br>
{<br>
   public static final String HOST = "127.0.0.1";<br>
   public static final int    PORT = 8421;<br>
<br>
   public static void main(String[] args) throws IOException<br>
   {<br>
      PyroSelector selector = new PyroSelector();<br>
      PyroServer server = selector.listen(new InetSocketAddress(HOST, PORT));<br>
      System.out.println("listening: " + server);<br>
<br>
      server.addListener(new PyroServerListener()<br>
      {<br>
         @Override<br>
         public void acceptedClient(PyroClient client)<br>
         {<br>
            System.out.println("accepted-client: " + client);<br>
<br>
            echoBytesForTwoSeconds(client);<br>
         }<br>
      });<br>
<br>
      while (true)<br>
      {<br>
         selector.select(100);<br>
      }<br>
   }<br>
<br>
   static void echoBytesForTwoSeconds(PyroClient client)<br>
   {<br>
      try<br>
      {<br>
         client.setTimeout(2 * 1000);<br>
      }<br>
      catch (IOException exc)<br>
      {<br>
         exc.printStackTrace();<br>
         return;<br>
      }<br>
<br>
      client.addListener(new PyroClientAdapter()<br>
      {<br>
         @Override<br>
         public void receivedData(PyroClient client, ByteBuffer buffer)<br>
         {<br>
            ByteBuffer echo = buffer.slice();<br>
<br>
            // convert data to text<br>
            byte[] data = new byte[buffer.remaining()];<br>
            buffer.get(data);<br>
            String text = new String(data);<br>
<br>
            // dump to console<br>
            System.out.println("received \"" + text + "\" from " + client);<br>
<br>
            client.write(echo);<br>
         }<br>
<br>
         @Override<br>
         public void disconnectedClient(PyroClient client)<br>
         {<br>
            System.out.println("disconnected");<br>
         }<br>
      });<br>
   }<br>
}<br>
</code></pre>

<h2>Example: Echo Client</h2>
<pre><code>public class RawEchoClient extends PyroClientAdapter<br>
{<br>
   public static final String HOST = "127.0.0.1";<br>
   public static final int    PORT = 8421;<br>
<br>
   public static void main(String[] args) throws IOException<br>
   {<br>
      RawEchoClient handler = new RawEchoClient();<br>
      PyroSelector selector = new PyroSelector();<br>
<br>
      InetSocketAddress bind = new InetSocketAddress(HOST, PORT);<br>
      System.out.println("connecting...");<br>
      PyroClient client = selector.connect(bind);<br>
      client.addListener(handler);<br>
<br>
      while (true)<br>
      {<br>
         // perform network I/O<br>
<br>
         selector.select();<br>
      }<br>
   }<br>
<br>
   @Override<br>
   public void connectedClient(final PyroClient client)<br>
   {<br>
      System.out.println("connected: " + client);<br>
<br>
      final String message = "hello there!";<br>
<br>
      System.out.println("client: yelling \"" + message + "\" to the server");<br>
<br>
      // send "hello there!"<br>
      client.write(ByteBuffer.wrap(message.getBytes()));<br>
<br>
      client.addListener(new PyroClientAdapter()<br>
      {<br>
         @Override<br>
         public void receivedData(PyroClient client, ByteBuffer buffer)<br>
         {<br>
            // convert data to text<br>
            byte[] data = new byte[buffer.remaining()];<br>
            buffer.get(data);<br>
            String text = new String(data);<br>
<br>
            System.out.println("server echoed: \"" + text + "\"");<br>
         }<br>
<br>
         @Override<br>
         public void droppedClient(PyroClient client, IOException cause)<br>
         {<br>
            System.out.println("lost connection");<br>
         }<br>
<br>
         @Override<br>
         public void disconnectedClient(PyroClient client)<br>
         {<br>
            System.out.println("disconnected");<br>
         }<br>
      });<br>
   }<br>
}<br>
</code></pre>