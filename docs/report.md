# UNIX V6 커널의 Pipe 구현 분석

## Ⅰ. 서론

### 1. 과제 소개와 분석 목적

이번 과제는 UNIX 6판(V6) 커널 소스 중 원하는 부분을 골라 C 코드 100줄 이상을 분석하는 것이다. 분석 주제는 **UNIX V6 커널의 Pipe 구현**으로 정했다.

파이프는 셸에서 `ls | wc`처럼 한 프로세스의 출력을 다른 프로세스의 입력으로 바로 넘겨 주는 기능으로, 프로세스 간 통신(IPC)의 대표적인 방법이다. 이 보고서의 목적은 다음 세 가지를 실제 코드로 확인하는 것이다.

- 커널이 파이프를 만들 때 어떤 자원(inode, file 구조체, 파일 디스크립터)을 할당하는가
- 데이터를 쓰고 읽을 때 버퍼가 가득 차거나 비면 어떻게 처리하는가
- 두 프로세스가 같은 파이프에 동시에 접근할 때 잠금과 `sleep`/`wakeup`으로 어떻게 동기화하는가

### 2. 분석 범위

분석 대상은 `pipe.c` 한 파일이고, 그 안의 함수 다섯 개를 중심으로 분석했다.

| 구분 | Lions 줄 번호 | 내용 | 전체 줄 | 실제 코드 줄 |
|---|---|---|---|---|
| `pipe.c` 전체 | 7700–7894 | include, `PIPSIZ`, 함수 5개 | 195 | **126** |
| `pipe()` | 7723–7753 | 파이프 생성 | 31 | 30 |
| `readp()` | 7758–7800 | 파이프에서 읽기 | 43 | 30 |
| `writep()` | 7805–7855 | 파이프에 쓰기 | 51 | 37 |
| `plock()` | 7862–7873 | 파이프(inode) 잠금 | 12 | 11 |
| `prele()` | 7882–7893 | 잠금 해제 | 12 | 11 |
| **함수 5개 합계** | | | 149 | **119** |

"실제 코드 줄"은 빈 줄과 주석만 있는 줄을 뺀 수이다. 두 가지 방법(줄 단위 판별, 주석을 지운 뒤 다시 세기)으로 세어 같은 결과를 얻었다. 따라서 과제 조건인 100줄 이상을 만족한다. `pipe.c` 전체 원본은 **부록**에 실었다.

이 밖에 `struct file`, `struct inode`, `rdwr()`, `main()` 등 다른 파일의 코드도 일부 인용했다. 이것들은 pipe 코드를 이해하기 위해 함께 본 **관련 코드**이고, 위의 분석 분량에는 넣지 않았다.

### 3. 원본 자료와 줄 번호

