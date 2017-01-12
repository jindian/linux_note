# start_kernl call stack

```start_kernl_call_stack

    start_kernel        # init/main.c:520
        |--smp_setup_processor_id        # init/main.c:496
        |--lockdep_init

```