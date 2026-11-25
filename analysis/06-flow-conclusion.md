> UNIX V6 Pipe 구현 분석 · [← 이전](./05-writep.md) · [목차](../README.md)

# Ⅳ. 운영체제 개념과의 연결

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

# Ⅴ. 전체 실행 흐름: 예제로 따라가기

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

# Ⅵ. 결론

## 1. 분석 정리

- 커널은 열린 파일을 **fd(프로세스별) → file 구조체(공유) → inode(공유)** 의 세 단계로 관리한다. 파이프는 이 구조를 그대로 써서 **이름 없는 inode 하나와 file 구조체 두 개**로 만들어진다.
- `pipe()`는 inode와 file 구조체를 고정 크기 표에서 할당하고, 실패하면 할당한 순서의 반대로 되돌린다. 두 fd는 레지스터 R0, R1로 돌려준다.
- 파이프의 데이터는 일반 파일처럼 `readi`/`writei`로 읽고 쓴다. 읽기 위치(`f_offset[1]`)와 쓰기 위치(`i_size1`) 두 값으로 버퍼를 관리하고, 다 읽으면 둘 다 0으로 되감는다. 크기는 `PIPSIZ`(4096)로 제한된다.
- 동기화는 두 겹이다. `plock`/`prele`는 inode를 한 번에 한 프로세스만 다루게 하는 **상호 배제**이고, `IREAD`/`IWRITE`와 `sleep`/`wakeup`은 버퍼가 비거나 찼을 때 **기다리고 깨우는** 역할을 한다.
- `main()`은 `pipe()`를 직접 부르지 않지만, `binit()`과 `iinit()`으로 파이프가 쓸 버퍼와 루트 장치를 준비한다. `pipe()`는 `trap()` → `sysent[42]`로, `readp`/`writep`는 `rdwr()`을 거쳐 실행된다.

## 2. 해설서와 비교하며 확인한 점

- 7736줄의 `r`에는 file 구조체가 아니라 **fd 번호**가 저장된다(6830줄에서 R0에 쓰인 값).
- `prele`는 **잠금을 기다리는** 프로세스만 깨운다. 데이터나 공간을 기다리는 프로세스는 `readp`/`writep`와 `closef`가 채널 `ip+1`, `ip+2`로 따로 깨운다.

# Ⅶ. 참고 자료

1. John Lions, *A Commentary on the UNIX Operating System*, University of New South Wales, 1977. (21장 Pipes)
2. Lions' Commentary 소스·해설 열람 사이트, https://warsus.github.io/lions-/
3. Diomidis Spinellis, *unix-history-repo*, Research-V6 브랜치, https://github.com/dspinellis/unix-history-repo

> `pipe.c` 전체 원본은 [source/pipe_lions.txt](../source/pipe_lions.txt)에 있습니다.

---

[← 이전](./05-writep.md) · [목차](../README.md)
