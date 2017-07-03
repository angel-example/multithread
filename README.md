# multithread
How to run Angel in multiple isolates.

The concept is pretty simple. A normal server would look like this:

```dart
var app = new Angel();
```

If you use the `Angel.custom` constructor, you can provide a custom `ServerGenerator`, which is
a typedef for a function that binds an HTTP server:

```dart
typedef Future<HttpServer> ServerGenerator(InternetAddress address, int port);
```

With this in mind, you can start a server passing the `shared` argument to `HttpServer.bind`:
```dart
new Angel.custom((address, port) => HttpServer.bind(address, port, shared: true));
```

`startShared` is a function that accomplishes this for you, since it's commonly-used functionality:
```dart
new Angel.custom(startShared);
```

## Multiple Isolates
To run in multiple isolates, the concept is simple as well:

```dart
import 'dart:convert';
import 'dart:io';
import 'dart:isolate';
import 'package:angel_common/angel_common.dart';

main(List<String> args) async {
  int concurrency = Platform.numberOfProcessors;

  // Start child isolates...
  for (int i = 1; i < concurrency; i++) {
    Isolate.spawn(serverMain, i);
  }

  // Spawn a server in the main isolate.
  serverMain(concurrency);
}

void serverMain(int id) {
  // Start shared!!!
  var app = new Angel.custom(startShared);
  
  app.get('/json', () => {'hello': 'world'});

  app.get('/db', () async {
    // Run a query...
    var connection = new PostgreSQLConnection(
        '127.0.0.1', 5432, 'wrk_benchmark',
        username: 'postgres', password: 'password');
    await connection.open();
    var rows = await connection.query('SELECT id, text from notes;');
    return rows.map((row) => {'id': row[0], 'text': row[1]});
  });

  app.startServer(InternetAddress.ANY_IP_V4, 3000).then((server) {
    print(
        'Instance #$id listening at http://${server.address.address}:${server.port}');
  });
}

```

