---
layout: post
title: "Obtaining the source code of GetAmped"
date: 2024-10-27 00:00:00 +0100
categories: [reverse-engineering]
---

Any online game, at some point, will go offline. Most commonly due to a decline in player base.
When these games go offline, the developers tend not to publish any server binaries or source code.

We're talking about a 3D fighting game called GetAmped (or FogsFighters / SplashFighters). It was in development since
2002 by Cyberstep, a Japanese company.
The game is popular in the Asian market, which lead up to a couple of releases in the west.

The Dutch release of this game was shut down in 2013, then got a couple of releases in different countries, but now only
exists in Asia.

I was able to obtain the source code of said game, 10 years after shutdown.
If you have some technical knowledge, you should be able to perform the
steps mentioned in this post yourself.

Want to download the game yourself? Try the [Internet Archive](https://web.archive.org/web/20141008085905/http://download.getamped.com/Getamped_EU/GetAmped_EU_setup.exe).

## Observing the client files

These are the available client files after installing the game:

{% highlight txt %}

> conf/
> dists/
> jre/
> music/
> option/
> update/
> amped.kar
> amped.exe
> amped_launcher.exe
> keel.dat
> keel_aw.dll
> keel_awtcx.dll
> keel_clazz.dll
> keel_...
> resources.kar

{% endhighlight %}

We can make some assumptions:

- The `jre` folder indicates that there's some involvement of Java (Java Runtime Environment).
- The `amped.exe` and `amped_launcher.exe` indicate that not all binaries are Java.
- The `amped.exe` binary is only ~200 KB in size, which is very small for a game like this.
- The `keel` related files might hint towards the game's engine.
- After opening the `keel.dat`, `amped.kar` and `resources.kar` files, it's clear that these are encrypted.

The game has 2 executables: `amped.exe` and `amped_launcher.exe`.

The launcher was the intended way of booting up the game,
it opens up an interface that originally was used to update the game. The server has been long gone, so any
connection attempts fail.

Luckily for us, the game boots up just fine when running `amped.exe` directly.

## Analyzing amped.exe

I always start with checking if there are any interesting strings present in the binary.
Usually strings will give away a lot of information on what the binary does.

Windows comes with a handy `strings` command that extracts strings from a binary.

{% highlight bash %}
D:\Cyberstep> strings amped.exe
...
java/lang/ClassLoader not found.
java/lang/ClassLoader
Couldn't create Java VM.
-Djava.library.path=.;libs
-Xincgc
-Xmx%dm
OKeelRuntimeWindow_GetAmped
JNI_CreateJavaVM
Error: jvm.dll not found.
Please reinstall this software.
Keel Launcher Exe
JRE\1.3.1\bin\hotspot\jvm.dll
...
{% endhighlight %}

After a quick search, the string `"JNI_CreateJavaVM"` indicates that it's using the Java Native Interface (JNI) to
create
the JVM.
The JNI is a library you can use in C to manage a JVM and interact with it.

At this point, I've opened up my disassembler to see what's going on.

We can locate one of the xrefs to the `"JNI_CreateJavaVM"` string and see where it's called.

{% highlight assembly %}
004223bf  68f8f64200         push    data_42f6f8  {"JNI_CreateJavaVM"}
004223c4  56                 push    esi {var_2d0_1}
004223c5  ff1508f04200       call    dword [GetProcAddress]
004223cb  8d54242c           lea     edx, [esp+0x2c {var_29c}]
004223cf  52                 push    edx {var_29c} {var_2cc_6}
004223d0  53                 push    ebx {var_2d0_2}
004223d1  50                 push    eax {var_2d4_1}
004223d2  e859f6ffff         call    sub_421a30
{% endhighlight %}

Although this only gets the address to `"JNI_CreateJavaVM"` function, after analyzing the `sub_421a30` function we
actually
start seeing some JNI calls in action:

{% highlight assembly %}
00421b09  52                 push    edx {var_14c} {vm_args}
00421b0a  8d44240c           lea     eax, [esp+0xc {var_164}]
00421b0e  894c242c           mov     dword [esp+0x2c {vm_args_1}], ecx {var_11c}
00421b12  50                 push    eax {var_164} {var_174_1}
00421b13  8d4c2424           lea     ecx, [esp+0x24 {var_150}]
00421b17  51                 push    ecx {var_150} {var_178_1}
00421b18  c744245cc0f64200   mov     dword [esp+0x5c {var_11c}], data_42f6c0
00421b20  c7442464a4f64200   mov     dword [esp+0x64 {var_114}], data_42f6a4  {"-Djava.library.path=.;libs"}
00421b28  c744242c02000100   mov     dword [esp+0x2c {var_14c}], 0x10002
00421b30  c744243003000000   mov     dword [esp+0x30 {var_148}], 0x3
00421b38  c644243800         mov     byte [esp+0x38 {var_140}], 0x0
00421b3d  ffd6               call    esi ; JNI_CreateJavaVM

{% endhighlight %}

Oracle has some documentation
about [The Invocation API](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/invocation.html) which covers
the call to `JNI_CreateJavaVM`.

`JNI_CreateJavaVM` has the following signature:

{% highlight c %}
jint JNI_CreateJavaVM(JavaVM **p_vm, void** p_env, void* vm_args);
{% endhighlight %}

So from the above assembly, we know `var_150` holds a pointer to a `JavaVM` struct and `var_164` holds a pointer to a `JNIEnv` struct.

After renaming these variables, we can find them easier in the disassembly.
Not much further down, the `"java/lang/ClassLoader"` string is referenced.

{% highlight assembly %}
00421b75  8b442408           mov     eax, dword [esp+0x8 {JNIEnv*}]
00421b79  8b10               mov     edx, dword [eax]
00421b7b  6870f64200         push    data_42f670 {var_170}  {"java/lang/ClassLoader"}
00421b80  50                 push    eax {var_174_3}
00421b81  8b4218             mov     eax, dword [edx+0x18]
00421b84  ffd0               call    eax
{% endhighlight %}

After reading some more JNI documentation, it becomes clear that this is a call to the `FindClass` function.

Any Java installation comes with header files for the JNI. If we'd import these, we'd get rid of those pesky "random" offsets.
I'm using Binary Ninja, so I can very easily retype `JNIEnv*` to its actual type, and the assembly will make a lot more sense:

{% highlight assembly %}
00421b75  8b442408           mov     eax, dword [esp+0x8 {JNIEnv*}]
00421b79  8b10               mov     edx, dword [eax]
00421b7b  6870f64200         push    data_42f670 {var_170}  {"java/lang/ClassLoader"}
00421b80  50                 push    eax {var_174_3}
00421b81  8b4218             mov     eax, dword [edx+0x18 {JNINativeInterface_::FindClass}]
00421b84  ffd0               call    eax
{% endhighlight %}

## Finding the compiled Java classes

The JNI is able to load compiled Java classes during runtime.
There's a function called `DefineClass` that takes in a class loader 
and a buffer containing raw bytes representing the compiled Java class.

Looking at the xrefs to `DefineClass`, we can see 10 calls to this function.

The xref takes us to a function where some data is being pushed onto the stack. 1 byte at a time...

{% highlight assemble %}
00401290  c68424d0250000c9   mov     byte [esp+0x25d0 {buf_8}], '\xc9'
00401298  c68424d125000071   mov     byte [esp+0x25d1 {arg_25c1}], 'q'
004012a0  c68424d225000046   mov     byte [esp+0x25d2 {arg_25c2}], 'F'
004012a8  c68424d325000007   mov     byte [esp+0x25d3 {arg_25c3}], '\x07'
004012b0  c68424d4250000d1   mov     byte [esp+0x25d4 {arg_25c4}], '\xd1'
004012b8  c68424d525000054   mov     byte [esp+0x25d5 {arg_25c5}], 'T'
004012c0  c68424d62500008b   mov     byte [esp+0x25d6 {arg_25c6}], '\x8b'
004012c8  c68424d7250000fe   mov     byte [esp+0x25d7 {arg_25c7}], '\xfe'
004012d0  c68424d8250000ae   mov     byte [esp+0x25d8 {arg_25c8}], '\xae'
004012d8  c68424d925000001   mov     byte [esp+0x25d9 {arg_25c9}], '\x01'
004012e0  c68424da250000b1   mov     byte [esp+0x25da {arg_25ca}], '\xb1'
004012e8  c68424db2500007b   mov     byte [esp+0x25db {arg_25cb}], '{'
{% endhighlight %}

It does this approximately 17,000 times. These bytes often don't even represent characters, so it's likely encrypted.

Reverse engineering the decryption algorithm is one option, 
but why bother when we can simply put an old-fashioned breakpoint on the `DefineClass` call?
The `DefineClass` function should still be called with valid bytes that represent a Java class after all, 
so the data must be decrypted before that.

Take this call for example:

{% highlight assembly %}
00421710  8b4014             mov     eax, dword [eax+0x14 {JNINativeInterface_::DefineClass}]
00421713  68a8050000         push    0x5a8 {len}
00421718  8d8c24c4100000     lea     ecx, [esp+0x10c4 {buf_5}]
0042171f  51                 push    ecx {buf_5} {var_2c_1}
00421720  53                 push    ebx {var_30_1}
00421721  8d942498410000     lea     edx, [esp+0x4198 {name_5}]
00421728  52                 push    edx {name_5} {var_34_1}
00421729  55                 push    ebp {var_38_1}
0042172a  ffd0               call    eax ; <-- BREAKPOINT
{% endhighlight %}

and the `DefineClass` signature:

{% highlight c %}
jclass DefineClass(JNIEnv *env, const char *name, jobject loader,
const jbyte *buf, jsize bufLen);
{% endhighlight %}

Binary Ninja already renamed some variables for us according to the header files of the JNI,
so we know that the class name is stored in `name_5`, and the class data in `buf_5`.

There aren't any anti-debugging techniques in place, so we can run the binary and wait until the breakpoint is hit.
After that, we can dump the decrypted class data from `buf_5`.

I've gone ahead and did this to all classes, and it gave me the following files:

{% highlight txt %}
> BootstrapException.class
> KarSource$CountableOutputStream.class
> KarSource$CryptKey.class
> KarSource$DecryptorInputStream.class
> KarSource$EncryptorOutputStream.class
> KarSource$KarEntry.class
> KarSource$KarInfo.class
> KarSource$KarInputStream.class
> KarSource.class
> KeelClassLoader.class
{% endhighlight %}

In Java, inner classes will result in a separate .class file. The hierarchy is denoted with a $, e.g.:

{% highlight java %}
class KarSource {
    class KarEntry {
    }
    class KarInfo {
    }
    // ...
}
{% endhighlight %}

## Decompiling the classes

Java .class files contain bytecode, which can be easily decompiled using the FernFlower decompiler.
The community edition of IntelliJ IDEA comes with this decompiler built-in. 
All we simply need to do is store all these .class files in a single folder and open it in IntelliJ:

![Keel Source Tree](/assets/img/keel.png)

It's obvious that this is not the source code of the game, 
but from now on its easy game as the assembly is not needed much longer.

## Decrypting the keel.dat file

The implementation of `KarSource` contains a constructor that takes in 2 arguments:

{% highlight java %}
public KarSource(String var1, String var2)
{% endhighlight %}

After reading the source code, I'm renaming the arguments to:

{% highlight java %}
public KarSource(String path, String password)
{% endhighlight %}

There's not much of the assembly left to reverse, but 2 strings are being referenced:

{% highlight assembly %}
00421c8c  bed8f44200         mov     esi, data_42f4d8  {"dhfuhsudfh98vhdsovnfdhiouer8u8hgjbkjciudsuifsjdiosajfn"}
{% endhighlight %}

and 

{% highlight assembly %}
00421cbe  68ccf44200         push    data_42f4cc {JNIEnv*}  {"keel.dat"}
{% endhighlight %}

isn't that suspicious?

We can include these compiled Java classes into our own Java program and use the `KarSource` class ourselves, 
and plug in these 2 strings:

{% highlight java %}
var file = "D:\\Cyberstep\\keel.dat";
var pass = "dhfuhsudfh98vhdsovnfdhiouer8u8hgjbkjciudsuifsjdiosajfn";
var source = new KarSource(file, pass);
{% endhighlight %}

and... it works! The `keel.dat` file is decrypted and the contents is all ours. The `KarSource` implementation exposes
a simple API to extract the contents, so I've just done that.

It appears that `keel.dat` contains encrypted byte code of the game's engine. The file contained 707 classes in total.

But what about the `amped.kar` and `resources.kar` files? We still need to find the encryption keys for those.

## Decrypting the amped.kar file

The "Keel" engine expects an `"-entrypath"` argument to be passed, at least that's what their source is hinting towards:

{% highlight java %}
if (appClassName == null) {
    System.out.println("Usage: KeelRuntime [-options] class [args...]");
    System.out.println("where options include:");
    System.out.println("-debug     debug graphics code");
    System.out.println("-entrypath <path or jar files...>");
    System.out.println("           set search path for keel application classes");
    System.exit(1);
}
{% endhighlight %}

and it so happened that the `-entrypath` is set to `"kar:program_getamped:kar"`.

but... the password is nowhere to be found and from the `KarSource` implementation, we know that a password is required.

After looking through the Keel source code, I've stumbled across this snippet:

{% highlight java %}
// var0 = "kar:program_getamped:kar"
public static Archive openArchive(String var0) throws IOException {
    if (var0.startsWith("kar:")) {
        return new Archive(createKarSource(var0, ':'));
    } else {
        return var0.startsWith("kar;") ? new Archive(createKarSource(var0, ';')) : openArchive(new File(var0));
    }
}

// var0 = "kar:program_getamped:kar"
// var1 = ':'
private static ArchiveSource createKarSource(String var0, char var1) throws IOException {
    String var2 = var0.substring(var0.indexOf(var1) + 1);
    String var3 = null;
    String var4;
    if (var2.indexOf(var1) >= 0) {
        var3 = var2.substring(0, var2.indexOf(var1));
        var4 = var2.substring(var2.indexOf(var1) + 1);
    } else {
        var4 = var2;
    }
    return new KarSource(var4, var3);
}
{% endhighlight %}

See that `var3` being passed as the 2nd argument to `KarSource`? That's the password, and we need it.
After analyzing the source code, it seems that the password is simply `"program_getamped"`.
That's it.

Remember that old `KarSource` implementation we used to decrypt the `keel.dat` file?
I tried using it on `amped.kar`, but it didn't work. It seemed to be able to extract the class names, but not the contents.

It appears that a newer version of the `KarSource` implementation is present in the `keel.dat` file.
So... here we go again:

{% highlight java %}
var file = "D:\\Cyberstep\\amped.kar";
var pass = "program_getamped";
var source = new KarSource(file, pass);
{% endhighlight %}

and that's it!

After dumping the contents of `amped.kar`, the game's source code has been restored.
There are over 7000 classes to be found in this one file.

![Amped Source Tree](/assets/img/source.png)

## Decrypting the rest of the kar files

Sorry folks, this one is rather anti-climatic. All the other passwords were super easy to find in the source code.

{% highlight java %}
SETTING_KAR_PW = "setting";
PROGRAM_KAR_PW = "program_getamped";
RESOURCE_KAR_PW = "resource_getamped";
DISTS_KAR_PW = "dists_getamped";
{% endhighlight %}

Go crazy!

## Conclusion

Having the source code allows reverse engineers to learn how the game was built. 
It allows us to easily create our own server emulator from scratch, which is easier with source code than reading assembly.

This game offers a lot of content, albeit by pay-walling most of the weapons and classes, there were hundreds of them.
After looking through `resources.kar`, I'm quite blown away by how much scripting work went into this game.

If you feel like picking up on this project, please reach out to me. I'd love to keep an eye on it!

Fun fact: The code contains server related files that definitely shouldn't be on the client-side.
