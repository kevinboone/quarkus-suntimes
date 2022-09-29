# quarkus-suntimes
 
Version 1.0.0
Kevin Boone, September 2022

## What is this?

`quarkus_suntimes` is a simple webservice implemented using the Camel
extension for Quarkus, that provides a REST interface to calculate sunrise
and sunset times. 

It is in a sense the Java counterpart of the C sample here:

    https://github.com/kevinboone/solunar_ws

These two webservices, the Java implementation and the one in C, aren't
exactly the same -- the C version provides a lot more detail -- but
they are both REST webservices that do similar computations in
similar ways. I wrote this version as a way to compare the efficiency
of the native code compilation of Quarkus, with a traditional native code
implementation in C. 

## REST API

To perform a calculation, invoke the webservice with a request URL like this:

    /suntimes/local/{city}/{date}

or

    /suntimes/local/{city}

to use the current date. The "local" part of these URLs reflects local time
at the selected city; conceivably other time formats could be added later. 

The Date format is "YYYY-MM-DD", e.g., "2022-08-01".

The city format is "region:city", e.g., "Europe:London". The city name is
case-sensitive. Note that a colon is used as the region separator rather than
the usual "/", because the forward slash is interpreted as a separator in
REST conventions. 

To get a (long) list of cities and their geographical coordinates:

    /suntimes/list

All results are output in JSON format.

To test the webservice you could run, for example:

    $ curl http://localhost:8080/suntimes/local/Europe:London/2022-01-28

And this should produce the following JSON output (layout may vary):

      {
      "rise":"2022-01-28T07:45:00Z",
      "set":"2022-01-28T16:42:00Z",
      "timezone":"Europe/London",
      "date":"2022-1-28"
      }

## Prerequisites

- Java JDK 11 or later
- Maven version 3.2.8 or later
- GraalVM with native compilation extension installed, to test the native-code compilation support

## Configuration

All configuration parameters are in `src/resources/application.properties`.

All the settings in `application.properties` -- including the ones I have not
explicitly set -- can be over-ridden at run time using environment variables.
For example, to set the HTTP port of the service, define the environment
variable `QUARKUS_HTTP_PORT`. 

## Building

    $ mvn clean package

This will build a "fat" JAR containing the application and all its
dependencies. 

## Building the city database

The Java JVM includes a list of timezones, but it has no geographical
information. To calculate sunrise and sunset times we need accurate latitude
and longitude.

This information is in the Java source `City.java`. The content is
derived from the file `/usr/local/zoneinfo/zone.tab`, which is found on most
Linux systems. However, this information can change over time.  In addition,
there's no guarantee that the list of cities known to Linux is a perfect
match for those known to the Java JVM. We can't use `zone.tab` at runtime,
anyway, because not all platforms have it.

So `City.java` is generated from `zone.tab` and the JVM's list of known
timezones using the class `BuildDB`, which has a `main()` method, and can be
invoked from the command line. So, after

    $ mvn compile

build `City.java` like this:

    $ java -classpath target/classes/ me.kevinboone.suntimes.BuildDB
 
Then build the whole package again.

There is no need to do this routinely, and it will only work on Linux
systems. The `City.java` in the source bundle should be sufficient for most
applications.

## Running

To run the self-contained JAR: 

   java -jar target/quarkus-suntimes-1.0.0-runner.jar 

To run in development mode, use:

    mvn quarkus:dev

A useful feature of development mode, apart from enabling remote debugging,
is that allows the log level to be changed using a keypress. It also enables
dynamic reloading, so changing a source file will cause the service to be
rebuilt and restarted automatically. This is often quicker than running `mvn
package` repeatedly.

## Native compilation

If GraalVM, or an equivalent, is installed, this application can be compiled
to a native executable, and will run without a JVM. The Quarkus maintainers
now recommend Mandrel for compiling Quarkus to native code:
https://github.com/graalvm/mandrel/releases.

To build the native executable it should only be necessary to use the
`-Pnative` profile with Maven:

	GRAALVM_HOME=/path/to/graalvm mvn clean package -Pnative

This compilation process takes a long time for such a small program: 
minutes to tens of minutes. 

`quarkus-suntimes` or, rather, the Camel framework that supports it makes
extensive use of Java reflection. This is problematic for native 
compilation and, in general, the compiler needs to be told which classes
will be loaded reflectively. These classes can be annotated with
`@RegisterForReflection` (there are other ways as well).

In a simple application like this, it's not too difficult to work out
which classes will be loaded reflectively -- essentially all of them.
A way to find out is to run the application using the GraalVM 
reflection agent, like this:

    java -agentlib:native-image-agent=config-output-dir=foo \
       -jar target/quarkus-suntimes-1.0.0-runner.jar 

This needs to be done using the GraalVM JVM. And, of course, the 
run of the application needs to exercise all possible execution paths,
to capture all possible uses of reflection.

The agent writes a file in the output directory called 
`reflection-config.json`. You can inspect this and use it as a basis
for annotation the relevant classes, or just include the whole
thing in the build (see the Quarkus documentation).

## Notes

1. _Native compilation_. This radically reduces start-up time, but we cannot 
expect the same level of serviceability as a Java JVM provides. Even
a thread dump is not particularly comprehensible. 

2. _Accuracy of computation_. The algorithm for calculating sunrise and
sunset times was derived from a book which I believe was called
"Astronomical computation in BASIC". I've used the method in various
projects over the last twenty years, and it's worked well. However, this
is just a demonstration program -- please don't rely on it for 
ocean navigation or anything like that.
 

