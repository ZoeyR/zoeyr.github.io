# Announcing Bincode 1.3

After a little over 6 months, bincode 1.3 is finally here! This version is a big milestone. It brings many new features and bugfixes, as well as introducing new maintainers
and a roadmap.

## What is Bincode

Bincode is a compact serialization library for Rust. It uses a zero-fluff encoding scheme where encoded objects will be the same size or smaller than what they take up in memory.

## Special Thanks

First of all I would like to thank the contributors to this release of bincode, without whom this release wouldn't be possible!

* [Jean Airoldie](https://github.com/jean-airoldie)
* [Kent Fredric](https://github.com/kentfredric)
* [Maciej Hirsz](https://github.com/maciejhirsz)
* Leonard Kramer
* [Nicole Mazzuca](https://github.com/strega-nil)
* [Joonatan Saarhelo](https://github.com/joonazan)

And I would also like to thank [Josh Matthews](https://github.com/jdm) and [Ty Overby](https://github.com/TyOverby) for trusting me to maintain bincode.

## New Maintainer

For those of you who don't know, Bincode has existed since before Rust hit 1.0. Over its existance it has had multiple maintainers. Ty Overby, David Tolnay, and Josh Matthews to name a few.
In the last year, development speed of Bincode has slowed down. It met most of its users' needs and developer mindshare shifted elsewhere to projects that were changing faster and had more 
room for experimentation.

The consequence of this is that issues on Bincode began to pile up. Even though most users' needs were met, there were still some requests that required new feature work as well as time
investigating how the features would fit into the overall design of bincode. There were also some areas of technical debt that could be cleaned up to improve the safety and speed of the
library overall.

A few months ago I got in contact with Ty and Josh to ask them if they would be interested in tranferring maintainership. I was a previous contributor to the library, having helped migrate it
through the massive breakage of serde 0.9. After a short discussion it was decided that I would take over the maintinence of bincode.

I don't take the role of bincode maintainer lightly. Bincode is a serialization library with millions of downloads and many high profile users. I am commited to keeping bincode stable, safe,
and performant. There may be a 2.0 in the far future, but my primary concern is not breaking the ecosystem that bincode has today.

## Plans for the future

In the coming months I have a few overarching plans for what to do with bincode. 

* Resolve outstanding major feature requests
* Write a specification for the format
* Create implementations in other languages

1.3 is the first step along that path. This release contains bugfixes and some of the most requested features.

## Release notes

### What's New?

1.3 adds three big features to bincode: constructable serializers/deserializers, a new config system that's even more performant than the last, and varint support.

#### New config system

This feature is a mostly technical one that won't affect most typical users of bincode. Bincode has a 0-cost configuration system. The config is expressed in the type system which removes any
runtime branching needed. This makes configuration performant. Previously you couldn't access these config types directly. Instead configuration was constructed through an intermediate object that would
create the actual config through a massive match statement at runtime. This meant that there was a slight penalty every time you ran bincode. The new config system allows the user access to the internal
config types directly, removing the need for a large match statement and improving performance (if only slightly). The main advantage of exposing these types is that it allows the internal config to be
stored, which leads into the next big feature.

#### Constructable Serializers/Deserializers

Being able to create an instance of the Serializer/Deserializer types directly is an important feature for interoperability. Libraries like `erased-serde` need access to a `Serializer` in
order to work correctly. Bincode didn't expose the Serializer/Deserializer structs before since it was thought that exposing them would lock us out of adding new configuration choices and options.
With the addition of the new config system there is no longer that risk, so the Serializer/Deserializer can be exposed to users!

#### Varint support

Varint support has been a long requested feature of bincode. Previously, enum discriminants and slice lengths were stored as u32 and u64 respectively. This was wasteful for most objects since it is
very uncommon to have an enum with millions of discriminants or a slice with billions of entries. Varint is a new config option for bincode that will store all values under 250 in a single byte, saving
a lot of space in the most common cases.

### What's Fixed?

#### UB while deserializing

There was a well known UB bug in bincode that could be triggered by safe code. This is due to the fact that Rust doesn't have a good way to read into uninitialized buffers (see the linked bug for more info).
The fix ended up impacting performance slightly, but since serialization libraries are often the source of security issues in other languages, the fix was judged to be worth the performance hit.

#### Poor codegen when deserializing from slices

The slice deserialization code was generating suboptimal assembly with a lot of unnecessary function calls. A small change was made to allow the optimizer to work more effectively.

## Closing

Again, thank you to all those who contributed to 1.3, I could not have done it without you. Thanks to Ty and Josh, for trusting me with such an important project. I can't wait to see where bincode goes
in the future.