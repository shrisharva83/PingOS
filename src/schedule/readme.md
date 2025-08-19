# Process Creation: `fork_process()`

## Purpose
`fork_process()` creates a new process by duplicating the current process or by setting up a new kernel thread.

## How it Works
The process of creating a new task involves several key steps:

1.  **Disable Preemption**: Preemption is disabled to prevent race conditions during process creation. This ensures that the process of duplicating the task is atomic and safe.

3.  **Allocate Resources**: Kernel memory is allocated for the new task's control block, known as `task_struct_t`. This structure holds all the necessary information about the new process.

4.  **Initialize Context**: The CPU registers and context for the new task are initialized.

5.  **Handle Task Type**: The function's behavior diverges based on whether a normal process is being forked or a kernel thread is being created:
    * **Kernel Thread (`PF_KTHREAD`)**: If flagged as a kernel thread, the CPU context is set up to begin execution of a specified function with its arguments.
    * **Normal Process Fork**:
        * The parent's CPU register state is copied to the child.
        * The child's return register is set to `0` to allow the child to distinguish itself from the parent.
        * The parent's virtual memory layout is copied to the child using `copy_virt_mem()`.

6.  **Set Task Properties**:
    * The new task's priority and flags are copied from the parent.
    * The task state is set to "running".

7.  **Set Up Execution**:
    * The program counter in the new task's context is set to `return_from_fork` to ensure proper resumption of execution.
    * The stack pointer is set to point to the new task's saved register context.

8.  **Finalization**:
    * The newly created task is added to the global task list.
    * The global task count is incremented.

9.  **Enable Preemption**: Preemption is re-enabled, allowing the scheduler to manage tasks again.

10.  **Return Value**:
    * The function returns the new task's identifier (PID) upon success.
    * If memory allocation fails, it returns `-1`.

# Process Transition: `move_to_user_mode()`

## Purpose
`move_to_user_mode()` prepares the current task to transition from its current state in kernel mode to an unprivileged user mode.

## How it Works
The transition to user mode involves a series of steps to set up the necessary context and memory for the user program:

1.  **Clear Saved Registers**: The saved registers in the process control block are cleared and zeroed out. This ensures that the user-mode execution starts with a clean slate.

2.  **Set Program Counter (PC)**: The program counter is set to the entry point of the user program. This is the address where execution will begin in user mode.

3.  **Configure Processor State**: The processor state register (`proc_state`) is configured to `EL0t` (ARM Exception Level 0, Thread mode). This switches the CPU to the lowest privilege level, which is required for user-mode execution.

4.  **Initialize Stack Pointer**: The stack pointer is initialized to a fixed user-mode stack location. The provided information specifies a 2-page offset for this location.

5.  **Allocate and Load Program**:
    * A page in user memory is allocated to hold the program code.
    * The executable image is copied from its source into this newly allocated user memory page.

6.  **Set Page Global Directory (PGD)**: The Page Global Directory (PGD) is set for the current task. This crucial step enables virtual memory and allows the task to access its own private address space.

7.  **Return Value**:
    * The function returns a success or failure status. The result depends on whether the memory allocation for the user program was successful.
  
# Utility Function: `get_ptr_to_regs()`

## Purpose
`get_ptr_to_regs()` is a utility function that returns the memory location of the saved CPU register context (`proc_regs`) for a given task. This context is stored on the task's kernel stack.

## How it Works
The function calculates the pointer to the register context based on a simple offset calculation:

1.  It starts with the base address of the task's kernel stack.
2.  It then applies an offset, which is calculated as `(thread_size - sizeof(proc_regs))`.
3.  This calculation effectively finds the exact location where the CPU registers were saved on the kernel stack, which is typically at the top of the stack.

By returning this pointer, the function allows other parts of the kernel to access and manipulate the task's saved register state, which is essential for context switching and returning from system calls or interrupts.
## Submodule: Memory Management and Page Handling

This submodule implements **basic page allocation, virtual memory mapping, and page fault handling** for the minimalistic Raspberry Pi 5 kernel using its ARM-based MMU.

---

### 1. Page Allocation

- **`allocate_free_page()`**:
  - Scans a bitmap `page_map[]` that tracks free and used physical memory pages.
  - Marks the first available page as used (1).
  - Calculates the physical start address using `PHYSICAL_SECOND_START` plus offset.
  - Clears memory for the allocated page in virtual address space (`mem_init_zero`) to avoid residual data.
  - Returns the physical address of the allocated page or 0 if no free pages remain.

- **`free_page()`**:
  - Marks the page as free in the bitmap by setting corresponding entry to 0, enabling reuse.

- **`allocate_kernel_page()`**:
  - Wrapper calling `allocate_free_page()` and converting to virtual address before returning.

- **`allocate_user_page()`**:
  - Allocates a free physical page and **maps** it into the user virtual address space for the specified task using `map_page()`.

---

### 2. Page Mapping

- **`map_page()`**:
  - Ensures the task has a valid Page Global Directory (PGD); allocates one if missing.
  - Walks the paging hierarchy:
    - PGD → PUD → PMD → PTE (levels of page tables in ARMv8 architecture).
  - For each level, `map_table()` either returns an existing table or allocates a new one.
  - Updates kernel pages bookkeeping in the task’s `mm` struct.
  - Calls `map_table_entry()` to map the specific virtual address to the allocated physical page with appropriate user-mode page flags.
  - Logs the user page for bookkeeping.

- **`map_table()`**:
  - Uses bit-shifting and masking to get the index in the given page table for the virtual address.
  - Allocates a new table page if no valid entry exists.
  - Returns the base address of the next level page table or existing entry.

- **`map_table_entry()`**:
  - Maps a virtual page table entry to a physical page address, along with access permissions and flags (like user mode).

---

### 3. Memory Copy (Fork Support)

- **`copy_virt_mem()`**:
  - Copies all user pages from the current process to a new destination task.
  - Allocates new user pages and duplicates memory content via `mem_copy()`.
  - Ensures process isolation by deeply copying address spaces during fork.

---

### 4. Page Fault Handler

- **`mem_abort_handler()`**:
  - Handles memory access faults triggered by attempts to access unmapped virtual addresses.
  - Decodes the fault status (`esr_val`) to check for **data aborts** due to page faults.
  - Tries to recover by allocating and mapping a new physical page to the faulted address.
  - Allows limited retries (counts `ind`), panics if unable to handle after 2 attempts.
  - Logs informational or panic messages accordingly.

