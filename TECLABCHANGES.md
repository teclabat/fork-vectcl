# TECLAB Changes: VecTcl 0.3 — Tcl 9 Migration

This document describes all changes made to migrate VecTcl (vectcl 0.3) from Tcl 8.6 to Tcl 9.

## Summary

Full Tcl 9 API migration plus Windows LLP64 64-bit integer correctness fixes. All 251 tests pass.

## Major Changes

### 1. Tcl 9 API Migration

#### Tcl_GetObjType Snooping

`Tcl_GetObjType()` returns NULL for core types in Tcl 9. Replaced with temp-object snooping pattern in `Vectcl_Init()`:

```c
// Before:
tclListType = Tcl_GetObjType("list");

// After:
tclListType = (Tcl_ObjType *) Tcl_GetObjType("list");
if (tclListType == NULL) {
    Tcl_Obj *tmp = Tcl_NewListObj(0, NULL);
    Tcl_IncrRefCount(tmp);
    tclListType = (Tcl_ObjType *)tmp->typePtr;
    Tcl_DecrRefCount(tmp);
}
```

Applied to `tclListType`, `tclDoubleType`, `tclIntType`, and `tclWideIntType`.

The snooped type pointers are stored as globals in `vectcl.c` and accessed from `nacomplex.c` via `extern` declarations, so the snooping runs once at init time rather than on every call.

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`, `generic/nacomplex.c`

#### Tcl_InitStringRep

Replaced direct `objPtr->bytes` / `objPtr->length` writes with `Tcl_InitStringRep()`:

```c
// Before:
naPtr->length = Tcl_DStringLength(&srep);
naPtr->bytes = ckalloc(naPtr->length + 1);
memcpy(naPtr->bytes, Tcl_DStringValue(&srep), naPtr->length + 1);

// After:
Tcl_InitStringRep(naPtr, Tcl_DStringValue(&srep), Tcl_DStringLength(&srep));
```

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`, `generic/nacomplex.c`

#### Tcl_FreeInternalRep

Replaced the private `TclFreeIntRep` macro:

```c
// Before:
#define TclFreeIntRep(objPtr) \
    if ((objPtr)->typePtr != NULL && ...

// After:
#define TclFreeIntRep(objPtr) Tcl_FreeInternalRep(objPtr)
```

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`, `generic/nacomplex.c`

#### int to Tcl_Size

Changed output parameters for Tcl API functions from `int` to `Tcl_Size`:

- `Tcl_ListObjLength` / `Tcl_ListObjGetElements` output variables
- `Tcl_GetByteArrayFromObj` output length
- `Tcl_GetStringFromObj` output length

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`, `generic/vectclapi.c`, `generic/bcexecute.c`, `generic/vmparser.c`

#### CONST to const

Replaced deprecated `CONST` macro with standard `const`:

```c
// Before:
Tcl_Obj* CONST* objv

// After:
Tcl_Obj *const *objv
```

**Files:** `generic/vmparser.c`

#### Tcl_InitStubs Version

```c
// Before:
Tcl_InitStubs(interp, TCL_VERSION, 0)

// After:
Tcl_InitStubs(interp, "8.6-", 0)
```

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`

#### Lowercase Init Alias

Tcl 9 package loading requires a lowercase init function:

```c
int vectcl_Init(Tcl_Interp* interp) { return Vectcl_Init(interp); }
```

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`

#### Vectcl_InitStubs Return Type

```c
// Before:
char* Vectcl_InitStubs(...)

// After:
const char* Vectcl_InitStubs(...)
```

**Files:** `generic/vectcl.h`, `generic/vectcl.tcl.h`, `generic/vectclapi.c`

### 2. Windows LLP64 64-bit Integer Fixes

On Windows (LLP64), `long int` is 32-bit even on 64-bit systems.

#### NaWideInt Typedef

```c
// Before:
typedef long int NaWideInt;

// After:
typedef long long NaWideInt;
```

**Files:** `generic/vectcl.h`, `generic/vectcl.tcl.h`

#### Tcl Integer API

Replaced 32-bit `long`-based Tcl API calls with 64-bit `WideInt` variants:

```c
// Before:
Tcl_NewLongObj(value)
Tcl_GetLongFromObj(interp, obj, &longvar)

// After:
Tcl_NewWideIntObj(value)
Tcl_GetWideIntFromObj(interp, obj, &widevar)
```

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`

#### labs to llabs

`labs()` operates on `long` (32-bit on Windows). Changed to `llabs()` for `long long`:

```c
// Before:
#define INTOP *result = labs(op);

// After:
#define INTOP *result = llabs(op);
```

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`

### 3. Bignum Support for Integer Conversions

Added proper handling of Tcl bignums (values exceeding int64 range) using the libtommath API through Tcl stubs.

#### TomMath Stubs Initialization

```c
#define TCL_NO_TOMMATH_H
#include <tclTomMath.h>
```

Added `Tcl_TomMath_InitStubs()` call in `Vectcl_Init()`.

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`

#### Bignum Detection in Type Scanning

In `ScanNumArrayDimensionsFromValue()`, after `Tcl_GetWideIntFromObj` fails, bignums that fit in 64 bits (`mp_count_bits <= 64`) are stored as `NumArray_Int` with exact 64-bit truncation via `mp_get_mag_u64()`. Larger bignums fall through to double conversion.

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`

#### Bignum Extraction in Buffer Creation

