> UNIX V6 Pipe 구현 분석 · [← 이전](./03-plock-prele.md) · [목차](../README.md) · [다음 →](./05-writep.md)

# 4. `readp()`: 파이프에서 읽기 (7756–7800)

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

---

[← 이전](./03-plock-prele.md) · [목차](../README.md) · [다음 →](./05-writep.md)
