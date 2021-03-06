# gelfd
D module for the GELF protocol.

## What is GELF?
GELF stands for the [Graylog Extended Logging Format](https://www.graylog.org/resources/gelf/).

It is an open standard logging format based on JSON. It is primarily used to pipe messages to [Graylog](www.graylog.org/overview/), the open source log management and analysis platform. 

Using GELF avoids the shortcomings of logging to plain syslog, most importantly the lack of a structured payload along with messages (stack traces, timeouts etc).

GELF is a pure JSON format. It describes how log messages should be structured. In addition, it also describes compression of messages and chunking of large messages over UDP.

This module aims to provide a simple, yet structured, way of generating log messages in GELF format. You can combine messages with arbitrary payload data and construct messages in multiple parts. Message contents are queryable as well. See examples below.

## Installation

### Using DUB

See documentation on the [project dub page](http://code.dlang.org/packages/gelfd)

### Using DMD/LDC/GDC

- Clone this repo
- Compile your source with the gelf sources. `dmd MYFILE.d gelf/*`

## Unittests

Run unittests like so :

````
rdmd -main -unittest gelf/protocol.d
````

## Usage

This is the simplest way to create a GELF message.
````
import gelf;

// A simple way of creating a GELF message
writeln(Message("localhost", "The error message"));
````

GELF messages are composed in a `Message` struct. The struct supports :
- `opString` - writing to a string generates a JSON string.
- `opDispatch` - payload data can be added as functions or properties. It can also be read as properties.
- `opIndexAssign` - payload data can be assigned like an associative array.

````
import std.stdio;
import std.datetime;
import std.socket;

import gelf;

void main() {
	
	writeln(Message("localhost","HUGE ERROR!")); //This creates a bare minimum GELF message
	writeln(Message("localhost","HUGE ERROR!", Level.ERROR)); //This example uses the overloaded contructor to report an error
	
	// Let's create a GELF message using properties
	auto m = Message("localhost","HUGE ERROR!");
	m.level = Level.ERROR;
	m.timestamp = Clock.currTime();
	m.a_number = 7;
	
	// Now let's add some environment variables in
	import std.process;
	foreach(v, k; environment.toAA())
		m[k] = v;
	
	writeln(m); // {"version":1.1, "host:"localhost", "short_message":"HUGE ERROR!", "timestamp":1447275799, "level":3, "_a_number":7, "_PATH":"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games", ...}
	
	// OR, use the fluent interface ..
	auto m1 = Message("localhost", "Divide by zero error").level(Level.ERROR).timestamp(Clock.currTime()).numerator(1000).PATH("/usr/bin/");
	
	// Values can be checked for conditions. Here we only send messages of Level.ERROR or more severity to Graylog 
	if(m1.level <= Level.ERROR) {
		auto s = new UdpSocket();
		s.connect(new InternetAddress("localhost", 11200));
		s.send(m1.toString());
	}
	
	writeln(m1); //{"version":1.1, "host:"localhost", "short_message":"Divide by zero error", "timestamp":1447274923, "level":3, "_numerator":1000, "_PATH":"/usr/bin/"}

}
````

## Chunking and Compression

Chunking and compression are supported automatically using the `sendChunked` function.

````
import gelf;
import std.socket;

auto s = new UdpSocket();
s.connect(new InternetAddress("localhost", 12200));

// Start netcat to watch the output : `nc -lu 12200`

s.sendChunked(m, 500); // Chunk if message is larger than 500 bytes
s.sendChunked(m, 500, true); // Same as above, but compresses the message (zlib) before chunking
````

## Licence
MIT License

Adil Baig