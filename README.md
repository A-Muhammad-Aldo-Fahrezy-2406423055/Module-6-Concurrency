# Reflection 1

In this milestone, I implemented a basic single-threaded web server in Rust that listens for incoming TCP connections on 127.0.0.1:7878. The TcpListener::bind() method binds the server to that address and port, and the incoming() method returns an iterator over connection attempts, allowing the server to handle each request sequentially in a loop.

I added a helper functin handle_connection that receives a TcpStream which represents the raw bidirectional communication channel between the server and the browser. Inside the function, a BufReader is wrapped around the stream to enable efficient, line-by-line reading of the incoming data, since raw TCP streams do not have built-in buffering.

The HTTP request is parsed by reading lines from the BufReader using .lines(), mapping each result to unwrap it, and collecting them until an empty line is encountered where this empty line signals the end of the HTTP request headers according to the HTTP/1.1 protocol. The result is stored as a Vec<String> which is then printed to the console for inspection.

From the output, I can see that a browser request consists of a request line (e.g., GET / HTTP/1.1), followed by several header fields such as Host, User-Agent, Accept, and Connection. These headers give the server information about who is making the request and what kind of response the client can handle.

One interesting observation is that the browser often sends multiple connections in quick succession. Upon investigation, I discovered that this behaviour is pretty much expected because modern browsers are designed to proactively retry or make additional requests (e.g., for a favicon), which is why multiple "Connection established!" messages appear even from a single browser tab visit. This highlights that even a simple web server must be prepared to handle more than one request per user interaction.

## Breaking Down the Request Output

From the actual output I received, here is a breakdown of it according to my research:

- **`GET / HTTP/1.1`** is the request line. It tells the server that the browser wants to `GET` the root path `/` using HTTP version 1.1.
- **`Host: 127.0.0.1:7878`** specifies the target host and port the browser is connecting to.
- **`Connection: keep-alive`** means that the browser requests the connection to stay open for potential follow-up requests, rather than closing it immediately after one response.
- **`sec-ch-ua`** is a Client Hints header introduced by Chromium-based browsers that tells the server which browser brand and version is being used. Here it indicates Chromium 147.
- **`sec-ch-ua-mobile: ?0`** indicates that the request is NOT coming from a mobile device (`?0` means false).
- **`sec-ch-ua-platform: "macOS"`** tells the server the operating system of the client, in this case macOS.
- **`Upgrade-Insecure-Requests: 1`** means the browser prefers to receive a secure (HTTPS) version of the page if available.
- **`User-Agent`** is a detailed string describing the browser and OS. Here it shows Chrome 147 running on macOS, built on the WebKit/537.36 engine.
- **`Accept`** lists the content types the browser can handle, in order of preference. It prefers HTML, then XHTML, XML, and also accepts image formats like `avif` and `webp`.
- **`Sec-Fetch-Site: cross-site`** indicates that this request is coming from a different origin than the target, likely because the browser was navigating from a different tab or address.
- **`Sec-Fetch-Mode: navigate`** tells the server this is a top-level navigation request (e.g., typing a URL in the address bar).
- **`Sec-Fetch-Dest: document`** means the browser expects to receive an HTML document as the response.
- **`Accept-Encoding: gzip, deflate, br, zstd`** lists the compression formats the browser supports, so the server can compress the response to save bandwidth.
- **`Accept-Language: en-GB,en-US;q=0.9,en;q=0.8`** means the browser prefers British English, then American English, then generic English as fallback.
- **`Cookie`** contains a `csrftoken`, which is a Cross-Site Request Forgery protection token. This was likely stored from a previous session on another local application (e.g., a Django app), and the browser automatically attached it since the host is `127.0.0.1`.

Overall, this output shows just how much metadata a browser sends with even the simplest request. All of this information is available to the server to decide how to respond, though for now our server only prints it and does nothing else with it.

## "Didn't Send Any Data" vs "Refused to Connect"

When visiting `127.0.0.1:7878` with the server running, the browser shows an error along the lines of **"the server didn't send any data"**. This is different from the **"refused to connect"** error that appears when the server is not running at all, and the distinction is important.

- **"Refused to connect"** means the TCP connection itself was rejected. In other words, no process is listening on that port, so the operating system immediately sends back a TCP RST (reset) packet. The browser never even gets to exchange data with anything.
- **"Didn't send any data"** means the TCP connection was successfully established. This indicates that our Rust server accepted it and even printed "Connection established!", but then sent nothing back. The browser completed the handshake, waited for an HTTP response, and received only silence before the connection closed. Since we haven't written any response logic yet in this milestone, our server simply drops the stream after printing the request, causing the browser to complain that it got no data.

This distinction confirms that our server is working correctly at the TCP level. The connection is being accepted and the request is being read. What is missing so far is the part where the server writes an HTTP response back to the stream, which will be addressed in the next milestone.