- [gcc __sync_fetch_and_add反汇编](#gcc-__sync_fetch_and_add反汇编)
- [gcc在命令行include一个头文件](#gcc在命令行include一个头文件)
- [基础类型sizeof](#基础类型sizeof)
- [gcc __builtin高级用法](#gcc-__builtin高级用法)

# gcc __sync_fetch_and_add反汇编
```c
int global_int = 0;

int main()
{
    __sync_fetch_and_add( &global_int, 1 );
}
```

```
gcc -O2 -march=mips64 -mabi=64 atomic.c && objdump -d a.out

00000001200008b0 <main>:
  1200008b0: 3c030002 lui v1,0x2
  1200008b4: 0079182d daddu v1,v1,t9
  1200008b8: 64638760 daddiu v1,v1,-30880
  1200008bc: dc628050 ld v0,-32688(v1)
  1200008c0: 0000000f sync
  1200008c4: c0410000 ll at,0(v0)
  1200008c8: 24210001 addiu at,at,1
  1200008cc: e0410000 sc at,0(v0)
  1200008d0: 1020fffc beqz  at,1200008c4 <main+0x14>
  1200008d4: 00000000 nop
  1200008d8: 0000000f sync
  1200008dc: 03e00008 jr ra
  1200008e0: 00000000 nop
```

```
gcc -O2 -march=octeon2 -mabi=64 atomic.c && objdump -d a.out

00000001200008b0 <main>:
  1200008b0: 3c030002 lui v1,0x2
  1200008b4: 0079182d daddu v1,v1,t9
  1200008b8: 64638760 daddiu v1,v1,-30880
  1200008bc: dc628050 ld v0,-32688(v1)
  1200008c0: 0000010f syncw
  1200008c4: 0000010f syncw
  1200008c8: c0410000 ll at,0(v0)
  1200008cc: 24210001 addiu at,at,1
  1200008d0: e0410000 sc at,0(v0)
  1200008d4: 1020fffc beqz  at,1200008c8 <main+0x18>
  1200008d8: 00000000 nop
  1200008dc: 03e00008 jr ra
  1200008e0: 00000000 nop
```

# gcc在命令行include一个头文件
```
gcc -include octeon-atomic.h simple-test.c
```

# 基础类型sizeof
```
sizeof char is 1
sizeof short is 2
sizeof int is 4
sizeof long is 8
sizeof long long is 8
sizeof void is 1
sizeof void * is 8
sizeof short int is 2
sizeof long int is 8
sizeof int long is 8
```

# gcc __builtin高级用法
https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#Other-Builtins
```c
#define foo(x)                                                    \
	({                                                            \
	  typeof (x) tmp = (x);                                       \
	  if (__builtin_types_compatible_p (typeof (x), long double)) \
		tmp = foo_long_double (tmp);                              \
	  else if (__builtin_types_compatible_p (typeof (x), double)) \
		tmp = foo_double (tmp);                                   \
	  else if (__builtin_types_compatible_p (typeof (x), float))  \
		tmp = foo_float (tmp);                                    \
	  else                                                        \
		abort ();                                                 \
	  tmp;                                                        \
	})
```
有了`__builtin_types_compatible_p`, 就可以根据数据宽度不同来做不同的动作, 比如
```c
#define __sync_fetch_and_add(ptr, value) \
({ \
	typeof(*(ptr)) octeon_ret; \
	if (__builtin_types_compatible_p(typeof(*(ptr)), long)) \
		octeon_ret = cvmx_atomic_fetch_and_add64((long *)(ptr), (value)); \
	else if (__builtin_types_compatible_p(typeof(*(ptr)), int)) \
		octeon_ret = cvmx_atomic_fetch_and_add32((int *)(ptr), (value)); \
	else \
		assert(sizeof(*(ptr)) == 8 || sizeof(*(ptr)) == 4); \
	octeon_ret; \
})
```
注意typeof的用法, typeof(*ptr)可以得到ptr指向数据的数据类型.
