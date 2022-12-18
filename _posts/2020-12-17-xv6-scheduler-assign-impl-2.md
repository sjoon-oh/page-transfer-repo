---
title:  "XV6 Scheduler Assignment Implementation 2"
date:   2020-12-18 13:00:00
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

# [Operating System] XV6 Scheduler Assignment Implementation 2

## 구현하기에 앞서

우선 주요 목표는 다시 한 번, Round Robin의 디폴트 스케줄러를 MLFQ(Multi Level Feedback Queue) 스타일의 스케줄러로 변경하는 것입니다. 여기서 '스타일'이라고 함은, 완벽한 MLFQ가 아니라 과제를 위해 몇 가지 조건이 추가되었기 때문에 사용한 용어입니다. 이에 대한 조건은 이 [포스트](https://sjoon-oh.github.io/archivers/xv6-scheduler-assign-guide)에서 확인할 수 있습니다.

MLFQ는 거창하게 Multi Level이 붙었지만 단순히 여러 개의 Queue만 있으면 이 Queue 사이를 이동하는 행위만 정의해주면 됩니다. Queue는 일반적으로 Linked List를 이용하므로 저도 Linked List를 이용하는게 좋을 것 같습니다.

<!--more-->

일단 xv6는 머신 부팅 시 상위 메모리 주소에 proc table에 대한 정보를 쌓아두는데, 이 크기는 정해져 있습니다. 또 앞선 포스트에 언급되어 있는 코드 외에 다른 부분을 확인하면 알겠지만 부팅 시 trap table이나 필요한 부가 정보를 메모리에 차곡차곡 쌓아올립니다. 그 말인 즉슨, 예를 들어 NPROC으로 정의되어 있는 최대 64개의 프로세스를 임의적으로 개수를 늘리거나, proc table을 kernel domain의 다른 메모리 주소에 올려놓기가 힘들겁니다. 아마 그러려면 모든 소스를 뜯어고쳐야 할 지도 모를 일입니다. (일부 주요 함수는 어셈블리어로 작성되어 있으니 그것 마져 고쳐야 할 지도...) 가변적인 링크드 리스트를 이용한 직접적인 초기화는 힘듭니다.

따라서 가능한 접근법은 부팅 시의 각종 정보 (proc table 사이즈 등등)은 유지하되, 관리만 Queue를 사용해야 더 쉽게 구현이 가능하다는 결론이 나옵니다. xv6가 생성할 수 있는 최대 개수 NPROC과 proc table은 그대로 놔 두고, 추가적으로 Queue를 사용하여 scheduler() 만 고치는거죠.

## 변수 준비

리스트로 프로세스를 관리하려면 다음 proc 구조체를 가리키는 포인터 변수가 필요합니다. 문제라면 이걸 어디에 끼워넣느냐가 될겁니다. 어차피 최대 개수가 NPROC으로 정해져 있는 요소를 관리하면 되는 문제이므로 다음과 같은 방법이 있을 수 있을 겁니다.

- struct proc* 타입의 배열을 추가 생성하여 circular queue 형식으로 관리하는 방법
- proc 구조체를 직접 수정하여 안에 struct proc* 타입의 멤버를 선언하고 linked list 형태로 구현하는 방법

첫 번째 방법은 proc 구조체를 건드리지 않고 외부에서 포인터 형식으로 각 proc 구조체를 독립적으로 관리하는 방법입니다. 이는 기존 코드를 고치는 작업이 많이 필요하지 않고 프로세스 정보만 포인터로 잘 관리하면 되기 때문이죠. 고정 길이 배열로 struct proc*을 관리하기 때문에 우선적으로 비용이 많이 줄어듭니다. 

두 번째 방법은 구조체 안에 직접 struct proc* 타입의 포인터 멤버 변수를 선언하는 것입니다. 각 Level에 해당하는 Queue 관리를 위한 방법이라고 하면 될 것 같습니다. 데이터구조 수업에서 배우는 가장 일반적인 방법이고 배열로 관리하는 방법과는 달리 추가적인 배열을 사용하지 않는다는 장점이 있습니다. 물론, 멤버를 추가하는 것이기 때문에 proc 구조체 자체의 크기가 늘어난다는 점에서 space 측면에서는 큰 차이가 없을 것이라 예상됩니다.

첫 번째 방법의 경우 제가 제일 처음 시도한 방법이고 충분히 구현 가능합니다. 그러나 이 과제에서는 일반적인 Queue를 구현하는 것을 목적으로 두고 있기 때문에 본 포스트에서는 두 번째 방법을 사용하도록 하겠습니다. 

### struct proc

앞서 설명한 struct proc 구조체는 아래와 같습니다. 이전과 다음 포스트를 가리킬 변수가 필요하므로 struct proc* 타입의 p_prev, p_next의 변수를 선언해줍니다. 

```cpp
// proc.c
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


// EEE3535 Operating System
// Assignment #4 Scheduling
// Author: SukJoon Oh, acoustikue@yonsei.ac.kr
#define SUKJOON
#ifdef SUKJOON
  int           p_id;           // This p_id holds which queue it is located in.
  uint          p_ticks[3];     // ticks.
  uint          p_stp;          // tick calculation starting point
  struct proc*  p_prev;         // Maybe unnecessary?
  struct proc*  p_next;         // Move to the next element.
  int           p_intr;
#endif
};
```

이 과제에서는 총 세 개의 Queue를 구현하는 것을 목적으로 두고 있습니다. 스케줄링 과정에서는 한 프로세스가 어느 Level의 Queue에 위치해있는지 알아야 하는 경우가 있습니다. 그래야 어떤 특정한 시점이 현재 프로세스를 어느 Queue에 올려두어야 하는지 알아야 하기 때문이죠. Queue는 Linked List로 구현되어 있기 때문에 프로세스를 옮겨야 하는 경우 그 Queue에 process가 상주해 있는지 찾을 필요가 있습니다. 최대 프로세스의 수가 64개로 정해져 있기 때문에 최악의 경우에도 64번만 리스트를 타고 검사하면 됩니다. 그렇지만 직관적이고 빠른 search를 위해 현재 프로세스가 저장되어 있는 Queue의 레벨을 저장하는 p_id 변수도 추가해줍니다.

p_ticks와 p_stp 등은 현재 프로세스가 CPU를 얼마나 점유하고 있는지를 나타내는 변수입니다. 이 변수들은 console 창으로 보여야하는 것들의 과제 목표 중 일부라 이에 대한 설명은 넘어가도록 하겠습니다.


### Queue

세 개의 Queue level이 존재하므로 Queue를 관리하는 변수도 각각 세 개면 충분합니다. 

```cpp
// proc.c
typedef struct __queue {
  int           q_id;           // Queue holds its ID. Q2 holds 2 for instance.
  int           q_cnt;          // Element counter.
  struct proc*  q_head;         // First element position.   
  struct proc*  q_tail;         // Last element position.
} _queue;
```

별 내용은 없습니다. q_id는 각 Queue의 레벨을 나타내는 확인용 변수이고, q_cnt는 queue 안에 몇 개의 요소를 가지고 있는지 카운트하는 변수입니다.


## 함수 준비

이 정도면 간단히 새로운 스케줄러를 구성하는 데는 충분합니다. 나머지는 구현하면서 추가하도록 합시다. 

이 과제에서는 어느 부분에서 action을 추가할지가 관건입니다. *main()* 함수에서 *procinit()* 부분에서 proc table을 제일 먼저 건드리고 있으니 그곳에서 queue를 초기화하도록 하죠. 

```cpp
// Initializes struct __queue. 
// This function should be called when the system was on booting.
// procinit(), for instance. Any scope in front of scheduler() function will do.
void init_queue(_queue*, int);
```

함수 원형은 아래와 같습니다.

```cpp
//
// Make it all zeros.
void init_queue(_queue* q, int id) { 
  q->q_id = id; 
  q->q_head = q->q_tail = 0; 
  q->q_cnt = 0;
};
```

이걸 아래 부분에다 추가하도록 합시다.

```cpp
// proc.c
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

// 추가!!
#ifdef SUKJOON
  // Set all values to zeros.
  init_queue(&q2, Q2);
  init_queue(&q1, Q1);
  init_queue(&q0, Q0);
#endif
}
```

코드의 내용은 별 것 없습니다. 각 Queue를 나타내는 구조체를 받아다가 모두 0으로 초기화시킵니다. 물론, 각 q_id는 Queue level 마다 맞게 설정해 주어야 하니 Q2, Q1, Q0를 인자로 넣어줍니다. 매크로 원형은 다음과 같아요.

```cpp
#define Q2    2
#define Q1    1
#define Q0    0
```

그리고 미리 enqueue와 dequeue 작업을 할 수 있도록 함수를 만들어주죠. 간단히 q_head와 q_tail, 그리고 각 프로세스안의 포인터 변수를 서로 연결해주는 작업입니다.

```cpp
// Queue controls
// 
int is_empty(_queue* q) { return (q->q_cnt == 0); } 

struct proc* get_head(_queue* q) { return q->q_head; }
struct proc* get_tail(_queue* q) { return q->q_tail; }
int get_cnt(_queue* q) { return q->q_cnt; }

// Simple enqueueing function.
struct proc* enqueue(_queue* q, struct proc* p) {
  color(p, q->q_id); // color it first
  
  if (is_empty(q)) { ground(p); q->q_head = q->q_tail = p; }
  else {
    q->q_tail->p_next = p;
    p->p_prev = q->q_tail;
    p->p_next = 0;
    q->q_tail = p;
  }

  q->q_cnt++;  
  return p;
}

// Simple dequeueing function.
struct proc* dequeue(_queue* q) {
  struct proc* p = q->q_head;
  if (is_empty(q)) return 0;

  // When single element exists
  if (q->q_cnt == 1) q->q_head = q->q_tail = 0;
  else {
    q->q_head = p->p_next;
    q->q_head->p_prev = 0;
  }
  uncolor(p); // remove color
  ground(p);

  q->q_cnt--;
  return p;
}
```

여기서 is_empty()는 queue가 비었는지 검사하는 함수입니다. 비었다면 1을 반환하기 때문에 미리 오류를 방지하도록 작성했습니다. 이 외에도 Queue 상태를 확인하도록 하는 함수들을 미리 만들어둡니다. color 함수와 uncolor 함수는 이 프로세스가 현재 어느 Queue에 존재하는지 기록/해제하는 함수입니다. 단순히 p_id를 수정해주면 되겠죠. ground 함수는 각 포인터를 0으로 초기화해줍니다.

```cpp
int color(struct proc* p, int id) {  p->p_id = id; return id; }
int uncolor(struct proc* p) {  p->p_id = UD; return UD; }
int ground(struct proc* p) { p->p_next = p->p_prev = 0; return 0;}
```

## allocproc()

다시 한 번 상기하자면, 목적은 scheduler() 함수를 교체하는 것입니다. 이전 포스트에서 보았듯, for 문을 돌면서 Round Robin 형태로 프로세스를 순차적으로 CPU에 제공하는 것을 볼 수 있습니다.

이 과제에서 요구하는 MLFQ에서는 프로세스가 제일 처음 만들어질 경우 가장 interactive 한 Queue, 즉 Q2에 먼저 집어넣도록 요구합니다. 따라서 프로세스가 생성되는 코드를 찾아 Q2에 밀어넣도록 합시다.

```cpp
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

#ifdef SUKJOON
  enqueue(&q2, p);
  p->p_ticks[Q2] = p->p_ticks[Q1] = p->p_ticks[Q0] = 0;
  p->p_stp = ticks;
  p->p_intr = 0;
#endif

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

*allocproc()* 함수는 본래 UNUSED로 state되어 있는 struct proc을 proc table에서 찾아 프로세스를 할당합니다. 오랜 만에 goto 문을 볼 수 있네요. 여기서 프로세스가 만들어지므로 다른 건 볼 필요 없고, Q2에 넣어주는 작업만 하면 됩니다. 아까 작성한 enqueue 함수를 사용하도록 하죠. 

xv6를 다루다 보면 예상하지 못한 곳에서 panic이 일어나는 경우가 잦습니다. 대부분의 경우 access하면 안되는 메모리에 접근한 경우입니다. enqueue 함수는 단순히 새로 만들어진 struct proc 의 주소만 queue에 기록하는 것이므로 별 다른 에러를 발생하지 않지만 proc의 내부 변수에 접근하는 경우는 신중해야 합니다. 따라서 이 경우 acquire과 release 함수를 적극 사용하는 것이 좋습니다.

혹시 모를 경우를 대비해 enqueue도 acquire과 release 사이에 위치시켰습니다.


## freeproc()

프로세스가 종료되는 경우 *freeproc()* 함수가 호출됩니다. 여기에서 새로운 함수가 필요한데, 현재 프로세스가 위치해 있는 Queue로 부터 자기 자신을 제거하는 remove 기능입니다. 물론 Queue의 구조 자체가 FIFO 형태이지만 MLFQ에서는 추가적으로 Queue 중간 요소에 대해 제거를 담당하는 기능도 필요합니다. Queue 사이에서는 Round Robin 형태로 돌아가면서 CPU 를 점유하기 때문입니다. 먼저 다 실행해버린 프로세스는 그때 그때 제거해야하죠.

Linked List 이기 때문에 remove는 별 다른 방법이 없습니다. 순차적으로 순회하면서 해당 프로세스를 찾아 제거해야합니다. 

```cpp
// This function removes a specific element from queue.
// It is necessary operation, since a process located in the middle of the queue
// might be freed early. It is not desirable to hold a dead process information
// in q1 or q2, so it should be removed.
struct proc* remove(_queue* q, struct proc* tp) { 
  struct proc* np = q->q_head;
  // search
  if (is_empty(q)) { return 0; } // case when empty
  while (np != tp && np != 0) np = np->p_next; // search for target
  if (np == 0) { return 0; } // not found

  if (q->q_cnt == 1) { q->q_head = q->q_tail = 0; }
  else {
    if (np->p_prev != 0) np->p_prev->p_next = np->p_next;  // not in front
    if (np->p_next != 0) np->p_next->p_prev = np->p_prev;  // not in rear
    if (np == q->q_head) q->q_head = np->p_next; 
    if (np == q->q_tail) q->q_tail = np->p_prev; 
  }

  // Delete the information needed for queueing.
  uncolor(np);
  ground(np);

  q->q_cnt--;
  return np;
}
```

process id로 찾는 것이 아닌 그 struct proc 자체의 주소가 동일한지 파악하는 것이 더 빠릅니다. while 구문에서 확인할 수 있듯이 next pointer로 이동하면서 동일한 주소값을 가지는지 보고 있습니다.

```cpp
// proc.c
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
#ifdef SUKJOON
#define RATIO(X) 100 * X / (p->p_ticks[Q2] + p->p_ticks[Q1] + p->p_ticks[Q0])
  
  printf("%s (pid=%d): Q2(%d%%), Q1(%d%%), Q0(%d%%)\n",
         p->name, p->pid, 
         RATIO(p->p_ticks[Q2]), RATIO(p->p_ticks[Q1]), RATIO(p->p_ticks[Q0]));

  p->p_ticks[Q2] = p->p_ticks[Q1] = p->p_ticks[Q0] = 0;
  p->p_stp = 0;
  p->p_intr = 0;

  if (p->state == ZOMBIE) {
    if (is_q2(p)) remove(&q2, p);
    else if (is_q1(p)) remove(&q1, p);
    else if (is_q0(p)) remove(&q0, p);
  }
#else
  // Print out the runtime stats of queue occupancy.
  printf("%s (pid=%d): Q2(%d%%), Q1(%d%%), Q0(%d%%)\n",
         p->name, p->pid, 0, 0, 0);
#endif

  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

*freeproc()*에서는 우선 ZOMBIE 상태인 경우에만 Queue에서 제거시킵니다. UNUSED나 그외의 경우는 *scheduler()* 함수 안에서 한번에 처리하도록 합니다.


## scheduler()

이제 본격적으로 *scheduler()* 함수를 재작성 해보겠습니다. 앞선 포스트에서 확인할 수 있듯이 kernel 프로그램 시작점 *main()*은 끝에 *scheduler()*를 호출하는 것으로 함수가 끝납니다. 외부 인터럽트, 또는 치명적인 오류(잘못된 메모리 접근 등등)가 있는 경우에는 *scheduler()* 함수가 멈추면서 프로그램이 종료됩니다. MLFQ도 이러한 조건을 일단 만족해주어야 합니다.

```cpp
void
scheduler(void)
{
#ifdef SUKJOON
  mlfq_like();
#else
}
```

헷갈리지 않게 *mlfq_like()* 함수를 별도로 만들어 작성해 보도록 하겠습니다. 


### mlfq_like(): Q2와 Q1

*mlfq_like()* 함수는 다행스럽게도 두 개의 Queue에 대해서만 요소 이동을 수행해주면 됩니다. 조건에서는 세 개의 Queue가 있지만 Q0는 SLEEP state의 process를 기록하는 용도이기 때문에 사실 Q0는 Queue 형태가 아니라도 상관없습니다. 오히려 SLEEP 상태에서 벗어나는 경우 그 process를 이동시켜야 하기 때문에 search operation을 하는 경우 리스트를 다 훓어야 하는 상황이 생길겁니다. 

일단은 조건에 맞추어 Q0도 Queue 형태를 갖게끔 해 봅시다.


```cpp
void mlfq_like() {
  struct proc *p;
  struct proc* rt = get_head(&q2); // run this
  struct proc* pb = get_head(&q1); // priority boost role (modified)
  struct cpu *c = mycpu(); // cpu information
  c->proc = 0;
```

세 개의 포인터 변수를 선언해 주겠습니다. struct proc* p는 기존 scheduler 함수와 동일하게 proc table을 순회하는 용도로 사용하는 변수입니다. struct proc* rt는 *run this* 약자로 다음 CPU에 넣어 줄 process를 가리키는 변수입니다. struct proc* pb는 priority boost의 약자로 Q1에서 머물고 있는 프로세스를 주기적으로 실행시키기 위한 변수입니다. 

*main()*에서 첫 프로세스를 만들면 Q2에 올라가게 됩니다. 이는 당연한게, 앞서 *procinit()* 함수를 수정하면서 제일 먼저 Q2에 집어넣도록 만들어 두었기 때문입니다. 따라서 scheduler 함수에 처음 진입하게 되면 적어도 하나의 프로세스는 Q2에 올라가 있는 상태가 되므로 Q2의 프로세스를 먼저 CPU에 제공해주면 됩니다. 

```cpp
  for(;;){
    intr_on(); // Avoid deadlock by ensuring that devices can interrupt.

    // If the system is on a boot at first, the queue will hold zero pointer. 
    // Kerneltrap initiates when accessing an unknown process, 
    // thus rt should never be zero.
    if (rt != 0) {
      acquire(&rt->lock);

      for(p = proc; p < &proc[NPROC]; p++) {
        // This is for searching.
```

인터럽트를 enable 시켜주고 acquire을 해 줍니다. *rt* 변수는 항상 CPU에 던저 줄 process를 가리키고 있으니 *rt*에 대한 lock을 aquire 해 줍니다. 물론 rt가 비정상적인 값이면 panic을 마구 뿜어대기 때문에 이에 대한 예외처리도 같이 해 줍니다. 

```cpp
        else if (  p == rt && is_q2(p) ) { // case when it is time for q2 to run.
          if(p->state == RUNNABLE) { __CONTEXT_SWITCH__(c, p); }
          else if (p->state == SLEEPING || p->state == ZOMBIE) { __RE_MOVE___(p, &q2, &q0); }
          else if (p->state == UNUSED) { remove(&q2, p); }
          break;
        }
```

priority boost가 아닌 경우, 즉 *rt*를 CPU에 제공해 주는 경우를 먼저 보겠습니다. 코드는 굉장히 간단하죠? SLEEPING과 UNUSED의 경우는 scheduler 함수 안에서 처리하는 것이 더 간단하기 때문에 else if 로 넣어줍니다. 만일 proc table을 순회하면서 찾은 process 정보가 state가 RUNNABLE 하고 Q2안에 존재한다면 p를 CPU에게 제공해주면 됩니다. 

context switch 시키는 코드는 기존 scheduler 함수에서 사용된 것과 동일합니다. 너무 길어서 매크로로 작성했어요. 간단하게 봅시다. 

```cpp
#define __CONTEXT_SWITCH__(C, P) \
                                        do { P->state = RUNNING; C->proc = P; swtch(&C->context, &P->context); C->proc = 0; } while(0)
#define __DE_MOVE___(P, SRC, DST)       do { dequeue(SRC); enqueue(DST, P); } while(0)
#define __RE_MOVE___(P, SRC, DST)       do { enqueue(DST, remove(SRC, P)); } while(0)

```

*MOVE* 시리즈 매크로는 Queue 사이에 process를 간단하게 이동시키기 위한 매크로입니다. 앞서 구현해두었던 dequeue, enqueuem, remove를 잘 조합하면 위와 같이 작성 가능합니다. Q2와 Q1사이를 이동하는 경우에는 **d**equeue 시킨 후 **e**nqueue를 해주어야 하니 __DE_MOVE___를 사용하면 되고, SLEEP 상태에서 벗어나는 경우 Q0에서 Q2 또는 Q1으로 이동시켜야 하니 **r**emove와 **e**nqueue를 사용해아겠죠. 이 때는 __RE_MOVE____를 사용하면 됩니다. 

Q2에 존재하는 process 를 실행시켰으면 Q1도 동일하게 실행시켜주면 되겠지요. 큰 틀에서 코드는 동일합니다. p가 현재 돌려야 할 process, *rt*이고 Q1에 존재하면 context switch 시켜주면 됩니다. 이는 Q2에 프로세스가 존재하지 않을 때 아래 level Queue를 보아야 하는 경우입니다. 

```cpp
        else if (  p == rt && is_q1(p) ) { // case when it is time for q2 to run.
          if(p->state == RUNNABLE) { __CONTEXT_SWITCH__(c, p); }
          else if (p->state == SLEEPING || p->state == ZOMBIE) { __RE_MOVE___(p, &q1, &q0); }
          else if (p->state == UNUSED) { remove(&q1, p); }  
          break;
        }
```


### mlfq_like(): Priority Boost

조금 복잡한 경우는 Priority Boost인 경우입니다. RUNNABLE이 아닌 state에서는 위의 Q2와 Q1의 경우와 동일하지만 *pb*가 가리키는 값을 설정하는 과정에서 부가적인 행동을 해 주어야 합니다. *pb*는 *rt*와 독립적으로 업데이트 해 주어야 하므로 context switch가 끝난 직후의 시점에서 새로운 값으로 바꾸어주는게 편합니다. *pb*는 Q1 안에서 Round Robin 형식으로 돌아가기 때문에 리스트의 끝 (p_next 포인터가 0로 묶여있는 경우)에 다다르면 Q1의 제일 처음으로 바꾸어 주면 됩니다. 조금은 복잡하지만 또 생각해보면 그렇게 어려운 경우는 아닙니다.

```cpp
        // First case: priority boosting
        // This assignment does not actually implement priority boosting, but
        // a different and similar way of preventing starvation.
        if (pb != 0 && p->p_next == 0 && is_q2(p)) {
          // This is the case when rt(run this pointer) is at the end of the q2.
          // It should run the process located in q1, but not changing its position.
          if (pb->state == RUNNABLE) {
            __CONTEXT_SWITCH__(c, pb); // defined in macro.
            pb = pb->p_next;
            if (pb == 0) pb = get_head(&q1);
          }
          else if (p->state == SLEEPING || p->state == ZOMBIE) { __RE_MOVE___(p, &q1, &q0); }
          else if (p->state == UNUSED) { remove(&q1, p); }
          break;
        }
```

### mlfq_like()

이 모든걸 합하면 아래와 같이 작성됩니다. 

```cpp
void mlfq_like() {
  struct proc *p;
  struct proc* rt = get_head(&q2); // run this
  struct proc* pb = get_head(&q1); // priority boost role (modified)
  struct cpu *c = mycpu(); // cpu information
  c->proc = 0;

  for(;;){
    intr_on(); // Avoid deadlock by ensuring that devices can interrupt.

    // If the system is on a boot at first, the queue will hold zero pointer. 
    // Kerneltrap initiates when accessing an unknown process, 
    // thus rt should never be zero.
    if (rt != 0) {
      acquire(&rt->lock);

      for(p = proc; p < &proc[NPROC]; p++) {
        // This is for searching.

        // First case: priority boosting
        // This assignment does not actually implement priority boosting, but
        // a different and similar way of preventing starvation.
        if (pb != 0 && p->p_next == 0 && is_q2(p)) {
          // This is the case when rt(run this pointer) is at the end of the q2.
          // It should run the process located in q1, but not changing its position.
          if (pb->state == RUNNABLE) {
            __CONTEXT_SWITCH__(c, pb); // defined in macro.
            pb = pb->p_next;
            if (pb == 0) pb = get_head(&q1);
          }
          else if (p->state == SLEEPING || p->state == ZOMBIE) { __RE_MOVE___(p, &q1, &q0); }
          else if (p->state == UNUSED) { remove(&q1, p); }
          break;
        }
        // For this operation, pb pointer exists for always pointing at the process ready to 
        // run at the next priority boost.

        else if (  p == rt && is_q2(p) ) { // case when it is time for q2 to run.
          if(p->state == RUNNABLE) { __CONTEXT_SWITCH__(c, p); }
          else if (p->state == SLEEPING || p->state == ZOMBIE) { __RE_MOVE___(p, &q2, &q0); }
          else if (p->state == UNUSED) { remove(&q2, p); }
          break;
        }
        else if (  p == rt && is_q1(p) ) { // case when it is time for q2 to run.
          if(p->state == RUNNABLE) { __CONTEXT_SWITCH__(c, p); }
          else if (p->state == SLEEPING || p->state == ZOMBIE) { __RE_MOVE___(p, &q1, &q0); }
          else if (p->state == UNUSED) { remove(&q1, p); }  
          break;
        }
      }  
      release(&rt->lock);
    }
    // run_this() function is the key point of this function. 
    // This is the one that actually decides what to run next.
    rt = run_this(rt);    
  }
}
```

### run_this()

*mlfq_like()* 함수에서 빠진 것이 있습니다. 바로 다음 *rt*, 어떠한 프로세스를 돌릴 건지 *rt* 값을 업데이트 해 주는 부분이죠. 조건문이 많아 아래 코드는 복잡해 보이지만 간단합니다. 순서대로 따라가 보겠습니다.


```cpp
// 
// This function is responsible in deciding the next process to run.
struct proc* run_this(struct proc* p) {
  struct proc* rp;
  
  // If the process is fixed to 0, the kernaltrap will be on. 
  // Thus, it should be prevented.
  if (p == 0) return get_head(&q2);
  acquire(&p->lock);

  if (p->p_next == 0) { // case when at the end of loop
    if (!is_empty(&q2)) rp = get_head(&q2);
    else rp = get_head(&q1);
    // If it is the end of the queue, 
    // the next process to run is always the head of the queue.
    // This holds true only when q2 is not empty.
  } else {
    if (!is_empty(&q2)) { // Case when q2 is empty, which is q1's turn.
      if (is_q1(p)) rp = get_head(&q2);
      else rp = p->p_next;
    } else rp = p->p_next;
  }
  release(&p->lock);
  return rp;
}
```

인자로 들어오는 *p*는 기존 *rt*값이 들어와주면 됩니다. proc 구조체 값을 읽는 것이므로 혹시 모를 panic을 대비 해 acquire lock 시켜줍니다. 

다음 요소를 가리키는 p_next가 0으로 묶여 있다면 리스트의 끝에 다다른 것 일테니, 이 경우는 맨 처음 요소로 업데이트 해 줍니다. 물론, Queue가 비어있지 않은 것을 확인하고 업데이트 해 주어야 합니다. Q2가 비어있는 경우 Q1의 head 요소로 업데이트 해 주면 되고, Q2가 비어있지 않는 경우에는 Q2가 우선순위를 가지고 있으므로 무조건 Q2로 업데이트 시켜줍니다. 그래서 크게 Q2가 비어있는지의 여부가 가장 외부에 있는 if의 조건으로 들어갑니다. 

*run_this()* 는 *rt*만 업데이트 할 뿐, priority boosting *pb*는 scheduler 함수 안에서 독립적으로 업데이트 됩니다. 


## yield()

*yield()* 함수는 CPU가 설정된 time slice 가 지나면 호출되는 함수 입니다. 원형을 보면 단순합니다. 현재 실행중인 proc의 정보를 가져오고 state를 RUNNABLE로 교체합니다. 앞서 보았던 *sched()* 함수는 scheduler 로 돌아가는 함수입니다. 

```cpp
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;

  sched();
  release(&p->lock);
}
```

이 과제에서 구현하는 MLFQ는 Q2에 process를 실행하는 경우 한 time slice를 모두 사용한다면 Q1으로 내려야 합니다. 그러기 위해서는 현재 실행중인 proc이 Q2에 있어야 합니다. 만일 Q1의 proc을 돌리고 있다면 SLEEP state로 변하지 않는 이상 그대로 Q1에 남아있으면 되기 때문에 건들지 않아도 됩니다. 하나의 if문으로 control이 가능합니다. 

p_intr 변수는 조금 후에 설명하겠습니다.

```cpp
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;

#ifdef SUKJOON
  if (is_q2(p) && p->p_intr == 0) __RE_MOVE___(p, &q2, &q1);
  //if (is_q0(p)) __RE_MOVE___(p, &q0, &q1);
  if (p->p_intr == 1) p->p_intr = 0;
#endif

  sched();
  release(&p->lock);
}
```

## interrupt: syscall()

Interrupt가 걸린다는 이야기는 syscall 호출이 일어난다는 말입니다. Interrupt가 잦을 수록 (예: 사용자와의 상호 작용, 키보드를 누르는 행위) Interactive한 작업이 됩니다. Interactive한 작업은 Q2위에 두어서 먼저 처리해야 사용자가 delay를 거의 느낄 수 없겠죠. 따라서 Q0 또는 Q1에 있는 process 중 syscall을 호출하는 process는 Q2로 바로 옮겨두어야 합니다. 

*syscall()* 함수는 *syscall.c* 파일에 정의되어 있습니다. 우선, 이제까지 작업한 변수나 함수의 원형은 *proc.c*에 정의되어 있으므로 extern으로 먼저 알려줍니다.

```cpp
// syscall.c
extern _queue q2;
extern _queue q1;
extern _queue q0;

extern int             is_q2(struct proc*);
extern int             is_q1(struct proc*);
extern int             is_q0(struct proc*);

extern struct proc*    enqueue(_queue*, struct proc*);
extern struct proc*    dequeue(_queue*);
extern struct proc*    remove(_queue*, struct proc*);
extern void            show_queue_status();
extern int             record_tick(struct proc*, int); 
```

 그리고 syscall 함수 중간에 옮겨주는 작업을 추가해 주면 완성입니다.

 ```cpp
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
  
#ifdef SUKJOON
  acquire(&p->lock);
  if (is_q2(p)) { }
  if (is_q1(p)) { remove(&q1, p); enqueue(&q2, p); }
  if (is_q0(p)) { remove(&q0, p); enqueue(&q2, p); }
  p->p_intr = 1; // Mark that interrupt has been occurred.
  release(&p->lock);
#endif
}
 ```
