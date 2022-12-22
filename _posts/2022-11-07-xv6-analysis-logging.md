---
title:  "[TA]xv6의 로깅 분석"
date:   2022-11-07 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
   show_overlay_excerpt: false
categories: 
   - TA
tags:
   - xv6
   - Operating System
   - Analysis
toc: true
toc_sticky: true
breadcrumbs: true
---

# [TA] xv6의 로깅(Logging) 분석

*이 포스트는 OSLab 연구 활동에서 작성한 개인 노트의 일부입니다. 공부 목적으로 작성된 내용이므로 많이 부족할 수 있습니다.*

## Overview 

본 글에서는 xv6 의 로깅 Logging 메커니즘에 대해 설명합니다.

## Logging Layer

xv6의 로깅(Logging) 의 작동을 확인하기에 앞서 xv6 파일 시스템의 로깅 레이어Logging Layer 를 먼저 짚고 넘어가겠습니다. xv6의 파일 시스템 구조는 다음 그림과 같으며, 로그는 세 번째 인덱스 2 블록에 위치하고 있습니다. 이에 대한 내용은 `mkfs`에서 확인할 수 있습니다. 이 섹션에서는 mkfs 에 의해 아래 그림에 대한 구조를 생성하는 과정에 대해 설명하겠습니다.

### `fs.img` : xv6 파일 시스템

xv6를 실행할 때 다음과 같은 명령어를 사용합니다.

```bash
# Launch XV6
$ make qemu-nox
```

위의 `make` 타겟(Target) `qemu-nox`를 확인하기 위해 `Makefile` 확인하면 QEMU 실행에 다음과 같은 `fs.img` 이름의 파일을 요구하는 것을 확인할 수 있습니다.

```
# ...

222: QEMUOPTS = -drive file=fs.img,index=1,media=disk,format=raw -drive file=xv6.img,index=0,media=disk,format=raw -smp $(CPUS) -m 512 $(QEMUEXTRA)
223: 

230: qemu-nox: fs.img xv6.img
231: 	$(QEMU) -nographic $(QEMUOPTS)

# ...
```

- `$(QEMU)` : 운영체제에 따른 QEMU 바이너리 경로
- `$(QEMUOPTS)` : QEMU 실행 옵션 지정

`fs.img` 의 레시피(recipe)는 아래와 같이 지정됩니다. 미리 컴파일된 사용자 프로그램에 대한 리스트가 `UPROGS`의 이름으로 존재하며, `mkfs`를 실행시키면서 인자로 `fs.img`, `README`, `UPROGS`를 지정하고 있습니다.

```
# ... makefile

168: UPROGS=\
169: 	_cat\
170: 	_echo\
171: 	_forktest\
172: 	_grep\
173: 	_init\
174: 	_kill\
175: 	_ln\
176: 	_ls\
177: 	_mkdir\
178: 	_rm\
179: 	_sh\
180: 	_stressfs\
181: 	_usertests\
182: 	_wc\
183: 	_zombie\
184: 
185: fs.img: mkfs README $(UPROGS)
186: 	./mkfs fs.img README $(UPROGS)

# ...
```

### `mkfs` : `fs.img` 의 생성

아래는 `mkfs.c` 에 정의된 `main` 함수를 나타내고 있습니다. `mkfs.c`의 코드는 `fs.img` 타겟이 실행하는 `mkfs` 바이너리로 컴파일됩니다. `main()` 에서의 `fsfd` 는 `fs.img` 를 작성할 파일 디스크립터 (File Descriptor) 입니다. `fsfd`는 전역으로 선언되어 있어 모든 함수가 접근이 가능합니다. `mkfs.c:87` 에서 `open()` 호출 시 첫 번째 인자로 `argv[1]` 을 지정하고 있으며, 이는 `Makefile:186` 에서 첫 번째 인자로 지정하는 `fs.img` 문자열입니다. 따라서 `mkfs` 바이너리의 첫 번째 인자는 파일 시스템의 이름 `fs.img`가 지정됩니다.

