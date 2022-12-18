---
title:  "XV6 Scheduler Assignment Implementation 1"
date:   2020-12-18 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
   show_overlay_excerpt: false
categories: 
   - Electronic Engineering
tags:
   - Electronic Engineering
   - C
   - xv6
toc: true
toc_sticky: true
breadcrumbs: true
---

# [Operating System] XV6 Scheduler Assignment Implementation

본 포스트는 제가 2020년 2학기에 수강한 운영체제론 수업에서 네 번째 프로그래밍 과제를 기반으로 하고 있습니다. 한 학기 동안 진행했던 총 7개의 과제 중 가장 많은 오류를 겪었고, 또 그만큼 OS에 대한 감각을 익히는데 가장 큰 도움이 되었던 과제였습니다. 지금부터 설명하는 xv6 구조에 더해 작성하는 코드의 의도를 파악하기 힘든경우 가볍게 넘어가셔도 무방하며, 과제에서 요구하는 가이드라인은 [xv6-scheduler-assign-guide](https://sjoon-oh.github.io/archivers/xv6-scheduler-assign-guide) 포스트의 Goal 부분을 참고하시면 이해에 도움이 되실 것 같습니다. 본 포스트에서는 Linux 환경에서 xv6를 사용하였으며, 설치 및 이용에 대한 설명은 제공하지 않습니다.

xv6는 RISC-V 프로세서를 기반으로 한 연습용 운영체체(OS)입니다. 2006년에 MIT PDOS lab에서 자체로 개발했습니다. xv6에 대한 자세한 사용법, 또는 가이드는 [링크](https://pdos.csail.mit.edu/6.828/2018/xv6.html)를 참고하면 좋을 것 같습니다. PDF 파일 형태로 제공하고 있으며 과제할 때 코드의 의도와 용도를 파악할 때 도움을 많이 받았습니다.

<!--more-->

## xv6 Default Scheduler

이 과제는 xv6의 스케줄러를 MLFQ 형태로 교체하는 것에 목적을 두고 있습니다. 본래 xv6는 Round Robin으로 프로세스를 관리합니다. Round Robin이란 순차적으로 돌아가면서 CPU를 점유할 기회를 주는 방법을 말합니다. 이에 대한 구현을 간단히 살펴보죠.

```cpp
// proc.c
#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "riscv.h"
#include "spinlock.h"
#include "proc.h"
#include "defs.h"

struct cpu cpus[NCPU];

struct proc proc[NPROC]; // 요놈
```

*proc.c* 파일의 최상단에 proc 이라는 이름의 구조체가 선언되어 있음을 볼 수 있습니다. proc 구조체는 하나의 프로세스를 나타내는 구조체라고 생각하면 편합니다. proc 구조체는 *proc.h* 헤더에 아래와 같이 선언되어 있습니다. 

```cpp
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

*state*, *parent*, 등등 굉장히 많은 멤버 변수들이 존재합니다. 일단 다른 것은 나중에 보기로 하고 저 *state* 변수가 가장 눈에 띕니다. 프로세스가 지금 어떤 상태인지 나타내는 아주 중요한 지표입니다. xv6 소스 안에 존재하는 프로세스를 관리하는 많은 함수들이 저 *state* 값을 읽고 다음 행동을 결정하기 때문입니다. 

열겨형(enum)으로 선언되어 있는 저 *procstate*는 아래와 같이 정의됩니다.

```cpp
enum procstate { UNUSED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
```

총 다섯 개의 상태를 열거하고 있고, 이름을 보면 직관적으로 각 프로세서가 어떤 상태에서 어떠한 값을 가지고 있을 지 예상이 됩니다. CPU를 차지하고 있는 프로세스는 RUNNING 상태일 것이고, 대기 중인 프로세스는 SLEEPING 상태이겠죠. ZOMBIE는 프로세스가 종료되었지만 아직 그에 대한 정보가 유지되고 있는 상태를 의미합니다. 

앞서 말했듯이 xv6는 Round Robin 형태로 프로세스를 관리합니다. 이를 구현하기 위해 앞서 언급한 proc 구조체가 필요한 것이죠. NPROC은 xv6가 최대로 관리할 수 있는 프로세스의 수를 의미하며 default로 64로 선언되어 있습니다. 이는 *param.h* 에서 확인할 수 있지요.

```cpp
// param.h
#define NPROC        64  // maximum number of processes
#define NCPU          8  // maximum number of CPUs
#define NOFILE       16  // open files per process
#define NFILE       100  // open files per system
#define NINODE       50  // maximum number of active i-nodes
#define NDEV         10  // maximum major device number
#define ROOTDEV       1  // device number of file system root disk
#define MAXARG       32  // max exec arguments
#define MAXOPBLOCKS  10  // max # of blocks any FS op writes
#define LOGSIZE      (MAXOPBLOCKS*3)  // max data blocks in on-disk log
#define NBUF         (MAXOPBLOCKS*3)  // size of disk block cache
#define FSSIZE       1000  // size of file system in blocks
#define MAXPATH      128   // maximum file path name
```

그러면 xv6 커널에서 부팅과 함께 프로세스를 어떻게 시작하는지 한번 보겠습니다. 우선 커널도 하나의 프로그램이니 시작점을 찾아 검색해봅시다. main 함수는 *main.c* 파일 안에 존재합니다. 총 30줄 정도 되는 짧은 코드를 볼 수 있습니다.

```cpp
// main.c

#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "riscv.h"
#include "defs.h"

volatile static int started = 0;

// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("EEE3535 Operating Systems: booting xv6-riscv kernel\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode cache
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}

```

함수 이름이 직관적이고 개발자가 친절하게 주석을 달아두어서 간략히 각 함숙가 무엇을 하고 있는지 알 수 있습니다. 이 또한 공식 xv6 book을 찾아보시면 자세한 설명이 제공되므로 참고하시면 좋을 것 같습니다.

교수님께서 과제 출제를 위해 추가한 몇 가지의 소스는 무시하고, 우선 저의 관심사는 프로세스의 관리, 구조체 proc 리스트의 관리입니다. 우선 시작점에서 page table initializing, trap vector initializing 등은 알아서 하겠거니 제쳐두고 *procinit()* 함수가 가장 눈에 띕니다. 여기서는 proc 리스트를 proc table 이라 부르고 있습니다.

```cpp
// initialize the proc table at boot time.
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");

      // Allocate a page for the process's kernel stack.
      // Map it high in memory, followed by an invalid
      // guard page.
      char *pa = kalloc();
      if(pa == 0)
        panic("kalloc");
      uint64 va = KSTACK((int) (p - proc));
      kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
      p->kstack = va;
  }
  kvminithart();
}

