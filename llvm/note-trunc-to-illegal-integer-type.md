## Handling `trunc` to an Unsupported Integer Type

### (Hexagon Example)

### 1. LSR Transformation

During **LoopStrengthReduce**, ScalarEvolution rewrites:

```llvm
%x = and i32 %src, 127
```

into:

```llvm
%x = trunc i32 %src to i7
```

This encodes the fact that only the **lower 7 bits** are semantically relevant.

On Hexagon, `i7` is not a supported integer type, so the correctness of this transformation depends on how `trunc i7` is handled during **SelectionDAG type legalization**.

---

### 2. Loop Preheader: `zext i7 → i32` (Mask Required)

In the loop preheader, the truncated value is later zero-extended:

```llvm
%18 = zext i7 %lsr.iv3 to i32
%19 = shl i32 %18, 2
```

#### Initial SelectionDAG

```
t7: i32 = CopyFromReg
t8: i7  = truncate t7
t9: i32 = zero_extend t8
```

#### Responsibility of Type Legalization

At this point:

* The `zext i7 → i32` requires that the **upper 25 bits be zero**.
* Therefore, **SelectionDAG type legalization is responsible for inserting an AND** to clear the upper bits.

This happens via DAG combining:

```
zext(trunc(x)) → and x, 127
```

which then combines further with the shift:

```
(shl (and x, 127), 2)
→
(and (shl x, 2), 508)
```

#### Resulting Hexagon Instruction

```asm
%9:intregs = S4_andi_asl_ri 508, %6:intregs, 2
```

**Key point:**
The masking semantics are preserved because the `zext` makes the upper bits observable and therefore forces legalization to materialize the mask.

---

### 3. Loop Exit: `trunc i7` Used Only by `add` (No Mask)

In the loop exit block, the IR contains:

```llvm
%34 = trunc i32 %2 to i7
%lsr.iv.next4 = add i7 %lsr.iv3, %34
```

After instruction selection and legalization, this becomes:

```asm
%22:intregs = A2_add %9:intregs, %33:intregs
%24:intregs = A2_add %7:intregs, %33:intregs
```

No explicit masking instruction is generated.

---

### 4. Reason No Mask Is Inserted for `add i7`

* Only the **lower 7 bits of the add result** are required to be correct.
* The **upper 25 bits of the inputs are not needed** to compute those lower 7 bits.
* Therefore, SelectionDAG does **not** insert an `and 127` when legalizing this pattern.

This is a deliberate behavior: the truncation semantics are satisfied without materializing a mask.

---

### 5. Contrast: Operations That Would Force Masking

If the truncated value were used by an operation where **upper bits affect the computation of the lower 7 bits**, then legalization would be required to preserve semantics.

Examples include:

* `udiv` / `sdiv`
* `urem` / `srem`
* `lshr` / `ashr`
* `icmp`

In such cases, SelectionDAG would introduce either:

* an explicit `and (2^N - 1)`, or
* a `sign_extend_inreg` / `zero_extend_inreg`

to ensure correctness.

---

### 6. Summary

* In the loop preheader, **`zext i7 → i32` makes upper bits observable**, so type legalization inserts an AND.
* In the loop exit, **`add i7` only requires correct lower 7 bits**, so no mask is inserted.
* The difference is entirely driven by whether the consuming operation needs the upper bits to compute the required lower bits.

This explains the observed asymmetry in masking behavior on Hexagon.

---

### 7. Appendix

#### Reproducer

```c
#define TRIM128(a) ((a) & 0xFFFFFF80u)

unsigned repro(float *array, int length,
               unsigned dim0_unpad, unsigned dim0_pad) {
    unsigned dst_idx = dim0_unpad;

    for (int d1 = 0; d1 < length / dim0_pad - 1; d1++) {
        for (int d0 = 0; d0 < dim0_unpad; d0++) {
            array[dst_idx - TRIM128(dst_idx) + d0] = array[128 + d0];
        }
        dst_idx += dim0_unpad;
    }
    return dst_idx;
}
```
The expression:
```
dst_idx - TRIM128(dst_idx)
```
is equivalent to:
```
dst_idx & 127
```
#### Dumps