```c
67: int
68: main(int argc, char *argv[])
69: {
70:   int i, cc, fd;
71:   uint rootino, inum, off;
72:   struct dirent de;
73:   char buf[BSIZE];
74:   struct dinode din;
75: 
76: 
77:   static_assert(sizeof(int) == 4, 
        "Integers must be 4 bytes!");
78: 
79:   if(argc < 2){
80:     fprintf(stderr, "Usage: mkfs fs.img files...\n");
81:     exit(1);
82:   }
83: 
84:   assert((BSIZE % sizeof(struct dinode)) == 0);
85:   assert((BSIZE % sizeof(struct dirent)) == 0);
86: 
87:   fsfd = open(argv[1], O_RDWR|O_CREAT|O_TRUNC, 0666);
88:   if(fsfd < 0){
89:     perror(argv[1]);
90:     exit(1);
91:   }
```

아래 나타나는 `sb` 변수는 `struct superblock` 타입의 구조체로, 전역으로 선언되어 있습니다. `fsfd` 파일 디스크립터에 새로운 파일을 열었다면, 전역 변수 sb 의 내부를 초기화합니다. (`mkfs.c:93니103`) 초기화 과정에서 필요한 기본 값들은 `param.h` 에 정의되어 있습니다.

```c
// main(), mkfs.c
23: int nbitmap = FSSIZE/(BSIZE*8) + 1;
24: int ninodeblocks = NINODES / IPB + 1;
25: int nlog = LOGSIZE;

093:   // 1 fs block = 1 disk sector
094:   nmeta = 2 + nlog + ninodeblocks + nbitmap;
095:   nblocks = FSSIZE - nmeta;
096: 
097:   sb.size = xint(FSSIZE);
098:   sb.nblocks = xint(nblocks);
099:   sb.ninodes = xint(NINODES);
100:   sb.nlog = xint(nlog);
101:   sb.logstart = xint(2);
102:   sb.inodestart = xint(2+nlog);
103:   sb.bmapstart = xint(2+nlog+ninodeblocks);
104: 
105:   printf("nmeta %d (boot, super, log blocks %u 
         inode blocks %u, bitmap blocks %u) blocks 
         %d total %d\n",
106:          nmeta, nlog, ninodeblocks, nbitmap, 
         nblocks, FSSIZE);
```

```c
// param.h
01: #define NPROC        64  
        // maximum number of processes
02: #define KSTACKSIZE 4096  
        // size of per-process kernel stack
03: #define NCPU          8  
        // maximum number of CPUs
04: #define NOFILE       16  // open files per process
05: #define NFILE       100  // open files per system
06: #define NINODE       50  
        // maximum number of active i-nodes
07: #define NDEV         10  
        // maximum major device number
08: #define ROOTDEV       1  
        // device number of file system root disk
09: #define MAXARG       32  // max exec arguments
10: #define MAXOPBLOCKS  10  
        // max # of blocks any FS op writes
11: #define LOGSIZE      (MAXOPBLOCKS*3)  
        // max data blocks in on-disk log
12: #define NBUF         (MAXOPBLOCKS*3)  
        // size of disk block cache
13: #define FSSIZE       1000  
        // size of file system in blocks
```

참고로, 초기화 과정에서 사용되는 `xint()` 함수는 인텔 기반 Byte Order (Little Endian) 에 맞추기 위해 순서를 바꾸는 함수입니다.

```c
// xint(), mkfs.c
55: uint
56: xint(uint x)
57: {
58:   uint y;
59:   uchar *a = (uchar*)&y;
60:   a[0] = x;
61:   a[1] = x >> 8;
62:   a[2] = x >> 16;
63:   a[3] = x >> 24;
64:   return y;
65: }
```

`struct superblock` 타입 `sb` 변수 초기화 과정을 통해 다음과 같은 `fs.img`에 대한 정보는 다음과 같이 생성됩니다. 로깅 레이어(Logging Layer)에 대한 정보는 아래와 같습니다.

