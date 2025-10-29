---
layout: single
title:  "The Chronicles of Virtual Table Table (a.k.a VTT)"
date: 2025-10-03
classes: wide
tags:
  - Reverse Engineering
  - Linux
  - RE Writeup
---


In this article, we're going to see how virtual inheritance works under the hood. To understand it, we will need to go deeper into the `Itanium C++ ABI`. We’ll reverse its hidden structures and watch how it keeps diamond inheritance from turning into a havoc. Lets start!

To get started, first lets get familiar, deeply familiar with a VTT.

### And... What exactly is a VTT?
Virtual Table Table, VTT, as defined in the Itanium C++ ABI specification:
>"An array of virtual table addresses, called the _**VTT**_, is declared for each class type that has indirect or direct virtual base classes. 

We'll take the following example from the Itanium ABI to explain the VTT:
```cpp
  class A1 { int i; };
  class A2 { int i; virtual void f(){} };
  class V1 : public A1, public A2 { int i; };
	// A2 is primary base of V1, A1 is non-polymorphic
  class B1 { int i; };
  class B2 { int i; };
  class V2 : public B1, public B2, public virtual V1 { int i; };
	// V2 has no primary base, V1 is secondary base
  class V3 {virtual void g(){} };
  class C1 : public virtual V1 { int i; };
	// C1 has no primary base, V1 is secondary base
  class C2 : public virtual V3, public virtual V2 { int i; };
	// C2 has V3 primary (nearly-empty virtual) base, V2 is secondary base
  class X1 { int i; };
  class C3 : public X1 { int i; };
  class D : public C1, public C2, public C3 { int i;  };
```
which gives us an inheritance structure like...
![diagram](/assets/images/Pasted image 20250930231334.png){: style="display:block; margin:auto; width:600px;" }
Lets take a look at the VTT created for this example by IDA.
![diagram](/assets/images/Pasted image 20250929134605.png)

### Dissecting VTT's layout...
It has a number of entries... Lets crack each entry. According to the VTT layout as described by Itanium ABI, 
> The elements of the VTT array for a class `D` are in this order:
>
> 1- **Primary virtual pointer**: address of the primary virtual table for the complete object `D`.
> 
> 2- **Secondary VTTs**: for each direct non-virtual proper base class `B` of `D` that requires a VTT, in declaration order, a sub-VTT for `B-in-D`, structured like the main VTT for `B`, with a primary virtual pointer, secondary VTTs, and secondary virtual pointers, but without virtual VTTs.  
> **NOTE**: This construction is applied recursively.
> 
> 3- **Secondary virtual pointers**: for each base class `X` which (a) has virtual bases or is reachable along a virtual path from `D`, and (b) is not a non-virtual primary base, the address of the virtual table for `X-in-D` or an appropriate construction virtual table.  
> 
> 4- **Virtual VTTs**: For each proper virtual base class in inheritance graph preorder, construct a sub-VTT as in (2) above.  
> **NOTE**: The virtual VTT addresses come last because they are only passed to the virtual base class constructors for the complete object.

Lets see, if the VTT we got from IDA, really does conform to these rules or not. The first entry in our VTT `unk_3A18`should point to the primary virtual table of our object.
![diagram](/assets/images/Pasted image 20250929135309.png)

and we see, it does bring us to the vtable for `D`. 
After our first entry for the primary virtual table for the complete object, now come the Sub-VTT, which are actually construction tables. 

### what on earth is a construction table?
There is a problem, and that is that a normal virtual table for a base class may not have the right information like virtual base offsets which are needed to access virtual bases of the complete object, because the offsets of a virtual base can of course be different between a base class object and the same base class when it's part of a derived class. And to resolve this, a **construction virtual table** is created, which is our Sub-VTT, which contains the necessary information that makes sure that virtual calls and RTTI queries resolve correctly during the construction and destruction of base class subobjects. 

When a base class subobject is being created, the object temporarily behaves like that specific base class even if its not, and as a result, RTTI queries and virtual calls within the base class constructor will resolve to the base class's functions, and not the complete derived class's functions. Enough on why we need Sub-VTT's, lets move forward now..

Now we see, that `D` inherits from `C1`, `C2` and `C3` in the following order:
```cpp
  class D : public C1, public C2, public C3 { int i;  };
```
The specification says:
> For each direct non-virtual proper base class **B** of D that requires a VTT, add a sub-VTT for B-in-D.

`C1` is a direct proper base of `D` and has a virtual base (`V1`), so `C1` requires a VTT. Therefore, we see `D`'s VTT having an entry for `C1-in-D`.
![diagram](/assets/images/Pasted image 20250929140022.png)

and it brings us right inside our `C1-in-D` construction vtable, at an offset right below our typeinfo for `C1`. So till now, we have a VTT like...

![diagram](/assets/images/Pasted image 20250930234436.png)