| 항목 | 내용 |
|---|---|
| 원본 저장소 | GitHub `dspinellis/unix-history-repo`, 브랜치 `Research-V6`, `usr/sys/ken/pipe.c` |
| 줄 번호 기준 | Lions 판 소스 목록 (https://warsus.github.io/lions-/) |
| 해설서 | John Lions, *A Commentary on the UNIX Operating System* 21장 "Pipes" |

두 판본에서 주석과 빈 줄을 지우고 비교해 보면 남은 코드 126줄이 완전히 같다. 차이는 주석 모양과 빈 줄뿐이다. 그래서 내용은 원본 저장소와 같다고 보고, 줄 번호는 해설서와 맞추기 쉬운 **Lions 판 번호**를 썼다. 보고서에 나오는 모든 줄 번호는 이 기준이다.

## Ⅱ. 배경 지식

### 1. UNIX V6와 커널

UNIX V6는 1975년 벨 연구소에서 나온 UNIX 6판으로, PDP-11이라는 16비트 컴퓨터에서 동작했다. 커널 대부분이 C로 작성되어 있고 분량이 크지 않아서, John Lions가 소스 전체에 해설을 붙인 책이 운영체제 교재로 쓰였다.

커널은 프로세스, 메모리, 파일, 장치를 관리하는 운영체제의 핵심 부분이다. 사용자 프로그램은 하드웨어나 커널 자료구조를 직접 건드리지 못하고 **시스템 콜**로 커널에 요청한다. 파이프도 `pipe`, `read`, `write`, `close` 시스템 콜을 통해서만 쓸 수 있다.

### 2. Pipe를 고른 이유

- **IPC를 코드로 직접 볼 수 있다.** 수업에서 다루는 프로세스 간 통신이 실제 커널에서 어떤 자료구조로 만들어지는지 확인할 수 있다.
- **한 파일 안에 생성, 읽기, 쓰기, 동기화가 모두 있다.** 다섯 함수가 서로를 호출하는 구조라서 전체 흐름을 따라가기 좋다.
- **운영체제의 핵심 개념이 함께 나온다.** 커널 자원 할당, 생산자-소비자 문제, `sleep`/`wakeup`, 상호 배제가 모두 120줄 정도 안에 들어 있다.

### 3. 열린 파일을 관리하는 3단 구조

파이프 코드를 읽으려면 커널이 "열린 파일"을 세 단계로 관리한다는 것을 먼저 알아야 한다. 도서관에 비유하면 다음과 같다.

| 도서관 | 커널 | 누구 것인가 |
|---|---|---|
| 내 대출증의 칸 번호 | fd (파일 디스크립터) | 프로세스마다 따로 |
| 대출 기록 카드 (어디까지 읽었는지) | file 구조체 | 시스템 전체가 공유 |
| 책 자체 (내용, 크기) | inode | 시스템 전체가 공유, 파일 하나에 하나 |

**① fd: `struct user`의 `u_ofile`**

원본: `source/related/user_struct.txt`
```c
0424:         int     u_procp;                /* pointer to proc structure */
0425:         char    *u_base;                /* base address for IO */
0426:         char    *u_count;               /* bytes remaining for IO */
0427:         char    *u_offset[2];           /* offset in file for IO */
...
0438:         int     u_ofile[NOFILE];        /* pointers to file structures of open files */
...
0451:         int     *u_ar0;                 /* address of users saved R0 */
```

`u`는 지금 실행 중인 프로세스의 정보이다. `u_ofile[NOFILE]`(0438)이 fd 표이다. **fd는 이 배열의 인덱스 번호**이고, 배열 칸에는 file 구조체의 주소가 들어 있다. `NOFILE`은 15이다. `u_base`, `u_count`, `u_offset`(0425–0427)은 입출력할 때 "어디로, 몇 바이트를, 파일의 어느 위치에서"를 담는 변수로, `readp`/`writep`가 직접 다룬다. `u_ar0`(0451)는 사용자 레지스터가 저장된 곳을 가리키며, 시스템 콜의 결과값을 돌려줄 때 쓴다.

**② file 구조체**

원본: `source/related/file_struct.txt`
```c
5507: struct  file
5508: {
5509:         char    f_flag;
5510:         char    f_count;        /* reference count */
5511:         int     f_inode;        /* pointer to inode structure */
5512:         char    *f_offset[2];   /* read/write character pointer */
5513: } file[NFILE];
...
5517: #define FREAD   01
5518: #define FWRITE  02
5519: #define FPIPE   04
```

`f_count`는 이 file을 가리키는 fd 수, `f_inode`는 연결된 inode의 주소, `f_offset`은 현재 읽기/쓰기 위치이다. `} file[NFILE];`은 구조체 정의와 전역 배열 선언을 한 번에 한 것이다. V6 커널은 필요할 때마다 메모리를 새로 할당하지 않고, **미리 만들어 둔 고정 크기 배열에서 빈 칸을 찾아 쓴다.** `f_offset`이 원소 2개인 것은 16비트 `int` 하나로 큰 위치 값을 담을 수 없어서 상위/하위로 나눈 것이다. 파이프는 최대 4096바이트라서 하위 `[1]`만 쓴다.

**③ inode 구조체 (메모리에 있는 것)**

원본: `source/related/inode_struct.txt`
```c
5659: struct  inode
5660: {
5661:         char    i_flag;
5662:         char    i_count;        /* reference count */
5663:         int     i_dev;          /* device where inode resides */
5664:         int     i_number;       /* i number, 1-to-1 with device address */
5665: 
5666:         int     i_mode;
5667:         char    i_nlink;        /* directory entries */
5668:         char    i_uid;          /* owner */
5669:         char    i_gid;          /* group of owner */
5670:         char    i_size0;        /* most significant of size */
5671:         char    *i_size1;       /* least sig */
5672:         int     i_addr[8];      /* device addresses constituting file */
5673:         int     i_lastr;        /* last logical block read (for read-ahead) */
5674: 
5675: } inode[NINODE];
...
5679: #define ILOCK   01              /* inode is locked */
...
5683: #define IWANT   020             /* some process waiting on lock */
...
5696: #define IREAD   0400            /* read, write, execute permissions */
5697: #define IWRITE  0200
```

inode 구조체는 디스크에 저장되는 모양(5605)과 메모리에서 쓰는 모양(5659) 두 가지가 있다. 파이프 코드는 메모리 쪽(5659)을 쓴다. 파이프에서 각 필드는 원래 뜻과 조금 다르게 쓰인다.

| 필드 | 원래 의미 | 파이프에서의 쓰임 |
|---|---|---|
| `i_count` | 이 inode를 가리키는 file 구조체 수 | 2이면 읽는 쪽과 쓰는 쪽이 모두 열려 있음 |
| `i_size1` | 파일 크기(하위 16비트) | **쓰기 위치** (지금까지 쓴 데이터의 끝) |
| `i_flag` | 상태 플래그 | `ILOCK`, `IWANT`: 잠금에 사용 |
| `i_mode` | 파일 종류와 권한 | `IREAD`, `IWRITE` 비트를 "기다리는 프로세스가 있음" 표시로 사용 |

### 4. `main()`에서 `pipe()`까지

과제 안내의 "main() code를 기준으로"에 맞추어, 커널이 시작되는 `main()`에서 파이프 코드까지 어떻게 이어지는지 먼저 정리했다. `main()`은 `pipe()`를 직접 호출하지 않는다. `main()`은 커널이 돌아갈 환경을 준비하고, `pipe()`는 나중에 사용자 프로그램이 시스템 콜로 부를 때 실행된다.

원본: `source/related/main.txt`
```c
1613:         cinit();
1614:         binit();
1615:         iinit();
1616:         rootdir = iget(rootdev, ROOTINO);
...
1627:         if(newproc()) {
1628:                 expand(USIZE+1);
1629:                 estabur(0, 1, 0, 0);
1630:                 copyout(icode, 0, sizeof icode);
1631:                 /*
1632:                  * Return goes to loc. 0 of user init
1633:                  * code just copied out.
1634:                  */
1635:                 return;
1636:         }
1637:         sched();
```

1614줄 `binit()`은 디스크 블록 버퍼를 준비한다. 파이프 데이터도 결국 `readi`/`writei`를 거쳐 이 버퍼에 저장된다. 1615줄 `iinit()`은 루트 장치를 준비하는데, `pipe()`의 첫 줄인 7728줄 `ialloc(rootdev)`가 바로 이 루트 장치에 inode를 만든다. 1627–1635줄은 1번 프로세스를 만들어 `/etc/init`을 실행시키는 부분이고, 이후 사용자 프로그램들이 실행되면서 `pipe` 시스템 콜을 부르게 된다.

원본: `source/related/trap_syscall.txt`
```c
2751:         case 6+USER: /* sys call */
...
2754:                 callp = &sysent[fuiword(pc-2)&077];
...
2771:                 trap1(callp->call);
...
2954:         0, &pipe,                       /* 42 = pipe */
```

사용자 프로그램이 시스템 콜 명령을 실행하면 커널 모드로 바뀌어 `trap()`이 실행된다. 2754줄은 방금 실행한 명령어의 하위 6비트(`&077`)를 시스템 콜 번호로 쓴다. 번호가 42이면 `sysent[42]`, 즉 `&pipe`가 골라져 2771줄에서 호출된다. `sysent`는 "번호 → 함수"를 연결하는 함수 포인터 배열이다.

원본: `source/related/rdwr.txt`
```c
5736:         fp = getf(u.u_ar0[R0]);
...
5746:         if(fp->f_flag&FPIPE) {
5747:                 if(m==FREAD)
5748:                         readp(fp); else
5749:                         writep(fp);
5750:         } else {
5751:                 u.u_offset[1] = fp->f_offset[1];
5752:                 u.u_offset[0] = fp->f_offset[0];
5753:                 if(m==FREAD)
5754:                         readi(fp->f_inode); else
5755:                         writei(fp->f_inode);
```

파이프가 만들어진 뒤의 `read`/`write`는 모두 `rdwr()`로 모인다. 5736줄 `getf()`가 fd 번호로 file 구조체를 찾고, 5746줄에서 file에 `FPIPE` 표시가 있으면 일반 파일 경로(`readi`/`writei`) 대신 `readp`/`writep`로 보낸다.

```text
[boot]  main() 1550
          |-- binit() 1614 / iinit() 1615 ....... buffers + root device for pipe
          '-- newproc() 1627 -> /etc/init -> shell (user programs)
                                   |
          pipe system call         |          read / write system call
                 v                 |                    v
         trap() 2754 -> sysent[42]           read() 5711 / write() 5720
                 v                                      v
            pipe() 7723                           rdwr() 5731
                                                        |  f_flag & FPIPE ?
                                              +---------+---------+
                                         readp() 7758        writep() 7805
                                              +--- plock() 7862 / prele() 7882 ---+
[close]  closef() 6643 : if FPIPE, wake up waiting processes
```

## Ⅲ. 함수별 코드 분석

### 1. 파일 앞부분과 `PIPSIZ` (7700–7715)

원본: `source/pipe_lions.txt`
```c
7700: #include "../param.h"
7701: #include "../systm.h"
7702: #include "../user.h"
7703: #include "../inode.h"
7704: #include "../file.h"
7705: #include "../reg.h"
7706: 
7707: /* 
7708:  * Max allowable buffering per pipe.
7709:  * This is also the max size of the
7710:  * file created to implement the pipe.
7711:  * If this size is bigger than 4096,
7712:  * pipes will be implemented in LARG
7713:  * files, which is probably not good.
7714:  */
7715: #define PIPSIZ  4096
```

필요한 헤더 여섯 개를 포함하고, 파이프 하나에 담을 수 있는 최대 크기 `PIPSIZ`를 4096바이트로 정한다. 주석에 따르면 이 값은 파이프를 구현하는 데 쓰는 파일의 최대 크기이기도 하다. 4096보다 크면 파일이 "큰 파일(LARG)" 방식으로 바뀌어 접근이 느려지기 때문에 이 크기로 제한했다.

**옛날 C 문법 몇 가지**

pipe.c는 지금의 C와 다른 문법이 섞여 있어서 먼저 정리해 둔다.

| pipe.c의 표현 | 지금 C로 쓰면 | 예 |
|---|---|---|
| `=\|`, `=&`, `=-` | `\|=`, `&=`, `-=` | 7789 `ip->i_mode =\| IREAD;` |
| `register *ip;` | `register int *ip;` | 형을 안 쓰면 `int`로 본다 |
| `rp->f_offset` (rp는 `int *`) | 구조체 포인터로 형 변환 후 접근 | 당시에는 구조체 멤버 이름을 어떤 포인터에도 쓸 수 있었다 |
| `readp(fp) int *fp; { … }` | `readp(int *fp) { … }` | 인자 형을 함수 이름 아래에 따로 적는다 |

### 2. `pipe()`: 파이프 생성 (7723–7753)

**함수의 목적**: `pipe` 시스템 콜의 본체이다. 파이프로 쓸 inode 한 개와 file 구조체 두 개(읽기용, 쓰기용)를 할당하고, 두 fd 번호를 사용자에게 돌려준다.

**1) inode와 첫 번째 file 구조체 할당 (7723–7735)**