- 파일 시스템의 크기 : `FSSIZE` 1000
- 메타 데이터 블록 수 (`nmeta`) : 2
- `nlog` : 3 * `MAXOPBLOCKS` = 30
- `ninodeblocks` : `NINODES` / `IPB` + 1 = `NINODES` / 200 / ((`BSIZE / sizeof(struct dinode`))) + 1 = 200 / (512 / 64) + 1 = 26
- `nbitmap` : `FSSIZE` /(`BSIZE`*8) + 1 = 1

이를 통해 앞서 확인한 파일 시스템의 구조를 확인할 수 있습니다.

###  More about `mkfs`

xv6가 사용하는 `fs.img` 파일 시스템의 구조에 대한 파악은 위의 섹션으로 충분하나, 이 섹션에서는 `fs.img` 가 어떻게 초기화되는지 확인해보겠습니다. 

`mkfs`는 메타 데이터 블록 수 만큼 우선 블록을 할당하기 위해 이를 `freeblock` 변수에 기록합니니. `wsect()` 함수는 주어진 첫 번째 인자 섹터 숫자 (Sector Number)에 두 번째 인자 내용으로 `write()` 를 진행합니다. 즉, `mkfs.c:110` 에서와 같이 `FSSIZE` 만큼의 모든 섹터를 0으로 초기화됩니다.

```c
// mkfs.c, main()
108:   freeblock = nmeta;     
         // the first free block that we can allocate
109: 
110:   for(i = 0; i < FSSIZE; i++)
111:     wsect(i, zeroes);
```

다음 단계 `mkfs.c:113~115`는 Superblock 에 대한 내용을 1번 인덱스의 섹터 (Sector 1)에 기록합니다.


```c
// mkfs.c, main()
113:   memset(buf, 0, sizeof(buf));
114:   memmove(buf, &sb, sizeof(sb));
115:   wsect(1, buf);
```

이후 블록 하나 디렉토리 타입 (`T_DIR`)으로 할당 받고 (`mkfs.c:117`), 최상위 경로 (Root Directory)에 대한 정보를 기록합니다. 코드에 나타나는 `de`는 `struct dirent` 타입으로, 디렉토리 엔트리에 대한 정보를 기록합니다. 최상위 경로에 대해 필요한 정보 (e.g. `“.”`, `“..”`) 를 기록합니다. (`mkfs.c:120~128`) (`iappend()` 함수는 주어진 inode 에 정보를 추가합니다. 즉, 할당 받았던 최상위 경로에 대한 정보를 추가합니다.)

```c
// mkfs.c, main()
117:   rootino = ialloc(T_DIR);
118:   assert(rootino == ROOTINO);
119: 
120:   bzero(&de, sizeof(de));
121:   de.inum = xshort(rootino);
122:   strcpy(de.name, ".");
123:   iappend(rootino, &de, sizeof(de));
124: 
125:   bzero(&de, sizeof(de));
126:   de.inum = xshort(rootino);
127:   strcpy(de.name, "..");
128:   iappend(rootino, &de, sizeof(de));
129: 

```

이후, `mkfs`는 인자로 받았던 두 번째 리스트의 파일을 `fs.img`로 복사를 진행합니다. 앞 섹션에서 `Makefile` 파일에 `mkfs` 인자로 `fs.img` 이름을 지정하고, 이후 `README`, `_cat`, `_echo`, `_forktest` 등의 문자열 리스트를 지정했습니다. `mkfs`는 xv6 부팅 후 이 파일을 가지고 있도록 `fs.img`로 복사하는 과정을 거칩니다. (`mkfs.c:130~156`) 단, 파일의 이름이 밑줄(`_`)로 시작하는 경우 이를 제거하고 복사를 진행합니다.

