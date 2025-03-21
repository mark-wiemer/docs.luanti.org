---
title: Persistence
aliases:
  - /api/persistence
---

# Persistence

This page aims to give an overview of the serialization options Luanti provides.

Your choice for persistence depends on the data you want to persist:

- Structure & types of the data: How is the data structured? What data types occur?
- Size of the data, frequency of changes to the data: Can it fit into memory? How expensive can updates be?
- Granularity needed: How often must the data be persisted?
- Required query-ability of the data: Are indices for fast lookups required?
- Required data access, objects the data is tied to: Items? Players?

## (De-)Serialization

### JSON

Luanti provides a JSON serializer based on `jsoncpp`.

The following types are supported by JSON:

- booleans
- numbers
- strings
- tables (objects/arrays)

{{< notice note >}}
Negative and positive infinity are represented by numbers which exceed the double number range by a lot (plus/minus `1e9999`).
{{< /notice >}}

{{< notice warning >}}
The empty table `{}` (ambiguous: might be either `{}` or `[]`) and `nan` aren't supported by JSON and will be turned into `null`.
{{< /notice >}}

Tables may either:

1. Contain only positive integer keys (represented as array) or
2. Contain only string keys (represented as object/dictionary)
   obviously, plenty of Lua tables aren't representable this way (boolean keys, mixed hash/list keys, fractional keys, inf keys)

#### `core.write_json(data, [styled])`

If the data is serializable as outlined above:: Returns a JSON string representing `data`. Formats the JSON nicely to improve readability if `styled` is truthy.
Else:: Returns `nil` and one of the following errors if it fails:

- `Can't use indexes with fractional part in JSON`
- `Lua key to convert to JSON is not a string or number`
- `Can't mix array and object values in JSON`

{{< notice warning >}}
Hash tables containing only positive integer keys - a table of `core.hash_node_position` hashes for instance -
will be turned into _an array with holes_, that is, all `nil` values from `1` to the maximum index will be filled with `null`,
which means terrible performance and possibly very large output generated from very small input.
_Do not allow direct access to `core.write_json`_ as it can be trivially used to DoS your server.
{{< /notice >}}

{{< notice tip >}}
Sparse hash maps, should always be converted to objects, using only string keys, for JSON. You might do this just for serialization.
{{< /notice >}}

{{< notice warning >}}
The JSON serializer is furthermore limited by its recursion depth: Only a recursion depth up to _16_ is allowed. Very nested data structures can't be (de)serialized. Circular references will cause the JSON serializer to recurse until the maximum recursion depth is exceeded.
{{< /notice >}}

{{< notice tip >}}
Use the `json = assert(core.write_json(data))` idiom to not silently ignore errors when serializing.
{{< /notice >}}

#### `core.parse_json(json, [nullvalue])`

If `json` is valid JSON:: Returns the Lua value represented by the JSON string `json`, with `null` values deserialized to `nullvalue` (which defaults to `nil`).
Else:: Returns `nil` and logs an error.

