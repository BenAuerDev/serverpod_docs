# Setup

Serverpod comes with built-in user management and authentication. It is possible to build a [custom authentication implementation](custom-overrides), but the recommended way to authenticate users is to use the `serverpod_auth` module. The module makes it easy to authenticate with email or social sign-ins and currently supports signing in with email, Google, Apple, and Firebase.

Future versions of the authentication module will include more options. If you write another authentication module, please consider [contributing](/contribute) your code.

![Sign-in with Serverpod](https://github.com/serverpod/serverpod/raw/main/misc/images/sign-in.png)

## Installing the auth module

Serverpod's auth module makes it easy to authenticate users through email or 3rd parties. The authentication module also handles basic user information, such as user names and profile pictures. Make sure to use the same version numbers as for Serverpod itself for all dependencies.


## Server setup

Add the module as a dependency to the server projects `pubspec.yaml`.

```yaml
dependencies:
  ...
  serverpod_auth_server: ^1.x.x
```

Add a nickname for the module in the `config/generator.yaml` file. This nickname will be used as the name of the module in the code.

```yaml
modules:
  serverpod_auth:
    nickname: auth
```

### Initialize the auth database

After adding the module to the server project, you need to initialize the database. First you have to create a new migration that includes the auth module tables. This is done by running the `serverpod create-migration` command line tool in the server project.

```bash
$ serverpod create-migration
```


Start your database container from the server project.

```bash
$ docker-compose up --build --detach
```

Then apply the migration by starting the server with the `apply-migration` flag.

```bash
$ dart run bin/main.dart --role maintenance --apply-migrations
```

The full migration instructions can be found in the [migration guide](../database/migrations).

## Client setup

Add the auth client in your client projects `pubspec.yaml`.

```yaml
dependencies:
  ...
  serverpod_auth_client: ^1.x.x
```

## App setup

First, add dependencies to your app's `pubspec.yaml` file for the methods of signing in that you want to support.

```yaml
dependencies:
  flutter:
    sdk: flutter
  serverpod_flutter: ^1.x.x
  auth_example_client:
    path: ../auth_example_client
  
  serverpod_auth_shared_flutter: ^1.x.x
```

Next, you need to set up a `SessionManager`, which keeps track of the user's state. It will also handle the authentication keys passed to the client from the server, upload user profile images, etc.

```dart
late SessionManager sessionManager;
late Client client;

void main() async {
  // Need to call this as we are using Flutter bindings before runApp is called.
  WidgetsFlutterBinding.ensureInitialized();

  // The android emulator does not have access to the localhost of the machine.
  // const ipAddress = '10.0.2.2'; // Android emulator ip for the host

  // On a real device replace the ipAddress with the IP address of your computer.
  const ipAddress = 'localhost';

  // Sets up a singleton client object that can be used to talk to the server from
  // anywhere in our app. The client is generated from your server code.
  // The client is set up to connect to a Serverpod running on a local server on
  // the default port. You will need to modify this to connect to staging or
  // production servers.
  client = Client(
    'http://$ipAddress:8080/',
    authenticationKeyManager: FlutterAuthenticationKeyManager(),
  )..connectivityMonitor = FlutterConnectivityMonitor();

  // The session manager keeps track of the signed-in state of the user. You
  // can query it to see if the user is currently signed in and get information
  // about the user.
  sessionManager = SessionManager(
    caller: client.modules.auth,
  );
  await sessionManager.initialize();

  runApp(MyApp());
}
```

The `SessionManager` has useful methods for viewing and monitoring the user's current state:

- The `signedInUser` will return a `UserInfo` if the user is currently signed in (or `null` if the user isn't signed in).
- Use the `addListener` method to get notified of changes to the user's signed in state.
- Sign out a user by calling the `signOut` method.

For example it can be useful to subscribe to changes in the `SessionManager` and force a rerender of your app.

```dart
@override
void initState() {
  super.initState();

  // Rebuild the page if signed in status changes.
  sessionManager.addListener(() {
    setState(() {});
  });
}
```