```c
// mkfs.c, main()
130:   for(i = 2; i < argc; i++){
131:     assert(index(argv[i], '/') == 0);
132: 
133:     if((fd = open(argv[i], 0)) < 0){
134:       perror(argv[i]);
135:       exit(1);
136:     }
137: 
138:     // Skip leading _ in name when writing to 
            file system.
139:     // The binaries are named _rm, _cat, etc. 
            to keep the
140:     // build operating system from trying to 
            execute them
141:     // in place of system binaries like rm and cat.
142:     if(argv[i][0] == '_')
143:       ++argv[i];
144: 
145:     inum = ialloc(T_FILE);
146: 
147:     bzero(&de, sizeof(de));
148:     de.inum = xshort(inum);
149:     strncpy(de.name, argv[i], DIRSIZ);
150:     iappend(rootino, &de, sizeof(de));
151: 
152:     while((cc = read(fd, buf, sizeof(buf))) > 0)
153:       iappend(inum, buf, cc);
154: 
155:     close(fd);
156:   }
```

모든 과정을 거치면 초기 파일이 복사된 파일 시스템 이미지 `fs.img` 가 완성됩니다.

## Logging Mechanism

### `struct log` : 로그(Log)

`struct logheader` 는 로그에 대한 정보를 관리하는 자료 구조입니다. 이는 `struct logheader` 필드 n개의 커밋(Commit) 되지 않은 블록의 수를 가지며, 커밋(Commit)되면 0으로 초기화됩니다. 또한, `block` 필드는 아직 커밋이 완료되지 않은 블록 인덱스를 배열 형태로 저장하고 있습니다.

```c
// log.c
32: // Contents of the header block, 
       used for both the on-disk header block
33: // and to keep track in memory of logged 
       block# before commit.
34: struct logheader {
35:   int n;
36:   int block[LOGSIZE];
37: };
```

`struct log` 는 전역 변수 `log` 으로 단일로 선언되어 있으며, `struct spinlock` 타입의 `lock` 변수와 메타 데이터를 저장하는 `start`, `size`, `outstanding`, `commiting`, `dev` 필드를 갖습니다. 내부에 `struct logheader` 필드 `lh` 를 하나 두어 관리합니다. 이 변수에 대한 쓰임은 다음 섹션에서 후술하겠습니다.

```c
// log.c
39: struct log {
40:   struct spinlock lock;
41:   int start;
42:   int size;
43:   int outstanding; 
        // how many FS sys calls are executing.
44:   int committing;  // in commit(), please wait.
45:   int dev;
46:   struct logheader lh;
47: };
48: struct log log;
```

### `initlog()` : 로그(Log) 의 초기화

로그의 초기화는 `initlog()` 함수에서 진행합니다. xv6의 로그는 위 섹션에서 언급한 `struct log` 자료 구조로 관리되며, 다음 코드의 형태를 가지고 있습니다. 

초기화의 과정은 다음과 같습니다. 우선 `struct superblock` 타입의 변수 `sb` 를 선언하여 블록의 내용을 읽어 로드(`readsb()`)하며, 이를 기반으로 전역 변수 `log`에 정보를 기록합니다. 또한 `log` 변수의 `lock` 을 초기화 하고 이후 `recover_from_log()` 함수를 호출합니다.

`recover_from_log()` 함수는 로그 영역의 블록을 디스크에 직접 쓰는 함수로, 이에 대한 자세한 내용은 `install_trans()` 함수 섹션에서 후술하겠습니다.

```c
// log.c
53: void
54: initlog(int dev)
55: {
56:   if (sizeof(struct logheader) >= BSIZE)
57:     panic("initlog: too big logheader");
58: 
59:   struct superblock sb;
60:   initlock(&log.lock, "log");
61:   readsb(dev, &sb);
62:   log.start = sb.logstart;
63:   log.size = sb.nlog;
64:   log.dev = dev;
65:   recover_from_log();
66: }
```

그러면, `initlog()` 함수가 호출되는 위치를 파악해 보겠습니다.  `forkret()` 함수는 `allocproc()` 함수에 의해 `fork()` 로 인한 새 프로세스가 실행되는 첫 코드입니다. (프로세스의 `p->context->eip`가 `forkret` 함수로 지정됩니다. `allocproc()` 의 세부 내용은 로깅에 대한 범위를 넘어가므로 생략하겠습니다.) 