원본: `source/pipe_lions.txt`
```c
7717: /*
7718:  * The sys-pipe entry.
7719:  * Allocate an inode on the root device.
7720:  * Allocate 2 file structures.
7721:  * Put it all together with flags.
7722:  */
7723: pipe()
7724: {
7725:         register *ip, *rf, *wf;
7726:         int r;
7727: 
7728:         ip = ialloc(rootdev);
7729:         if(ip == NULL)
7730:                 return;
7731:         rf = falloc();
7732:         if(rf == NULL) {
7733:                 iput(ip);
7734:                 return;
7735:         }
```

- **7725–7726**: `ip`는 inode를, `rf`/`wf`는 읽기용/쓰기용 file 구조체를 가리킬 포인터이다. `r`은 fd 번호를 잠시 저장할 변수이다.
- **7728**: `ialloc(rootdev)`가 루트 장치에 새 inode를 하나 할당한다. `ialloc`은 안에서 `iget`을 부르므로, 돌려받은 inode는 `i_count`가 1이고 `ILOCK`으로 잠긴 상태이다(7302–7303, 7316–7317). 또 `ialloc`이 `i_mode`부터 `i_addr`까지 0으로 지우므로(7082–7083) 링크 수 `i_nlink`도 0이다. 디렉터리 어디에도 이름이 없는 **이름 없는 파일**이 되는 것이다.
- **7729–7730**: inode가 없으면 그냥 돌아간다. 오류 코드는 `ialloc` 쪽에서 이미 `u.u_error`에 넣어 둔다.
- **7731**: `falloc()`이 빈 fd 칸과 빈 file 구조체를 찾아 연결한다(6847). 이때 `falloc` 안의 `ufalloc`이 고른 fd 번호를 `u.u_ar0[R0]`에 써 둔다(6830).
- **7732–7735**: file 구조체가 없으면 앞에서 받은 inode를 `iput()`으로 돌려주고 끝낸다.

**2) 두 번째 file 구조체 할당과 실패 처리 (7736–7743)**

원본: `source/pipe_lions.txt`
```c
7736:         r = u.u_ar0[R0];
7737:         wf = falloc();
7738:         if(wf == NULL) {
7739:                 rf->f_count = 0;
7740:                 u.u_ofile[r] = NULL;
7741:                 iput(ip);
7742:                 return;
7743:         }
```

- **7736**: `r = u.u_ar0[R0]`는 첫 번째 `falloc`이 R0에 써 둔 **읽기용 fd 번호**를 `r`에 옮겨 둔다. 다음 줄의 두 번째 `falloc`이 R0를 다시 덮어쓰기 때문이다. 해설서는 이 줄을 "file table entry를 r로 기억한다"고 설명하지만, 코드를 따라가 보면 `r`에 들어가는 것은 file 구조체 주소가 아니라 **fd 번호**이다(6830 `u.u_ar0[R0] = i`).
- **7737**: 쓰기용 file 구조체를 할당한다. 이번에 고른 fd 번호는 R0에 들어간다.
- **7738–7742**: 실패하면 앞에서 받은 자원을 **거꾸로** 돌려준다. 읽기용 file 구조체를 빈 칸으로 만들고(`f_count = 0`), fd 칸을 비우고(`u_ofile[r] = NULL`), inode를 반납한다. 커널은 고정 크기 표를 쓰기 때문에 이렇게 정리하지 않으면 그 칸이 영영 "사용 중"으로 남는다.

**3) 결과 돌려주기와 연결 (7744–7753)**

원본: `source/pipe_lions.txt`
```c
7744:         u.u_ar0[R1] = u.u_ar0[R0];
7745:         u.u_ar0[R0] = r;
7746:         wf->f_flag = FWRITE|FPIPE;
7747:         wf->f_inode = ip;
7748:         rf->f_flag = FREAD|FPIPE;
7749:         rf->f_inode = ip;
7750:         ip->i_count = 2;
7751:         ip->i_flag = IACC|IUPD;
7752:         ip->i_mode = IALLOC;
7753: }
```

