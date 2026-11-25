> UNIX V6 Pipe 구현 분석 · [← 이전](./02-pipe.md) · [목차](../README.md) · [다음 →](./04-readp.md)

# 3. `plock()`과 `prele()`: 파이프 잠금 (7858–7893)

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

---

[← 이전](./02-pipe.md) · [목차](../README.md) · [다음 →](./04-readp.md)