만일 `static` 변수 `first` 가 `1`의 값을 갖는 경우, 즉 처음 forkret() 함수가 실행 되어 1로 초기화되는 경우 `if` 범위 (`proc.c:403`)으로 진입하게 되며, `proc.c:409` 에서 `initlog()` 함수를 호출합니다. 즉, 첫 번째 `forkret()` 함수가 호출되는 지점에서 로깅이 활성화됩니다. 

`allocproc()` 의 `forkret()` 지정은 다음을 참고합니다.

```c
// proc.c
394: // A fork child's very first 
        scheduling by scheduler()
395: // will swtch here.  "Return" to user space.
396: void
397: forkret(void)
398: {
399:   static int first = 1;
400:   // Still holding ptable.lock from scheduler.
401:   release(&ptable.lock);
402: 
403:   if (first) {
404:     // Some initialization functions 
            must be run in the context
405:     // of a regular process 
           (e.g., they call sleep), and thus cannot
406:     // be run from main().
407:     first = 0;
408:     iinit(ROOTDEV);
409:     initlog(ROOTDEV);
410:   }
411: 
412:   // Return to "caller", actually trapret 
          (see allocproc).
413: }

// proc.c
073: static struct proc*
074: allocproc(void)
075: {
076:   struct proc *p;
077:   char *sp;
078: 

...

112:   memset(p->context, 0, sizeof *p->context);
113:   p->context->eip = (uint)forkret;
114: 
115:   return p;
116: }
```

### `begin_op()` : 로그(Log) 개시

xv6의 시스템 콜(system call) 다음과 같은 패턴으로 작성됩니다.

```c
begin_op(); 
...
bp = bread(...);
bp->data[...] = ...;
log_write(bp); 
...
end_op();
```

이 섹션에서는 `begin_op()` 함수에 대해 다룬다. `begin_op()` 함수는 세 가지의 조건을 처리하는 무한 루프가 존재합니다.

1. 첫 번째 `if` 범위(scope) (`log.c:130`)는 현재 로그가 커밋 중인지를 파악하고, 사용 중인 경우 대기 상태로 진입합니다. (`sleep()` 호출)
2. 두 번째 `if` 범위(scope) (`log.c:132`)는 사용 중인 로그의 크기가 큰 경우 커밋이 일어날 때 까지 대기합니다. (`sleep()` 호출) 이 때 `struct log` 의 `outstanding` 필드는 처리되지 않은 로깅의 요청의 수를 나타냅니다. 
3. 마지막 `if` 범위(scope) (`log.c:135`), 즉 앞의 두 조건이 아닌 경우 로그의 요청의 수를 하나 늘리고 (`log.c:136`), 락을 릴리스한 후 무한 루프에서 빠져나갑니다.

```c
// log.c
124: // called at the start of each FS system call.
125: void
126: begin_op(void)
127: {
128:   acquire(&log.lock);
129:   while(1){
130:     if(log.committing){
131:       sleep(&log, &log.lock);
132:     } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
133:       // this op might exhaust log space; 
              wait for commit.
134:       sleep(&log, &log.lock);
135:     } else {
136:       log.outstanding += 1;
137:       release(&log.lock);
138:       break;
139:     }
140:   }
141: }
```

### `log_write()` : 로그(Log)의 작성

