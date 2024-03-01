---
layout: post
title: Java class loader
date: 2024-03-01 00:20 -0800
categories: [programming-language, java]
tags: [java, classloader]
---

Recently, I was reading Kafka connect source code, and was trying to figure out
how Kafka connect loads plugins provided by users. It turns out Kafka connect
uses the standard ServiceLoader to load plugin implementations. The question
then becomes how ServiceLoader works.

## ServiceLoader

Below is how Kafka connect uses ServiceLoader to load plugin implementations
[[code](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/isolation/DelegatingClassLoader.java#L424-L424)].

```
ServiceLoader<T> serviceLoader = ServiceLoader.load(klass, loader);
for (Iterator<T> iterator = serviceLoader.iterator(); iterator.hasNext(); ) {
    T pluginImpl;
    try {
        pluginImpl = iterator.next();
    } catch (ServiceConfigurationError t) {
        log.error("Failed to discover {}{}", klass.getSimpleName(), reflectiveErrorDescription(t.getCause()), t);
        continue;
    }
    result.add(pluginDesc((Class<? extends T>) pluginImpl.getClass(),
        versionFor(pluginImpl), loader));
}
```

It firsts creates a `ServiceLoader` object by calling the `.load` method. Then
it iterates this service loader. OK. So service loader provides some iterator
interface. See
[here](https://github.com/openjdk/jdk/blob/dc08216f0ef55970c96df43bcc86ebd5792d486e/src/java.base/share/classes/java/util/ServiceLoader.java#L1299-L1299).
It has two different kinds of service providers: module based and lazy class
path based. I am not familiar with Java module, so we focus on class path
service providers. Continuing reading the implementation, you see it finds
source files with name prefix `META-INF/services/`, and then read each file
line by line. It trims each line and discards lines start with `#`. Each line
represents a full class name. And finally, it loads the corresponding class for
this name
[[code](https://github.com/openjdk/jdk/blob/dc08216f0ef55970c96df43bcc86ebd5792d486e/src/java.base/share/classes/java/util/ServiceLoader.java#L1217-L1217)].

```
try {
    return Class.forName(cn, false, loader);
} catch (ClassNotFoundException x) {
    fail(service, "Provider " + cn + " not found");
    return null;
}
```

OK. `Class.forName(cn, flase, loader)` is the core part of how ServiceLoader
works!

Method `Class.forName` is a simple wrapper on top of `Class.forName0` which is
a native function. The corresponding C implementation is
[here](https://github.com/openjdk/jdk/blob/dc08216f0ef55970c96df43bcc86ebd5792d486e/src/java.base/share/native/libjava/Class.c#L106-L106).
Following the call chain, it finally comes to
[this part](https://github.com/openjdk/jdk/blob/dc08216f0ef55970c96df43bcc86ebd5792d486e/src/hotspot/share/classfile/systemDictionary.cpp#L1382-L1382)

```
// Call public unsynchronized loadClass(String) directly for all class loaders.
// For parallelCapable class loaders, JDK >=7, loadClass(String, boolean) will
// acquire a class-name based lock rather than the class loader object lock.
// JDK < 7 already acquire the class loader lock in loadClass(String, boolean).
JavaCalls::call_virtual(&result,
                        class_loader,
                        spec_klass,
                        vmSymbols::loadClass_name(),
                        vmSymbols::string_class_signature(),
                        string,
                        CHECK_NULL);

```

Basically, `Class.forName` calls virtual function `ClassLoader.loadClass` to
load class.

## ClassLoader

Coming back to Kafka connect, the class loader is `PluginClassLoader`. The
inheritance is
`PluginClassLoader -> URLClassLoader -> SecureClassLoader -> ClassLoader`.
`PluginClassLoader` overwrites the `loadClass` method only for some plugins
that need isolation. For the rest, it simply calls the method in the base
class, i.e., `ClassLoader`.

If you read `ClassLoader.loadClass` source code, you find a funny thing that it
delegates the class loading job to its parent class loader, or the bootstrap
class loader. If not found, then it calls its own `findClass` method. hmm... a
delegation pattern! Not sure about why design it this way.

OK. Then everything is clear now. For kafka connect, loading a plugin is just
implementing `findClass` method. `PluginClassLoader` inherits this method from
`URLClasLoader`.

`URLClasLoader` opens the plugin Jars and load the corresponding class for the
give service name.

## What lesson we learnt

After analyzing how Kafka connect loads a plugin. We learnt a design pattern.
If we want to write an extensible system that allows users writing their own
plugins, namely, allow a user to write an implementation for an interface, then
we can use a `URLClassLoader` to load all plugins inside a folder. We only need
agree with users on where the folder is, and ask them to put their jars inside
this folder.
