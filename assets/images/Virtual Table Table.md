gef> break main
gef> run
the first call in main, is call to our constructor (not new because our object is created on the stack not on the heap).
![[Pasted image 20250921034511.png]]
after this we call the first ctor D. 
In D() we add 0x20 offset to rax 
![[Pasted image 20250921034559.png]]
this will be rdi for A()->de40.
now we have to load the address of vtable for A class as we are in A's ctor.
![[Pasted image 20250921034728.png]]
if we analyze the contents at the vtable for A
![[Pasted image 20250921034824.png]]
we know that it is the first virtual function in our vtable. (this will already be given in ida. first give ida view and then gdb)
![[Pasted image 20250921035026.png]]
in A we first stored the address of our vtable A inside the object to which this pointer was pointing to on the stack, and then at +8 offset we stored A's data member whose value gets set to 1 in the constructor.
![[Pasted image 20250921035246.png]]
after returning from A, we load the address of our Virtual table table in rdx. this rdx(address of VTT) will be passed onto other base constructors for D as arguments.
well lets peek into it
![[Pasted image 20250921042031.png]]
lets see what each entry in vtt holds

![[Pasted image 20250921042446.png]]
this is our vtt. it has
1- b's first vfunc address in B-in-D
2- b's first vfunc address in D vtable
3- c's first vfunc address in C-in-D
4- c's first vfunc address in D-vtable
5- a's first vfunc address in D-vtable
6- a's first vfunc address in C-in-D
7- a's first vfunc address in B-in-D
to see...
![[Pasted image 20250921042841.png]]
Structure of B-in-D
![[Pasted image 20250921042922.png]]
first is the offset to i dont know what*. then come all zeros*. then comes the B typeinfo object
then comes the first vfunc of B, 
![[Pasted image 20250921043222.png]]
### What is a **VTT (Virtual Table Table)**?

- A **VTT** is **not a vtable itse
- lf**, but rather a table of **pointers to construction vtables**.
    
- Think of it as a lookup table that tells constructors:
    
    > “If you’re constructing `Base1` inside `Derived`, here’s the vtable you should install in `Base1`’s vptr at this stage.”
    
- The complete object constructor for `Derived` loads the right entry from the VTT and passes it down to `Base1`’s constructor.
with rsi holding the vtt address, and rdi holding the this pointer, we dive inside B().
![[Pasted image 20250921045033.png]]
we have our rax holding the address of the vtable. when i see the contents of my vtable using x/gx i get some address... that address is actually my construction table for B-in-D inside VTT. Then from there i get the address at this location (B in D). when i see it, it's my first vfunc of B.
![[Pasted image 20250921045635.png]]
now howevere ive stored the address of my B-in-D inside my object. 
rax already held the address of construction table for B in D. We went down 18h (sub 18h). there we got the offset to top... we add it to our rdx(rdx has the offset to top that is 20h and rax has my current this ptr for this constructor that is 0x7fffde40). 
![[Pasted image 20250921050324.png]]
we'll be storing A_ka_show() inside 0x7fffde60, that's why we added 0x20 to get to this address.
![[Pasted image 20250921050710.png]]
anyway we have our addresses stored like this.
anyway.... moving further
![[Pasted image 20250921050852.png]]
rsi------------>addres of VTT----------->addres of C-in-D-------->Address of C_ka_show(first virtual func)
![[Pasted image 20250921051840.png]]
till now... to set the pointers in my objects.. vtt was used as it had references to them.  then in my object the construction table relevant addresses were stored and those came from vtt. 
(try showing calls to functions at this stage)
the offset in our construction tables analyzed earlier was to reach the base object (A).
now rdx is storing the address of vtable of D, this is after we have returned from constructor calls of A, B and C. 
![[Pasted image 20250921052309.png]]
first we were this way as shown below
![[Pasted image 20250921052441.png]]
and now whatever was in rdx has been stored in rax, and rax has 0x7fffde40
![[Pasted image 20250921052536.png]]
now we see, that 0x7ffde40 now holds address of B from the D vtable.
moving forward,
then now we add some offset +20 to our rax and then...
![[Pasted image 20250921052701.png]]
now it stores address from vtable D see below:
![[Pasted image 20250921052739.png]]
then we do...
![[Pasted image 20250921052901.png]]

After all our ctor calls to A,B, and C are done, we rewrite all the addresses in our object. We'd see overwriting thrice and not four times although A,B,C, & D, classes are 4. that is because B's vtable, recall primary vtable, which has derived's vfuns too. so that is why 