`log_write()`는 `begin_op()` 이후에 호출됩니다. `log_write()` 함수는 우선 로그 헤더 `struct logheader lh` 의 `n` 필드를 검사하고 트랜잭션(Transaction) 이 가능한 크기인지 확인합니다. (`log.c:218~219`) 만일 대기 중인 요청의 수 log.`outstanding` 이 있다면 (`log.c:220`) 락(Lock) 획득 후 로그 변수 `log::block` 배열에 버퍼 `b->blockno` 블록 인덱스가 존재하는지 확인하는 과정을 거치게 됩니다. 즉, 로그 배열 `log::block` 의 첫 배열 부터 검사하여 동일한 블록 숫자 (Block Number)가 있다면 (동일한 블록에 여러 번 쓰기가 일어나는 경우), 그 로그 배열의 동일한 위치 (인덱스)에 로그를 기록합니다. 즉, 동일한 블록 숫자 (Block Number)가 로그에 이미 존재한다면, 그 인덱스부터 시작하여 이후에 작성된 로그는 덮어 씌우게 됩니다. 이를 Log Absorption 이라 부릅니다. 만일 동일한 블록 숫자가 존재하지 않는 경우 마지막 인덱스에 추가(Append) 합니다. (`log.c:230`)

로그에 기록 후 버퍼의 플래그 `b->flag` 에 `B_DIRTY` 를 마킹함으로써 데이터가 지워지지 않도록 하고 락을 해제합니다. 이 과정은 아래에서 확인할 수 있습니다.

```c
// log.c
204: // Caller has modified b->data and 
        is done with the buffer.
205: // Record the block number and pin in the cache 
        with B_DIRTY.
206: // commit()/write_log() will do the disk write.
207: //
208: // log_write() replaces bwrite(); a typical use is:
209: //   bp = bread(...)
210: //   modify bp->data[]
211: //   log_write(bp)
212: //   brelse(bp)
213: void
214: log_write(struct buf *b)
215: {
216:   int i;
217: 
218:   if (log.lh.n >= LOGSIZE || 
           log.lh.n >= log.size - 1)
219:     panic("too big a transaction");
220:   if (log.outstanding < 1)
221:     panic("log_write outside of trans");
222: 
223:   acquire(&log.lock);
224:   for (i = 0; i < log.lh.n; i++) {
225:     if (log.lh.block[i] == b->blockno)   
           // log absorption
226:       break;
227:   }
228:   log.lh.block[i] = b->blockno;
229:   if (i == log.lh.n)
230:     log.lh.n++;
231:   b->flags |= B_DIRTY; // prevent eviction
232:   release(&log.lock);
233: }
234: 
235: 
```

### `end_op()` : 로그(Log) Checkpointing

`end_op()` 함수는 가장 로그Log 작성 과정에서 가장 마지막으로 호출되는 함수이다. 로그 작성 요청의 대기 수 (Outstanding Calls)가 존재하지 않는다면 커밋(commit)을 진행합니다.

`end_op()` 의 동작은 다음과 같습니다. `end_op()` 이 호출되는 순간은 `log_write()` 이후의 시점이므로, 우선 락을 획득하여 로그 log의 outstanding 값을 감소시킵니다. (`log.c:151`) 만일 `outstanding` 값이 `0`, 또는 대기 중인 요청이 없는 경우 (감소시킨 요청이 마지막 요청인 경우) `do_commit` 과 `log.commiting` 변수를 `1`로 활성화합니다.

만일 현재 `outstanding` 요청이 존재하는 경우 (`log.c:157`) `begin_op()` 에서 `log` 채널로 `sleep()` 대기 중에 있던 프로세스를 깨우게됩니다. 이 경우 `do_commit` 이 활성화되지 않으므로 다음의 `log.c:165` 의 `if` 범위(scope)는 실행하지 않습니다.

`do_commit` 이 활성화 된 경우, `commit()` 함수를 호출하고, `commiting` 을 초기화하고 락Lock을 릴리스 합니다.

```c
// log.c
143: // called at the end of each FS system call.
144: // commits if this was the last 
        outstanding operation.
145: void
146: end_op(void)
147: {
148:   int do_commit = 0;
149: 
150:   acquire(&log.lock);
151:   log.outstanding -= 1;
152:   if(log.committing)
153:     panic("log.committing");
154:   if(log.outstanding == 0){
155:     do_commit = 1;
156:     log.committing = 1;
157:   } else {
158:     // begin_op() may be waiting for log space,
159:     // and decrementing log.outstanding 
            has decreased
160:     // the amount of reserved space.
161:     wakeup(&log);
162:   }
163:   release(&log.lock);
164: 
165:   if(do_commit){
166:     // call commit w/o holding locks, since 
            not allowed
167:     // to sleep with locks.
168:     commit();
169:     acquire(&log.lock);
170:     log.committing = 0;
171:     wakeup(&log);
172:     release(&log.lock);
173:   }
174: }
```

