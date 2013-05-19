# Records

<note><title>A note to reviewers</title>

**IMPORTANT**: This section is going to describe how records and
variants will behave in OCaml 4.01.  If you want to follow along fully
with the examples here, you'll need to make sure you install a
bleeding-edge release, which you can do as follows:

```
opam switch 4.01.0dev+trunk
```

Once you make the switch, you'll need to install the packages you
need.  Note that switching back and forth between different compilers
and their packages is a fast operation after the initial build.

</note>

One of OCaml's best features is its concise and expressive system for
declaring new datatypes.  Two key elements of that system are
_records_ and _variants_, both of which we discussed briefly in
[xref](#a-guided-tour).  In this chapter we'll cover records in more
depth, covering more of the details of how they work, as well as
advice on how to use them effectively in your software designs.

A record represents a collection of values stored together as one,
where each component is identified by a different field name.  The
basic syntax for a record type declaration is as follows.

```ocaml
type <record-name> =
  { <field-name> : <type-name> ;
    <field-name> : <type-name> ;
    ...
  }
```

Here's a simple example, a `host_info` record that summarizes
information about a given computer.

```ocaml
# type host_info =
    { hostname   : string;
      os_name    : string;
      os_release : string;
      cpu_arch   : string;
      timestamp  : Time.t;
    };;
```

We can construct a `host_info` just as easily.  The following code
uses the `Shell` module from `Core_extended` to dispatch commands to
the shell to extract the information we need about the computer we're
running on, and the `Time.now` call from Core's `Time` module.

```ocaml
# open Core_extended.Std;;
# let my_host =
    let sh = Shell.sh_one_exn in
    { hostname   = sh "hostname";
      os_name    = sh "uname -s";
      os_release = sh "uname -r";
      cpu_arch   = sh "uname -p";
      timestamp  = Time.now ();
    };;
val my_host : host_info =
  {hostname = "yevaud.local"; os_name = "Darwin"; os_release = "12.3.0";
   cpu_arch = "i386"; timestamp = 2013-04-13 06:39:17.806527}
```

You might wonder how the compiler inferred that `my_host` is of type
`host_info`.  The hook that the compiler uses in this case to figure
out the type is the record field name.  Later in the chapter, we'll
talk about what happens when there is more than one record type in
scope with the same field name.

Once we have a record value in hand, we can extract elements from the
record field using dot-notation.

```ocaml
# my_host.cpu_arch;;
- : string = "i386"
```

## Patterns and exhaustiveness

Another way of getting information out of a record is by using a
pattern match, as in the definition of `host_info_to_string` below.

```ocaml
# let host_info_to_string { hostname = h; os_name = os;
                            os_release = r; cpu_arch = c;
                            timestamp = ts;
                          } =
       sprintf "%s (%s %s / %s, on %s)" h os r c (Time.to_sec_string ts);;
    val host_info_to_string : host_info -> string = <fun>
# host_info_to_string my_host;;
- : string = "yevaud.local (Darwin 12.3.0 / i386, on 2013-04-13 06:39:17)"
```

Note that the pattern that we used had only a single case, rather than
using several cases separated by `|`s.  We needed only one pattern
because record patterns are _irrefutable_, meaning that a record
pattern match will never fail at runtime.  This makes sense, because
the set of fields available in a record is always the same.  In
general, patterns for types with a fixed structure, like records and
tuples, are irrefutable, unlike types with variable structure like
lists and variants.

Another important characteristic of record patterns is that they don't
need to be complete; a pattern can mention only a subset of the fields
in the record.  This can be convenient, but it can also be error
prone.  In particular, this means that when new fields are added to
the record, code that should be updated to react to the presence of
those new fields will not be flagged by the compiler.

As an example, imagine that we wanted to add a new field to our
`host_info` record called `os_version`, as shown below.

```ocaml
# type host_info =
    { hostname   : string;
      os_name    : string;
      os_release : string;
      cpu_arch   : string;
      os_version : string;
      timestamp  : Time.t;
    } ;;
```

The code for `host_info_to_string` would continue to compile without
change.  In this particular case, it's pretty clear that you might
want to update `host_info_to_string` in order to include `os_version`,
and it would be nice if the type system would give you a warning about
the change.

Happily, OCaml does offer an optional warning for missing fields in a
record pattern.  With that warning turned on (which you can do in the
toplevel by typing `#warnings "+9" `), the compiler will warn about
the missing field.

```ocaml
# #warnings "+9";;
# let host_info_to_string { hostname = h; os_name = os;
                            os_release = r; cpu_arch = c;
                            timestamp = ts;
                          } =
    sprintf "%s (%s %s / %s, on %s)" h os r c (Time.to_sec_string ts);;
Characters 24-183:
val host_info_to_string : host_info -> string = <fun>
Warning 9: the following labels are not bound in this record pattern:
os_version
Either bind these labels explicitly or add '; _' to the pattern.
```

We can disable the warning for a given pattern by explicitly
acknowledging that we are ignoring extra fields.  This is done by
adding an underscore to the pattern, as shown below.

```ocaml
# let host_info_to_string { hostname = h; os_name = os;
                            os_release = r; cpu_arch = c;
                            timestamp = ts; _
                          } =
    sprintf "%s (%s %s / %s, on %s)" h os r c (Time.to_sec_string ts);;
val host_info_to_string : host_info -> string = <fun>
```

It's a good idea to enable the warning for incomplete record matches,
and to explicitly disable it with an `_` where necessary.

## Field punning

When the name of a variable coincides with the name of a record field,
OCaml provides some handy syntactic shortcuts.  For example, the
pattern in the following function binds all of the fields in question
to variables of the same name.  This is called _field punning_.

```ocaml
# let host_info_to_string { hostname; os_name; os_release; cpu_arch } =
     sprintf "%s (%s %s / %s)" hostname os_name os_release cpu_arch;;
  val host_info_to_string : host_info -> string = <fun>
```

Field punning can also be used to construct a record.  Consider the
following code for generating a `host_info` record.

```ocaml
# let my_host =
    let sh cmd = Shell.sh_one_exn cmd in
    let hostname   = sh "hostname" in
    let os_name    = sh "uname -s" in
    let os_release = sh "uname -r" in
    let cpu_arch   = sh "uname -p" in
    let os_version = sh "Uname -v" in
    let timestamp  = Time.now () in
    { hostname; os_name; os_release; cpu_arch; os_version; timestamp };;
val my_host : host_info =
  {hostname = "yevaud.local"; os_name = "Darwin"; os_release = "12.3.0";
   cpu_arch = "i386";
   os_version =
    "Darwin Kernel Version 12.3.0: Sun Jan  6 22:37:10 PST 2013; root:xnu-2050.22.13~1/RELEASE_X86_64";
   timestamp = 2013-04-13 06:49:57.771755}
```

In the above code, we defined variables corresponding to the record
fields first, and then the record declaration itself simply listed the
fields that needed to be included.

You can take advantage of both field punning and label punning when
writing a function for constructing a record from labeled arguments,
as shown below.

```ocaml
# let create_host_info ~hostname ~os_name ~os_release ~cpu_arch ~os_version =
    { os_name; os_release; cpu_arch; os_version;
      hostname = String.lowercase hostname;
      timestamp = Time.now () };;
val create_host_info :
  hostname:string ->
  os_name:string ->
  os_release:string ->
  cpu_arch:string -> os_version:string -> timestamp:Time.t -> host_info =
  <fun>
```

This is considerably more concise than what you would get without
punning at all.

```ocaml
# let create_host_info
    ~hostname:hostname ~os_name:os_name ~os_release:os_release
    ~cpu_arch:cpu_arch ~os_version:os_version =
    { os_name = os_name; os_release = os_release;
     cpu_arch = cpu_arch; os_version = os_version;
      hostname = String.lowercase hostname;
      timestamp = Time.now () };;
val create_host_info :
  hostname:string ->
  os_name:string ->
  os_release:string ->
  cpu_arch:string -> os_version:string -> timestamp:Time.t -> host_info =
  <fun>
```

Together, labeled arguments, field names, and field and label punning,
encourage a style where you propagate the same names throughout your
code-base.  This is generally good practice, since it encourages
consistent naming, which makes it easier to navigate the source.

## Reusing field names

Defining records with the same field names can be problematic.  Let's
consider a simple example: building types to represent the protocol
used for a logging server.

We'll describe three message types: `log_entry`, `heartbeat` and
`logon`.  The `log_entry` message is used to deliver a log entry to
the server; the `logon` message is sent to initiate a connection, and
includes the identity of the user connecting and credentials used for
authentication; and the `heartbeat` message is periodically sent by
the client to demonstrate to the server that the client is alive and
connected.  All of these messages include a session id and the time
the message was generated.

```ocaml
# type log_entry =
    { session_id: string;
      time: Time.t;
      important: bool;
      message: string;
    }
  type heartbeat =
    { session_id: string;
      time: Time.t;
      status_message: string;
    }
  type logon =
    { session_id: string;
      time: Time.t;
      user: string;
      credentials: string;
    }
;;
```

Reusing field names can lead to some ambiguity.  For example, if we
want to write a function to grab the session_id from a record, what
type will it have?

```ocaml
# let get_session_id t = t.session_id;;
val get_session_id : logon -> string = <fun>
```

In this case, OCaml just picks the most recent definition of that
record field.  We can force OCaml to assume we're dealing with a
different type (say, a `heartbeat`) using a type annotation.

```ocaml
# let get_heatbeat_session_id (t:heartbeat) = t.session_id;;
val get_heatbeat_session_id : heartbeat -> string = <funambulate>
```

While it's possible to resolve ambiguous field names using type
annotations, the ambiguity can be a bit confusing.  Consider the
following functions for grabbing the session id and status from a
heartbeat.

```ocaml
# let status_and_session t = (t.status_message, t.session_id);;
val status_and_session : heartbeat -> string * string = <fun>
# let session_and_status t = (t.session_id, t.status_message);;
Error: The record type logon has no field status_message
# let session_and_status (t:heartbeat) = (t.session_id, t.status_message);;
val session_and_status : heartbeat -> string * string = <fun>
```

Why did the first definition succeed without a type annotation and the
second one fail?  The difference is that in the first case, the
type-checker considered the `status_message` field first and thus
concluded that the record was a `heartbeat`.  When the order was
switched, the `session_id` field was considered first, and so that
drove the type to be considered to be a `logon`, at which point
`t.status_message` no longer made sense.

We can avoid this ambiguity altogether, either by using
non-overlapping field names or, more generally, by minting a module
for each type.  We'll talk more about modules more in
[xref](#files-modules-and-programs), but for now, you can think of a
module as a way of collecting values and types together in a named
package.

Packing types into modules is a broadly useful idiom (and one used
quite extensively by Core), providing for each type a name-space
within which to put related values.  When using this style, it is
standard practice to name the type associated with the module `t`.
Using this style we would write:

```ocaml
# module Log_entry = struct
    type t =
      { session_id: string;
        time: Time.t;
        important: bool;
        message: string;
      }
  end
  module Heartbeat = struct
    type t =
      { session_id: string;
        time: Time.t;
        status_message: string;
      }
  end
  module Logon = struct
    type t =
      { session_id: string;
        time: Time.t;
        user: string;
        credentials: string;
      }
  end;;
```

Now, our heartbeat-creation function can be rendered as follows.

```ocaml
# let create_log_entry ~session_id ~important message =
     { Log_entry.time = Time.now (); Log_entry.session_id;
       Log_entry.important; Log_entry.message }
  ;;
val create_log_entry :
  session_id:string -> important:bool -> string -> Log_entry.t = <fun>
```

The module name `Log_entry` is required to qualify the fields, because
this function is outside of the `Log_entry` module where the record
was defined.  OCaml only requires the module qualification for one
record field, however, so we can write this more concisely.

```ocaml
# let create_log_entry ~session_id ~important message =
     { Log_entry. time = Time.now (); session_id; important; message }
  ;;
val create_log_entry :
  session_id:string -> important:bool -> string -> Log_entry.t = <fun>
```

This is not restricted to constructing a record; we can use the same
trick when pattern-matching:

```ocaml
# let message_to_string { Log_entry. important; message; _ } =
    if important then String.uppercase message else message
  ;;
val message_to_string : Log_entry.t -> string = <fun>
```

When using dot-notation for accessing record fields, we can qualify
the field by the module directly:

```ocaml
# let is_important t = t.Log_entry.important;;
val is_important : Log_entry.t -> bool = <fun>
```

For functions defined within the module where a given record is
defined, the module qualification goes away entirely.

## Functional updates

Fairly often, you will find yourself wanting to create a new record
that differs from an existing record in only a subset of the fields.
For example, imagine our logging server had a record type for
representing the state of a given client, including when the last
heartbeat was received from that client.  The following defines a type
for representing this information, as well as a function for updating
the client information when a new heartbeat arrives.

```ocaml
# type client_info =
   { addr: Unix.Inet_addr.t;
     port: int;
     user: string;
     credentials: string;
     last_heartbeat_time: Time.t;
   };;
# let register_heartbeat t hb =
      { addr = t.addr;
        port = t.port;
        user = t.user;
        credentials = t.credentials;
        last_heartbeat_time = hb.Heartbeat.time;
      };;
val register_heartbeat : client_info -> Heartbeat.t -> client_info = <fun>
```

This is fairly verbose, given that there's only one field that we
actually want to change, and all the others are just being copied over
from `t`.  We can use OCaml's _functional update_ syntax to do this
more tersely.  The syntax of a functional update is as follows.

```ocaml
{ <record> with <field> = <value>;
                <field> = <value>;
                ...
}
```

The purpose of the functional update is to create a new record based
on an existing one, with a set of field changes layered on top.

Given this, we can rewrite `register_heartbeat` more concisely.

```ocaml
# let register_heartbeat t hb =
    { t with last_heartbeat_time = hb.Heartbeat.time };;
```

Functional updates make your code independent of the identity of the
fields in the record that are not changing.  This is often what you
want, but it has downsides as well.  In particular, if you change the
definition of your record to have more fields, the type system will
not prompt you to reconsider whether your update code should affect
those fields.  Consider what happens if we decided to add a field for
the status message received on the last heartbeat.

```ocaml
# type client_info =
   { addr: Unix.Inet_addr.t;
     port: int;
     user: string;
     credentials: string;
     last_heartbeat_time: Time.t;
     last_heartbeat_status: string;
   };;
```

The original implementation of `register_heartbeat` would now be
invalid, and thus the compiler would warn us to think about how to
handle this new field.  But the version using a functional update
continues to compile as is, even though it incorrectly ignores the new
field.  The correct thing to do would be to update the code as
follows.

```ocaml
# let register_heartbeat t hb =
    { t with last_heartbeat_time   = hb.Heartbeat.time;
             last_heartbeat_status = hb.Heartbeat.status_message;
    };;
val register_heartbeat : client_info -> Heartbeat.t -> client_info = <fun>
```

## Mutable fields

Like most OCaml values, records are immutable by default.  You can,
however, declare individual record fields as mutable.  For example, we
could take the `client_info` type and make the fields that may need to
change over time mutable, as follows.

```ocaml
# type client_info =
   { addr: Unix.Inet_addr.t;
     port: int;
     user: string;
     credentials: string;
     mutable last_heartbeat_time: Time.t;
     mutable last_heartbeat_status: string;
   };;
```

We then use the `<-` operator for actually changing the state.  The
side-effecting version of `register_heartbeat` would be written as
follows.

```ocaml
# let register_heartbeat t hb =
    t.last_heartbeat_time   <- hb.Heartbeat.time;
    t.last_heartbeat_status <- hb.Heartbeat.status_message
  ;;
val register_heartbeat : client_info -> Heartbeat.t -> unit = <fun>
```

Note that `<-` is not needed for initialization, because all fields of
a record, including mutable ones, must be specified when the record is
created.

OCaml's policy of immutable-by-default is a good one, but imperative
programming does have its place.  We'll discuss more about how (and
when) to use OCaml's imperative features in
[xref](#imperative-programming).

## First-class fields

Consider the following function for extracting the usernames from a
list of `Logon` messages.

```ocaml
# let get_users logons =
     List.dedup (List.map logons ~f:(fun x -> x.Logon.user));;
val get_users : Logon.t list -> string list = <fun>
```

Here, we wrote a small function `(fun x -> x.Logon.user)` to access
the `user` field.  This kind of accessor function is a common enough
pattern that that it would be convenient to generate them
automatically.  The `fieldslib` syntax extension that ships with
`Core` does just that.

`fieldslib` is invoked by putting the `with fields` annotation at the
end of the declaration of a record type.  So, for example, we could
have defined `Logon` as follows.

```ocaml
# module Logon = struct
    type t =
      { session_id: string;
        time: Time.t;
        user: string;
        credentials: string;
      }
    with fields
  end;;
```

Given that definition, we can use the function `Logon.user` to extract
the user field from a logon message.

```ocaml
# let get_users logons = List.dedup (List.map logons ~f:Logon.user);;
val get_users : Logon.t list -> string list = <fun>
```

In addition to generating field accessor functions, `fieldslib` also
creates a sub-module called `Fields` that contains a first class
representative of each field, in the form of a value of type
`Field.t`.  The `Field` module provides the following functions:

* `Field.name`, which returns the name of a field
* `Field.get`, which returns the content of a field
* `Field.fset`, which does a functional update of a field
* `Field.setter`, which returns `None` if the field is not mutable or
  `Some f` if it is, where `f` is a function for mutating that field.

A `Field.t` has two type parameters: the first for the type of the
record, and the second for the type of the field in question.  Thus,
the type of `Logon.Fields.session_id` is `(Logon.t, string) Field.t`,
whereas the type of `Logon.Fields.time` is `(Logon.t, Time.t)
Field.t`.  Thus,  if you call `Field.get` on `Logon.Fields.user`,
you'll get a function for extracting the `user` field from a `Logon.t`.

```ocaml
# Field.get Logon.Fields.user;;
- : Logon.t -> string = <fun>
```

Thus, first parameter of the `Field.t` corresponds to the record you
pass to `get`, and the second argument corresponds to the value
contained in the field, which is also the return type of `get`.

The type of `Field.get` is a little more complicated than you might
naively expect from the above, as you can see below.

```ocaml
# Field.get;;
- : ('b, 'r, 'a) Field.t_with_perm -> 'r -> 'a = <fun>
```

The extra type parameter `'b` is there to keep track of permissions on
the `Field.t`.  As we'll see later when we cover

The type is `Field.t_with_perm` rather than a simple `Field.t` because
fields have a notion of access control associated with them because
there are some special cases where we may expose a field but not
expose the ability to read a field but not the ability to do a
functional update.

We can use first class fields to do things like write a generic
function for displaying a record field.

```ocaml
# let show_field field to_string record =
    let name = Field.name field in
    let field_string = to_string (Field.get field record) in
    name ^ ": " ^ field_string
  ;;
val show_field : ('a, 'b) Field.t -> ('b -> string) -> 'a -> string = <fun>
```

This takes three arguments: the `Field.t`, a function for converting
the contents of the field in question to a string, and a record from
which the field can be grabbed..

Here's an example of `show_field` in action.

```ocaml
# let logon = { Logon.
                session_id = "26685";
                time = Time.now ();
                user = "yminsky";
                credentials = "Xy2d9W"; }
  ;;
# show_field Logon.Fields.user Fn.id logon;;
- : string = "user: yminsky"
# show_field Logon.Fields.time Time.to_string logon;;
- : string = "time: 2012-06-26 18:44:13.807826"
```

As a side note, the above is our first use of the `Fn` module (short
for "function") which provides a collection of useful primitives for
dealing with functions.  `Fn.id` is the identity function.

`fieldslib` also provides higher-level operators, like `Fields.fold`
and `Fields.iter`, which let you iterate over all the fields of a
record.  The following function uses `Logon.Fields.iter` and
`show_field` to print out all the fields of a `Logon` record.

```ocaml
# let print_logon logon =
    let print to_string field =
      printf "%s\n" (show_field field to_string logon)
    in
    Logon.Fields.iter
      ~session_id:(print Fn.id)
      ~time:(print Time.to_string)
      ~user:(print Fn.id)
      ~credentials:(print Fn.id)
  ;;
val print_logon : Logon.t -> unit = <fun>
# print_logon logon;;
session_id: 26685
time: 2012-06-26 18:44:13.807826
user: yminsky
credentials: Xy2d9W
- : unit = ()
```

One nice side effect of this approach is that it helps you refactor
your code when changing the fields of a record.  In particular, if you
add a field to `Logon`, the type of `iter` will change with it,
acquiring a new argument.  Any code using `iter` won't be able to
compile until it's fixed to take this new argument into account, and
these compilation failiures will point out places you need to adapt
your code.

Field iterators are useful for a variety of record-related tasks, from
building record validation functions to scaffolding the definition of
a web-form from a record type, all with a guarantee that you've
considered all fields of the record type in question.
