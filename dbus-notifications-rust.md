## Creating a Notification Daemon for Linux Desktops

### What were going to do...

This article is going to follow the notification spec layed out by Gnome and Freedesktop (Version 1.2) [here](https://people.gnome.org/~mccann/docs/notification-spec/notification-spec-latest.html).

Our plan of action is...

- Implement freedesktop spec methods and requirements
- Receive notifications over D-Bus, print them to the console
- Show basic UI popup
- Add Icons
- Add formatting for markup
- Implement actions
- Implement hints

We're going to use the `dbus-rs` d-bus implementation at [diwic/dbus-rs](https://github.com/diwic/dbus-rs) to communicate with D-Bus. We're also going to use GTK to display the notifications using [Gtk-rs](https://gtk-rs.org/).

### How linux desktop notifications work

Most linux distrubutions use D-Bus as a type of RPC for interprocess communications.

> D-Bus is a message bus system, a simple way for applications to talk to one another. In addition to interprocess communication, D-Bus helps coordinate process lifecycle; it makes it simple and reliable to code a "single instance" application or daemon, and to launch applications and daemons on demand when their services are needed.

From [dbus.freeesktop.org](https://dbus.freeesktop.org)

Most [desktop environments](https://wiki.archlinux.org/index.php/Desktop_environment) and applications follow this spec and use d-bus as a way to send notifications instead of sending messages directly to the notifications daemon, wether directly or through a notification library ( libnotify, etc ).

This spec specifys that notifications should be sent and received over a D-Bus bus...

> The server should implement the org.freedesktop.Notifications interface on an object with the path "/org/freedesktop/Notifications". This is the only interface required by this version of the specification.

### The components of a Notification

- Application Name [optional] - Formal Application Name
- Replaces ID [optional] - ID of previous notification to replace
- Notification Icon [optional] - The notification icon
- Summary - UTF-8, < 40 characters, title of notification
- Body [optional] - UTF-8, Multiline, Allows some [markup](https://people.gnome.org/~mccann/docs/notification-spec/notification-spec-latest.html#markup)
- Actions - Tuple of ("default", "Default Action", key, action, key, action, ...)
- Hints - [Extra information](https://people.gnome.org/~mccann/docs/notification-spec/notification-spec-latest.html#hints) for us to implement if we want
- Expiration Timeout - Timeout time in milliseconds for the notification to close, -1 is default, 0 never expires

### D-Bus Notification Spec Requirements

Must implement methods:

- **GetCapabilities** -> string[] ( array of server capabilities )
- **Notify** app_name: string, replaces_id: u32, app_icon: string, summary: string, body: string, actions: tuple, hints: struct, expire_timeout: i32 -> u32 ( id of the notification )
- **CloseNotification** u32 -> void
- **GetServerInformation** -> name: string, vendor: string, version: string, spec_version: string ( server information )

Optional signals:

- **NotificationClosed** with id: u32, reason: u32 ( from options in signals definition ) _were probably going to ignore this one_
- **ActionInvoked** with id: u32, action_key: string ( from actions of notification )

### Generating some code using D-Bus Introspection XML and D-Bus Codegen

D-Bus provides a way to introspect the interface that is provided by specific services. This results in some XML being returned. This XML can also work the other way with `dbus-codegen` crate after installing.

`./dbus-notifications-one $ dbus-codegen-rust < org.freedesktop.Notifications > src/mod.rs`
