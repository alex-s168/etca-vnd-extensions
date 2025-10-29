# etca-variable-vlen-vector-ext

## Terminology
- "target": the etc.a core that the code runs on
- "vlen": number of elements in a vector; exact meaning depends on context
- "target vlen": the maximum number of elements in a vector, which is specified by the target, for the given data type
- "eltw": for the current vector configuration, the number of bits or bytes (depends on context) of the configured vector element type

## Extensions
- base vector extension: defines basic configuration, data movement, and mask ops
- base integer vector extension:
  does not define a set of supported element types, but defines the integer vector instructions
- base floating point vector extension:
  does not define a set of supported element types, but defines the floating point instructions
- integer data types extensions: ...
- floating point data types extensions: ...

## Goals
- minimal overhead for scalar floating point code: only overhead is requiring a `setvl` instruction once
- easy and cheap to implement for scalar floating point
- vectororized code should be portable
- vectorized integer loops, even on vlen=1, shouldn't have much overhead compared to scalar integer loops,
  to hopefully increase vector extension adaption in the future (even if it's just vlen=1), so that programs
  don't need unvectorized versions of code

## Operands
### vector register
```c
typedef u4 vector;
```

there are 16 vector registers (TODO: can reduce, rvv has 32...),
to prevent spilling in code that makes heavy use of vectors or floats.

these have to be able to store any vector configuation that is legal on that target.

### vector mask register
```c
typedef u2 vector_mask;
```

There are 2 vector mask registers, and two hard-wired vector masks:
- `0b10`: all elements are `1`
- `0b11`: only the first element is `1`, the others are `0`; motivation: working with scalar floats in vectors

these have one bit elements, and have the same number of elements as the currently configured vlen.

### vector memory operand
```c
// reg(base) + vlen * imm(u3) - SS * imm(u3)
//
// motivation:
// 1. being able to offset by vlen allows one to cheaply
//    unroll multiple vectorized loop iterations.
//    see Dot-product example
// 2. being able to subtract a constant amount
//    makes it cheap to spill vectors to/from the stack
struct __v_ldmo {
  register void* base : 4;
  u3 plus_this_times_vlen;
  i3 minus_this_times_ss;
};
```

## Instruction encoding
### "VB"
`out = a op b`

if `only_scalar` is set, this op only operates on the first element, and the other elements in the destination register are zeroed.

```c
struct {
  u?? opc;
  u1 only_scalar;

  u4 out;
  u4 a;
  u4 b;
};
```

### "VM"
for loads: `reg = {mask} [mo]`

for stores: `{mask} [mo] = reg`

```c
struct {
  u?? opc;

  // 16 bits:
  u4 reg;
  __v_ldmo mo : 10;
  u2 mask;
};
```

### Design Considerations
- Implementing the vector extension for one element vlen should be really easy!


## Instructions
### setvl
```c
// pseudocoe
size_t __setvl (__vector_dtype_t imm(dty), size_t regOrImm(num), size_t imm(chunksize)) {
  size_t supported = readcr(__VLEN_ + dty);
  if (num != 0) {
    if (supported < num) num = supported;
  } 
  return floordiv(num, chunksize);
}
```

TODO

When reducing vlen, the elements stored in the vector that are not anymore accessible,
have to stay stored there, so that, when extending vlen again, they are still accessible.

It is recommended for compilers to emit `v.drop_upper`, at the end of heavily vectorized code.

When setting the dtype to for example `f32`, integer vector operations will operator on the vector elements as `i32`,
IF the core supports `i32` elements in vectors (otherwise it will fault),
and the other way arround.


### v.drop_upper
```c
enum __v_drop_upper_m  {
  ALL = 0, // all elements in all vectors are affected
  AFTER_VLEN = 1,  // all elements that are not in current vlen are affected
  AFTER_FIRST = 2, // all elements, except for the first one, are affected
};
void __v_drop_upper (__v_drop_upper_m imm(mode));
```

When this instruction is used, an implementation can fill the affected
vector elements with any value.

An implementation may ignore this instruction.

This should be emitted by compilers at the end of code that increased the vlen by a lot,
so that specific advanced out-of-order cores can reuse the upper elements of those registers for other code.

It is recommended for compilers to use `AFTER_FIRST`, if they know that vlen is 1, and the dtype is correct.


### v.ld
```c
// masked load vlen*eltw
//
// only loads where the mask is true,
// filling the other elements with zero,
// and not faulting for those elements that are not loaded
vector __v_ld(__v_mo imm(mo), vector_mask m);
```

### v.ldnf
```c
// don't fault when can't acces ptr+(N>1) element,
// but will fault if can't access [ptr] element.
// every value after the first fault is UB
//
// motivation:
// - strlen
vector __v_ldnf(__v_mo mo);
```

### v.ld_brdcst
```c
// load one element of eltw, and broadcast to all elements
vector __v_ld_brdcst(__v_mo imm(mo));
```

### vm.any
```c
void __vm_any(vector_mask m) {
  if (any bit set in m) {
    test(0, 0)
  } else {
    test(1, 1)
  }
}
```

### vm.all
```c
void __vm_all(vector_mask m) {
  if (all bits set in m) {
    test(0, 0)
  } else {
    test(1, 1)
  }
}
```

### vm.or
### vm.xor
### vm.and
### vm.ctlz
### vm.cttz?? TODO
### vm.popcnt


## Examples
### strlen
```c
size_t strlen(char const* s) {
  size_t o = 0;
  for (; *s; s++) o ++;
  return o;
}
```

Any automatically loop vectorizing compiler would output:
```c
size_t strlen(char const* s) {
  if (*s == 0) return 0;
  __setvl (__VEC_I8, 0, 1);
  size_t num = 0;
  // we could easily (and should, if we know vlen is small), unroll this loop
  while (true) {
    vector x = __v_ldnf(s);
    mask z = __v_cmp_eq(x, __broadcast_arg 0);
    if (__vm_any(z)) {
      return num + __vm_ctlz(z);
    }

    // we could cache result of last setvl op, because will be the same,
    // but this reduces register usage, and is only one op with const args anyways
    s += __setvl (__VEC_I8, 0, 1);
    num += __setvl (__VEC_I8, 0, 1);
  }
}
```

### Dot-product
```c
__attribute__((hot))
float dot(float* a, float* b, size_t len) {
  if (__builtin_unlikely(len < 1024)) {}
  float o = 0;
  for (size_t i = 0; i < len; i ++)
    o += a[i] * b[i];
  return o;
}
```

A "smart" compiler would output:
```c
float dot(float* a, float* b, size_t len) {
  // would insert a check for vec ext & fp32 vecs here

  // scalar floats are stored in vec regs too
  // since loop vlen is more than 1 float probably, remaining elements are zeroed
  vector sum = __v_ld(0.0f, MASK_FIRST);
  while (true) { // unrolled(4) loop
    size_t num = __setvl (__VEC_F32, len, 4);
    if (num == 0) break;

    vector c, d;
    c = __v_ld(a); d = __v_ld(d); vector s1 = __v_fmul(c,d);
    // load ops support offset `addr_reg + C + vlen * N`
    c = __v_ld(a + __vlen*1); d = __v_ld(b + __vlen*1);   __v_fma(&s1,c,d);
    c = __v_ld(a + __vlen*2); d = __v_ld(b + __vlen*2);   __v_fma(&s1,c,d);
    c = __v_ld(a + __vlen*3); d = __v_ld(b + __vlen*3);   __v_fma(&s1,c,d);
    vector x = __v_fhsum(s1); // horizontal reduce: sum
    sum = __v_fadd(sum, x);

    a += num; b += num; len -= num;
  }

  // remainder loop
  while (len) {
    sum += *a * *b;
    a ++; b ++; len --;
  }

  return sum;
}
```


### 1-dimensional convolution
```c
void conv1d_3(float kernel[3], float* out, float* arr, size_t len) {
  for (size_t i = 0; i < len; i ++) {
    float prev = i == 0 ? 0 : arr[i - 1];
    float this = arr[i];
    float next = i + 1 == len ? 0 : arr[i + 1];
    out[i] = kernel[0] * prev + kernel[1] * this + kernel[2] * next;
  }
}
```

A "smart" compiler would output something like this:
```c
void conv1d_3(float kernel[3], float* out, float* arr, size_t len) {
  if (len == 0) return;

  __setvl (__VEC_F32, 1, 1);

  /* first iter */ {
    vector this = __v_ld(arr);
    vector next = len == 1 ? 0 : arr[1];
    out[0] = kernel[1] * this + kernel[2] * next;
  }

  size_t i = 1;
  while (true) {
    size_t num = __setvl (__VEC_F32, len - 1, 1);
    if (num == 0) break;
    vector prev = __v_ld(arr + i - 1);
    vector this = __v_ld(arr + i);
    vector next = __v_ld(arr + i + 1);
    vector res = __v_mul(prev, __v_ld_brdcst(kernel));
    __v_fma(&res, this, __v_ld_brdcst(kernel + 1));
    __v_fma(&res, next, __v_ld_brdcst(kernel + 2));
    __v_st(out + i, res);
    i += num;
  }

  // don't need another setvl here, because dty doesn't change, and can simply mark ops as SCALAR

  /* last iter */ {
    size_t i = len - 1;
    vector prev = 0;
    if (i != 0) {
      prev = __v_ld SCALAR(arr + i - 1);
    }
    // last iter using vector, SCALAR
  }
}
```

Existing compilers supporting this:
- [LLVM targetting RISC-V vector](https://godbolt.org/z/j6qEbns85)
- [LLVM targetting ARM SVE](https://godbolt.org/z/eWsKP865T)
- [GCC targetting RISC-V vector](https://godbolt.org/z/G7nvT6M4q) (requires manually peeling)



### reciprocal square root approximation
https://en.wikipedia.org/wiki/Fast_inverse_square_root
```c
float rsqrt( float number )
{
	const float threehalfs = 1.5F;

	float x2 = number * 0.5F;
	float y  = number;
	long i  = * ( long * ) &y;
	i  = 0x5f3759df - ( i >> 1 );
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );
	return y;
}
```

compiled for scalar floating point (via vector float extension):
```c
vector rsqrt(vector number)
{
  __setvl (__VEC_F32, 1, 1);
  uint32_t i = __v_extract(number, 0); // in this case: performs f32 bitcast into 32bit int reg
  number = __v_fmul(number, __v_ld(0.5F, MASK_FIRST));
  i >>= 1;
  i = __rsub(i, load(0x5f3759df));
  vector y;
  __v_insert(&y, i, 0);

  vector threehalfs = __v_ld(1.5F, MASK_FIRST);
  number = __v_fmul(number, y);
  number = __v_fmul(number, y);
  number = __v_fsub(__v_ld(1.5F, MASK_FIRST), number);
  number = __v_fmul(number, y);
  return number;
}
```

BUT if we know that the target CPU supports `i32` vector elements, we can instead compile the function like this:
```c
// require vec elt ty = F32 or I32
vector rsqrt(vector number)
{
  vector y = number; // in this case: performs f32 bitcast into 32bit int reg
  vector x2 = __v_fmul(number, __v_ld(0.5F, MASK_FIRST));
  y = __v_shr(i, 1);
  y = __v_sub(__v_ld_brdcst(0x5f3759df), i);

  vector threehalfs = __v_ld(1.5F, MASK_FIRST);
  number = __v_fmul(x2, y);
  number = __v_fmul(number, y);
  number = __v_fsub(__v_ld(1.5F, MASK_FIRST), number);
  number = __v_fmul(number, y);
  return number;
}
```
and now, this function can be used in both vectorized loops that require rsqrt, and for scalar floats;
this might also have better performance on some implementations



TODO: 
- document SCALAR bit
- float immediates
- standard vector & float ABIs


## Other Architectures
RVV: https://riscv.org/wp-content/uploads/2024/12/15.20-15.55-18.05.06.VEXT-bcn-v1.pdf

TODO