- **7744–7745**: 두 fd를 사용자에게 돌려준다. R1에는 쓰기용 fd(두 번째 `falloc`이 R0에 쓴 값)를, R0에는 읽기용 fd(`r`)를 넣는다. 시스템 콜에서 돌아가면 이 두 레지스터 값이 사용자 쪽으로 전달된다.
- **7746–7749**: 쓰기용 file에는 `FWRITE|FPIPE`, 읽기용 file에는 `FREAD|FPIPE`를 넣고 둘 다 같은 inode를 가리키게 한다. `FPIPE`가 있어야 `rdwr()`이 `readp`/`writep`로 보낸다(5746).
- **7750**: `i_count = 2`. 이 inode를 file 구조체 두 개가 가리킨다는 뜻이다. 나중에 `readp`/`writep`는 이 값이 2보다 작아지면 "반대쪽이 닫혔다"고 판단한다.
- **7751**: `i_flag = IACC|IUPD`. 대입(`=`)이라서 `iget`이 걸어 둔 `ILOCK`도 함께 지워진다. 즉 이 줄에서 inode의 잠금이 풀린다.
- **7752**: `i_mode = IALLOC`. 사용 중인 일반 파일이라는 표시만 남기고 권한 비트는 모두 0이다. 이후 `IREAD`/`IWRITE` 비트는 권한이 아니라 "기다리는 프로세스" 표시로만 쓰인다.

```text
process u_ofile[]        file[]  (system-wide)            inode[]  (system-wide)
+----+
| 3  |--------------> +-- file (read) ----------+
| 4  |---+            | f_flag  = FREAD|FPIPE   |---+
+----+   |            | f_count = 1             |   |     +-- inode (pipe) ------+
         |            | f_offset[1] = read pos  |   +---> | i_count = 2          |
         |            +-------------------------+   |     | i_size1 = write pos  |
         +----------> +-- file (write) ---------+   |     | data: disk blocks    |
                      | f_flag  = FWRITE|FPIPE  |---+     +----------------------+
                      | f_count = 1             |
                      +-------------------------+
```

**관련 운영체제 개념**: 커널 자원 할당(고정 크기 표에서 빈 칸 찾기), 실패 시 되돌리기, 시스템 콜 반환값 전달. 파이프는 새로운 자료구조가 아니라 **이름 없는 일반 파일 하나에 file 구조체 두 개를 붙인 것**이라는 점이 핵심이다. 해설서도 파이프를 "또 하나의 파일 종류"로 설명한다.

### 3. `plock()`과 `prele()`: 파이프 잠금 (7858–7893)

`readp`/`writep`는 둘 다 이 두 함수로 inode를 잠그고 푼다. 그래서 먼저 분석한다.

**1) `plock()`: 잠금 (7862–7873)**

원본: `source/pipe_lions.txt`
```c
7858: /* Lock a pipe.
7859:  * If its already locked,
7860:  * set the WANT bit and sleep.
7861:  */
7862: plock(ip)
7863: int *ip;
7864: {
7865:         register *rp;
7866: 
7867:         rp = ip;
7868:         while(rp->i_flag&ILOCK) {
7869:                 rp->i_flag =| IWANT;
7870:                 sleep(rp, PPIPE);
7871:         }
7872:         rp->i_flag =| ILOCK;
7873: }
```

- **7868**: 이미 누가 잠갔으면(`ILOCK`이 켜져 있으면)
- **7869**: "기다리는 사람이 있다"는 `IWANT` 표시를 남기고
- **7870**: `sleep(rp, PPIPE)`로 잠든다. 첫 번째 인자 `rp`(inode 주소)는 **대기 채널**, 즉 "무엇을 기다리는지"를 나타내는 이름표이다.
- **7872**: 잠금이 풀려 있으면 `ILOCK`을 켜고 돌아간다.

`if`가 아니라 `while`인 이유는 `wakeup`이 같은 채널에서 잠든 프로세스를 **전부** 깨우기 때문이다(2122–2123). 여러 프로세스가 함께 깨어나도 먼저 실행된 하나만 잠금을 얻고, 나머지는 다시 검사해서 또 잠들어야 한다.

**2) `prele()`: 잠금 해제 (7882–7893)**

원본: `source/pipe_lions.txt`
```c
7876: /* Unlock a pipe.
7877:  * If WANT bit is on,
7878:  * wakeup.
7879:  * This routine is also used
7880:  * to unlock inodes in general.
7881:  */
7882: prele(ip)
7883: int *ip;
7884: {
7885:         register *rp;
7886: 
7887:         rp = ip;
7888:         rp->i_flag =& ~ILOCK;
7889:         if(rp->i_flag&IWANT) {
7890:                 rp->i_flag =& ~IWANT;
7891:                 wakeup(rp);
7892:         }
7893: }
```

- **7888**: `ILOCK`을 끈다.
- **7889–7892**: `IWANT`가 켜져 있으면, 즉 잠금을 기다리며 잠든 프로세스가 있으면 표시를 지우고 `wakeup(rp)`로 깨운다.

주석에 있듯이 `prele`는 파이프뿐 아니라 일반 inode 잠금 해제에도 쓰인다. 실제로 `iput()`도 `prele`를 부른다(7358, 7363). 해설서는 7799줄의 `prele`를 "파이프를 기다리는 프로세스를 깨운다"고 설명하지만, 코드상 `prele`가 깨우는 것은 **잠금을 기다리는 프로세스(채널 `ip`)뿐**이다. 데이터를 기다리는 reader(채널 `ip+2`)와 빈 공간을 기다리는 writer(채널 `ip+1`)는 `readp`/`writep`가 직접 깨운다.

**3) `sleep`과 `wakeup`**

원본: `source/related/sleep_wakeup.txt`
```c
2066: sleep(chan, pri)
2067: {
2068:         register *rp, s;
2069: 
2070:         s = PS->integ;
2071:         rp = u.u_procp;
2072:         if(pri >= 0) {
2073:                 if(issig())
2074:                         goto psig;
2075:                 spl6();
2076:                 rp->p_wchan = chan;
2077:                 rp->p_stat = SWAIT;
2078:                 rp->p_pri = pri;
2079:                 spl0();
...
2113: wakeup(chan)
2114: {
2115:         register struct proc *p;
2116:         register c, i;
2117: 
2118:         c = chan;
2119:         p = &proc[0];
2120:         i = NPROC;
2121:         do {
2122:                 if(p->p_wchan == c) {
2123:                         setrun(p);
2124:                 }
2125:                 p++;
2126:         } while(--i);
2127: }
```

