# Load Reference Barrier of OpenJDK Shenandoah GC

```Assembly
;; AArch64

<load object reference>:
  0x0000ffffa0fdea3c:   ldr	w11, [x1, #16]      ;; load an object reference
<load gc state>:
  0x0000ffffa0fdea40:   ldrb	w10, [x28, #40]     ;; load gc_state and then test
  0x0000ffffa0fdea44:   lsl	x0, x11, #3
  0x0000ffffa0fdea48:   tbnz	w10, #0, 0x0000ffffa0fdea64

<continue>:
  0x0000ffffa0fdea4c:   ...

<test if object is in cset>:
  0x0000ffffa0fdea64:   lsr	x10, x0, #23
  0x0000ffffa0fdea68:   add	x10, x10, #0x10, lsl #12
  0x0000ffffa0fdea6c:   ldrsb	w11, [x10]
  0x0000ffffa0fdea70:   cbz	w11, 0x0000ffffa0fdea4c
<call LRB slowpath>:
  0x0000ffffa0fdea74:   add	x1, x1, #0x10
  0x0000ffffa0fdea78:   adr	x9, 0x0000ffffa0fdea90
  0x0000ffffa0fdea7c:   mov	x8, #0x7cd0           ;   {runtime_call ShenandoahRuntime::load_reference_barrier_strong_narrow(oopDesc*, narrowOop*)}
  0x0000ffffa0fdea80:   movk	x8, #0xb2e9, lsl #16
  0x0000ffffa0fdea84:   movk	x8, #0xffff, lsl #32
  0x0000ffffa0fdea88:   stp	xzr, x9, [sp, #-16]!
  0x0000ffffa0fdea8c:   blr	x8
  0x0000ffffa0fdea90:   b <continue>
```

```Assembly
;; RISC-V 64

<load object reference>:
  0x000000401385caf8:   lwu	t3,12(a1)           ;; load an object reference
<load gc state>:
  0x000000401385cafc:   lbu	t2,40(s7)           ;; load gc_state and then test
  0x000000401385cb00:   slli	a0,t3,0x3
  0x000000401385cb04:   andi	t3,t2,1
  0x000000401385cb08:   bnez	t3,0x000000401385cb20

<continue>:
  0x000000401385cb0c:   ...

<test if object is in cset>:
  0x000000401385cb20:   srli	t2,a0,0x17
  0x000000401385cb24:   lui	t3,0x10
  0x000000401385cb26:   add	t2,t2,t3
  0x000000401385cb28:   lb	t2,0(t2)
  0x000000401385cb2c:   beqz	t2,0x000000401385cb0c
<call LRB slowpath>:
  0x000000401385cb30:   addi	a1,a1,12
  0x000000401385cb32:   auipc	t1,0x0
  0x000000401385cb36:   addi	t1,t1,34 # 0x000000401385cb54
  0x000000401385cb3a:   lui	t0,0x200                    ;   {runtime_call ShenandoahRuntime::load_reference_barrier_strong_narrow(oopDesc*, narrowOop*)}
  0x000000401385cb3e:   addi	t0,t0,352 # 0x0000000000200160
  0x000000401385cb42:   slli	t0,t0,0xb
  0x000000401385cb44:   addi	t0,t0,324
  0x000000401385cb48:   slli	t0,t0,0x6
  0x000000401385cb4a:   addi	t0,t0,6
  0x000000401385cb4e:   addi	sp,sp,-16
  0x000000401385cb50:   sd	t1,8(sp)
  0x000000401385cb52:   jalr	t0
  0x000000401385cb54:   j <continue>
```

Summaries:
* AArch64: 15 instructions (without the load instruction itself)
* RISCV64: 22 instructions (without the load instruction itself, and if without RVC)

RISCV64: 46% more than AArch64.
