---
layout: post
title: "Injecting custom TNL NetEvents into QPang"
date: 2025-11-17 00:00:00 +0100
categories: [ reverse-engineering, tnl ]
---

We wanted to add custom TNL NetEvents into the game we are modding, without hooking & patching existing events.
Something one can't easily do without the original game's source code, yet we did it anyway.
The game in question is QPang.

To preface; This post is not a tutorial on how to use TNL nor how to reverse engineer it,
instead it is meant for that _one_ person who might have the same idea in the future, 
decides to Google and somehow lands on this post.

For that one person, I hope this is resourceful.

## Torque Network Library (TNL)

TNL is a C++ network library targeted at game development.
It was in development around the 90's and early 00's, and originally 
built for the Torque Game Engine.
Some other engines like Virtools 4 gave you the option to use TNL, 
but you could also use it as a standalone library.
In the case of QPang, it was used as a standalone library.

TNL provides a concept named "NetEvent".
A NetEvent is a class that gives you the building blocks
to write & read data over the network.

A really simply example of a NetEvent declaration from
the [NetEvent docs](https://opentnl.sourceforge.net/doxytree/index.html):

{% highlight cpp %}
// A simple NetEvent to transmit a string over the network.
class SimpleMessageEvent : public NetEvent
{

    typedef NetEvent Parent;
    char *msg;

public:
    SimpleMessageEvent(const char *message = NULL);
    ~SimpleMessageEvent();

    virtual void pack   (EventConnection *conn, BitStream *bstream)
    virtual void unpack (EventConnection *conn, BitStream *bstream);
    virtual void process(EventConnection *conn);
   
    TNL_DECLARE_CLASS(SimpleMessageEvent);

};

TNL_IMPLEMENT_NETEVENT(SimpleMessageEvent, NetClassGroupGameMask,0);

{% endhighlight %}

The `TNL_DECLARE_CLASS` and `TNL_IMPLEMENT_NETEVENT` do some macro magic
to add some (static) member variables/methods that TNL uses internally.

There's nothing more to it. This NetEvent will be understood both by the client & server.
But that's also already the caveat to TNL; the library doesn't provide an API to
add new NetEvents during run-time. NetEvents must be declared during build-time.

So? If you don't have access to the source code, how would you go
about adding custom NetEvents to your game?

In theory, you can hope that the game you're modding
has a NetEvent that's reading a large string, or is making use of TNL's `ByteBuffer`.
You'd be able to hook the NetEvent's `process` function and implement some custom client logic
that unpacks the string into a custom data structure.
However, this is, in practice, hacky to implement on both client & server side, and might be off limits for some games.

For us, with the amount of custom features that we are working on, 
it made more sense to find an approach that allows us to add NetEvents the intended way.

## TNL Internals

TNL doesn't offer documentation on _how_ these compiled NetEvent objects end up being recognized by TNL
(at least not on the doxy docs pages).
All the developer needs to know is to call these magic TNL macros, build, and voilà.

The magic lies in static initialization. Internally, TNL keeps track of a static "Class Link List",
which falls under the `NetClassRep` scope. As the name implies, it's a linked list of `NetClassRep` instances,
or more precisely, `NetClassRepInstance` instances. A `NetClassRepInstance` provides an API to construct a
`NetEvent` by name and/or internal id.

If you expand the `TNL_DECLARE_CLASS` macro, it will declare a static `NetClassRepInstance`, like so:

{% highlight cpp %}
#define TNL_DECLARE_CLASS(className) \
static TNL::NetClassRepInstance<className> dynClassRep;      \
virtual TNL::NetClassRep* getClassRep() const
{% endhighlight %}

And in the constructor of `NetClassRepInstance`, the "Class Link List", AKA `mClassLinkList` is updated to include the
`NetEvent` in the linked list:

{% highlight cpp %}
NetClassRepInstance(const char *className, U32 groupMask, NetClassType classType, S32 classVersion)
{
    // Store data about ourselves
    mClassName = strdup(className);
    mClassType = classType;
    mClassGroupMask = groupMask;
    mClassVersion = classVersion;
    for(U32 i = 0; i < NetClassGroupCount; i++)
        mClassId[i] = 0;

    // link the class into our global list
    mNextClass = mClassLinkList;
    mClassLinkList = this;

}
{% endhighlight %}

Then, after all these instances have been statically initialized,
there's a function named `NetClassRep::initialize` which is responsible for parsing this linked list,
assigning unique ids to the instances, and ultimately storing them in a table so that TNL can handle the events
the moment they come in over the network.
This function is called whenever a `NetInterface` is constructed during run-time.

_snippet (simplified):_
{% highlight cpp %}
void NetClassRep::initialize()
{
    if(mInitialized)
        return;
    Vector<NetClassRep *> dynamicTable;

    NetClassRep *walk;
    
    for (U32 group = 0; group < NetClassGroupCount; group++)
    {
        U32 groupMask = 1 << group;
        for(U32 type = 0; type < NetClassTypeCount; type++)
        {
            for (walk = mClassLinkList; walk; walk = walk->mNextClass)
            {
                if(walk->getClassType() == type && walk->mClassGroupMask & groupMask)
                    dynamicTable.push_back(walk);
            }
        }
    }

}
{% endhighlight %}

After this function gets called in the binary, it will be difficult to register new events without patch work
or without a lot of effort. So we must register our NetEvents before this function call.

## Finding the ClassLinkList

To add new NetEvents, you need to locate the static `NetClassRep::mClassLinkList` variable in the binary.
If this offset is located, you can simply append your own `NetClassRepInstance` instances
to the game's `NetClassRep::mClassLinkList` before `NetClassRep::initialize` is called.

The offset of `NetClassRep::mClassLinkList` is a blatant giveaway.
The function that references this offset contains a string literal that will be baked into the binary,
so finding it is not a difficult task.

{% highlight txt %}
"Class Group: %d Class Type: %d count: %d"
{% endhighlight %}

With a simple string search `NetClassRep::initialize` can be found,
and therefore also `NetClassRep::mClassLinkList`:

_snippet from top of `NetClassRep::initialize`:_
{% highlight assembly %}
0063b930  void NetClassRep::initialize()

0063b930  a07c4e7c00         mov     al, byte [0x7c4e7c]
0063b935  83ec30             sub     esp, 0x30
0063b938  84c0               test    al, al
0063b93a  0f8574030000       jne     0x63bcb4

...

0063b970  b801000000         mov     eax, 0x1
0063b975  d3e0               shl     eax, cl
0063b977  c744241000000000   mov     dword [esp+0x10 {Type}], 0x0
0063b97f  89442428           mov     dword [esp+0x28 {var_18_1}], eax

0063b983  8b0d484e7c00       mov     ecx, dword [0x7c4e48] <--- CLASS LINK LIST !
{% endhighlight %}

In my case, the address `0x7c4e48` points to `NetClassRep::mClassLinkList`.

## Injecting custom NetEvents

Our SDK links against TNL, which allows us to use TNL in our SDK.

Declaring our custom NetEvent is done the regular TNL way:

{% highlight cpp %}
class CustomEvent : public NetEvent
{
    typedef NetEvent Parent;

public:
    CustomEvent();
    ~CustomEvent();

    virtual void pack   (EventConnection *conn, BitStream *bstream);
    virtual void unpack (EventConnection *conn, BitStream *bstream);
    virtual void process(EventConnection *conn);
   
    TNL_DECLARE_CLASS(CustomEvent);

};

TNL_IMPLEMENT_NETEVENT(CustomEvent, NetClassGroupGameMask,0);
{% endhighlight %}

The `CustomEvent` still needs to be injected into the game's class link list.
Our SDK's TNL linkage will cause a 2nd TNL "instance" to exist. One in the SDK's DLL and one in the game's binary.
AKA, there are now 2 class link lists, and we must make the game's class link list aware about the new one.

In the SDK, we have created a hook that hooks into the `NetClassRep::initialize` function.
This ensures our custom NetEvents will be registered before they are initialized:

{% highlight cpp %}
// This class list contains our custom net events, coming from our own TNL "instance".
// In our example: "CustomEvent".

TNL::NetClassRep* customClassList = TNL::NetClassRep::mClassLinkList;

// Important !!!
// TNL comes with 3 RPC GhostConnection related net events built-in.
// These net events need to be skipped, to avoid duplicate net events.
// As they will also exist in the game's binary...

for (auto i = 0; i < 3; i++)
{
    customClassList = customClassList->mNextClass;
}

TNL::NetClassRep* originalClassList = *reinterpret_cast<TNL::NetClassRep**>(0x7c4e48);
for (auto* walk = originalClassList; walk; walk = walk->mNextClass)
{
    // Appends our custom class link list at the end of the game's class link list.

    if (!walk->mNextClass)
    {
        walk->mNextClass = customClassList;
        break;
    }

}
{% endhighlight %}

TNL sorts the NetEvents based on their name before assigning internal ids,
so with the above setup, we don't have to worry about our custom event getting in the way.

The last step is on you. The custom event also needs to be implemented on the server side.
TNL will abort the connection during the handshake phase when the class count between client & server mismatch.

TNL also offers RPC Events. An RPCEvent is essentially just a NetEvent with a lot of sugar on top.
The above setup will also work for RPCEvents out of the box.

## Sources

- [OpenTNL DoxyDocs](https://opentnl.sourceforge.net/doxytree/index.html)
- [OpenTNL Source](https://github.com/kocubinski/opentnl)