`sleep(chan, pri)`은 현재 프로세스의 대기 채널을 `chan`으로 적고(2076) 대기 상태로 바꾼 뒤(2077) `swtch()`로 CPU를 다른 프로세스에 넘긴다. `wakeup(chan)`은 프로세스 표 전체를 돌면서 대기 채널이 `chan`인 프로세스를 모두 실행 가능 상태로 만든다(2121–2126). 채널은 그냥 **주소값**이라서, 파이프 코드는 `ip`, `ip+1`, `ip+2`라는 서로 다른 주소 세 개를 이름표로 써서 세 종류의 기다림을 구분한다.

| 채널 | 기다리는 것 | 잠드는 곳 | 깨우는 곳 |
|---|---|---|---|
| `ip` | inode 잠금 | `plock` 7870 | `prele` 7891 |
| `ip+1` | 파이프에 빈 공간 (writer) | `writep` 7838 | `readp` 7778, `closef` 6652 |
| `ip+2` | 파이프에 데이터 (reader) | `readp` 7790 | `writep` 7852, `closef` 6653 |

`PPIPE`는 1로 0 이상인 우선순위이다(0157). `sleep`은 우선순위가 0 이상이면 잠들기 전후에 시그널을 검사한다(2072–2074, 2085–2086). 그래서 파이프에서 기다리는 프로세스는 시그널(예: Ctrl+C)로 깨어날 수 있다.

**관련 운영체제 개념**: 상호 배제(mutual exclusion), 잠금 변수와 대기 표시, 바쁜 대기(busy waiting) 대신 잠드는 방식. V6 커널은 프로세스가 커널 모드에서 실행되는 동안 스스로 `sleep`하기 전에는 다른 프로세스로 바뀌지 않는다. 그래서 7868줄 검사와 7872줄 설정 사이에 다른 프로세스가 끼어들 수 없고, 특별한 원자적 명령 없이도 이 잠금이 동작한다(단일 CPU 기준).

### 4. `readp()`: 파이프에서 읽기 (7756–7800)

**함수의 목적**: 읽기용 file 구조체를 받아, 파이프에 데이터가 있으면 읽고 없으면 데이터가 들어올 때까지 기다린다.

파이프는 위치 두 개로 관리된다. **읽기 위치**는 읽기용 file의 `f_offset[1]`, **쓰기 위치**는 inode의 `i_size1`(파일 크기)이다.

```text
0                                                          4096 (PIPSIZ)
|######## already read ########|~~~~ unread data ~~~~|      free       |
                               ^                      ^
                     read pos (f_offset[1])    write pos (i_size1)

 read pos == write pos  : empty -> reset both to 0, reader sleeps on ip+2
 write pos == 4096      : full  -> writer sleeps on ip+1
```

**1) 준비와 잠금 (7756–7768)**

원본: `source/pipe_lions.txt`
```c
7756: /* Read call directed to a pipe.
7757:  */
7758: readp(fp)
7759: int *fp;
7760: {
7761:         register *rp, *ip;
7762: 
7763:         rp = fp;
7764:         ip = rp->f_inode;
7765: loop:
7766:         /* Very conservative locking.
7767:          */
7768:         plock(ip);
```

- **7763–7764**: file 구조체에서 inode를 꺼낸다.
- **7765**: `loop:`는 잠들었다 깨어났을 때 다시 시작할 위치이다.
- **7768**: 주석대로 "아주 보수적인" 잠금이다. 읽기와 쓰기가 동시에 inode를 건드리지 못하게 한다.

**2) 파이프가 비었을 때 (7769–7792)**

원본: `source/pipe_lions.txt`
```c
7769:         /* If the head (read) has caught up with
7770:          * the tail (write), reset both to 0.
7771:          */
7772:         if(rp->f_offset[1] == ip->i_size1) {
7773:                 if(rp->f_offset[1] != 0) {
7774:                         rp->f_offset[1] = 0;
7775:                         ip->i_size1 = 0;
7776:                         if(ip->i_mode&IWRITE) {
7777:                                 ip->i_mode =& ~IWRITE;
7778:                                 wakeup(ip+1);
7779:                         }
7780:                 }
7781: 
7782:                 /* If there are not both reader and
7783:                  * writer active, return without
7784:                  * satisfying read.
7785:                  */
7786:                 prele(ip);
7787:                 if(ip->i_count < 2)
7788:                         return;
7789:                 ip->i_mode =| IREAD;
7790:                 sleep(ip+2, PPIPE);
7791:                 goto loop;
7792:         }
```

- **7772**: 읽기 위치가 쓰기 위치를 따라잡았으면, 즉 **읽을 데이터가 없으면** 이 블록에 들어온다.
- **7773–7775**: 위치가 0이 아니면 두 위치를 모두 0으로 되돌린다. 다 읽은 데이터는 버리고 파일 앞부분부터 다시 쓰게 하는 것이다. 원형 버퍼가 아니라 **"다 비면 처음으로 되감는"** 방식이다.
- **7776–7779**: `IWRITE`가 켜져 있으면 파이프가 꽉 차서 잠든 writer가 있다는 뜻이다. 이제 공간이 생겼으므로 표시를 지우고 `wakeup(ip+1)`로 깨운다.
- **7786**: 잠들기 전에 잠금을 먼저 푼다. 잠금을 쥔 채 잠들면 writer가 영영 들어올 수 없다.
- **7787–7788**: `i_count`가 2보다 작으면 쓰는 쪽이 모두 닫힌 것이다. 더 기다려도 데이터가 오지 않으므로 아무것도 읽지 않고 돌아간다. 사용자에게는 0바이트가 돌아가고, 이것이 **파일의 끝(EOF)** 신호가 된다.
- **7789–7791**: writer가 살아 있으면 `IREAD` 표시를 남기고 채널 `ip+2`에서 잠든다. 깨어나면 `goto loop`로 처음부터 다시 검사한다.

**3) 데이터가 있을 때 (7793–7800)**

원본: `source/pipe_lions.txt`
```c
7793:         /* Read and return
7794:          */
7795:         u.u_offset[0] = 0;
7796:         u.u_offset[1] = rp->f_offset[1];
7797:         readi(ip);
7798:         rp->f_offset[1] = u.u_offset[1];
7799:         prele(ip);
7800: }
```

