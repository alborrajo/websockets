# Websockets in c++

This project is the implementation of WebSockets in c++ using lib WebSockets - [lws](https://github.com/warmcat/libwebsockets.git). I wrote [post](https://yairgadelov.me/websockets-with-c-/) about it in my blog.

## Download & install libwebsockets

The project uses CMake as a build system.

Clone this repository and then fetch the git submodule dependencies:

```bash
git submodule update --init --recursive
```

And use this to build it:

```bash
cd websockets &&  mkdir Debug && cd Debug && cmake -DCMAKE_BUILD_TYPE=Debug .. && make
```

## Add as a CMake dependency

In your own project that uses CMake as a build system, clone this repository and add the following in CMakeLists.txt

```cmake
target_link_libraries(${TARGET} PRIVATE websockets)
target_link_libraries(${TARGET} PRIVATE WebSockets)
add_subdirectory(websockets EXCLUDE_FROM_ALL)
```

## Usage example

```cpp
#include "WebSockets.h"
#include <iostream>
#include <thread>

int main()
{
	WebSockets ws;

	// Called for every message received, with the protocol it came from.
	// Fragments are buffered, so this always gets a whole message.
	// Returning false closes that connection.
	auto on_message = [](std::string msg, std::string protocol_name) -> bool {
		std::cout << protocol_name << ": " << msg << std::endl;
		return true;
	};

	// Optional, fired both when a connection fails to come up and when an
	// established one closes. There is no automatic reconnection.
	auto on_disconnect = [](std::string protocol_name) -> void {
		std::cerr << protocol_name << " went down" << std::endl;
	};

	// Called once every protocol is connected.
	auto on_connect = [&ws]() -> void {
		// Write() defaults to LWS_WRITE_BINARY but takes any lws_write_protocol,
		// e.g. LWS_WRITE_TEXT, LWS_WRITE_PING, or LWS_WRITE_CONTINUATION with
		// LWS_WRITE_NO_FIN for fragmented sends.
		ws["bitstamp"].Write(
			"{\"event\": \"bts:subscribe\","
			" \"data\": {\"channel\": \"diff_order_book_btcusd\"}}",
			LWS_WRITE_TEXT);
	};

	// name, host, path, port, callbacks. TLS is on with full certificate
	// validation, both callbacks are optional.
	ws.AddProtocol("bitstamp", "ws.bitstamp.net", "/", 443, on_message, on_disconnect);

	// A second overload takes an ssl_connection argument after the port, passed
	// straight to lws_client_connect_info, so any LCCSCF_* combination works.
	// 0 gives a plain ws:// connection.
	ws.AddProtocol("dev", "192.168.1.10", "/ws", 443,
		       LCCSCF_USE_SSL | LCCSCF_ALLOW_SELFSIGNED, on_message);

	// Creates the lws context and blocks until every protocol is connected.
	ws.Connect(on_connect);

	// Run() is a blocking service loop, so give it a thread if the caller has to
	// stay responsive. RunStep() does a single pass instead, for when you drive
	// an event loop yourself.
	std::thread t([&ws] { ws.Run(); });

	// ws.Connected() is true once every protocol is up, ws["name"].Connected()
	// reports one of them, and ws["name"].last_update_time is refreshed on every
	// received message, which helps spot a stalled feed.

	ws.Stop();
	t.join();
}
```

Watch out for a few rough edges: `Connect()` spins forever if a host never comes up,
`Write()` copies into a fixed 8092 byte buffer without checking, protocol names must stay
under 16 characters, and mbedTLS builds need `info.client_ssl_ca_filepath` set in
`CreateContext()`. The whole wrapper is one header plus one source file, so read them
when in doubt.