{{< notice tip >}}
If you must use JSON (to interact with a web interface, for instance), consider using your JSON library of choice; Lua-only implementations such as [`lunajson`](https://github.com/grafi-tt/lunajson/) are available.
{{< /notice >}}

### Lua

Luanti alternatively offers a serializer implemented in Lua, which serializes to Lua source code, available as `core.serialize` and `core.deserialize`. The following values are supported:

- `nil`
- booleans
- numbers
- strings
- tables, including circular references
- functions: _not recommended_; uses `string.dump` internally
  - Deserialization of functions will error unless mod security is disabled
  - Lua-implementation and platform-specific bytecode: Functions serialized by PUC Lua won't work on LuaJIT and vice versa; no cross-platform portability
  - Upvalues or the context of the function (function environment) aren't preserved
  - C functions which lack Lua bytecode (such as `math.sqrt` or most Luanti API functions) can't be (de)serialized at all

Userdata objects (like `ItemStack` for example) aren't supported. Threads (coroutines) aren't supported either.

#### `core.serialize(data)`

Serializes `data`, which may be a value of any of the above types.

Returns a Lua chunk in `string` form that returns `data` if executed.

Will error with `Can't serialize data of type <type>` if it encounters an unsupported type.

Fully supports circular references.

#### `core.deserialize(str, [safe])`

If `str` is not of type `string`:: Returns `nil, "Cannot deserialize type '<type>'. Argument must be a string."`
If `str` starts with `"\27"`:: Returns `nil, "Bytecode prohibited"`
If `str` triggers a Lua error at run- or load-time:: Returns `nil, error`
If `str` returns without errors:: Returns the first value returned by executing the chunk `str` without arguments

If `safe` is truthy, serialized functions will be deserialized to `nil`.
This will trigger an error if functions are used as table keys (`{[function()end] = true}`).
Otherwise, serialized functions will get an empty function environment set - only being able to operate on literals and arguments.

{{< notice tip >}}
Use of the `data = assert(core.deserialize(lua, safe))` idiom is recommended.
{{< /notice >}}

{{< notice warning >}}
[`core.deserialize` errors on large objects on LuaJIT](https://github.com/luanti-org/luanti/issues/7574)
{{< /notice >}}

## Engine-provided default persistence

Nodes (consisting of nodename, param1, param2, meta) are persisted automatically as part of mapblocks.
A handful of player properties (HP, breath, position, look direction, inventory, meta) are persisted as well.
Granularity is controlled by the `server_map_save_interval` setting.

## Storage options

### Database server / the ominous cloud

You can use Luanti's HTTP library to communicate with web servers, which might store data for you.

Other ways of Inter-Process Communication that can be leveraged to communicate with a database include _sockets_,
provided through the `luasockets` library (requiring an insecure environment and an accessible installation).
If the database server runs on the same machine, you might decide to use file bridges for IPC.

### Lightweight database library

Requires an insecure environment and an installation of the database library that is accessible to Luanti.
SQLite3, available through the `lsqlite3` LuaRocks package,
is a popular choice here and used for instance by the [sban](https://github.com/shivajiva101/sban) mod.

### String stores

#### Entity staticdata

Tied to entities. The serialized string must be returned by `get_staticdata` and is passed to `on_activate`.

#### File store

Usually tied to world or mod paths. The simplest approach reads the file at load time and writes it on shutdown.
As `on_shutdown` may however not be called in the case of a crash -
or even worse, a power outage might abruptly shut down the server without calling anything -
this provides a rather poor granularity, as all changes to the data during the uptime may be lost.

You may simply serialize your data and write it to a file on every update.
If your data is rather larger or gets updated frequently, a full serialization might negatively impact performance.
Performance can be improved at the expense of granularity by saving periodically and choosing "long" periods.
A transaction log improves performance by only storing changes, at the expense of disk space.

{{< notice tip >}}
A mix of both approaches can provide satisfying results, logging only changes and rewriting the logfile periodically to keep disk space waste acceptable.
{{< /notice >}}

For special cases like logging, an append-only file may be the ideal solution if using the global `core.log` is not desirable.

### Key-value store

#### Filesystem

On systems that provide a decent filesystem implementation (that is, everything except Windows),
you can use filenames/filepaths as keys and files as values.
On poor filesystems, you might be heavily limited by absolute path character limits;
lots of small files might lead to fragmentation.

A nested hierarchical key-value store is possible through directory structures, which can be managed and traversed using:

- `core.mkdir`
- `core.rmdir`
- `core.cpdir`
- `core.mvdir`
- `core.get_dir_list`

If you want to mitigate the risk of data loss, you can use `core.safe_file_write` when (re)writing files.

#### Configuration files

The `Settings` object allows you to operate on configuration files, getting & setting key-value entries and saving the file.
The main `Settings` object `core.settings` can be used to persist a few settings "globally" - bleeding everywhere.
This is horribly abused by the mainmenu to store stuff like the last selected game. Don't be like the mainmenu;
distinguish persistent game data and settings/configuration properly and _namespace_ your data properly.

That said, `Settings` can be used to read from & write to simple & limited key-value store files.
The main advantage of them is that they are presumably easy to edit for users,
but this is usually not a requirement for game data which is "edited" by in-game interactions;
indeed, you might not want to tempt players to cheat in singleplayer by editing their "saves".

#### MetaData

Luanti provides metadata objects which all provide a simple string key-value store, tied to four different game "objects":

1. ItemStacks: `ItemStackMetaData`: Fully sent to clients; serialized within inventories, which may be serialized within mapblocks
2. Node positions: `NodeMetaData`: Sent to clients, but fields can be marked as private; serialized somewhere within mapblocks
3. Players: `PlayerMetaData`: SQLite-backed key-value storage, however only available while the player is online
4. Mods: `ModStorage`: SQLite-backed key-value storage; older versions use a JSON-backed store unsuitable for large data (as serialization will block the main thread)

Utilities for setting & getting non-string data types like integers and floats are provided; the datatype is however not stored with the entries.
The granularity of all these key-value stores is determined by the `server_map_save_interval` setting.