- **7795–7796**: 읽기 위치를 `u.u_offset`에 넣는다. `u_base`(사용자 버퍼 주소)와 `u_count`(읽을 바이트 수)는 `rdwr()`이 이미 채워 두었다(5743–5744).
- **7797**: `readi(ip)`가 일반 파일처럼 데이터를 읽는다. `readi`는 파일 크기를 넘어서 읽지 않으므로(6243–6247) **쓰기 위치까지만** 읽힌다. 그래서 요청한 양보다 적게 읽힐 수 있다.
- **7798**: 읽은 만큼 늘어난 위치를 읽기 위치로 저장한다.
- **7799**: 잠금을 푼다.

### 5. `writep()`: 파이프에 쓰기 (7803–7855)

**함수의 목적**: 쓰기용 file 구조체를 받아, 요청한 바이트를 모두 쓸 때까지 "쓸 수 있는 만큼 쓰고, 꽉 차면 기다리기"를 반복한다.

**1) 준비와 종료 조건 (7803–7820)**

원본: `source/pipe_lions.txt`
```c
7803: /* Write call directed to a pipe.
7804:  */
7805: writep(fp)
7806: {
7807:         register *rp, *ip, c;
7808: 
7809:         rp = fp;
7810:         ip = rp->f_inode;
7811:         c = u.u_count;
7812: loop:
7813:         /* If all done, return.
7814:          */
7815:         plock(ip);
7816:         if(c == 0) {
7817:                 prele(ip);
7818:                 u.u_count = 0;
7819:                 return;
7820:         }
```

- **7811**: 앞으로 써야 할 전체 바이트 수를 `c`에 저장한다. `readp`와 달리 여러 번 나눠 쓰기 때문에 남은 양을 따로 센다.
- **7815**: 반복할 때마다 먼저 잠근다.
- **7816–7820**: 다 썼으면(`c == 0`) 잠금을 풀고 `u_count`를 0으로 만든 뒤 돌아간다. `rdwr()`은 `요청량 - u_count`를 결과로 돌려주므로(5758) 요청한 양을 모두 썼다고 전달된다.

**2) 읽는 쪽이 없을 때 (7821–7830)**

원본: `source/pipe_lions.txt`
```c
7821:         /* If there are not both read and
7822:          * write sides of the pipe active,
7823:          * return error and signal too.
7824:          */
7825:         if(ip->i_count < 2) {
7826:                 prele(ip);
7827:                 u.u_error = EPIPE;
7828:                 psignal(u.u_procp, SIGPIPE);
7829:                 return;
7830:         }
```

- **7825**: `i_count`가 2보다 작으면 읽는 쪽이 모두 닫힌 것이다.
- **7826–7829**: 아무도 읽지 않을 데이터이므로 잠금을 풀고 `EPIPE` 오류를 넣은 뒤 `psignal`로 자기 자신에게 `SIGPIPE` 시그널을 보낸다. 해설서는 쓰는 쪽이 `write`의 결과를 확인하지 않을 수도 있어서 시그널까지 보낸다고 설명한다. 읽는 쪽이 EOF(0바이트)로 조용히 끝나는 것과 대조된다.

**3) 파이프가 꽉 찼을 때 (7831–7840)**

원본: `source/pipe_lions.txt`
```c
7831:         /* If the pipe is full,
7832:          * wait for reads to deplete
7833:          * and truncate it.
7834:          */
7835:         if(ip->i_size1 == PIPSIZ) {
7836:                 ip->i_mode =| IWRITE;
7837:                 prele(ip);
7838:                 sleep(ip+1, PPIPE);
7839:                 goto loop;
7840:         }
```

- **7835**: 쓰기 위치가 `PIPSIZ`(4096)에 닿았으면 더 쓸 수 없다.
- **7836–7839**: `IWRITE` 표시를 남기고, 잠금을 풀고, 채널 `ip+1`에서 잠든다. reader가 데이터를 다 읽어 위치를 되감을 때(7773–7778) 깨워 준다.

**4) 쓸 수 있는 만큼 쓰기 (7841–7855)**

원본: `source/pipe_lions.txt`
```c
7841:         /* Write what is possible and
7842:          * loop back.
7843:          */
7844:         u.u_offset[0] = 0;
7845:         u.u_offset[1] = ip->i_size1;
7846:         u.u_count = min(c, PIPSIZ-u.u_offset[1]);
7847:         c =- u.u_count;
7848:         writei(ip);
7849:         prele(ip);
7850:         if(ip->i_mode&IREAD) {
7851:                 ip->i_mode =& ~IREAD;
7852:                 wakeup(ip+2);
7853:         }
7854:         goto loop;
7855: }
```

- **7844–7845**: 쓰는 위치는 항상 현재 데이터의 끝(`i_size1`)이다.
- **7846**: 이번에 쓸 양은 "남은 양"과 "4096까지 남은 공간" 중 작은 값이다.
- **7847**: 남은 양에서 이번에 쓸 양을 뺀다.
- **7848**: `writei(ip)`가 일반 파일처럼 데이터를 쓴다. 쓴 끝 위치가 파일 크기보다 커지면 `writei`가 `i_size1`을 늘리므로(6312–6316) 쓰기 위치가 그만큼 앞으로 간다.
- **7849–7853**: 잠금을 풀고, `IREAD`가 켜져 있으면(데이터를 기다리며 잠든 reader가 있으면) `wakeup(ip+2)`로 깨운다.
- **7854**: 아직 남은 양이 있을 수 있으므로 처음으로 돌아간다.

**관련 운영체제 개념**: 생산자-소비자 문제(유한 버퍼), 블로킹 입출력. writer는 생산자, reader는 소비자이고, 버퍼가 차면 생산자가, 비면 소비자가 잠든다. 수업에서 세마포어로 푸는 이 문제를 V6는 잠금(`plock`/`prele`)과 표시 비트(`IREAD`/`IWRITE`), `sleep`/`wakeup`으로 풀었다. 해설서에 따르면 파이프는 동시에 꽉 차 있으면서 비어 있을 수 없으므로 `IREAD`와 `IWRITE`가 동시에 켜지는 일은 없어야 한다.

## Ⅳ. 운영체제 개념과의 연결

| 개념 | pipe.c에서 나타나는 곳 |
|---|---|
| 프로세스 간 통신(IPC) | 두 프로세스가 같은 inode를 가리키는 서로 다른 file 구조체로 데이터를 주고받음 (7746–7749) |
| 커널 자원 할당 | `ialloc`, `falloc`으로 고정 크기 표에서 빈 칸을 찾음 (7728, 7731, 7737) |
| 자원 반납 (오류 처리) | 실패하면 할당한 순서의 반대로 되돌림 (7733, 7739–7741) |
| 상호 배제 | `ILOCK`으로 한 번에 한 프로세스만 inode를 다룸 (7868–7872) |
| 생산자-소비자 | 꽉 차면 writer가, 비면 reader가 잠듦 (7835–7839, 7772–7791) |
| `sleep` / `wakeup` | 채널 `ip`, `ip+1`, `ip+2`로 세 종류의 기다림을 구분 |
| 파일 디스크립터 | `pipe()`가 fd 두 개를 R0, R1로 돌려줌 (7744–7745) |
| EOF와 SIGPIPE | 반대쪽이 닫히면 reader는 0바이트, writer는 `EPIPE`와 `SIGPIPE` (7787–7788, 7825–7829) |