In `createNumArraySharedBufferFromTypedList()`, when `Tcl_GetWideIntFromObj` fails for a `NumArray_Int` value, the bignum is extracted via `Tcl_GetBignumFromObj` and the low 64 bits are stored using `mp_get_mag_u64()` with proper sign handling.

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`

#### Float64 to Int64/Uint64 Wrapping

Added special-case conversion in `NumArrayConvertToType()` for Float64 to Int64/Uint64 using `fmod()` to avoid undefined behavior when casting out-of-range doubles:

```c
double mod = fmod(val, 18446744073709551616.0);  // mod 2^64
if (mod >= 9223372036854775808.0) mod -= 18446744073709551616.0;
*bufptr++ = (int64_t)mod;
```

**Files:** `generic/vectclapi.c`

### 4. myTcl_GetDoubleFromObj Rewrite

Replaced the function that accessed Tcl internal representation fields (`objPtr->internalRep.doubleValue`, `.longValue`, `.wideValue`) with one using only public APIs:

```c
static int myTcl_GetDoubleFromObj(Tcl_Interp *interp, Tcl_Obj *objPtr, double *dblPtr) {
    if (Tcl_GetDoubleFromObj(interp, objPtr, dblPtr) == TCL_OK) {
        return TCL_OK;
    }
    const char *str = Tcl_GetString(objPtr);
    char *endptr;
    *dblPtr = strtod(str, &endptr);
    if (endptr != str && *endptr == '\0') {
        return TCL_OK;
    }
    return TCL_ERROR;
}
```

Also removed deprecated `register` keyword from parameters.

**Files:** `generic/vectcl.c`, `generic/vectcl.tcl.c`

### 5. const Correctness in vmparser.c

Fixed `const` qualifier warning for `rde_param_data` call by adding explicit cast:

```c
rde_param_data(p, (char *)buf, len);
```

Changed `char* buf` to `const char* buf` to match `Tcl_GetStringFromObj` return type.

**Files:** `generic/vmparser.c`

## Files Modified

| File | Key Changes |
|------|-------------|
| `generic/vectcl.h` | `const char*` return type for `Vectcl_InitStubs`, `long long` for `NaWideInt` |
| `generic/vectcl.tcl.h` | `long long` for `NaWideInt` |
| `generic/vectcl.c` | Tcl 9 API migration, LLP64 fixes, bignum support, TomMath stubs, `myTcl_GetDoubleFromObj` rewrite, `llabs`, lowercase init alias |
| `generic/vectcl.tcl.c` | Same changes as `vectcl.c` (template source of truth) |
| `generic/vectclapi.c` | `const char*` return, `Tcl_Size`, Float64-to-Int64/Uint64 wrapping with `fmod` |
| `generic/nacomplex.c` | `extern` for snooped type pointers (removed per-call snooping), `Tcl_InitStringRep`, `Tcl_FreeInternalRep`, `Tcl_GetString` |
| `generic/bcexecute.c` | `Tcl_Size` for `Tcl_GetByteArrayFromObj` and `Tcl_ListObjLength` |
| `generic/vmparser.c` | `CONST` to `const`, `Tcl_Size`, `const char*` for buf, cast for `rde_param_data` |

### 6. Fixed-Width Type Support in Core Operations

Several core functions only handled the three base types (Int, Float64, Complex128), causing segfaults or spurious "Unknown data type" output when fixed-width types (Bool, Int8, Uint8, ..., Uint64, Float32, Complex64) were used in operations like `hstack`.

#### NumArray_UpcastType — Skip Fixed-Width Types

Changed from linear enum increment (`base+1`) to direct chain `Int → Float64 → Complex128`. Fixed-width types are only reachable via explicit conversion functions, not from Tcl string parsing. This eliminates 9–10 spurious "Unknown data type" printf messages per type-retry cycle.

**Files:** `generic/vectclapi.c`

#### NumArrayGetScalarValueFromObj — All Types

Extended the switch to read all fixed-width types from the buffer. Integer-like types are stored in the `value.Int` union slot, Float32 in `value.Float64`, Complex64 in `value.Complex128`. Also guarded the default case against `interp == NULL` to prevent segfault.

**Files:** `generic/vectclapi.c`

#### NumArraySetValue — All Destination Types

Added fill loops for all fixed-width destination types using a macro-generated pattern for integer types and explicit blocks for Float32 and Complex64.

**Files:** `generic/vectclapi.c`

#### NumArrayCopy — Generic Same-Type Copy

Added a byte-level `memcpy` copy path for same-type source and destination, handling all fixed-width types that the existing typed COPYLOOP macros did not cover.

**Files:** `generic/vectclapi.c`

#### createNumArraySharedBufferFromTypedList — Proper Error

Replaced bare `printf("Unknown data type\n")` with `RESULTPRINTF(...)` to produce a proper Tcl error message.

**Files:** `generic/vectcl.c`

## Test Results

```
all.tcl:  Total  257  Passed  257  Skipped  0  Failed  0
```

6 new tests (`hstack-fixed-1` through `hstack-fixed-6`) cover `hstack` with uint8, int8, int16, int32, uint64, and bool operands.

## Backward Compatibility

Tcl 8.6 compatibility is maintained via shims in `generic/vectcl.h`: `Tcl_Size` typedef, `Tcl_InitStringRep` static inline, and a `TclFreeIntRep` macro for Tcl < 9.