Lets jump to the third entry (actually 2nd since we're dealing with 0 based indexing) at `0x3A98` (`off_3B20`) now which brings us inside our `C1-in-D` again...
![diagram](/assets/images/Pasted image 20250929141249.png)

This introduces us to the **secondary vptr**.

### Meet the Secondary vptrs
 In the specification, it says...
> For each base class X which (a) has virtual bases or is reachable along a virtual path, and (b) is not a non-virtual primary base, include the address of X-in-D.

... a secondary vptr needs to be created.
and what is a virtual path...
>X is reachable along a virtual path from D if there exists a path X, B1, B2, ..., BN, D in the inheritance graph such that at least one of X, B1, B2, ..., or BN is a virtual base class.

Here, `V1` is a **virtual base** of `C1`, so `C1`’s sub-VTT contains a **secondary virtual pointer** for `V1-in-C1 in D`. Lets demangle it, and get the name for the entry at offset `off_3B20`. 
```
off_3B20        dq offset A2::f(void)
```
Recall, that `V1` inherited from both `A1` and `A2`....
```cpp
  class V1 : public A1, public A2;
```
... and `A2` is the primary base of `V1` because `A2` is a polymorphic class (has virtual functions) and is inherited non-virtually at offset 0 in `V1`'s object layout. Since `V1` places `A2` at the beginning of its memory layout, `V1`'s vptr shares the same location as `A2`'s vptr, allowing them to use the same vtable, and this is why `A2::f()` appears as the first virtual function entry in `V1`'s vtable. `A2` occupies the primary base position at offset 0, making it `V1`'s primary base class.

![diagram](/assets/images/Pasted image 20251001003617.png)

Till now in the VTT, we’ve set up the primary vptr that points to `D`'s vtable at [0] and then **Sub VTT** or **construction table** for `C1-in-D`:
- The first slot [1] points to the construction vtable for the `C1` part of `D`.
- The second slot [2] points to the secondary vptr for `V1-in-C1-in-D`.

### Down the Rabbit Hole of it all...
Lets deep dive in it so that the reason of creating a VTT becomes clearer... During `D`'s construction, `D`'s constructor first constructs virtual base `V1` directly, and then constructs `C1` subobject. `C1`'s constructor gets a pointer to `C1-in-D` (`D_VTT`[1]) which points to the sub-VTT for `C1-in-D`. `C1`'s constructor uses `D_VTT`[1] to set its vptr and then uses `D_VTT`[2] which is `V1-in-C1 in D` secondary vptr. If we take a look at the disassembly of `C1`'s constructor:

```asm 
push    rbp
mov     rbp, rsp    ;function prologue
mov     [rbp+this], rdi
mov     [rbp+C_in_D], rsi
mov     rax, [rbp+C_in_D]    ;rax = C_in_D pointer
mov     rdx, [rax]           ;rdx = qword at C_in_D Sub-VTT
mov     rax, [rbp+this]      ;rax = this
mov     [rax], rdx           ;*(this) = primary_vptr
mov     rax, [rbp+this]      ;rax = this
mov     rax, [rax]           ;rax = value stored in object (C-in-D)
sub     rax, 18h             ;rdx = -24 offset from where we currently are
```

So if we remember, we landed at `0x3B08` following our `C1-in-D` entry in the VTT. Our `rax` so far stores the address `0x3B08`, and when we go back 24 bytes, we reach at `0x3AF0`, where we have `28h`...

```asm
mov     rax, [rax]           ;rax = qword located at C-in-D - 0x18 = 28h
mov     rdx, rax             ;rdx=rax so rdx=0x28
mov     rax, [rbp+this]      ;rax=this
add     rdx, rax             ;move forward 28h from where we currently are
mov     rax, [rbp+C_in_D]    ;rax=C-in-D
mov     rax, [rax+8]         ;Reach the entry V1-in-C1 in D in Sub-VTT 
mov     [rdx], rax           ;move V1-in-C1 at offset 0x28 from actual this.
pop     rbp
retn
```

Now our `C1` has correct virtual base offsets for the `C1-in-D` context and it resets the shared virtual base `V1`'s vptr to ensure correct virtual dispatch during `C1`'s construction phase...  `V1`'s vptr will be updated again by subsequent constructors (`C2`, then `D`).

Lets move to our next entry now.
![diagram](/assets/images/Pasted image 20251001022954.png)

our fourth (technically third) entry at `0x3AA0` in the VTT (`off_3B58`) brings us to `C2-in-D`.
![diagram](/assets/images/Pasted image 20250929142050.png)

We know why it's here.. Because we need to create a sub-VTT `B-in-D` for each direct non-virtual proper base class `B` of `D`. `D` inherits directly from `C1`, is a proper base, and has a virtual base `V1`, so we create `C2-in-D`. 
![diagram](/assets/images/Pasted image 20251001022621.png)

Now at our fifth entry (technically fourth), we have `off_3B58` which brings us to...
![diagram](/assets/images/Pasted image 20250929173955.png)

We've already seen that `off_3B58` is inside our construction table `C2-in-D`. It points at `V3::g(void)`, means `V3` is our primary base for `C2`, so `V3-in-C2` is a primary vptr (look into this). This is because we can clearly see that `C2` is sharing its virtual table with `V3`. Our entry for Sub-VTT `C2-in-D` was also at `off_3B58`, and the entry immediately after it in the main VTT is also `off_3B58`, so we can safely conclude that `V3-in-D` is our primary vptr. Also, according to the Itanium ABI, a primary class is..
> If C has a dynamic base class, attempt to choose a primary base class B. It is the first (in direct base class order) non-virtual dynamic base class, if one exists. Otherwise, it is a nearly empty virtual base class, the first one in (preorder) inheritance graph order which is not an indirect primary base class if any exist, or just the first one if they are all indirect primaries.

A nearly empty virtual base class is...
> A class that contains a virtual pointer, but no other data except (possibly) virtual bases. In particular, it:
>- has no non-static data members and no non-zero-width unnamed bit-fields,
>- has no direct base classes that are not either empty, nearly empty, or virtual,
>- has at most one non-virtual, nearly empty direct base class, and
>- has no proper base class that is empty, not morally virtual, and at an offset other than zero.
Such classes may be primary base classes even if virtual, sharing a virtual pointer with the derived class.

`V3` completes the checklist for a nearly empty class. If we see the preorder traversal for our inheritance hierarchy, it comes as...
>_inheritance graph order_
The ordering on a class object and all its subobjects obtained by a depth-first traversal of its inheritance graph, from the most-derived class object to base objects, where:
> - No node is visited more than once. (So, a virtual base subobject, and all of its base subobjects, will be visited only once.) 
>- The subobjects of a node are visited in the order in which they were declared. (So, given  `class A : public B, public C`, A is walked first, then B and its subobjects, and then C and its subobjects.)
Note that the traversal may be preorder or postorder. Unless otherwise specified, preorder (derived classes before their bases) is intended...

Since we are supposed to do a depth first traversal from the most derived class object to base objects, so we turn our graph upside down, unlike the trees/graphs in basic data structures where we start from our root node which is usually the parent and go down to its children nodes. And all the remaining preorder depth first traversal rules are applied.
![diagram](/assets/images/Pasted image 20251001024747.png)

The preorder traversal for this graph would be:
```
D -> C1 -> V1 -> A1 -> A2 -> C2 -> V3 -> V2 -> B1 -> B2 -> C3 -> X1
```
We see, that `V3` also comes before `V2` in the preorder, so now we can safely conclude that `V3` is the primary base for `C2`... So now...
![diagram](/assets/images/Pasted image 20251001025338.png){: style="display:block; margin:auto;" }

Lets jump to our sixth entry (technically 5th)... 

![diagram](/assets/images/Pasted image 20250929185933.png){: style="display:block; margin:auto;" }

It points at `unk_3B78`. 
Rule 2 from the specification said that for each direct non-virtual proper base class `B` of `D` that requires a VTT, in declaration order, a sub-VTT for `B-in-D`, structured like the main VTT for `B`, with a primary virtual pointer, secondary VTTs - stop right there! We've seen `C2`'s primary virtual pointer, but we aren't going to see any secondary VTT here, because `C2` has no direct non-virtual proper base class as stated in the _rules_. But we might have secondary virtual pointers here...We know according to rule 3 that we have a secondary virtual pointers  for each base class `X` which... 
(a) has virtual bases or is reachable along a virtual path from `D`, and 
(b) is not a non-virtual primary base.

`C2` inherits virtually from 2 base classes...
```cpp
  class C2 : public virtual V3, public virtual V2 { int i; };
```
...`V3` and `V2`. `V3` won't get a secondary vptr, since it's our primary base.. Moving on to `V2`, it does have a virtual base, and it is not a non-virtual primary base for `C1`, so it gets a secondary vptr. If we follow `unk_3B78`, it brings us to the entry `V2-in-C2 in D`.
![diagram](/assets/images/Pasted image 20250929202909.png)

This is where it brings us, right at `off_3B78`, below the typeinfo which we see in Construction table `C2-in-D` for the second time, and above it is a negative offset which is a `vcall offset` for virtual inheritance adjustments. 
![diagram](/assets/images/Pasted image 20251001030124.png){: style="display:block; margin:auto;" }

But our secondary vptrs don't actually stop here, because `V2` inherits from `V1`. This brings us to the 7th (technically 6th) VTT entry, which makes us land at `off_3B90`.
![diagram](/assets/images/Pasted image 20250929203420.png)

and it takes us to `V1-in-C2`. 
![diagram](/assets/images/Pasted image 20250929203623.png)

We see `A2::f(void)`, well that's because `V1` inherits from `A2` and `A2` has a virtual function `f(void)`. `V1` gets a secondary vptr because firstly it's reachable along a virtual path, and secondly it's not a non-virtual primary base of `C2`. If we go up the inheritance hierarchy, we don't stop at `V1` alone, but go upto `A1` and `A2` classes too, since `V1` inherits from them.. not only this, but `V2` also inherits from `B1` and `B2`, but we're not going to have secondary vptrs for them..

 `A1` is direct base of `V1`. It doesn't have virtual bases, but it is reachable along a virtual path from `D`.  Also, it is not a non-virtual primary base. So it completes all the requirements for having a secondary vptr, but `A1` itself is non-polymorphic (no virtuals at all).  So `A1` does not even need a vptr and hence no entry in the VTT.  

 For`A2`, its  a base of `V1` and is polymorphic. It is also reachable along a virtual path. But! we know that _primary non-virtual bases don’t get secondary vptrs_.  And inside `V1`, `A2` is the primary base (polymorphic and also comes first in preorder), so no entry is going to be created for that. 
As far as `B1` and `B2` are concerned, they are non virtual and non-polymorphic, so no entry! 

![diagram](/assets/images/Pasted image 20251001031112.png){: style="display:block; margin:auto;" }

Our third base `D` inherits from is `C3`. And `C3` doesn't even get a Sub-VTT, because it has no direct or indirect virtual bases.

### More secondary vptrs
Now since we are done with Sub-VTT's for our main VTT, next come the secondary virtual pointers. You might have noticed, We had followed the same layout for our Sub-VTT's, as we have been doing for the main VTT. This is what we meant by being recursive in point 2, with the exception of Virtual VTT's (Virtual virtual table table, good heavens), which we'll later see. 
![diagram](/assets/images/Pasted image 20250929210149.png)

Now we are at the 8th entry (technically 7th), which takes us to... 
![diagram](/assets/images/Pasted image 20250929210336.png)

We see that it's inside secondary vtable in `D`'s vtable. It's `V1-in-C2 in D`. Actually (`V1` is a not a non-virtual primary base and it is reachable along a virtual path).... One thing to remember, is that we can't have a secondary vptr to `C1-in-D`, because it's the primary base for `D`. 
![diagram](/assets/images/Pasted image 20251001032017.png){: style="display:block; margin:auto;" }

Later comes entry at `off_3A48` which is our 9th (technically 8th) entry...
![diagram](/assets/images/Pasted image 20250929210635.png)

which brings us here.....

![diagram](/assets/images/Pasted image 20250929210549.png)

This is `C2-in-D`.. `C2` is not reachable via a virtual path, but it does have virtual bases, and it is not the non-virtual primary base for `D`, so it gets an entry. The `V3::g(void)` we see here, is the virtual function which was declared and defined in `V3`, which is inherited by `C2`.     
Jumping on to our 10th (technically 9th) VTT entry which as we can see in the above screenshot, is again `off_3A48`. This is our `V3-in-D`. We see that the `V3-in-D` entry will be the same as the `C2-in-D` entry. Again, `V3` is reachable along a virtual path and is definitely not `D`'s primary non-virtual base.

![diagram](/assets/images/Pasted image 20251001034949.png){: style="display:block; margin:auto;" }

Now onto our 11th (technically) VTT entry... which should theoretically be a secondary vptr to `V2-in-D`, but that's missing in the VTT compiler generated for my binary. The reason can possibly be a compiler optimization, as we'll soon see, where the Virtual VTT entry [11] serves the purpose that a `V2-in-D` secondary virtual pointer would have served. Now since we're done with our secondary vptrs, we'll move on to our fourth point, which as I just mentioned, is the virtual VTT.

### The unsung heroes: Virtual VTT 
>_Virtual VTTs_: For each proper virtual base classes in inheritance graph preorder, construct a sub-VTT as in (2) above.

The Virtual VTT's are required to be constructed just as we construced our Sub-VTT's. Their addresses come at the end of the VTT because they are only passed to the virtual base class constructors for the complete object. According to the specification Rule 3:
>_A virtual base with no virtual bases of its own does not require a VTT, but does require a virtual pointer entry in the VTT._

This means, a class needs a VTT when it has to deal with virtual base class construction, which requires multiple vtable addresses at different stages of construction, otherwise it's not needed. So only those of our virtual bases classes will need a virtual VTT, which themselves have virtual bases. 
The virtual bases we have are `V1`, `V2` and `V3`. From these, only `V2` is a virtual base which itself is inheriting virtually from `V1`, hence it'll get a virtual VTT entry in the main VTT. Moving forward with our VTT in IDA:
![diagram](/assets/images/Pasted image 20250930002213.png)

following the offset `unk_3BB0` brings us to....

![diagram](/assets/images/Pasted image 20250930002303.png)

which indeed is our Virtual VTT. 
Again applying the same rules here... For Virtual VTT of `V2-in-D`, we have our primary vptr (look into this)pointing to offset `0x3BB0`. Which checks our rule 1, that the first entry needs to hold the address of the primary virtual table... For rule 2, since we don't have any direct non-virtual proper base class `B` of `D` that requires a VTT, we won't create any. For rule 3, of having secondary vptrs, since `V1` is reachable along a virtual path, and is not the primary base for `V2`, because neither is it non-virtual, nor is it nearly empty because of its data member `i` which is an integer, so we are going to have a `V1-in-V2 in D` entry, and we aren't going to have a virtual VTT for it.. Lets take a look at the final entry of our main VTT in IDA for this:

![diagram](/assets/images/Pasted image 20250930003457.png)

going over to `off_3BC8`:

![diagram](/assets/images/Pasted image 20250930003553.png)

We reach inside our construction vtable for `V2-in-D`. Why can we see `A2:f(void)`, well that is because `A2` has a virtual function `f(void)`, which gets inherited by `V1`. 
![diagram](/assets/images/Pasted image 20251001035732.png){: style="display:block; margin:auto;" }

#### Had the A's tried to be sneaky..
Had either `A1` or `A2` been virtual bases, we would've had some more entries in our VTT. We then would've had another Virtual VTT for `V1-in-D`, more secondary vptrs etc. 

### Making the diamond behave...
Now lets take a look at how we solve the diamond problem using virtual inheritance. This is the same example used in [[Parents & Children in C++]], with one difference, that here we've used the virtual keyword while inheriting from class `A`.
```cpp
class A {
public:
    int var_a;
    
    virtual void foo() { }
};
class B : virtual public A {
public:
    int var_b;
    
    virtual void baz() { }
};
class C : virtual public A {
public:
    int var_c;
    
    virtual void bar() { }
    void foo() { }
};
class D : public B, public C {
public:
    int var_d;
    
    virtual void qux() { }
};
int main() {
    D obj;
    return 0;
}
```
Firstly our object `obj` resides at `0x7fffffffde50` on the stack. 

![diagram](/assets/images/Pasted image 20250930015559.png)

From main we call our `D`'s constructor...

```asm 
mov     rdi, rax        ; this
call    D::D(void)
```

and then from `D`, we invoke `A`'s constructor first of all with an addition of `0x20` to our object's starting address, rather than of `D`'s base class B and C, thanks to virtual inheritance.

```asm 
mov     rax, [rbp+this]
add     rax, 20h ; ' '
mov     rdi, rax        ; this
call    A::A(void)
```

and then inside `A`'s constructor, `obj` stores the address of our `A`'s vtable (grandparent class) where our this pointer is, at `0x7fffffffde70`.

```asm
mov     [rbp+this], rdi
lea     rdx, off_3D18
mov     rax, [rbp+this]
mov     [rax], rdx
```

Seeing our vtable in IDA

![diagram](/assets/images/Pasted image 20250930005218.png)

This makes our object look like..

![diagram](/assets/images/Pasted image 20250930015800.png)

Then on returning from `A::A()`:

```asm
call    A::A(void)
mov     rax, [rbp+var_8]
lea     rdx, off_3C50
mov     rsi, rdx
mov     rdi, rax        ; this
call    B::B(void)
```

We see that `rdi` has the address of our object `obj`  at `0x7fffffffde50`, and we set `rsi` as `0x555555557c50`.

![diagram](/assets/images/Pasted image 20250930005658.png)

If we analyze the contents at `rsi`, we see that..

![diagram](/assets/images/Pasted image 20250930005918.png)

The constructor for the complete class `D`, passes to its proper base class  `B` constructor a pointer to the appropriate place in the VTT where the proper base class B constructor can find its set of virtual tables. Here's where VTT has stepped in, lets take a look at how it looks like:

![diagram](/assets/images/Pasted image 20250930034238.png)

-  The first entry at `0x3C48`, holds a pointer to the address of the primary virtual table for the complete object `D`.
-  The second entry at `0x3C50`, holds our sub-VTT, `B-in-D`. `B` inherits virtually from `A`, so it requires a VTT of its own then, so at `0x3C50`, is our `B-in-D`'s construction table.
- The third entry is basically our secondary vptr in the construction table `B-in-D`. As we'll see ahead, it'll be holding a reference to `A::foo(void)`. `A-in-B in D` gets to be our secondary vptr because firstly, it's reachable through a virtual path, and secondly, it's not a primary base for B. This is because it is virtual and it also not nearly empty because of the presence of `    int var_a`. 
- The fourth entry is our `C-in-D` Sub-VTT. It is needed, because `C` has virtual bases. 
- The fifth entry is our secondary vptr for the construction table `C-in-D`. We'll too see this ahead that it holds a reference to a thunk for `C::foo()`, because `C` is overriding `A`'s `foo()`. Again it's needed, because all the conditions are satisfied as we've discussed above, to make `A-in-C in D` our secondary vptr. (check overriding)
- The sixth entry is again a pointer to our `thunk C::foo()`, consider it `A::foo()`, overridden by `C::foo()`, which is actually true. 
- The final seventh entry is our `C-in-D` secondary vptr.
We don't notice `B` anywhere in our secondary vptr list, well that's because `B` is `D`'s primary base, since it's non-virtual and comes first in the inheritance order.

Lets dig into the disassembly. We are sending the address of `B-in-D` construction table as our argument to `B`'s constructor. Below is the disassembly for `B`'s constructor.  

```asm
mov     [rbp+this], rdi
mov     [rbp+vtable_D], rsi
mov     rax, [rbp+vtable_D]
mov     rdx, [rax]
mov     rax, [rbp+this]
mov     [rax], rdx
```

We see that it stores the address where `[rbp+vtable_D]` points to at the address of our object stored in `rax`. This makes our object look like...

![diagram](/assets/images/Pasted image 20250930022644.png)

where the address at `0x7fffffffde50` is the address of our construction table `B-in-D`. Moving on with our `B`'s constructor..

```asm
mov     rax, [rbp+this]
mov     rax, [rax]
sub     rax, 18h
mov     rax, [rax]
mov     rdx, rax
```

We notice we're getting the address of our sub VTT `B-in-D`, going back 24 bytes, and storing whatever we get in `rdx`.

![diagram](/assets/images/Pasted image 20250930023954.png)

We see that it has taken us to the very start of the construction table... 

![diagram](/assets/images/Pasted image 20250930024134.png)

We pick the `20h` from here, and...

```asm
mov     rax, [rbp+this]
add     rdx, rax
mov     rax, [rbp+VTT_D]
mov     rax, [rax+8]
mov     [rdx], rax
```

Add it to our this pointer, which takes us to `0x7fffffffde70` again..  And then we add to `off_3C50`, which was our pointer to construction table `B-in-D`, an offset of 8, which bring us to `0x3C58`..

![diagram](/assets/images/Pasted image 20250930024736.png)

Which is where `A::foo(void)` is at... We've replaced the memory where `A`'s constructor had previously written its vtable's address, with the address of `A` in sub-VTT `B-in-D`, making our object now look like...

![diagram](/assets/images/Pasted image 20250930025003.png)

Great! Now lets return back to `D`'s constructor.

```asm
mov     rax, [rbp+var_8]
add     rax, 10h
lea     rdx, off_3C60
mov     rsi, rdx
mov     rdi, rax        ; this
call    C::C(void)
```

now we've added `0x10` to our object's address, which brings us to `0x7fffffffde60`, and load the address at `off_3C60` in `rdx`, which probably now would be our pointer to construction table `C-in-D` inside our VTT for D, since next we're calling `C`'s constructor. 

![diagram](/assets/images/Pasted image 20250930025309.png)

it's the 4th entry in our VTT, and it does take us to...

![diagram](/assets/images/Pasted image 20250930025349.png)

Lets also confirm this stuff in GDB:

![diagram](/assets/images/Pasted image 20250930025615.png)

This is the address of the first virtual function in `C`. Great, lets dig into `C()`'s disassembly.

```asm
mov     [rbp+this], rdi
mov     [rbp+VTT_D], rsi
mov     rax, [rbp+VTT_D]
mov     rdx, [rax]
mov     rax, [rbp+this]
mov     [rax], rdx
```

We're simply storing the address of our construction table `C-in-D` in our object. 

```asm
mov     rax, [rbp+this]
mov     rax, [rax]
sub     rax, 18h
mov     rax, [rax]
mov     rdx, rax
```

And now, we're simply getting the contents at where our this pointer points to right now, which is `0x7fffffffde60`, and subtracting 24 again (`0x18`)... The memory contents at `0x7fffffffde60` point to `off_3CD8`, which we earlier saw is an offset where our `C::bar(void)` is, and subtracting `0x18` from it brings us to `0x3CC0`, which is right the start of our construction table.

![diagram](/assets/images/Pasted image 20250930030438.png)

So we're basically get the 16 from here, `0x10`, which is our offset to `A`'s part in our object from `C`.

![diagram](/assets/images/Pasted image 20250930030542.png)

we just got a confirmation from GDB. Lets continue with the disassembly..

```asm
mov     rax, [rbp+this]
add     rdx, rax
```

We add the offset to our this pointer... landing at.. 

![diagram](/assets/images/Pasted image 20250930030739.png)

and then...

```asm
mov     rax, [rbp+VTT_D]
mov     rax, [rax+8]
mov     [rdx], rax
```

We're simply rewriting some address, at an offset of `+0x08` from our construction table `C-in-D`, at the area `A` entries have occupied till now..

![diagram](/assets/images/Pasted image 20250930031038.png)

It brings us to `0x3C68`, which holds a pointer to...

![diagram](/assets/images/Pasted image 20250930031200.png)

It's simply a thunk to `C::foo(void)`, which was initially `A::foo(void)`, overridden by `C`. So our object now looks like...

![diagram](/assets/images/Pasted image 20250930031457.png)

Where we have two new entries, `0x0000555555557cd8` which is our `C::bar(void)`, inside our sub-VTT C-in-D, and `0x0000555555557d00`, which is the thunk to `C::foo(void)` we just saw. Lets return to `D`'s constructor now, where `D` things will come into action.

```asm
lea     rdx, off_3BF0
mov     rax, [rbp+var_8]
mov     [rax], rdx
```

At `off_3BF0`, we have..

![diagram](/assets/images/Pasted image 20250930032055.png)

and we're simply replacing our old entry `0x0000555555557c98`, which we've just seen in the GDB dump, which was actually the address of the construction table of `B-in-D` which further pointed to `B::baz(void)`, with the entry of `B::baz(void)` in our vtable for `D`. This also shows us, that `B` is the primary base class for `D`, since it shares the virtual table slot with it at offset 0. So now our object seems something like...

![diagram](/assets/images/Pasted image 20250930032249.png)

with a new entry at the start of the object... moving further..

```asm
mov     rax, [rbp+this]
add     rax, 20h ; ' '
lea     rdx, off_3C40
mov     [rax], rdx
```

We now jump to `0x7fffffffde70` by adding `0x20` to our object's starting address, get some stuff at `off_3C40`, and dump it in our object.

![diagram](/assets/images/Pasted image 20250930032500.png)

which is actually again our thunk to `C::foo(void)`, but this time in our vtable for `D`, unlike before where it was in `C-in-D` construction table. 

![diagram](/assets/images/Pasted image 20250930032649.png)

and now we see a new address, address to our thunk in `D`'s vtable, at `0x7fffffffde70`. Continuing with our disassembly...

```asm
lea     rdx, off_3C18
mov     rax, [rbp+this]
mov     [rax+10h], rdx
```

lets go to `off_3C18`

![diagram](/assets/images/Pasted image 20250930032818.png)

which is again, inside our `D`'s vtable, making our object giving a final look like:

![diagram](/assets/images/Pasted image 20250930032909.png)

So now we've how virtual inheritance is taming classes... but before wrapping up, lets take a look at how each of the typeinfo for our classes would look like..

### typeinfo, not your average table...
Firstly if we see our typeinfo for class `A`:

![diagram](/assets/images/Pasted image 20250930161323.png)

We see the type is `__class_type_info`, and below that we have our `char *__name` which gets inherited by `__class_type_info` from `__type_info` as we've already seen in [Parents and Children in C++](https://mahamaryam.github.io/Parents-and-Children-in-C++/), and then we have the typeinfo name for `A`. Pretty straightforward... Lets see our typeinfo for `B`.

![diagram](/assets/images/Pasted image 20250930162037.png)

Since our class `B` inherits virtually from class `A`, so we see the our typeinfo object type is `__vmi_class_type_info`, and its vptr points to the vtable of `__vmi_class_type_info`(as can be seen at `0x3D80`) (see [The Virtual Functions](https://mahamaryam.github.io/The-Virtual-Functions/)). The `+0x10` we see, skips the RTTI pointer in the vtable itself. This means, our object is going to further hold of course, the typeinfo name, as `__vmi_class_type_info` inherits from `__class_type_info` which further inherits the `*__name` from `__type_info` and passes it down to its children, as well as `unsigned in __flags` which carries the details about the class hierarchy, `unsigned int __base_count`, which tells us the number of direct bases, and `__base_class_type_info __base_info[1]` (add this to virtual functions) -- which actually uses a technique called _struct hack_, to create a variable length array at the end of the class. Despite being declared as an array of size `[1]`, this member actually is supposed to hold a variable number of base class entries, which is determined by the number of elements as in `__base_count` --. So after our vptr, comes typeinfo name for B, and then come our `__flags` which here are 0, and then the `__base_count` which is 1. 
At offset `0x3D98`, starts our `__base_class_type_info` which was held in `__base_info[1]` which we know is a variable sized array and it holds base class entries. Since we just saw that our `__base_count` is 1, so we're going to hold only one entry inside `__base_info`, which is going to be for `A`. Lets take a look at how this structure looks like from C++ standard library.

```cpp
class __base_class_type_info
{
public:
const __class_type_info* __base_type; // Base class type.
#ifdef _GLIBCXX_LLP64
long long __offset_flags; // Offset and info.
#else
long __offset_flags; // Offset and info, we're using this.
#endif
};
```

We see that it has a `__class_type_info* __base_type` and either a long long datatype for `__offset_flags`, or long, but since we're using `LP64` where a long is 8 bytes and not `LLP64` where a long is 4 bytes (hence defined twice as long long to complete 8 bytes, 4+4), we'll be going with `long __offset_flags`. So at `0x3D98`, we have our `__class_type_info* __base_type` pointing to typeinfo for `A`, and then our `__offset_flags`. 
In our `__base_class_type_info`, we see,
```cpp
enum __offset_flags_masks
{
__virtual_mask = 0x1,
__public_mask = 0x2,
__hwm_bit = 2,
__offset_shift = 8 // Bits to shift offset.
};
```
`__hwm_bit` is used in the following way as defined in `__class_type_info..
```cpp
// Contained within us.
__contained_mask = 1 << __base_class_type_info::__hwm_bit
```
within the `__base_class_type_info`, we have
```cpp
// Implementation defined member functions.

bool __is_virtual_p() const
{ return __offset_flags & __virtual_mask; }
  
bool __is_public_p() const
{ return __offset_flags & __public_mask; }
```
Which tells whether our inheritance is virtual or public respectively. The `__offset_flags` is set by the compiler, we can't set it manually. And then...
```cpp
ptrdiff_t
__offset() const
{
// This shift, being of a signed type, is implementation
// defined. GCC implements such shifts as arithmetic, which is
// what we want.
return static_cast<ptrdiff_t>(__offset_flags) >> __offset_shift;
}
```
Here we can see that the memory offset of the base class gets extracted. So we've seen that the `__offset_flags` variable stores two things in one integer, the  low bits store the flags (virtual, public etc) and the high bits store the memory offset. So the negative offset we just saw, actually comes from here. Since our offset if `0xFFFFFFFFFFFFE8` and now we see that our flags take up only 8 bits, that is a byte, so lets decode the original `__offset_flags` we had.
`0xFFFFFFFFFFFFE803` is our `__offset_flags` which in two's complement signed bit integer we have as -6141 which doesn't really make much sense yet. We've seen that our `__offset_shift` is 8, which means, we need to shift 8 bits to the right, and sign extend the result, so from here we deduce that the lower 8 bits are actually our flags. To extract the flags...
```maths
flags = 0xFFFFFFFFFFFFE803 & 0xFF = 0x03
```
which in binary is `00000011`. Since we know that in our enum `__offset_flags_masks`, we have
```
_virtual_mask = 0x1
__public_mask = 0x2
```
So to find out whether our inheritance is virtual we do...
```
00000011 & 00000001 = 00000001
```
Which gives us 1.... Means yes, our inheritance is virtual. We can see the same thing in our function `__is_virtual_p()`.. And for public we do...
```
00000011 & 00000010 = 00000010
```
Which gives sets our `__public_mask` as well, as also shown in `__is_public_p`. This means we have a public virtual inheritance. We can see that these methods get used in our `__vmi_class_type_info`, and recall, that it was our `__vmi_class_type_info` which had..
```cpp
__base_class_type_info __base_info[1]; // Array of bases.
```
As an example, this is how it's being used in the function `__vmi_class_type_info::__do_upcast`...
```cpp
for (std::size_t i = __base_count; i--;)
{
/*some stuff*/
ptrdiff_t offset = __base_info[i].__offset ();
bool is_virtual = __base_info[i].__is_virtual_p ();
bool is_public = __base_info[i].__is_public_p ();
/*some stuff*/
}
```
Anyway, lets move on to our offset part now. Which after sign extension and 8 bit right shift leaves us with `0xFFFFFFFFFFFFFFE8` which is actually -24. While seeing the construction of our object we had noticed a -24 offset from `B` to reach `A`. This is exactly what that is.
We're done with the dissection of `B`'s typeinfo, and `C`'s typeinfo is going to be the same as `B`'s, since these are independent typeinfo objects. Lets see `D`'s typeinfo now..
![diagram](/assets/images/Pasted image 20250930191453.png)
`B`'s typeinfo is of type `__vmi_class_type_info`, and the first entry at `0x3D20` is a vtable pointer to `__vmi_class_type_info` vtable which is of 8 bytes.
Then at `0x3D28`, we have the typeinfo name for `D`. And then comes our `unsigned int __flags`, which is 4 bytes... We have seen in [The Virtual Functions](https://mahamaryam.github.io/The-Virtual-Functions/),
```cpp
enum __flags_masks
{
__non_diamond_repeat_mask = 0x1, // Distinct instance of repeated base.
__diamond_shaped_mask = 0x2, // Diamond shaped multiple inheritance.
__flags_unknown_mask = 0x10
};
```
that `__diamond_shaped_mask` holds the second bit in our `__flags_mask`. Since `D` is a diamond shaped multiple inheritance, so we have 2 in our `__flags` field... To make use of our `__flags_masks`, we simply `&` it with our `__flags`.
- Next at `0x3D34`, we have the count of base classes which is `__base_count`. 
-  Now comes the first entry in our `__base_info[0]` which is `B` base class. We're going to follow this same class for it...
```cpp
class __base_class_type_info
{
public:
const __class_type_info* __base_type; // Base class type.
long __offset_flags; // Offset and info, we're using this.
};
```
   Because we have an array of `__base_class_type_info` (`__base_class_type_info __base_info[1]`), so each index is going to hold an object of type `__base_class_type_info`. 
- At `0x3D38`, we have our typeinfo for `B`, which is our `__base_type`.
- At `0x3D40`, start our flags. Since we've simply inherited `B` publicly, so it's 2 (`__public_mask = 0x2` second bit set), and since `B` is our primary base for `D`, so the offset is 0.
- At `0x3D48`, comes `__base_info[1]`, which holds a pointer to typeinfo for `C`.
-  At `0x3D50`, comes 2, which again is for public inheritance.
-  After type of inheritance comes the offset, which in this case is `10h`, which are 16 bytes. It's the byte offset of `C`’s subobject inside `D`. Means if we take a `D*` and cast it to `C*`, the compiler will add `0x10` (16 bytes) to the pointer.
And here, we get finished with our typeinfos for all classes. 

And this is it for now.

Happy Reversing!