`end_op()` 함수에서 마지막으로 호출하는 `commit()` 함수는 다음 코드와 같다. `commit()` 함수는 다음 세 가지의 함수를 순차적으로 호출한다. 각 단계에 대해 요약하면 아래와 같습니다.

- `write_log()`:  캐시(Cache)에 존재하는 블록을 로그에 씁니다.
- `write_head()`:  디스크에 로그 헤더를 씁니다.
- `install_trans()`:  실제 디스크 위치에 로그에 내역을 작성합니다.
- `write_head()`:  트랜잭션(Transaction)에 대한 내용을 로그로부터 지웁니다.

```c
// log.c
192: static void
193: commit()
194: {
195:   if (log.lh.n > 0) {
196:     write_log();     
           // Write modified blocks from cache to log
197:     write_head();    
           // Write header to disk -- the real commit
198:     install_trans(); 
           // Now install writes to home locations
199:     log.lh.n = 0;
200:     write_head();    
           // Erase the transaction from the log
201:   }
202: }
```

`write_log()` 함수는 로그 블록의 시작부터 끝 (`log.lh.n`) 까지 순회하며 (`log.c:182`) 버퍼를 읽습니다. 우선 `struct buf` 타입의 `to` 와 `from` 변수에 각 블록에 대해 `bread()` 로 정보를 가져옵니다. 이때 `to` 변수는 `log.start + tail + 1` 로 지정되므로 로그가 저장되는 블록이며, `from` 변수는 `struct log`의 `block` 필드를 이용해 캐시에 저장된 수정된 블록 정보를 가져옵니다. (`log.c:183~184`) 이후 하나의 블록 사이즈 `BSIZE` 만큼 `from` 버퍼 캐시에 존재하는 `data` 필드를 `to->data` 로 이동 후, 로그 영역에 대해 디스크에 기록합니다. `bwrite()`, `log.c:186` 함수는 버퍼에 있는 데이터를 디스크로 기록하는 함수입니다. 기록이 끝나면 버퍼를 해제합니다. (`log.c:187~188`)

`bread()` 함수는 버퍼가 유효하다면 (메모리에 존재한다면) 이를 반환하고, 아닌 경우 `iderw()` 를 통해 디스크로부터 읽습니다.

```c
// log.c
176: // Copy modified blocks from cache to log.
177: static void
178: write_log(void)
179: {
180:   int tail;
181: 
182:   for (tail = 0; tail < log.lh.n; tail++) {
183:     struct buf *to = bread(log.dev, 
           log.start+tail+1); // log block
184:     struct buf *from = bread(log.dev, 
           log.lh.block[tail]); // cache block
185:     memmove(to->data, from->data, BSIZE);
186:     bwrite(to);  // write the log
187:     brelse(from);
188:     brelse(to);
189:   }
190: }
```

다음으로 호출되는 `write_head()` 함수는 아래와 같습니다. `write_head()` 함수는 디스크에 로그 헤더의 내용을 기록하는 함수입니다. 전체적인 형태는 `write_log()` 함수와 비슷하나, 블록 위치에 대한 정보와 기록 내용만 다르다는 차이점을 가지고 있습니다.

`write_log()` 함수와 동일하게 우선 버퍼(Buffer) 에서 `log.start` 블록에 대해 `bread()` 를 호출합니다. `write_log()` 함수는 `log.start + 1` 이후의 인덱스 부터 버퍼를 읽었으나, xv6 로깅 시스템은 로그 첫 번째 블록에 헤더가 위치하기 때문에 한 개의 블록으로부터 버퍼를 읽습니다. 버퍼(Buffer) 의 데이터는 `struct logheader` 타입으로 치환되어 기록되고, (`log.c:105~110`) `write_log()` 함수와 동일하게 디스크에 기록되고 버퍼가 해제됩니다.

