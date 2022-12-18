---
title:  "XV6 Scheduler Assignment Guide"
date:   2020-12-17 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
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

# [Operating System] XV6 Scheduler Assignment Guideline

## Goal (Scheduling Policy)

- A new process is first placed in the Q2 by assuming that it would be an interactive one.
- If a process invokes a system call or voluntarily gives up the CPU during a timer interrupt interval, theprocess stays in the Q2. But, if a process occupies a whole time slice, then it relocates to the Q1.
- When a process is enqueued, it is placed at the end of list.
<!--more-->
- If a process buffered in theQ1makes a system call or voluntarily gives up the CPU, it immediatelymoves to the Q2. Otherwise, it continues to stay in the Q1.  
- When a process changes its state to a non-runnable state (i.e.,sleep or zombie), the process is placed in the Q0. Thus, scheduling is made only among processes in the Q2 or Q1 because those in theQ0arenot runnable processes.
- Since the Q2 includes interactive jobs, processes in this queue should have higher priority for scheduling than those in the Q1.
-  However,  executing only Q2 processes makes those in the Q1 starve for scheduling. Although thestarvation problem can be alleviated by introducing a priority boosting method, we will not exploit thepriority boosting in this scheduling policy.
- Instead, processes in the Q2 and Q1 are scheduled as follows. To simplify explanations, let us assumethat processes do not change their states or move between queues.
- Processes in the Q2 are sequentially executed in the order they are queued, i.e.,P1, P0, P4, and then P6. If the scheduler reaches the end of Q2 process chain (i.e., linked list), it lets one process in the Q1 run in the next time slice. Since the P3 is the head of Q1 list, this process is executed.
- After the P3, the scheduler again runs processes in the Q2 from P1 to P6. When the scheduler reaches the end of Q2 process chain again, it triggers the next process in the Q1 to run, i.e., P7.  When the Q1 has reached the end of list after the P7, it loops back to the head of queue. The P3 will be the nextprocess to run when it becomes the Q1’s turn to go.
- In reality, processes may dynamically hop around different queues and change their states, and thuslinked-list pointers of process queues must be carefully handled to avoid erroneous operations


## Implementation

To implement the described scheduling policy using multiple process queues, you first need to understand how the xv6-riscv kernel handles processes. All process-related operations in xv6-riscv occur in *kernel/proc.c*, so the *proc.handproc.c* files are the only ones you will have to read to understand how process scheduling in xv6-riscv works.

Process creation–The first process of xv6-riscv is created during boot time in theuserinit()function in *proc.c*. The first process (i.e., *initproc*) forks a child process and makes it run a shell program namedsh. Every new process except for the *initproc* is created via the *fork()* function in *proc.c*.   The *fork()* calls *allocproc()* to create and initialize a new process. Technically  processes  are  not  created  in  the  xv6-riscv  kernel.   They  are  statically  allocated  as  *struct proc proc[NPROC]* near  the  beginning  of *proc.c*,  where *NPROC = 64*. It  means  that xv6-riscv can handle only up to 64 processes at a time. 

Elements of the statically-allocated *proc* array are used for storing the information of active processes. If an array entry has *UNUSED* state,  it does not contain a valid process; a new process later can be allocated to this entry. If the entry has other states (i.e., *SLEEPING*, *RUNNABLE*, *RUNNING*, or *ZOMBIE*), it includes an active process. When  a  process  is  completed,  it  dies  in  the *freeproc()* function. The  xv6-riscv  kernel  sets  the corresponding entry of *proc* array as *UNUSED* to indicate that the entry is available for a new process


## Timer Interrupt

When a timer interrupt occurs, it falls to the *yield()* function in *proc.c*. This function makes the interrupted process give up the CPU and performs context switching. The *yield()* function may be a good place to check if the interrupted process has occupied the entiretime slice and thus needs to move to the Q1 if it was previously in the Q2. The *yield()* function is called either from *usertrap()* or *kerneltrap()* in *trap.c*.  Note fromthe code that a timer interrupt is called only when the CPU is running a user process.  If the CPU is idle or executing the kernel thread, the timer interrupt is silenced. Thus, when xv6-riscv runs short programs such as *echo*, *ls*, *andwc*, you may find that timer interrupt occur infrequently and aperiodically instead of having 1ms periodic intervals because these programs rarely occupy the CPU for extensive period of time.

The interval of timer interrupt is initialized in the *timerinit()* function *instart.c*. The default xv6-riscv defines *interval = 1000000*, and it says in a comment that it is about 1/10thof a second. It means the default time unit is 100ns. You should notice in your xv6-riscv copy that the *interval = 10000*,  which indicates the timerinterrupt interval is set to 1ms in this assignment. The wall-clock time in the xv6-riscv kernel can be probed by reading the ticks variable.  This variable increments every 1ms in the *clockintr()* function in *trap.c* regardless of xv6-riscv kernel activities or timer interrupts.

