# HBridge

HBridge is a MQTT bridge for forwarding messages of certain topics from one broker to another one, written in `Haskell`.

**It is under development and is really simple now. Everything including structure can vary at any time.**

## Introduction

HBridge supports two types of connection:

- MQTT connection: MQTT connection is the most-used type of connection for performing inter-broker MQTT message forwarding. It supports standard MQTT publishing messages.
- Bare TCP connection: Although MQTT is (mostly) based on TCP connection, the bridge also supports bare TCP connection. For convenience, the one who connects to the bridge by TCP is also called "broker" no matter what it does. Messages from bare TCP connections can only be forwarded to other TCP ones (but it can receive messages from other MQTT brokers, which will be talked about later). It only supports a very simple type of message:

  + A simple type of message `PlainMsg` containing only a payload field in plain text and a topic. It is mostly used for testing purposes.

Both types of connection supports **multi-broker** work mode. It means that a bridge can connect to multiple brokers at the same time, and these connections can be different types. By proper configuration which will be talked later, a message from a broker can be forwarded to many other brokers.

The concept of the bridge is shown below:

![structure](./docs/images/structure.png)

HBridge also provides some extra features:

- Message processing: It means that HBridge can do some processing on the message. It currently supports a simple `SQL`-like grammar, which can be found at **Configuration** part.

- Mountpoint: It means that you can set a mountpoint for each broker the bridge connects to. The mountpoint will be combined with the original topic when a message is forwarded. This is useful to prevent messages from looping between brokers. For example, mountpoint `mp/test/` can modify topic `home/room/temp` to `mp/test/home/room/temp`.

## Build and Test

### Build
To build from source, haskell build tool `stack` is required. Type
```
$ stack build
```

### Test

There is a very simple test program in `test/Spec.hs`. To run the test with default configuration, it requires several MQTT brokers at running status. We recommend `EMQ X` broker:
```
$ docker run -d --name emqx1883 -p 1883:1883 -p 8083:8083 -p 8883:8883 -p 8084:8084 -p 18083:18083 emqx/emqx
$ docker run -d --name emqx1885 -p 1885:1883 -p 8085:8083 -p 8885:8883 -p 8086:8084 -p 18085:18083 emqx/emqx
```

Then run the test program:
```
$ stack test
```

Now the main program can be executed:
```
$ stack run -- HBridge --config etc/config.yaml +RTS -T
```

In this example, there exists two MQTT brokers `B1` and `B2` and two TCP ones `B3` and `B4` that connects to the bridge. The bridge forwards messages from `B1` and `B2` to other brokers. Also, `B3` sends messages to `B4` and vice versa.

The system running status can be found at `localhost:22333`. This is provided by `ekg` package. `+RTS -T` is not required if you do not need this function.

The status of the bridge can be obtained by

- `curl -X GET localhost:8999/funcs`
- `curl -X GET localhost:8999/metrics`

## Configuration

**CAUTION**: This part is currently at very early stage and can change at any time.

The configuration file is at `etc/` in `YAML` format. A default configuration file `etc/config.yaml` is also provided. A configuration file contains the following fields:

- `logFile`: Log file path.
- `logLevel`: Log level, it can be one of `DEBUG`, `INFO`, `WARNING` and `ERROR`. Logs whose level is lower than it will not be recorded.
- `logToStdErr`: It decides whether to log to console. It can be one of `true` and `false`.
- `crossForward`: It decides whether to forward MQTT messages to TCP brokers.
- `brokers`: A list of configurations of all brokers to connect to. It will be discussed later.
- `sqlFiles`: A list of names of message processing SQL files. It will be discussed later.

The `brokers` field in configuration is a list of configurations of all brokers. Each of it contains:

- `brokerName`: Name of the broker.
- `brokerURI`: URI of the broker to connect to. Due to some reasons, it is shown in a more complex form.
- `connectType`: Type of the connection. It can be one of `MQTTConnection` and `TCPConnection`.
- `brokerMount`: Mountpoint of the broker. It will be added to the topic of messages it forwards.
- `brokerFwds`: Topics this broker forwards. It is a list of strings (topics).
- `brokerSubs`: Topics this broker subscribes. It is a list of strings (topics).

The `sqlFiles` field in configuration is a list of name of SQL-like files. A SQL file contains three parts:
- `SELECT`
- `WHERE`
- `MODIFY`

The `WHERE` and `MODIFY` part are optional. The format of each part is as following:
- `SELECT field_1 [as alias_1], field_2 [as alias_2], ...`
- `WHERE name op const`, name can be selected field or alias; op can be one of `>`, `<`, `==`, `/=`, `=~` (`=~` means topic matching); const can be integer, double, string, char or boolean type, but it should match the name and oprator.
- `MODIFY name TO const`, name and const are same as `WHERE` part. Also, they should match correct type.

A sample SQL file is at `etc/sql.sql`, or seems like:
```
SELECT temp AS t, humidity AS h, location
WHERE temp     >  25.0,
      humidity <  85,
      topic    =~ "home/#"
MODIFY topic  TO "home/summary",
       QoS    TO "QoS2",
       retain TO True,
       temp   TO 20.0
```

Besides write configuration file manually, you can also write several lines of code in `test/Spec.hs`. Then run
```
$ stack repl
...
*Main Environment Extra Types> :l test/Spec.hs
*Main> writeConfig
```
The configuration file will be written at `etc/config.yaml`.