```
*** IR Dump After Remove sign extends (reargs) ***
; Function Attrs: nofree norecurse nosync nounwind memory(argmem: readwrite)
define dso_local i32 @repro(ptr noundef captures(none) %0, i32 noundef %1, i32 noundef %2, i32 noundef %3) local_unnamed_addr #0 {
  %5 = udiv i32 %1, %3
  %6 = add i32 %5, -1
  %7 = icmp eq i32 %6, 0
  br i1 %7, label %.loopexit1, label %8

8:                                                ; preds = %4
  %9 = and i32 %2, 7
  %10 = and i32 %2, -8
  %11 = trunc i32 %2 to i7
  %12 = lshr i32 %2, 3
  %13 = mul i32 %12, -8
  %cgep = getelementptr i8, ptr %0, i32 16
  %cgep3 = getelementptr i8, ptr %0, i32 540
  br label %14

14:                                               ; preds = %.loopexit, %8
  %lsr.iv3 = phi i7 [ %lsr.iv.next4, %.loopexit ], [ %11, %8 ]
  %15 = phi i32 [ 0, %8 ], [ %36, %.loopexit ]
  %16 = phi i32 [ %2, %8 ], [ %35, %.loopexit ]
  %17 = icmp eq i32 %2, 0
  %18 = zext i7 %lsr.iv3 to i32
  %19 = shl nuw nsw i32 %18, 2
  %20 = zext i7 %lsr.iv3 to i32
  %21 = shl nuw nsw i32 %20, 2
  %cgep25 = getelementptr i8, ptr %cgep, i32 %21
  br i1 %17, label %.loopexit, label %22

22:                                               ; preds = %14
  %23 = icmp ult i32 %2, 8
  br i1 %23, label %27, label %.preheader

.loopexit1:                                       ; preds = %.loopexit, %4
  %24 = phi i32 [ %2, %4 ], [ %35, %.loopexit ]
  ret i32 %24

25:                                               ; preds = %.preheader
  %26 = icmp eq i32 %9, 0
  br i1 %26, label %.loopexit, label %27

27:                                               ; preds = %25, %22
  %28 = phi i32 [ 0, %22 ], [ %10, %25 ]
  %29 = shl i32 %28, 2
  %30 = add i32 %29, %19
  %cgep4 = getelementptr i8, ptr %0, i32 512
  %cgep5 = getelementptr i8, ptr %cgep4, i32 %29
  %cgep6 = getelementptr i8, ptr %0, i32 %30
  br label %31

31:                                               ; preds = %31, %27
  %lsr.iv32 = phi i32 [ %lsr.iv.next33, %31 ], [ %9, %27 ]
  %lsr.iv30 = phi ptr [ %cgep8, %31 ], [ %cgep5, %27 ]
  %lsr.iv26 = phi ptr [ %cgep7, %31 ], [ %cgep6, %27 ]
  %32 = load float, ptr %lsr.iv30, align 4, !tbaa !9
  store float %32, ptr %lsr.iv26, align 4, !tbaa !9
  %lsr.iv.next33 = add nsw i32 %lsr.iv32, -1
  %33 = icmp eq i32 %lsr.iv.next33, 0
  %cgep7 = getelementptr i8, ptr %lsr.iv26, i32 4
  %cgep8 = getelementptr i8, ptr %lsr.iv30, i32 4
  br i1 %33, label %.loopexit, label %31, !llvm.loop !11

.loopexit:                                        ; preds = %31, %25, %14
  %34 = trunc i32 %2 to i7
  %35 = add i32 %16, %2
  %36 = add nuw nsw i32 %15, 1
  %lsr.iv.next4 = add i7 %lsr.iv3, %34
  %37 = icmp ult i32 %36, %6
  br i1 %37, label %14, label %.loopexit1, !llvm.loop !13

.preheader:                                       ; preds = %22, %.preheader
  %lsr.iv1 = phi i32 [ %lsr.iv.next2, %.preheader ], [ %13, %22 ]
  %lsr.iv11 = phi ptr [ %cgep24, %.preheader ], [ %cgep3, %22 ]
  %lsr.iv6 = phi ptr [ %cgep23, %.preheader ], [ %cgep25, %22 ]
  %cgep9 = getelementptr i8, ptr %lsr.iv11, i32 -28
  %38 = load float, ptr %cgep9, align 4, !tbaa !9
  %cgep10 = getelementptr i8, ptr %lsr.iv6, i32 -16
  store float %38, ptr %cgep10, align 4, !tbaa !9
  %cgep11 = getelementptr i8, ptr %lsr.iv11, i32 -24
  %39 = load float, ptr %cgep11, align 4, !tbaa !9
  %cgep12 = getelementptr i8, ptr %lsr.iv6, i32 -12
  store float %39, ptr %cgep12, align 4, !tbaa !9
  %cgep13 = getelementptr i8, ptr %lsr.iv11, i32 -20
  %40 = load float, ptr %cgep13, align 4, !tbaa !9
  %cgep14 = getelementptr i8, ptr %lsr.iv6, i32 -8
  store float %40, ptr %cgep14, align 4, !tbaa !9
  %cgep15 = getelementptr i8, ptr %lsr.iv11, i32 -16
  %41 = load float, ptr %cgep15, align 4, !tbaa !9
  %cgep16 = getelementptr i8, ptr %lsr.iv6, i32 -4
  store float %41, ptr %cgep16, align 4, !tbaa !9
  %cgep17 = getelementptr i8, ptr %lsr.iv11, i32 -12
  %42 = load float, ptr %cgep17, align 4, !tbaa !9
  store float %42, ptr %lsr.iv6, align 4, !tbaa !9
  %cgep18 = getelementptr i8, ptr %lsr.iv11, i32 -8
  %43 = load float, ptr %cgep18, align 4, !tbaa !9
  %cgep19 = getelementptr i8, ptr %lsr.iv6, i32 4
  store float %43, ptr %cgep19, align 4, !tbaa !9
  %cgep20 = getelementptr i8, ptr %lsr.iv11, i32 -4
  %44 = load float, ptr %cgep20, align 4, !tbaa !9
  %cgep21 = getelementptr i8, ptr %lsr.iv6, i32 8
  store float %44, ptr %cgep21, align 4, !tbaa !9
  %45 = load float, ptr %lsr.iv11, align 4, !tbaa !9
  %cgep22 = getelementptr i8, ptr %lsr.iv6, i32 12
  store float %45, ptr %cgep22, align 4, !tbaa !9
  %lsr.iv.next2 = add i32 %lsr.iv1, 8
  %46 = icmp eq i32 %lsr.iv.next2, 0
  %cgep23 = getelementptr i8, ptr %lsr.iv6, i32 32
  %cgep24 = getelementptr i8, ptr %lsr.iv11, i32 32
  br i1 %46, label %25, label %.preheader, !llvm.loop !15
}

# *** IR Dump After Hexagon DAG->DAG Pattern Instruction Selection (hexagon-isel) ***:
# Machine code for function repro: IsSSA, TracksLiveness
Function Live Ins: $r0 in %31, $r1 in %32, $r2 in %33, $r3 in %34

bb.0 (%ir-block.4):
  successors: %bb.4(0x30000000), %bb.1(0x50000000); %bb.4(37.50%), %bb.1(62.50%)
  liveins: $r0, $r1, $r2, $r3
  %34:intregs = COPY $r3
  %33:intregs = COPY $r2
  %32:intregs = COPY $r1
  %31:intregs = COPY $r0
  ADJCALLSTACKDOWN 0, 0, implicit-def $r29, implicit-def dead $r30, implicit $r31, implicit $r30, implicit $r29
  $r0 = COPY %32:intregs
  $r1 = COPY %34:intregs
  J2_call &__hexagon_udivsi3, <regmask $d8 $d9 $d10 $d11 $d12 $d13 $r16 $r17 $r18 $r19 $r20 $r21 $r22 $r23 $r24 $r25 $r26 $r27>, implicit-def dead $pc, implicit-def dead $r31, implicit $r29, implicit $r0, implicit $r1, implicit-def $r29, implicit-def $r0
  ADJCALLSTACKUP 0, 0, implicit-def dead $r29, implicit-def dead $r30, implicit-def dead $r31, implicit $r29
  %35:intregs = COPY $r0
  %0:intregs = A2_addi %35:intregs, -1
  %36:predregs = C2_cmpeqi %0:intregs, 0
  J2_jumpt killed %36:predregs, %bb.4, implicit-def dead $pc
  J2_jump %bb.1, implicit-def dead $pc

bb.1 (%ir-block.8):
; predecessors: %bb.0
  successors: %bb.2(0x80000000); %bb.2(100.00%)

  %1:intregs = A2_andir %33:intregs, 7
  %2:intregs = A2_andir %33:intregs, -8
  %3:intregs = COPY %33:intregs
  %4:intregs = A2_subri 0, %2:intregs
  %5:intregs = A2_addi %31:intregs, 16
  %6:intregs = A2_addi %31:intregs, 540
  %37:intregs = A2_tfrsi 0

bb.2 (%ir-block.14):
; predecessors: %bb.1, %bb.8
  successors: %bb.8(0x30000000), %bb.3(0x50000000); %bb.8(37.50%), %bb.3(62.50%)

  %7:intregs = PHI %3:intregs, %bb.1, %24:intregs, %bb.8
  %8:intregs = PHI %37:intregs, %bb.1, %23:intregs, %bb.8
  %9:intregs = PHI %33:intregs, %bb.1, %22:intregs, %bb.8
  %38:predregs = C2_cmpeqi %33:intregs, 0
  %10:intregs = S4_andi_asl_ri 508, %7:intregs(tied-def 0), 2
  %11:intregs = A2_add %5:intregs, %10:intregs
  J2_jumpt killed %38:predregs, %bb.8, implicit-def dead $pc
  J2_jump %bb.3, implicit-def dead $pc

bb.3 (%ir-block.22):
; predecessors: %bb.2
  successors: %bb.6(0x40000000), %bb.9(0x40000000); %bb.6(50.00%), %bb.9(50.00%)

  %40:predregs = C2_cmpgtui %33:intregs, 7
  %41:predregs = C2_not killed %40:predregs
  %39:intregs = A2_tfrsi 0
  J2_jumpt killed %41:predregs, %bb.6, implicit-def dead $pc
  J2_jump %bb.9, implicit-def dead $pc

bb.4..loopexit1:
; predecessors: %bb.0, %bb.8

  %12:intregs = PHI %33:intregs, %bb.0, %22:intregs, %bb.8
  $r0 = COPY %12:intregs
  PS_jmpret $r31, implicit-def dead $pc, implicit $r0

bb.5 (%ir-block.25):
; predecessors: %bb.9
  successors: %bb.8(0x30000000), %bb.6(0x50000000); %bb.8(37.50%), %bb.6(62.50%)

  %51:predregs = C2_cmpeqi %1:intregs, 0
  J2_jumpt killed %51:predregs, %bb.8, implicit-def dead $pc
  J2_jump %bb.6, implicit-def dead $pc

bb.6 (%ir-block.27):
; predecessors: %bb.3, %bb.5
  successors: %bb.7(0x80000000); %bb.7(100.00%)

  %13:intregs = PHI %39:intregs, %bb.3, %2:intregs, %bb.5
  %52:intregs = S2_asl_i_r %13:intregs, 2
  %14:intregs = S4_addaddi %31:intregs, %52:intregs, 512
  %15:intregs = M2_acci %31:intregs(tied-def 0), %52:intregs, %10:intregs

bb.7 (%ir-block.31):
; predecessors: %bb.6, %bb.7
  successors: %bb.8(0x04000000), %bb.7(0x7c000000); %bb.8(3.12%), %bb.7(96.88%)

  %16:intregs = PHI %1:intregs, %bb.6, %19:intregs, %bb.7
  %17:intregs = PHI %14:intregs, %bb.6, %21:intregs, %bb.7
  %18:intregs = PHI %15:intregs, %bb.6, %20:intregs, %bb.7
  %53:intregs, %21:intregs = L2_loadri_pi %17:intregs(tied-def 1), 4 :: (load (s32) from %ir.lsr.iv30, !tbaa !9)
  %20:intregs = S2_storeri_pi %18:intregs(tied-def 0), 4, killed %53:intregs :: (store (s32) into %ir.lsr.iv26, !tbaa !9)
  %19:intregs = nsw A2_addi %16:intregs, -1
  %54:predregs = C2_cmpeqi %19:intregs, 0
  %55:predregs = C2_not killed %54:predregs
  J2_jumpt killed %55:predregs, %bb.7, implicit-def dead $pc
  J2_jump %bb.8, implicit-def dead $pc

bb.8..loopexit:
; predecessors: %bb.2, %bb.5, %bb.7
  successors: %bb.2(0x7c000000), %bb.4(0x04000000); %bb.2(96.88%), %bb.4(3.12%)

  %22:intregs = A2_add %9:intregs, %33:intregs
  %23:intregs = nuw nsw A2_addi %8:intregs, 1
  %24:intregs = A2_add %7:intregs, %33:intregs
  %56:predregs = C2_cmpgtu %0:intregs, %23:intregs
  J2_jumpt killed %56:predregs, %bb.2, implicit-def dead $pc
  J2_jump %bb.4, implicit-def dead $pc

bb.9..preheader:
; predecessors: %bb.3, %bb.9
  successors: %bb.5(0x04000000), %bb.9(0x7c000000); %bb.5(3.12%), %bb.9(96.88%)

  %25:intregs = PHI %4:intregs, %bb.3, %28:intregs, %bb.9
  %26:intregs = PHI %6:intregs, %bb.3, %30:intregs, %bb.9
  %27:intregs = PHI %11:intregs, %bb.3, %29:intregs, %bb.9
  %42:intregs = L2_loadri_io %26:intregs, -28 :: (load (s32) from %ir.cgep9, !tbaa !9)
  S2_storeri_io %27:intregs, -16, killed %42:intregs :: (store (s32) into %ir.cgep10, !tbaa !9)
  %43:intregs = L2_loadri_io %26:intregs, -24 :: (load (s32) from %ir.cgep11, !tbaa !9)
  S2_storeri_io %27:intregs, -12, killed %43:intregs :: (store (s32) into %ir.cgep12, !tbaa !9)
  %44:intregs = L2_loadri_io %26:intregs, -20 :: (load (s32) from %ir.cgep13, !tbaa !9)
  S2_storeri_io %27:intregs, -8, killed %44:intregs :: (store (s32) into %ir.cgep14, !tbaa !9)
  %45:intregs = L2_loadri_io %26:intregs, -16 :: (load (s32) from %ir.cgep15, !tbaa !9)
  S2_storeri_io %27:intregs, -4, killed %45:intregs :: (store (s32) into %ir.cgep16, !tbaa !9)
  %46:intregs = L2_loadri_io %26:intregs, -12 :: (load (s32) from %ir.cgep17, !tbaa !9)
  S2_storeri_io %27:intregs, 0, killed %46:intregs :: (store (s32) into %ir.lsr.iv6, !tbaa !9)
  %47:intregs = L2_loadri_io %26:intregs, -8 :: (load (s32) from %ir.cgep18, !tbaa !9)
  S2_storeri_io %27:intregs, 4, killed %47:intregs :: (store (s32) into %ir.cgep19, !tbaa !9)
  %48:intregs = L2_loadri_io %26:intregs, -4 :: (load (s32) from %ir.cgep20, !tbaa !9)
  S2_storeri_io %27:intregs, 8, killed %48:intregs :: (store (s32) into %ir.cgep21, !tbaa !9)
  %49:intregs = L2_loadri_io %26:intregs, 0 :: (load (s32) from %ir.lsr.iv11, !tbaa !9)
  S2_storeri_io %27:intregs, 12, killed %49:intregs :: (store (s32) into %ir.cgep22, !tbaa !9)
  %28:intregs = A2_addi %25:intregs, 8
  %50:predregs = C2_cmpeqi %28:intregs, 0
  %29:intregs = A2_addi %27:intregs, 32
  %30:intregs = A2_addi %26:intregs, 32
  J2_jumpt killed %50:predregs, %bb.5, implicit-def dead $pc
  J2_jump %bb.9, implicit-def dead $pc

# End machine code for function repro.
```