# start_kernel part I

start_kernel routine is the most important routine of linux startup, it includes many subroutine of the initialization, it's devided into several parts to study, this is the first part.

```start_kernel

init/main.c:520

asmlinkage void __init start_kernel(void)
{
    ......
}
```

# Links