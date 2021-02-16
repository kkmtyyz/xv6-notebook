# 5.14. pinit関数
この関数はプロセステーブルのロックを初期化する。

コメントの通り。

param.h
```c
#define NPROC        64  // maximum number of processes
```

proc.c
```c
struct {
  struct spinlock lock;
  struct proc proc[NPROC];
} ptable;

/* 略 */

void
pinit(void)
{
  initlock(&ptable.lock, "ptable");
}
```

## proc構造体
プロセスを表す構造体。
コメントの通り。

types.h
```c
typedef uint pde_t;
```

param.h
```c
#define NOFILE       16  // open files per process
```

proc.h
```c
enum procstate { UNUSED, EMBRYO, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };

// Per-process state
struct proc {
  uint sz;                     // Size of process memory (bytes)
  pde_t* pgdir;                // Page table
  char *kstack;                // Bottom of kernel stack for this process
  enum procstate state;        // Process state
  int pid;                     // Process ID
  struct proc *parent;         // Parent process
  struct trapframe *tf;        // Trap frame for current syscall
  struct context *context;     // swtch() here to run process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

