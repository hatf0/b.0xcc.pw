+++
title = "Virtual Virtual Classes"
date = 2021-02-09T13:09:37-05:00
draft = false
description = "or how I learned how to abuse RtDynamicCast"
author = "hatf0"

[[repo]]
type = "github"
spec = "hatf0/classtest"

+++
{{< repo >}}

I was inspired by my friend, Val, on a concept: 
> What if we could add custom classes into Blockland?

I was immediately intrigued on this -- I've reversed Blockland for years, and am familiar with the internals. 

For some context, Blockland is a block-based building game built on the Torque Game Engine (and really, quite an old version..). The Torque Game Engine has an interesting type system, which allows for some neat tricks.

## Torque's Type System
Internally, every class that is exposed to the TorqueScript engine has both a static and a dynamic class representation 
(those being `AbstractClassRep` and `ConcreteClassRep`). These are defined with the `DECLARE_CONOBJECT` macro, which is defined as:
```cpp
#define DECLARE_CONOBJECT(className)                    \
   static ConcreteClassRep<className> dynClassRep;      \
   static AbstractClassRep* getParentStaticClassRep();  \
   static AbstractClassRep* getStaticClassRep();        \
   virtual AbstractClassRep* getClassRep() const
```

They are then implemented with the use of the `IMPLEMENT_CONOBJECT` macro, which is defined as:
```cpp
#define IMPLEMENT_CONOBJECT(className)                                                            \
   AbstractClassRep* className::getClassRep() const { return &className::dynClassRep; }           \
   AbstractClassRep* className::getStaticClassRep() { return &dynClassRep; }                      \
   AbstractClassRep* className::getParentStaticClassRep() { return Parent::getStaticClassRep(); } \
   ConcreteClassRep<className> className::dynClassRep(#className, 0, -1, 0, className::getParentStaticClassRep())
```

Example usage:
```cpp
// CustomClassObject.h
class CustomClassObject : public SimObject {
    typedef SimObject Parent;
public:
    // foo

    static void consoleInit();
    static void initPersistFields();

    DECLARE_CONOBJECT(CustomClassObject);
}

// CustomClassObject.cpp
#include "CustomClassObject.h"

// method implementations here

IMPLEMENT_CONOBJECT(CustomClassObject);
```
One would assume it's just as easy to add a new file, define our class, and be on our way. This is correct.. if you have the source code. 
Unfortunately, the most source code that we have is from the IDA Decompiler, and the engine source code. We're working from a DLL, with several critical methods we need inlined.. 

How do we go about this, then?

## A Primer On DLLs
A DLL, or a Dynamic Link Library, is the Windows-equivalent of a `.so` object on Linux. We can either:
* modify the PE header to have our DLL loaded before the program gains control

or

* inject the DLL at runtime, causing our code to be loaded into memory.

This allows us to inject code, modify execution flow, etc. Once we have a DLL injected, we have virtually infinite control over the program. 

## Beating the AbstractClassRep engine at instantiation
Great! Using that primer, we've built our DLL and injected it in. Within our `init`, we call
```cpp
AbstractClassRep::registerClassRep(CustomClassObject::getStaticClassRep());
```
And all is good in the world! We have class support!

But wait, something isn't right... 
`consoleInit()` is never getting called -- neither is `initPersistFields()`. What's going on? 

If we look in `console/consoleObject.cc` of the engine's source code, we can get a clue.
```cpp
void AbstractClassRep::registerClassRep(AbstractClassRep* in_pRep)
{
   AssertFatal(in_pRep != NULL, "AbstractClassRep::registerClassRep was passed a NULL pointer!");
   in_pRep->nextClass = classLinkList;
   classLinkList = in_pRep;
}
```

Notice how `registerClassRep` never sets up the class representation? `registerClassRep` was designed with the intention that:
* every class would register the class representation before `AbstractClassRep::initialize` is called
* `AbstractClassRep::initialize` will handle allocation / instantiation, don't handle it here

That's handled in `AbstractClassRep::initialize`... but `AbstractClassRep::initialize` is called at the *very beginning*, before everything else. 

To solve this? We just add our DLL to the PE stub, and don't inject it manually. Now we're running before `AbstractClassRep::initialize` is called.

Unnnnfortunately, we're now crashing. It turns out that modifying an engine variable before the engine has even initialized isn't such a good idea. Now what? 

## polyhook & Chill

[`polyhook`](https://github.com/stevemk14ebr/PolyHook_2_0) is a "x86/x64 C++ Hooking Library" which easily allows us to intercept function calls. 

We use this library to hook `AbstractClassRep::initalize` so we can register our class rep as early as possible, but not *too early*. 

How do we do this? The library inserts a "trampoline", and effectively detours the function call, giving us control before any other code can run.
```cpp
    #define ACR_INITIALIZE_OFFSET 0x11231 //invalid
    typedef void (*ACR_InitializeFn)();
    ACR_InitializeFn ACR_Initialize;
    PLH::x86Detour* ACR_InitializeDetour;
    uint64_t ACR_InitializeTramp;

    // within our init function
    ACR_Initialize = (ACR_InitializeFn)(ImageBase + ACR_INITIALIZE_OFFSET);
    ACR_InitializeDetour = new PLH::x86Detour((char*)ACR_Initialize, (char*)&h_ACRInit, &ACR_InitializeTramp, dis);
```
And while, yes, we could write our own trampoline layer, I find it easier to just use a library such as PolyHook / Detours. 

## Abusing RTDynamicCast to win
Wonderful! We've registered our AbstractClassRep, we've had `consoleInit` and `initPersistFields` called, and we're now ready to create our object.


## Don't Copy Those Vtables!


{{< figure src="https://f.0xcc.pw/palo-alto/1kVFKgfTSgsc.png" caption="Success! (spoilers!)" >}}

## We won, but how do we add member methods?

## How about adding fields?

## Fin

