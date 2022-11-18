# Considering RISC-V's jal instruction in managed runtimes

## Background
As we know the jal instruction in RISC-V can jump to a pc-relative ±1M range, and its counterpart in AArch64, the bl instruction, is able to reach a pc-relative ±128M range. Particularly, this leads to some strategies in managed runtimes like OpenJDK being less efficient in RV than in ARM under some of our evaluations.

Instruction patching regarding cross-modifying code[1] is heavily used in managed runtimes such as OpenJDK. Alike its AArch64 port, the trampoline logic in its RISC-V port is similar, as an example compiled Java method shown below.


```
[Code Segment]
...
<..010>:  jal <the real 64-bit address target(if offset reachable)> or <trampoline(if offset unreachable)>   // 4-byte aligned patchable callsite
<..014>:  ...
...
 
 
[Stub Segment]
...
# <trampoline>:
<..304>:  aupic t0, 0x0
<..308>:  ld t0, 12(t0)      // load the real target address from memory (dcache) <..320>.
<..31c>:  jr t0              // jump to the real target address (another compiled Java method).
<..320>:  <0x80000000        // 8-byte aligned patchable target address: 0x0000007f80000000. 
<..324>:   0x0000007f>
<..328>:  ...
...


# <the real 64-bit address target>:
<0x7f80000000>: ...          // I am the real target.
...

```

One thing that deserves noticing is the address `<..010>` and `<..320>` can be both potentially patched when another hart wants to redirect the call target while the code is running at full speed in the current hart (cross-modifying code). One would consider why the call site is not an auipc+jalr pair instead of this trampoline strategy: it is because we can not patch two instructions at one time when other harts are running the same piece of code. A hart might just finish the execution of the auipc instruction and want to execute the jalr, while the instruction pair is patched by another hart, the current hart might see a different jalr instruction and the pc would fly away. Therefore, the call site can only contain one 4-byte aligned jal for instruction patching purposes.

Such a trampoline design is for high performance's sake: if a thread wants to overwrite the jal at `<..010>` to another address, one of the two things will happen:
1. if the jal is reachable to the target address:
The jal can be directly patched to the target address. (results in a fast path: the jal becomes a plain 
jump with the highest performance)
2. if not reachable:
The real target address is patched at `<..320>` in the trampoline, and then the trampoline address is patched on the jal, as in the above example. (results in a slow path: it requires another short jump and an extra load from memory)

## Other references
Some references[2][3] show that considering the overall performance including the fast path, the trampoline strategy is the most recommended one compared to several other ones. The bl instruction in AArch64 can reach ±128M around the pc, so the performance is great. However, considering RISC-V's jal instruction can only reach ±1M around the pc, control flow would reach the slowest trampoline very often when the application is big.

## Evaluations
We made some experiments on AArch64 hardware to figure out the performance gain if we have one instruction bearing the same jump range as bl, by limiting the software trampoline to use its slow path when the target address is in range ±1M from the pc where the jal resides, instead of the normal ±128M range, to see how much regression we have.

### SPECjbb2015
```
* SPECjbb2015 benchmark[4] on Raspberry Pi 4b, 4 core, 1.5GHz (both tested 30 times)
The benchmark score decreases from an average 1186 to 1153: showing -2.74%.

* SPECjbb2015 benchmark on Kunpeng 920 server machine, 96 core, 2.6GHz  (both tested 20 times)
The benchmark score decreases from an average 66287 to 64134: showing -3.25%.

* SPECjbb2015 benchmark on ARM Neoverse-N1 server machine, 8 core, 2.6GHz  (both tested 20 times)
The benchmark score decreases from an average 15487 to 15288: showing -1.29%.
```

### JMH

We also have written a micro benchmark using JMH[5] considering such cases by generating ~300 methods, warming up them to the compiled code, and then forcing them to call a common method, which may locate at different offsets relative to the ~300 callers. The data before limiting the ±128M to ±1M and after that shows:

```
* The Kunpeng 920 server machine:
The JMH score decreases from 847.392 ops/s to 191.564 ops/s: showing -77.39%.

* The ARM Neoverse-N1 server machine:
The JMH score decreases from 865.753 ops/s to 332.795 ops/s: showing -61.56%.
```

The latter ones are stressfully testing the jump operation itself, so a bigger figure.

## Conclusion

So if we have nearly the same instruction as the AArch64 one, we may simply speculate that the additive inverse of the data above could be the performance gain on at least some managed runtimes like OpenJDK.

(We have observed some thoughts about the jal as well, though from some considerations[6] of code size reduction.)

----
[1] http://cr.openjdk.java.net/~jrose/jvm/hotspot-cmc.html

[2] https://acticloud.eu/storage/app/uploads/public/5d9/ee4/b8e/5d9ee4b8ebae8225702509.pdf

[3] https://dl.acm.org/doi/fullHtml/10.1145/3546568

[4] https://www.spec.org/jbb2015/

[5] https://github.com/openjdk/jmh

[6] https://github.com/riscv/riscv-code-size-reduction/blob/main/existing_extensions/Huawei%20Custom%20Extension/riscv_longjump_extension.rst