파이프를 닫을 때는 `closef()`가 기다리는 프로세스를 모두 깨운다.

원본: `source/related/closef.txt`
```c
6643: closef(fp)
6644: int *fp;
6645: {
6646:         register *rfp, *ip;
6647: 
6648:         rfp = fp;
6649:         if(rfp->f_flag&FPIPE) {
6650:                 ip = rfp->f_inode;
6651:                 ip->i_mode =& ~(IREAD|IWRITE);
6652:                 wakeup(ip+1);
6653:                 wakeup(ip+2);
6654:         }
6655:         if(rfp->f_count <= 1)
6656:                 closei(rfp->f_inode, rfp->f_flag&FWRITE);
6657:         rfp->f_count--;
6658: }
```

파일이 파이프이면 `IREAD`/`IWRITE` 표시를 지우고 두 채널(`ip+1`, `ip+2`)을 모두 깨운다(6649–6653). 깨어난 프로세스는 `goto loop`로 돌아가 `i_count`를 다시 검사하고, 반대쪽이 닫혔으면 EOF나 `EPIPE`로 끝난다. 양쪽이 모두 닫혀 `iput`이 마지막 참조를 놓으면, 링크 수가 0인 inode라서 데이터와 inode가 함께 해제된다(7350–7356).

## Ⅴ. 전체 실행 흐름: 예제로 따라가기

빈 파이프에 writer W가 `write(fd, buf, 5000)`을 하고, reader R이 `read(fd, buf, 3000)`을 반복하는 경우를 코드 순서대로 따라가 보았다. (`PIPSIZ` = 4096)

| 순서 | 누가 | 코드 | 상태 변화 |
|---|---|---|---|
| ① | W | 7811, 7844–7848 | 남은 양 `c` = 5000. 이번에 쓸 양 = min(5000, 4096−0) = 4096. 쓰기 위치 0 → 4096, `c` = 904 |
| ② | W | 7835–7838 | 쓰기 위치가 4096이라 꽉 참. `IWRITE`를 켜고 `ip+1`에서 잠듦 |
| ③ | R | 7772, 7795–7798 | 읽기 위치 0 ≠ 4096이므로 읽기. 3000바이트를 읽고 읽기 위치 0 → 3000 |
| ④ | R | 7797–7798 | 다시 `read`. 남은 데이터가 1096바이트뿐이라 1096바이트만 읽힘. 읽기 위치 → 4096 |
| ⑤ | R | 7772–7778, 7789–7790 | 다시 `read`. 읽기 위치 = 쓰기 위치이므로 둘 다 0으로 되감음. `IWRITE`가 켜져 있어 W를 깨움. 데이터가 없으니 `IREAD`를 켜고 `ip+2`에서 잠듦 |
| ⑥ | W | 7839 → 7844–7853 | 깨어나 `loop`로. 남은 904바이트를 위치 0부터 씀. 쓰기 위치 → 904. `IREAD`가 켜져 있어 R을 깨움 |
| ⑦ | W | 7816–7819 | `c` = 0이므로 `u_count` = 0으로 하고 끝. `write`는 5000을 돌려줌 |
| ⑧ | R | 7791 → 7795–7798 | 깨어나 `loop`로. 904바이트를 읽고 돌아감 |

이 예제에서 다음을 확인할 수 있다.

- writer는 요청한 양을 다 쓸 때까지 **여러 번 나눠 쓰며 기다린다.** 반면 reader는 **있는 만큼만 읽고 바로 돌아간다**(④에서 1096, ⑧에서 904).
- 버퍼 공간은 reader가 데이터를 **모두** 읽어 두 위치를 0으로 되감을 때(⑤) 다시 생긴다. ③처럼 일부만 읽은 상태에서는 writer가 깨어나지 않는다.
- 실제 순서는 스케줄러가 정하므로 ③과 ②의 순서 등은 바뀔 수 있다. 하지만 어느 순서든 `plock`/`prele` 덕분에 두 프로세스가 inode를 동시에 바꾸는 일은 없다.

## Ⅵ. 결론

### 1. 분석 정리

- 커널은 열린 파일을 **fd(프로세스별) → file 구조체(공유) → inode(공유)** 의 세 단계로 관리한다. 파이프는 이 구조를 그대로 써서 **이름 없는 inode 하나와 file 구조체 두 개**로 만들어진다.
- `pipe()`는 inode와 file 구조체를 고정 크기 표에서 할당하고, 실패하면 할당한 순서의 반대로 되돌린다. 두 fd는 레지스터 R0, R1로 돌려준다.
- 파이프의 데이터는 일반 파일처럼 `readi`/`writei`로 읽고 쓴다. 읽기 위치(`f_offset[1]`)와 쓰기 위치(`i_size1`) 두 값으로 버퍼를 관리하고, 다 읽으면 둘 다 0으로 되감는다. 크기는 `PIPSIZ`(4096)로 제한된다.
- 동기화는 두 겹이다. `plock`/`prele`는 inode를 한 번에 한 프로세스만 다루게 하는 **상호 배제**이고, `IREAD`/`IWRITE`와 `sleep`/`wakeup`은 버퍼가 비거나 찼을 때 **기다리고 깨우는** 역할을 한다.
- `main()`은 `pipe()`를 직접 부르지 않지만, `binit()`과 `iinit()`으로 파이프가 쓸 버퍼와 루트 장치를 준비한다. `pipe()`는 `trap()` → `sysent[42]`로, `readp`/`writep`는 `rdwr()`을 거쳐 실행된다.

### 2. 해설서와 비교하며 확인한 점

- 7736줄의 `r`에는 file 구조체가 아니라 **fd 번호**가 저장된다(6830줄에서 R0에 쓰인 값).
- `prele`는 **잠금을 기다리는** 프로세스만 깨운다. 데이터나 공간을 기다리는 프로세스는 `readp`/`writep`와 `closef`가 채널 `ip+1`, `ip+2`로 따로 깨운다.

## Ⅶ. 참고 자료

