# StackdriverLogging
A [SwiftLog](https://github.com/apple/swift-log)  `LogHandler` that logs GCP Stackdriver formatted JSON.

For more information on Stackdriver structured logging, see: https://cloud.google.com/logging/docs/structured-logging and [LogEntry](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry)

## Dependencies 
This Stackdriver `LogHandler` depends on [SwiftNIO](https://github.com/apple/swift-nio) which is used to create and save your new log entries in a non-blocking fashion. 

## How to install

### Swift Package Manager

```swift
.package(url: "https://github.com/Brainfinance/StackdriverLogging.git", from: "4.0.0"),
```
In your target's dependencies add `"StackdriverLogging"` e.g. like this:
```swift
.target(name: "App", dependencies: ["StackdriverLogging"]),
```

### Vapor 4
Here's a bootstrapping example for a standard Vapor 4 application.
```swift
import App
import Vapor

var env = try Environment.detect()
try LoggingSystem.bootstrap(from: &env) { (logLevel) -> (String) -> LogHandler in
    return { label -> LogHandler in
        var logger = StackdriverLogHandler(destination: .stdout)
        logger.logLevel = logLevel
        return logger
    }
}
let app = Application(env)
defer { app.shutdown() }
try configure(app)
try app.run()
```

## Logging JSON values using `Logger.MetadataValue`
To log metadata values as JSON, simply log all JSON values other than `String` as a `Logger.MetadataValue.stringConvertible` and, instead of the usual conversion of your value to a `String` in the log entry, it will keep the original JSON type of your values whenever possible.

For example:
```Swift
var logger = Logger(label: "Stackdriver")
logger[metadataKey: "jsonpayload-example-object"] = [
    "json-null": .stringConvertible(NSNull()),
    "json-bool": .stringConvertible(true),
    "json-integer": .stringConvertible(1),
    "json-float": .stringConvertible(1.5),
    "json-string": .string("Example"),
    "stackdriver-timestamp": .stringConvertible(Date()),
    "json-array-of-numbers": [.stringConvertible(1), .stringConvertible(5.8)],
    "json-object": [
        "key": "value"
    ]
]
logger.info("test")
```
Will log the non pretty-printed representation of:
```json
{  
   "sourceLocation":{  
      "function":"boot(_:)",
      "file":"\/Sources\/App\/boot.swift",
      "line":25
   },
   "jsonpayload-example-object":{  
      "json-bool":true,
      "json-float":1.5,
      "json-string":"Example",
      "json-object":{  
         "key":"value"
      },
      "json-null":null,
      "json-integer":1,
      "json-array-of-numbers":[  
         1,
         5.8
      ],
      "stackdriver-timestamp":"2019-07-15T21:21:02.451Z"
   },
   "message":"test",
   "severity":"INFO"
}
```

## Logging from a managed platform
If your app is running inside a managed environment such as Google Cloud Run or a container based Compute Engine, logging to stdout should get you up and running automatically.

## Stackdriver logging agent + fluentd config 
If you prefer logging to a file, you can use a file destination `StackdriverLogHandler.Destination.file` in combination with the Stackdriver logging agent https://cloud.google.com/logging/docs/agent/installation and a matching json format
google-fluentd config (/etc/google-fluentd/config.d/example.conf) to automatically send your JSON logs to Stackdriver for you. 

Here's an example google-fluentd conf file that monitors a json based logfile and send new log entries to Stackdriver:
```
<source>
    @type tail
    # Format 'JSON' indicates the log is structured (JSON).
    format json
    # The path of the log file.
    path /var/log/example.log
    # The path of the position file that records where in the log file
    # we have processed already. This is useful when the agent
    # restarts.
    pos_file /var/lib/google-fluentd/pos/example-log.pos
    read_from_head true
    # The log tag for this log input.
    tag exampletag
</source>
```
