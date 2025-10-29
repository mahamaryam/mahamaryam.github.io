zThe diamond problem can be fixed by using virtual inheritance. We do virtual inheritance in the following way:
```cpp
class A {
public:
    int var_a;
    
    virtual void foo()
    {
    	//some operation
    }
};
class B : virtual public A {
public:
    int var_b;
    
    virtual void bar()
    {
    	//some operation
    }
};
class C : virtual public A {
public:
    int var_c;
    
    virtual void baz()
    {
    	//some operation
    }
    void foo()
    {
    
    }
};
class D : public B, public C {
public:
    int var_d;
    
    virtual void qux()
    {
    	//some operation
    }
};
```
Here we see that both `B` and `C` inherit from `A` virtually. if we run it, we see that it calls A only once.
Lets analyze our binary using both static and dynamic analysis side by side. We've created an object on the stack as we saw earlier and call our D's constructor.  
```asm main
lea     rax, [rbp+var_40]
mov     rdi, rax        ; this
call    D::D(void)
```
The address of our object `obj_D`on the stack is `0x7fffffffde40`.
![[Pasted image 20250925010105.png]]
Once we're inside D's constructor, now we have to invoke A's constructor which is the _basest_ class, or the grandparent class. 
```asm D ctor
add     rax, 20h ; ' '
mov     rdi, rax        ; this
call    A::A(void)
```
Our rax has the address of our object on the stack. Note that we are adding 0x20 to it, resulting in address `0x7fffffffde60`.
![[Pasted image 20250925010308.png]]
lets continue..
```asm A
mov     [rbp+var_8], rdi
lea     rdx, off_3D18
mov     rax, [rbp+var_8]
mov     [rax], rdx
```
we see in the above assembly code snippet, that we're loading an address, most likely of A's vtable in rdx, which then gets positioned inside our object at `0x7fffffffde60`, making our object look something like..
![[Pasted image 20250925010452.png]]
What is at `off_3D18`? 
![[Pasted image 20250925010733.png]]
We see, it's A's vtable, and we just loaded the address of the first virtual function in A, that is foo() (it also has a single virtual function). Back to our disassembly now..
So now our object looks something like this...
![[Pasted image 20250925010910.png]]
Lets return back to our D's constructor. 
```asm D
mov     rax, [rbp+var_8]
lea     rdx, off_3C50
mov     rsi, rdx
mov     rdi, rax        ; this
call    B::B(void)
```
Once we return to D:D(), we notice that something is getting loaded at rdx from `off_3C50`, and then it's getting set as the second argument to our B:B().... lets take a look at what this can possibly be.. Double click on it, and it brings us to the VTT.. what in the world is that..
![[Pasted image 20250925011317.png]]
we've landed at `off_3C50` in VTT, but what are these other entries in the VTT, and firstly, what is VTT itself? VTT is actually, our magnificent Virtual Table Table. VTT is a table that holds virtual table pointers to ensure that they are set correctly during the construction of base classes in virtual inheritance. We can say, it's a table of pointers to virtual tables, and we'll soon see what those virtual tables actually are. What we're loading in rdx right now is
```bash
.data.rel.ro:0000000000003C50 off_3C50        dq offset off_3C98
```
if we go to `off_3C98`... it brings us to..
```bash
.data.rel.ro:0000000000003C98 off_3C98        dq offset B::baz(void) 
```
So vtt is actually holding a pointer to our B:baz(void). If we take a look in GDB:
![[Pasted image 20250925012718.png]]
our rdx is loaded with address 0x555555557c50 which is actually an address at offset +0x08 in the VTT. 
![[Pasted image 20250925012840.png]]
If we examine the memory contents at VTT+8, we get an address for construction vtable for B-in-D? what the heck is that? If we see in IDA
![[Pasted image 20250925013728.png]]