1. John Lions, *A Commentary on the UNIX Operating System*, University of New South Wales, 1977. (21장 Pipes)
2. Lions' Commentary 소스·해설 열람 사이트, https://warsus.github.io/lions-/
3. Diomidis Spinellis, *unix-history-repo*, Research-V6 브랜치, https://github.com/dspinellis/unix-history-repo

## 부록. `pipe.c` 전체 원본 (Lions 7700–7894)

원본: `source/pipe_lions.txt`
```c
7700: #include "../param.h"
7701: #include "../systm.h"
7702: #include "../user.h"
7703: #include "../inode.h"
7704: #include "../file.h"
7705: #include "../reg.h"
7706: 
7707: /* 
7708:  * Max allowable buffering per pipe.
7709:  * This is also the max size of the
7710:  * file created to implement the pipe.
7711:  * If this size is bigger than 4096,
7712:  * pipes will be implemented in LARG
7713:  * files, which is probably not good.
7714:  */
7715: #define PIPSIZ  4096
7716: 
7717: /*
7718:  * The sys-pipe entry.
7719:  * Allocate an inode on the root device.
7720:  * Allocate 2 file structures.
7721:  * Put it all together with flags.
7722:  */
7723: pipe()
7724: {
7725:         register *ip, *rf, *wf;
7726:         int r;
7727: 
7728:         ip = ialloc(rootdev);
7729:         if(ip == NULL)
7730:                 return;
7731:         rf = falloc();
7732:         if(rf == NULL) {
7733:                 iput(ip);
7734:                 return;
7735:         }
7736:         r = u.u_ar0[R0];
7737:         wf = falloc();
7738:         if(wf == NULL) {
7739:                 rf->f_count = 0;
7740:                 u.u_ofile[r] = NULL;
7741:                 iput(ip);
7742:                 return;
7743:         }
7744:         u.u_ar0[R1] = u.u_ar0[R0];
7745:         u.u_ar0[R0] = r;
7746:         wf->f_flag = FWRITE|FPIPE;
7747:         wf->f_inode = ip;
7748:         rf->f_flag = FREAD|FPIPE;
7749:         rf->f_inode = ip;
7750:         ip->i_count = 2;
7751:         ip->i_flag = IACC|IUPD;
7752:         ip->i_mode = IALLOC;
7753: }
7754: /* ---------------------------       */
7755: 
7756: /* Read call directed to a pipe.
7757:  */
7758: readp(fp)
7759: int *fp;
7760: {
7761:         register *rp, *ip;
7762: 
7763:         rp = fp;
7764:         ip = rp->f_inode;
7765: loop:
7766:         /* Very conservative locking.
7767:          */
7768:         plock(ip);
7769:         /* If the head (read) has caught up with
7770:          * the tail (write), reset both to 0.
7771:          */
7772:         if(rp->f_offset[1] == ip->i_size1) {
7773:                 if(rp->f_offset[1] != 0) {
7774:                         rp->f_offset[1] = 0;
7775:                         ip->i_size1 = 0;
7776:                         if(ip->i_mode&IWRITE) {
7777:                                 ip->i_mode =& ~IWRITE;
7778:                                 wakeup(ip+1);
7779:                         }
7780:                 }
7781: 
7782:                 /* If there are not both reader and
7783:                  * writer active, return without
7784:                  * satisfying read.
7785:                  */
7786:                 prele(ip);
7787:                 if(ip->i_count < 2)
7788:                         return;
7789:                 ip->i_mode =| IREAD;
7790:                 sleep(ip+2, PPIPE);
7791:                 goto loop;
7792:         }
7793:         /* Read and return
7794:          */
7795:         u.u_offset[0] = 0;
7796:         u.u_offset[1] = rp->f_offset[1];
7797:         readi(ip);
7798:         rp->f_offset[1] = u.u_offset[1];
7799:         prele(ip);
7800: }
7801: /* ---------------------------       */
7802: 
7803: /* Write call directed to a pipe.
7804:  */
7805: writep(fp)
7806: {
7807:         register *rp, *ip, c;
7808: 
7809:         rp = fp;
7810:         ip = rp->f_inode;
7811:         c = u.u_count;
7812: loop:
7813:         /* If all done, return.
7814:          */
7815:         plock(ip);
7816:         if(c == 0) {
7817:                 prele(ip);
7818:                 u.u_count = 0;
7819:                 return;
7820:         }
7821:         /* If there are not both read and
7822:          * write sides of the pipe active,
7823:          * return error and signal too.
7824:          */
7825:         if(ip->i_count < 2) {
7826:                 prele(ip);
7827:                 u.u_error = EPIPE;
7828:                 psignal(u.u_procp, SIGPIPE);
7829:                 return;
7830:         }
7831:         /* If the pipe is full,
7832:          * wait for reads to deplete
7833:          * and truncate it.
7834:          */
7835:         if(ip->i_size1 == PIPSIZ) {
7836:                 ip->i_mode =| IWRITE;
7837:                 prele(ip);
7838:                 sleep(ip+1, PPIPE);
7839:                 goto loop;
7840:         }
7841:         /* Write what is possible and
7842:          * loop back.
7843:          */
7844:         u.u_offset[0] = 0;
7845:         u.u_offset[1] = ip->i_size1;
7846:         u.u_count = min(c, PIPSIZ-u.u_offset[1]);
7847:         c =- u.u_count;
7848:         writei(ip);
7849:         prele(ip);
7850:         if(ip->i_mode&IREAD) {
7851:                 ip->i_mode =& ~IREAD;
7852:                 wakeup(ip+2);
7853:         }
7854:         goto loop;
7855: }
7856: /* ---------------------------       */
7857: 
7858: /* Lock a pipe.
7859:  * If its already locked,
7860:  * set the WANT bit and sleep.
7861:  */
7862: plock(ip)
7863: int *ip;
7864: {
7865:         register *rp;
7866: 
7867:         rp = ip;
7868:         while(rp->i_flag&ILOCK) {
7869:                 rp->i_flag =| IWANT;
7870:                 sleep(rp, PPIPE);
7871:         }
7872:         rp->i_flag =| ILOCK;
7873: }
7874: /* ---------------------------       */
7875: 
7876: /* Unlock a pipe.
7877:  * If WANT bit is on,
7878:  * wakeup.
7879:  * This routine is also used
7880:  * to unlock inodes in general.
7881:  */
7882: prele(ip)
7883: int *ip;
7884: {
7885:         register *rp;
7886: 
7887:         rp = ip;
7888:         rp->i_flag =& ~ILOCK;
7889:         if(rp->i_flag&IWANT) {
7890:                 rp->i_flag =& ~IWANT;
7891:                 wakeup(rp);
7892:         }
7893: }
7894: /* ---------------------------       */
```
