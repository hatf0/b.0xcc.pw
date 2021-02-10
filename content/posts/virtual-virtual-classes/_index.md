+++
title = "Virtual Virtual Classes"
date = 2021-02-09T13:09:37-05:00
draft = false
description = "or how I learned how to abuse VirtualProtect/memcpy"
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
// TestClass.h
class TestClass : public SimObject {
    typedef SimObject Parent;
public:
    // foo

    static void consoleInit();
    static void initPersistFields();

    DECLARE_CONOBJECT(TestClass);
}

// TestClass.cpp
#include "TestClass.h"

// method implementations here

IMPLEMENT_CONOBJECT(TestClass);
```
One would assume it's just as easy to add a new file, define our class, and be on our way. This is correct, but only if you have the source code. 
Unfortunately, the most source code that we have is from the IDA Decompiler, and the engine source code. We're working from a DLL, with several critical methods we need inlined. 

How, exactly, do we go about this?

First, we need to talk about DLLs.

## A Primer On DLLs
A DLL, or a Dynamic Link Library, is the Windows-equivalent of a `.so` object on Linux. We can either:
* modify the PE header to have our DLL loaded before the program gains control

or

* inject the DLL at runtime, causing our code to be loaded into memory.

This allows us to inject code, modify execution flow, etc. Once we have a DLL injected, we have virtually infinite control over the program. 

## Beating the AbstractClassRep engine at instantiation
Great! Using that primer, we've built our DLL and injected it in. Within our `init`, we call
```cpp
AbstractClassRep::registerClassRep(TestClass::getStaticClassRep());
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

To solve this? We just add our DLL to the PE stub, and don't inject it manually. 

## PE Header Modification
This is the simplest part. To modify the PE header, we use a tool such as CFF Explorer, and simply patch the imports to add our DLL:
{{< figure src="pe_modify.jpg" >}}

Now we're running before `AbstractClassRep::initialize` is called.
Unnnnfortunately, we're now crashing. It turns out that modifying an engine variable before the engine has even initialized isn't such a good idea. Now what? 


## polyhook & Chill

[`polyhook`](https://github.com/stevemk14ebr/PolyHook_2_0) is a "x86/x64 C++ Hooking Library" which easily allows us to intercept function calls. 

We use this library to hook `AbstractClassRep::initialize` so we can register our class rep as early as possible, but not *too early*. 

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

## Don't Copy Those Vtables!
Wonderful! We've registered our AbstractClassRep, we've had `consoleInit` and `initPersistFields` called, and we're now ready to create our object.
We'll just...
{{< figure src="wtf.jpeg" caption="!?!?!?!?!" >}}

...

The funny thing is... we're missing core functionality, and we're missing a *ton* of functions that the game depends on. My solution? Stubs.

{{< figure src="https://f.0xcc.pw/palo-alto/YdLbWPYtf0jC.png" caption="Stubs stubs stubs!" >}}

Great. We've populated the vtable, so we shouldn't crash... aaaaand....
{{< figure src="wtf.jpeg" caption="AGAIN??????" >}}

It turns out that you have to "populate" these stubs in order to restore functionality. Thankfully for us, finding the SimObject vtable is simple - IDA even handily labels it for you!

Now, *sigh*, we have to overwrite the vtable of our class to fix the vtable pointers. Thankfully, vtables are just copied around, so once we patch it once, we'll never need to patch it again.
```cpp
TestClass::TestClass()
{
	fixVTable();
}

static int simObjectVFTSize = (17 * sizeof(void*));
static bool vftFixed = false;
void TestClass::fixVTable()
{
	if (vftFixed) return;
	Printf("Fixing vTable");

	// vtables are always located in .RDATA, so we need to patch them to enable R/W
	void* ourVtable = *(void**)this;
	DWORD oldProtection;
	VirtualProtect(ourVtable, simObjectVFTSize, PAGE_READWRITE, &oldProtection);
	// Ignore getClassRep() and our dtor, only copy the rest
	// If you want to override any other functions, you *need* to do it here and restore them
	memcpy((void*)((DWORD)ourVtable + 0x8), (void*)((DWORD)pSimObjectVTable + 0x8), 15 * sizeof(void*));
	VirtualProtect(ourVtable, simObjectVFTSize, oldProtection, &oldProtection);
	vftFixed = true;
}
```
And with these changes...
{{< figure src="dump_works.jpg" caption="Success!" >}}

Great! But how do we make this class usable and add methods / variables?
## Adding Methods
Adding methods is the simplest - you just have to call your `RegisterMethod` function from `TestClass::consoleInit()`. By now, `AbstractClassRep::initialize` has created you a custom namespace -- we use this to register functions.
```cpp
ConsoleMethod(void, testMethod)
{
	TestClass* o = (TestClass*)obj;
	Printf("Member method called");
	Printf("%d", o->testInt);
	/* Walk the class list and print it out as a proof of concept */
	for (AbstractClassRep* walk = AbstractClassRep::getClassList(); walk; walk = walk->nextClass)
	{
		Printf("%s", walk->mNamespace->mName);
		Printf("%s", walk->mClassName);
	}
}

void TestClass::consoleInit()
{
	Namespace* ns = TestClass::getStaticClassRep()->mNamespace;
	RegisterInternalMethod(ns, testMethod, "() - Test method", 2, 2);
}
```
Simple enough -- no mess, no drama.
{{< figure src="https://f.0xcc.pw/palo-alto/1kVFKgfTSgsc.png" caption="Success!" >}}

But now we want to add custom fields.

## Calling Convention Hell (Adding Fields)
Unfortunately for us, the compiler has decided to convert *every* single field function into a `__fastcall` -- some even beyond `__fastcall` and straight up breaking specification.

As such... to call the main function to add a field (`ConsoleObject::addField`), you have to call it in this manner:
```cpp
void __fastcall ConsoleObject::addField(const char* in_pFieldname,
    const U32     in_fieldType,
    const size_t in_fieldOffset,
    const U32     in_elementCount,
    EnumTable* in_table,
    const char* in_pFieldDocs)
{
	/*
	 * This function doesn't fix the stack (naked call?)
	 * We need to fix ESP so we don't mess anything up   
	 */
	__asm {
        push in_pFieldDocs;
        push in_table;
        push in_elementCount;
        push in_fieldOffset;
        mov edx, in_fieldType;
        mov ecx, in_pFieldname;
        call ConsoleObjectAddField;
        add esp, 10h;
	}
}
```
MSVC, wtf?

## Fin
After that fix, we just simply declare our member variables in `TestClass::initPersistFields()`
```cpp
void TestClass::initPersistFields()
{
	addGroup("Test");
	addField("test", TypeInt, Offset(testInt, TestClass), 1, 0, "Test");
	endGroup("Test");
}
```
aaand...

{{< figure src="both_work.jpg" caption="Success!" >}}

Sweet. Everything works.

----------------

This has been a really fun project and a good waste of a few days, and I can't say I didn't enjoy it.
{{< video src="https://f.0xcc.pw/palo-alto/B9j1l4rSQuzS.mp4" width="100%" height="auto" caption="My IDA names window after finishing this project">}}
All credits to Val for blazing the way and giving me the idea. 


