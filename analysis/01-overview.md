> UNIX V6 Pipe 구현 분석 · [목차](../README.md) · [다음 →](./02-pipe.md)

# Ⅰ. 서론

## 1. 과제 소개와 분석 목적

이번 과제는 UNIX 6판(V6) 커널 소스 중 원하는 부분을 골라 C 코드 100줄 이상을 분석하는 것이다. 분석 주제는 **UNIX V6 커널의 Pipe 구현**으로 정했다.

파이프는 셸에서 `ls | wc`처럼 한 프로세스의 출력을 다른 프로세스의 입력으로 바로 넘겨 주는 기능으로, 프로세스 간 통신(IPC)의 대표적인 방법이다. 이 보고서의 목적은 다음 세 가지를 실제 코드로 확인하는 것이다.

- 커널이 파이프를 만들 때 어떤 자원(inode, file 구조체, 파일 디스크립터)을 할당하는가
- 데이터를 쓰고 읽을 때 버퍼가 가득 차거나 비면 어떻게 처리하는가
- 두 프로세스가 같은 파이프에 동시에 접근할 때 잠금과 `sleep`/`wakeup`으로 어떻게 동기화하는가

## 2. 분석 범위

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

## 3. 원본 자료와 줄 번호

| 항목 | 내용 |
|---|---|
| 원본 저장소 | GitHub `dspinellis/unix-history-repo`, 브랜치 `Research-V6`, `usr/sys/ken/pipe.c` |
| 줄 번호 기준 | Lions 판 소스 목록 (https://warsus.github.io/lions-/) |
| 해설서 | John Lions, *A Commentary on the UNIX Operating System* 21장 "Pipes" |

두 판본에서 주석과 빈 줄을 지우고 비교해 보면 남은 코드 126줄이 완전히 같다. 차이는 주석 모양과 빈 줄뿐이다. 그래서 내용은 원본 저장소와 같다고 보고, 줄 번호는 해설서와 맞추기 쉬운 **Lions 판 번호**를 썼다. 보고서에 나오는 모든 줄 번호는 이 기준이다.

# Ⅱ. 배경 지식

## 1. UNIX V6와 커널

UNIX V6는 1975년 벨 연구소에서 나온 UNIX 6판으로, PDP-11이라는 16비트 컴퓨터에서 동작했다. 커널 대부분이 C로 작성되어 있고 분량이 크지 않아서, John Lions가 소스 전체에 해설을 붙인 책이 운영체제 교재로 쓰였다.

커널은 프로세스, 메모리, 파일, 장치를 관리하는 운영체제의 핵심 부분이다. 사용자 프로그램은 하드웨어나 커널 자료구조를 직접 건드리지 못하고 **시스템 콜**로 커널에 요청한다. 파이프도 `pipe`, `read`, `write`, `close` 시스템 콜을 통해서만 쓸 수 있다.

## 2. Pipe를 고른 이유

- **IPC를 코드로 직접 볼 수 있다.** 수업에서 다루는 프로세스 간 통신이 실제 커널에서 어떤 자료구조로 만들어지는지 확인할 수 있다.
- **한 파일 안에 생성, 읽기, 쓰기, 동기화가 모두 있다.** 다섯 함수가 서로를 호출하는 구조라서 전체 흐름을 따라가기 좋다.
- **운영체제의 핵심 개념이 함께 나온다.** 커널 자원 할당, 생산자-소비자 문제, `sleep`/`wakeup`, 상호 배제가 모두 120줄 정도 안에 들어 있다.

## 3. 열린 파일을 관리하는 3단 구조

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

## 4. `main()`에서 `pipe()`까지

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

# Ⅲ. 함수별 코드 분석

## 1. 파일 앞부분과 `PIPSIZ` (7700–7715)

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

---

[목차](../README.md) · [다음 →](./02-pipe.md)