```

xv6에서 몇몇 함수들은 어셈블리어로 작성되어 있기 때문에 소스만 보고 의미를 파악하기 쉽지 않습니다. *procinit()* 에서는 for문으로 proc table을 돌면서 kalloc()으로 커널 level에서 메모리를 할당하고 먼저 메모리 상위 주소에 채워둡니다. 일단 main 함수의 초반에 proc table 을 이렇게 for문으로 돌면서 접근하고 있다는 것을 보고 넘어가도록 하죠. 

모든 함수 호출이 끝나면 main 함수의 마지막에 *scheduler()* 를 호출합니다. scheduler 함수는 *proc.c* 안에서 찾을 수 있습니다. 

```cpp
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;

  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
      }
      release(&p->lock);
    }
  }
}

```

xv6 스케줄러의 핵심은 이 *scheduler()* 함수입니다. 한 눈에 Round Robin 으로 돌아가고 있는 것이 보입니다. 외부 인터럽트가 발생하지 않는 이상 (예를 들어 xv6 강제 종료, ctrl + x + a) 조건 없는 for 루프를 돌면서 proc table 안의 프로세스 정보를 탐색합니다. 프로세스 정보를 읽어 올때는 *acquire()*, *release()*를 사용하여 다수 쓰레드가 접근하는 것을 막은 후, 상태가 RUNNABLE 이라면 RUNNING 상태로 교체 및 CPU에게 넘기는 형식입니다. 

자세히 볼 필요는 없고, swtch() 함수를 잠깐 확인하고 넘어갑시다. *swtch.S* 파일에서 확인할 수 있습니다.

```assembly
.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```

뭔가 장황하게 어셈블리어로 작성되어 있습니다. swtch() 함수는 context switch를 담당하는 함수로, 프로세스 정보를 CPU에게 던집니다. 

```cpp
swtch(&c->context, &p->context);
```

그러면 CPU는 프로세스를 time slice동안 실행하고 *sched()* 함수를 호출합니다. 주석에 자세히 설명되어 있듯, *sched()*는 *scheduler()*로 돌아가는 함수입니다. 

여기까지 scheduler() 함수에서 중점적으로 보아야 하는 점은 크게 두 가지 입니다. <u>scheduler() 함수가 어떻게 proc 정보를 이용하고 있는지 (Round Robin)</u>, 그리고 <u>어떠한 과정으로 CPU에게 프로세스를 넘기는지</u>겠지요. 목적은 커널 전체를 뜯어 고치는 것이 아니라 단순히 MLFQ 스타일의 스케줄링만 하도록 하는 것이기 때문에 제공된 소스를 충실히 활용할 예정입니다.

조건을 맞추기 위한 timer 및 syscall은 다음 포스트에서 계속 다루겠습니다.

[계속: XV6 Scheduler Assignment Implementation 2](https://sjoon-oh.github.io/archivers/xv6-scheduler-assign-impl-2)