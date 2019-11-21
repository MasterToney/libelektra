# Specification Language

This tutorial introduces you to the concepts of the specification language which are used by Elektra. The specification language
can be used to apply a validation for the configuration that you want to ship for your application.

## How Elektra perfroms validation

We encourage the reader to be first familiar with the tutorial of [namespaces](namespaces.md) and [mounting](mount.md). A basic
understanding of [set](../help/kdb-set.md) and [meta-set](../help/kdb-meta-set.md) is also helpful for understanding this tutorial.
When you [mount](mount.md) a configuration such as `kdb mount test.conf /tests dump` you usually use one of the four standard namespaces
`user`, `dir`, `proc` or `system`.

There is a 5th namespace called the `spec` namespace and has a special behavior. Any validation behavior
will be written to the `spec` namespace. You can use `meta-set` to set any metadata to the spec namespace.

```sh
kdb meta-set /tests/bool 'type' boolean
```

Now you can mount a configuration under the `/tests` path and be assured that the key `bool` will
have a boolean value validation applied:

```sh
# An example configuration which will by default gets mounted to the 'user' namespace
kdb mount test.conf /tests dump type

kdb set /tests/bool Batman
# RET:5
# Using name user/tests/bool
# Sorry, module type issued the error C03200:
# Validation Semantic: The value 'uaud' of key 'user/tests/bool' could not be converted into a boolean

# Cleanup
kdb umount /tests
kdb rm -r /tests
```

Elektra takes all configuration settings which the user or admin provides and parses it into a structured internal format.
This structure is called a `KeySet` and is format agnostic (meaning that validation can be applied to any file format
such as `ini`, `yml`, etc.). After successful parsing into the KeySet, Elektra then passes this KeySet to various
validation plugins. You can get a picture of all plugins [here](/src/plugins). Every plugin performs validation
of the KeySet depending on the metadata provided (more detail about this will come in the next section) and emits warnings
or errors. If any error happened, the changes of the configuration settings are not applied.

E.g. you can see
that in the upper example we also mounted the [type](/src/plugins/type/README.md) plugin for validation checks. This
allowed us to check for wrong types such as `Batman` being no valid boolean.

Please note as of Elektra v0.9 you cannot add an unlimited amount of plugins. You can see if this issue
still remains in our issue tracker([#2133](https://github.com/ElektraInitiative/libelektra/issues/2133)).

## Specification writing

There are multiple ways how you can set constraints for configuration settings. All possibilities though set `metadata` to certain
configuration settings. Setting metadata should be done in the spec namespace. A metadata is just additional data describing the configuration setting.
One way to set metadata is the command line tool [kdb meta-set](../help/kdb-meta-set.md) which we have used in the upper example.
For the [validation plugin](../../src/plugins/validation/README.md) which supports regular expression checks there exists even its own
command line tool [kdb vset](../help/kdb-vset.md).

If you are writing validation for dozens of configuration settings, the command line might not be the most convenient way. You can also
write specifications in any file format of your desire. Examples can be seen [here](../../examples/spec). We recommend the INI
file format as it natively supports metadata in an easy and understandable way. So the upper example
could be rewritten in an INI file in the following way:

```ini
[]
mountpoint = test-specification.ini
infos/plugins = type

[bool]
type = boolean
```

To integrate a configuration into the key database, we mount it by using the line `mountpoint = test-specification.ini`
in the configuration specification at the top. This is a mandatory information for the path under which the configuration is mounted
(also called the `parent key`).
This will be mounted into the spec namespace with the following command:

```bash
kdb mount </absolute/path/to/ini/file.ini> spec/tests ni
kdb spec-mount /tests
```

The `spec-mount` command tells Elektra to load all relevant plugins. It is smart enough to automatically detect the needed plugins
by looking at all metadata. Sine `type` for is given under the `[bool]` key, Elektra will load the `type` plugin.
You might need to remove some metadata to not load certain plugins that cause the exceeding of the maximum
plugin number.

What remains to know are the metadata names and meanings. We have used the `type` plugin which provides the metadata `type` metadata along with
many other metadata such as `check/enum`. You can read all available metadata in the plugin READMEs which all have to have a
so called `contract` at the beginning of each file. This contract also contains the `infos/metadata` data which lists all available
metadatas. We recommend to look through those plugins READMEs to know how they are intended to use.

As another example you may take the [range](/src/plugins/range/README.md) plugin. The `infos/metadata` show that
`check/range` is provided by the plugin:

```sh
kdb mount test.conf /tests dump range
kdb set /tests/range 5
kdb meta-set /tests/range 'check/range' '1-10'
kdb set /tests/range 12
# RET: 5
# Using name user/tests/range
# Sorry, module range issued the error C03200:
# Validation Semantic: Value '12' of key 'user/tests/range' not within range 1-10
```

You can also find all available metadata in the [METADATA.ini](/doc/METADATA.ini) file. It shows which
values certain metadatas can take such as `check/ipaddr` only accepting either `ipv4` or `ipv6`. This can be seen
in the `type` field. The file also shows which plugins or tools rely on this metadata such as the `check/ipaddr` is used by
both the `network` as well as the `ipaddr` plugin. A short description as well as examples are also given the majority of the entries.
For a more detailed information please visit our [metadata](../dev/metadata.md) tutorial.

Don't forget to clean up your tutorial executions:

```sh
kdb umount /tests
kdb rm -r /tests
```

### Nice to know

Many plugins write metadata with arrays in them such as the `type` plugin which allows to list all enumerations for configuration settings.
We have a nice [tutorial](../tutorials/arrays.md) for you to get used to arrays.

You can also set validation for multiple configurations settings at once by `_` or `#`. More information can be read in the documentation
of the [spec plugin](/src/plugins/spec/README.md).

If you decide to write metadata via a file (and especially the INI format) you should be aware of multiline strings that can cause
parsing issues. Descriptions of metadata often take up multiple lines and
there are multiple INI format standards which tell you differently how to write them.
Elektra also has two ini plugins (`ni` and `ini`) which also handle multilines differently. We recommend the `ni` plugin which
does not face this problem and also works correctly with our provided [examples](../../examples/spec).
