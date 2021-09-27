# SurPlus
### A fix for the race condition that MacOS 11.3+ exhibits on unsupported Macs

26 September 2021

It took me six months, a lot of sleepless nights, and I ended up having to write my own debugger to get the answers that I needed, but I finally found the source of the race condition that's been plaguing older Mac Pros (and others) using Big Sur 11.3+ and Monterey.  ~~I still don't know why this problem doesn't seem to appear on newer systems;  my focus has been on finding the problem and creating a solution, and the question of newer systems' success seems like an academic exercise for someone with more time on their hands.~~ (Thanks to insight from @vit9696, it seems that newer Macs don't suffer from this problem because those CPUs support the `rdrand` instruction, meaning they don't need floating-point access during early boot.)

The patch posted here is intended to be incorporated into an OpenCore `config.plist` file.  If you need assistance with this task, please use one (or more) of the following resources:
* [The SurPlus thread on MacRumors](https://forums.macrumors.com/threads/surplus-the-big-sur-monterey-fix-youve-been-waiting-for.2313858/)
* [The OpenCore documentation](https://dortania.github.io/docs/latest/Configuration.html)
* [The OpenCore thread on MacRumors](https://forums.macrumors.com/threads/opencore-on-the-mac-pro.2207814/)
* [The OpenCore Legacy Patcher (OCLP) site](https://dortania.github.io/OpenCore-Legacy-Patcher/)
* [The OpenCore site](https://github.com/acidanthera/OpenCorePkg)

The patch itself appears at the bottom of this page, in a format suitable for cutting-and-pasting into the appropriate location in an OpenCore `config.plist` file.

Questions, comments, or discussion about the "race condition" bug or this patch should be directed to [the SurPlus thread on MacRumors](https://forums.macrumors.com/threads/surplus-the-big-sur-monterey-fix-youve-been-waiting-for.2313858/).

If this information or the patch extends the usable life of your classic Mac, or you find other value in them, [donations are gratefully accepted](https://www.paypal.com/biz/fund?id=YK86GAKSYK3SU).  (To be clear, I am not a tax-exempt organization, charity, or non-profit, and there is no tax benefit to you should you choose to donate - just my heartfelt thanks.)

------------

<h2>The problem and solution for non-programmers:</h2>

MacOS consists of many separate parts working together.  Two of those parts, cryptography and kernel memory management, are interdependent (each makes calls to the other).  Starting with 11.3, there is a circular dependency between them - crypto needs something from memory management, and that same part of memory management needs something from crypto.  If that dependency is encountered at the wrong time during boot, a deadlock occurs - crypto is waiting on memory management, and memory management is waiting on crypto.  The boot process is then hung until the system is forcibly stopped.

There are several possible solutions, some better than others.  I've refined the solution I'm now posting to where it only modifies three bytes of code, and it doesn't impair the functionality or security of the system.

------------

<h2>The problem and solution for more technical folks:</h2>

First, a bit of relevant background:

* Intel does something interesting with floating point.  "Floating point" (FP) includes not only the obvious (floating point math), but also all of the vector and SIMD instructions (SSE, AVX, basically anything that touches the XMM/YMM/ZMM registers).  When an operating system does a context switch (where the CPU pauses execution of one thread to continue execution of another), it has to save and restore all the registers so nothing gets lost.  When you add in all the SIMD registers, that's a lot of overhead - so Intel lets the OS assume that FP is unlikely to happen, and will throw a #NM exception if any FP operations occur.  That way, context switches can just deal with the "normal" registers and ignore the SIMD registers unless a thread actually uses them;  if an FP operation occurs, the system catches the #NM exception, allocates some memory to save the SIMD registers, and saves/restores those registers when switching contexts for that thread.

* Since at least El Capitan (and probably earlier), the MacOS kernel has utilized "zone allocation" (`zalloc`) for internal dynamic memory allocation.  `zalloc` has evolved slowly over the years, but it changed rapidly during Big Sur's various releases;  Apple added both refinements and security.  While earlier versions of `zalloc` simply worked with contiguous chunks of memory, in 11.3 Apple introduced random gaps in the memory pages (presumably for security reasons, although it's a bit unclear to me what vulnerabilities this scheme mitigates - if an attacker has access low enough for that to matter, you've already lost the battle).

* The `corecrypto` kext handles most cryptography tasks for MacOS, including random number generation.  Like many other kexts, it tests the hardware it's running on and uses the most advanced instructions the hardware will support (e.g. AVX2 on newer machines, AVX1 or AES-NI or SSE3 on older machines).  I was surprised at the extent to which `corecrypto` is used - many/most MacOS subsystems touch `corecrypto` at some point.

OK, so here's the gist of the problem.  At startup, MacOS launches multiple threads to initialize the system - to discover and configure the hardware, etc.  At some point, a thread will call `IOLockAlloc()` (to allocate a lock group for IOKit), which will in turn call `zalloc` to allocate memory.  At this early stage of booting, `zalloc` has not yet initialized, so it does that now.  Remember those random memory gaps I mentioned?  `zalloc` initialization needs random numbers to make the gaps random, so it calls `early_random()` to get some random numbers.

So far, so good.  Now, at this point in the boot, `corecrypto` may or may not have been initialized yet.  If it has *not* yet been initialized, `early_random()` will just use its own SHA1 random number generator.  This is the "success" case, because that generator doesn't use any floating point instructions, so `early_random()` will return a random number, `zalloc` will finish initializing, and everybody's happy.

Unfortunately, `corecrypto`'s initialization code is short and sweet, so most of the time, `corecrypto` is already initialized when that call to `early_random()` occurs.  In that case, `early_random()` hands the request to `corecrypto`, which chooses the best instructions to use (on the Mac Pro 5,1-era machines, that's SSE3 and AES-NI, both of which are floating-point as far as the CPU is concerned).  `corecrypto` then acquires a thread lock (the necessity of which is not entirely clear to me), then it starts generating the random number by executing a floating point/SIMD instruction - which throws a #NM exception so MacOS knows to keep track of the SIMD registers.  That exception calls an OS routine called `fpnoextflt()`, which (among other things) allocates a buffer to save the SIMD registers for this thread.  Where does it get that buffer?  `zalloc`, of course.

As you'll recall, the only reason we're executing anything in `corecrypto` at all is because `zalloc` needed some random numbers to initialize itself.  Therefore, when `fpnoextflt()` wants to allocate a buffer, `zalloc` is still uninitialized, and it tries to request some random numbers from `corecrypto`.  (Sound familiar?)

This unhappy set of circumstances would lead to an infinite loop of `zalloc` requesting random numbers and `corecrypto`'s FP instructions triggering #NM exceptions that call `zalloc`, except for the `corecrypto` thread lock I mentioned.  The second time through, `corecrypto` tries to acquire that same lock again.  The lock mechanism doesn't care that the same thread already holds that lock, it just knows that the lock is held, and the current request has to wait for the lock to be released, which effectively puts the thread to sleep while it waits for a lock it can never acquire.

The "race" here is between the `zalloc` and `corecrypto` subsystems getting initialized (or, more accurately, between `zalloc` getting initialized and the execution of any SIMD or floating-point instruction, which is most likely to occur in `corecrypto`).  If a floating-point/SIMD instruction occurs at any point before an attempt to use `zalloc`, `fpnoextflt()` will invoke `zalloc`, which will invoke `corecrypto`, which will deadlock (as described above).  If *no* floating-point/SIMD instructions occur before `zalloc` is invoked/initialized, the boot proceeds normally, and everybody's happy.  For all the current problematic MacOS versions (11.3-11.6 and every 12.x Monterey beta to date), the solution is to delay using `corecrypto` until the `zalloc` subsystem is initialized.

After quickly rejecting a kext-based solution (because any kext would just become part of the race condition), I developed a "blunt instrument" solution that just skipped over the `zalloc` code that introduces random gaps in the memory map (by modifying `zone_allocate_va()` to NOP out the call to `zone_allocate_random_early_gap()`).  That produced no functional changes to the system, and, most likely, no reduction in system security.  However, "most likely" isn't good enough, and since we don't know exactly *why* Apple chose to add those gaps, we shouldn't cavalierly remove them.  In addition, this solution was too narrow - it covered the most likely race case, but there could be other similar cases on differently-configured systems that would require separate patching.

I then settled on a more elegant "scalpel" solution, which is what is included below.  It only modifies three bytes of code.  The idea here is to modify the early random number generator ("early" meaning it's only called during early boot) to only use its SHA1 method and never call `corecrypto` during early boot.  (This involves modifying `read_erandom()`, which the compiler inlines into `early_random()`.)  This way, `zalloc` can still introduce those gaps in the memory map, and system security is not compromised.  Also, because it covers all of the random number generation during the early boot process, it should handle most or all future changes to MacOS that might introduce a new variant of this same race condition (such as increased use of SIMD instructions during early boot).  This patch has minimal impact on the system, and will hopefully have some longevity.

As an aside, I had originally considered APFS as my prime suspect for being the root cause of this problem.  As it turns out, APFS is both a victim and an agent provocateur, but not a direct cause.  Because cryptography is fundamental to APFS, as soon as an APFS disk is detected and the APFS kext gets involved, both `zalloc` and `corecrypto` come into play, as well as SIMD instructions that the APFS kext itself may use.  Booting successfully then boils down to timing and luck.  By staggering the timing a bit, `latebloom` helped improve the luck.  By circumventing the actual cause, luck becomes irrelevant (to this particular problem, anyway).

**Note** that future releases of MacOS might introduce new variants of this race condition, if new early boot code uses SIMD instructions, or if Apple continues to tinker with the `zalloc` initialization sequence.  If that happens, at least we know where to start looking.

------------

## The gory details, for anyone wanting to make their own patch
The public portions of the MacOS source code are at [https://opensource.apple.com/](https://opensource.apple.com/).
The relevant random number code is in `osfmk/prng/prng_random.c`.  The relevant memory allocation code is in `osfmk/kern/zalloc.c`.  Unfortunately, many of the relevant functions are declared as `static`, so their symbols are not public, and it takes more effort to locate their code in the kernel binary or at runtime.  In addition, the compiler often inlines the relevant functions, so there is no discrete function call to manipulate; in those cases, you're stuck locating the code *in situ*, possibly in multiple places.
### The "blunt instrument" patch (included here only for completeness)
During `zalloc` initialization, `zone_expand_locked()` will eventually be called, which will call `zone_allocate_va()`, which will call `zone_allocate_random_early_gap()`, which is where those random gaps in the memory allocation map are inserted (which calls `early_random()`, which almost always invokes `corecrypto`, which throws a #NM floating-point exception, which eventually causes a hang, as described earlier).  We circumvent that call by locating the `zone_allocate_random_early_gap()` code, then finding the only call to that address.  In 11.5, `zone_allocate_random_early_gap()` looks like this:
<pre>
ffffff80002d94e0   55              pushq   %rbp
ffffff80002d94e1   48 89 e5        movq    %rsp, %rbp
ffffff80002d94e4   41 56           pushq   %r14
ffffff80002d94e6   53              pushq   %rbx
ffffff80002d94e7   49 89 fe        movq    %rdi, %r14
ffffff80002d94ea   e8 81 fa 0a 00  callq   _early_random
ffffff80002d94ef   48 89 c3        movq    %rax, %rbx
ffffff80002d94f2   83 e3 0f        andl    $0xf, %ebx
ffffff80002d94f5   83 fb 0b        cmpl    $0xb, %ebx
ffffff80002d94f8   72 14           jb      0xffffff80002d950e
ffffff80002d94fa   31 ff           xorl    %edi, %edi
ffffff80002d94fc   83 fb 0f        cmpl    $0xf, %ebx
ffffff80002d94ff   40 0f 94 c7     sete    %dil
ffffff80002d9503   ff c7           incl    %edi
ffffff80002d9505   5b              popq    %rbx
ffffff80002d9506   41 5e           popq    %r14
ffffff80002d9508   5d              popq    %rbp
ffffff80002d9509   e9 b2 0d 00 00  jmp     0xffffff80002da2c0
ffffff80002d950e   83 fb 04        cmpl    $0x4, %ebx
ffffff80002d9511   77 28           ja      0xffffff80002d953b
ffffff80002d9513   8d 74 1b 06     leal    0x6(%rbx,%rbx), %esi
ffffff80002d9517   4c 89 f7        movq    %r14, %rdi
ffffff80002d951a   e8 21 0f 00 00  callq   0xffffff80002da440
ffffff80002d951f   83 fb 01        cmpl    $0x1, %ebx
ffffff80002d9522   77 17           ja      0xffffff80002d953b
ffffff80002d9524   e8 47 fa 0a 00  callq   _early_random
ffffff80002d9529   83 e0 0f        andl    $0xf, %eax
ffffff80002d952c   8d 70 06        leal    0x6(%rax), %esi
ffffff80002d952f   4c 89 f7        movq    %r14, %rdi
ffffff80002d9532   5b              popq    %rbx
ffffff80002d9533   41 5e           popq    %r14
ffffff80002d9535   5d              popq    %rbp
ffffff80002d9536   e9 05 0f 00 00  jmp     0xffffff80002da440
ffffff80002d953b   5b              popq    %rbx
ffffff80002d953c   41 5e           popq    %r14
ffffff80002d953e   5d              popq    %rbp
ffffff80002d953f   c3              retq
</pre>
By searching for a call to `0xffffff80002d94e0`, we can find the `zone_allocate_va()` code (which is inlined):
<pre>
ffffff80002d8e1a   80 3d 33 e1 92 00 00    cmpb    $0x0, 0x92e133(%rip)
ffffff80002d8e21   0f 89 b9 fb ff ff       jns     0xffffff80002d89e0
ffffff80002d8e27   48 8b 7d d0             movq    -0x30(%rbp), %rdi
ffffff80002d8e2b   e8 b0 06 00 00          callq   0xffffff80002d94e0
ffffff80002d8e30   e9 ab fb ff ff          jmp     0xffffff80002d89e0
</pre>
That `callq 0xffffff80002d94e0` needs to go, so we just NOP it out (overwrite `e8 b0 06 00 00` with `90 90 90 90 90`).  Now the random gaps don't get inserted, and `zalloc` can initialize without involving `corecrypto`.
Note that all of the code in the second block there (from `cmpb` through `jmp`) is RIP-relative, meaning the linker and loader won't alter it, and we can search for those exact bytes without having to mask them.  (Sadly, the nearest public symbol prior to this is `_work_interval_port_type_render_server`, which is more than 2kB away, and unrelated to `zalloc`)

.
### The "scalpel" patch (as included below)
During early boot, the function `early_random()` is called for random number generation.  The first time through, it sets things up for `corecrypto`, then calls `read_erandom()` to generate a random number. `read_erandom()` looks at the static int flag `prng_ready` to see if `corecrypto` is ready for use;  if `prng_ready` is set, `read_erandom()` passes the request off to `corecrypto`.  If `prng_ready` is not set, `read_erandom()` calls `ccdrbg_generate(&erandom...)` to generate the random number; `ccdrbg_generate(&erandom...)` uses SHA1 and does *not* use any floating-point instructions that will throw a #NM exception.  Note that it's the `erandom` structure which contains pointers to the generator function and the digest data.

When `corecrypto` initializes, it calls a function in osfmk/prng/prng_random.c called `register_and_init_prng()`.  That function initializes pointers to functions and data passed in by `corecrypto`, then zeros out the `erandom` structure (erasing the pointer to the SHA1 generator and data).

With a simple patch, we can prevent `register_and_init_prng()` from zeroing out the `erandom` structure, so the SHA1 generator remains intact.

Fortunately, `read_erandom()` is only called from two places, and it gets inlined in each - meaning that we can patch the instance embedded in `early_random()` without affecting the instance inlined in `read_frandom()`, which is used elsewhere in the kernel during normal (post-boot) operation.

In 11.3, the `read_erandom()` instance inlined in `early_random()` contains this code:
<pre>
ffffff8000388de6  c6 05 43 89 ac 00 01  movb $0x1, 0xac8943(%rip)    // init = 1; (from early_random())
ffffff8000388ded  83 3d 7c 91 af 00 00  cmpl $0x0, 0xaf917c(%rip)    // if (prng_ready) {
ffffff8000388df4  74 23                 jz   0xffffff8000388e19      // (jmp to 0 case)
ffffff8000388df6  48 8b 3d 63 89 ac 00  movq 0xac8963(%rip), %rdi    // (beginning of !0 case)
</pre>
We do a one-byte patch to change "je <zero_case>" to "jmp <zero_case>" (i.e. change `74 23` to `eb 23`), so this instance of `read_erandom()` always behaves as if `prng_ready` is 0.  (Note that to decrease the odds of a false pattern match, the patch uses more than just those two bytes (it looks for `00 74 23 48 8b` and replaces it with `00 eb 23 48 8b`)).

In 11.3, the `register_and_init_prng()` function contains this code at the end of the function:
<pre>
ffffff80003890fd  ba 20 00 00 00        movl    $0x20, %edx
ffffff8000389102  48 89 df              movq    %rbx, %rdi
ffffff8000389105  31 f6                 xorl    %esi, %esi
ffffff8000389107  e8 a4 7f d7 ff        callq   _secure_memset        // cc_clear(sizeof(bootseed), bootseed);
ffffff800038910c  48 8d 3d cd f9 87 00  leaq    0x87f9cd(%rip), %rdi
ffffff8000389113  ba 48 01 00 00        movl    $0x148, %edx
ffffff8000389118  31 f6                 xorl    %esi, %esi
ffffff800038911a  e8 91 7f d7 ff        callq   _secure_memset        // cc_clear(sizeof(erandom), &erandom);
ffffff800038911f  48 83 c4 10           addq    $0x10, %rsp
ffffff8000389123  5b                    popq    %rbx
ffffff8000389124  41 5e                 popq    %r14
ffffff8000389126  5d                    popq    %rbp
ffffff8000389127  c3                    retq
</pre>
To prevent the `erandom` structure from getting zeroed out by `_secure_memset`, we just jump over that function call.  The patch searches for `ba 48 01 00 00 31 f6` (movl $0x148,%edx; xorl %esi,%esi) and replaces it with `ba 48 01 00 00 eb 05` (movl $0x148,%edx; jmp $+5).

When both of these patches are applied, we effectively make the early boot random number generator always use its non-floating-point SHA1 code.  We don't prevent `zalloc` from introducing its random gaps, or make any other functional changes to the system.  By avoiding the "floating point before zalloc initialization" deadlock, we allow Big Sur and Monterey to boot just as smoothly as their predecessors.

And now, the beautiful part - the handful of bytes that we look for and modify have not changed between 11.3 and 12b7, meaning we don't (currently, at least) need to have different patches for different versions of MacOS.  This will probably change at some point, but for now, we got very lucky.

------------

## The actual SurPlus patch
The patch below is intended to be inserted in the `Kernel - Patch` array of the OpenCore `config.plist` file.

Classic Macs that don't use OpenCore will need something to patch their kernel.  I'll happily work with anyone who wishes to develop such a patcher (creating a new patcher isn't on my to-do list, since OpenCore makes it simple).

**Remember to disable or uninstall latebloom if you have it installed!**

The SurPlus patch for both `Big Sur` and `Monterey` (through Beta 7, at least):
```
			<dict>
				<key>Arch</key>
				<string>x86_64</string>
				<key>Base</key>
				<string>_early_random</string>
				<key>Comment</key>
				<string>SurPlus v1 - PART 1 of 2 - Patch read_erandom (inlined in _early_random)</string>
				<key>Count</key>
				<integer>1</integer>
				<key>Enabled</key>
				<true/>
				<key>Find</key>
				<data>
				AHQjSIs=
				</data>
				<key>Identifier</key>
				<string>kernel</string>
				<key>Limit</key>
				<integer>800</integer>
				<key>Mask</key>
				<data>
				</data>
				<key>MaxKernel</key>
				<string>21.1.0</string>
				<key>MinKernel</key>
				<string>20.4.0</string>
				<key>Replace</key>
				<data>
				AOsjSIs=
				</data>
				<key>ReplaceMask</key>
				<data>
				</data>
				<key>Skip</key>
				<integer>0</integer>
			</dict>
			<dict>
				<key>Arch</key>
				<string>x86_64</string>
				<key>Base</key>
				<string>_register_and_init_prng</string>
				<key>Comment</key>
				<string>SurPlus v1 - PART 2 of 2 - Patch register_and_init_prng</string>
				<key>Count</key>
				<integer>1</integer>
				<key>Enabled</key>
				<true/>
				<key>Find</key>
				<data>
				ukgBAAAx9g==
				</data>
				<key>Identifier</key>
				<string>kernel</string>
				<key>Limit</key>
				<integer>256</integer>
				<key>Mask</key>
				<data>
				</data>
				<key>MaxKernel</key>
				<string>21.1.0</string>
				<key>MinKernel</key>
				<string>20.4.0</string>
				<key>Replace</key>
				<data>
				ukgBAADrBQ==
				</data>
				<key>ReplaceMask</key>
				<data>
				</data>
				<key>Skip</key>
				<integer>0</integer>
			</dict>
```


**Note** that this is a two-part patch (there are two separate patches included), and both parts need to successfully load or the kernel will most likely panic.  Assuming your installation of OpenCore is relatively new (and stable), this should not be an issue.

------------

## What should Apple do about this?

First, let's be realistic - it's unreasonable to expect Apple to make modifications to MacOS because of a problem that seems to only affect unsupported systems.  Patches like this are probably the best path forward for these older systems.

~~That being said, I have yet to see any reason why this problem could not manifest itself on a supported system.  It may simply be that the newer systems are fast enough that this problem never shows up, or there may be some piece of this that I've overlooked.  In any case, it's a definite flaw for there to be a circular dependency like this, and I hope Apple will consider removing it (there are several possible means to accomplish this).  (I'm not holding my breath, though.)~~

Update: Thanks to insight from @vit9696, it seems that newer models are unaffected by this bug because those CPUs support the `rdrand` instruction, meaning they don't require FP access during early boot.  From Apple's perspective, the race condition fixed by the SurPlus patch is not a bug, since it doesn't affect supported systems.  Therefore, Apple has no reason to address this issue at all.
