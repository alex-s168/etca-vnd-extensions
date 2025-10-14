# etca-variable-vlen-vector-ext

```c
// pseudocoe
size_t __setvl (comptime __vector_dtype_t dty, comptime size_t num, comptime size_t chunksize) {
  size_t supported = readcr(__VLEN_ + dty);
  if (num != 0) {
    if (supported < num) num = supported;
  } 
  return floordiv(num, chunksize);
}

// don't fault when can't acces ptr+(N>1) element,
// but will fault if can't access [ptr] element.
// every value after the first fault is UB
vector __v_ldnf(void* ptr);

// example compiler input:
size_t strlen(char const* s) {
  size_t o = 0;
  for (; *s; s++) o ++;
  return o;
}
// example compiler output:
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


// example compiler input:
__attribute__((hot))
float dot(float* a, float* b, size_t len) {
  if (__builtin_unlikely(len < 1024)) {}
  float o = 0;
  for (size_t i = 0; i < len; i ++)
    o += a[i] * b[i];
  return o;
}

// example compiler output:
float dot(float* a, float* b, size_t len) {
  // would insert a check for vec ext & fp32 vecs here

  // scalar floats are stored in vec regs too
  // since loop vlen is more than 1 float probably, remaining elements are zeroed
  vector sum = __v_mov_one(0.0f);
  while (true) { // unrolled(4) loop
    size_t num = __setvl (__VEC_F32, len, 4);
    if (num == 0) break;

    vector c, d;
    c = __v_ld(a); d = __v_ld(d); vector s1 = __v_fmul(c,d);
    // load ops support offset `addr_reg + C + vlen * N`
    c = __v_ld(a + __vlen*1); d = __v_ld(b + __vlen*1);   __v_fma(&s1,c,d);
    c = __v_ld(a + __vlen*2); d = __v_ld(b + __vlen*2);   __v_fma(&s1,c,d);
    c = __v_ld(a + __vlen*3); d = __v_ld(b + __vlen*3);   __v_fma(&s1,c,d);
    vector x = __v_fhsum(s1);
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