```c
// log.c
098: // Write in-memory log header to disk.
099: // This is the true point at which the
100: // current transaction commits.
101: static void
102: write_head(void)
103: {
104:   struct buf *buf = bread(log.dev, log.start);
105:   struct logheader *hb = 
         (struct logheader *) (buf->data);
106:   int i;
107:   hb->n = log.lh.n;
108:   for (i = 0; i < log.lh.n; i++) {
109:     hb->block[i] = log.lh.block[i];
110:   }
111:   bwrite(buf);
112:   brelse(buf);
113: }
```

다음은 `install_trans()` 함수를 나타내고 있습니다. `install_trans()` 함수는 `write_log()` 함수와 동일한 구조를 갖습니다. 로그 블록의 시작부터 끝 (`log.lh.n`) 까지 순회하며 (`log.c:74`) 버퍼를 읽기 위해 `struct buf` 타입의 `lbuf` 와 `dbuf` 변수에 각 블록에 대해 `bread()` 로 정보를 가져옵니다. 이때 `lbuf` 변수는 `log.start + tail + 1` 로 지정되므로 캐시가 저장되는 블록이며, `dbuf` 변수는 `struct log`의 `block` 필드를 이용해 캐시에 저장된 수정된 블록 정보를 가져옵니다.

로그 블록을 모두 읽고 나면 `lbuf` 의 내용을 `dbuf` 로 옮긴 후 (`memmove()`, `log.c:77`) `bwrite()`으로 디스크게 기록 및 버퍼를 해제합니다.

```c
// log.c
68: // Copy committed blocks from log to their 
       home location
69: static void
70: install_trans(void)
71: {
72:   int tail;
73: 
74:   for (tail = 0; tail < log.lh.n; tail++) {
75:     struct buf *lbuf = bread(log.dev, 
          log.start+tail+1); 
          // read log block
76:     struct buf *dbuf = bread(log.dev, 
          log.lh.block[tail]); // read dst
77:     memmove(dbuf->data, lbuf->data, BSIZE);  
          // copy block to dst
78:     bwrite(dbuf);  // write dst to disk
79:     brelse(lbuf);
80:     brelse(dbuf);
81:   }
82: }
83: 
```

`install_trans() `함수가 모두 호출 되었다면 `commit()` 함수는 로그의 헤더 `log.lh.n` 필드를 `0`으로 초기화 후 `write_head()` 함수를 한번 더 호출한다. 앞에서 언급했던 `write_head()` 함수는 메모리 위에 존재하는 `log` 변수를 디스크에 기록하는 함수이므로, 현재 로그(Log) 블록의 수가 `0` 이라는 내용을 기록함으로써 이전의 로그 내역을 초기화한다.

### `recover_from_log()` : 로그(Log) 복구

앞서 `initlog()` 함수 내에서 `recover_from_log()` 함수를 호출합니다. 이는 디스크에 기록된 로그 내역으로부터 복구하는 함수이며, 다음과 같습니다.

우선 `read_head()` 함수를 호출하여 디스크로부터 로그 블록을 읽어들입니다. 이후 기록된 로그의 내용을 바탕으로 `install_trans()` 함수를 호출하여 로그의 내용을 해당 디스크 위치에 기록합니다. 기록이 끝났다면 `commit()` 함수의 두 번째 `write_head()` 동작과 같이 `log.lh.n` 필드를 초기화 하고 디스크에 로그가 비었다는 것을 기록합니다.

```c
115: static void
116: recover_from_log(void)
117: {
118:   read_head();
119:   install_trans(); 
       // if committed, copy from log to disk
120:   log.lh.n = 0;
121:   write_head(); // clear the log
122: }
```

