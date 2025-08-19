This part deals with the process creation(forking) and transition from kernel space into unser mode

Functions:
1.fork_process()
purpose: Creates a new process by setting up a kernel thread
how it works: 1.Disables preemption to avoid race conditions
2.
