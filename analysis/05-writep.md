> UNIX V6 Pipe 구현 분석 · [← 이전](./04-readp.md) · [목차](../README.md) · [다음 →](./06-flow-conclusion.md)

# 5. `writep()`: 파이프에 쓰기 (7803–7855)

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

---

[← 이전](./04-readp.md) · [목차](../README.md) · [다음 →](./06-flow-conclusion.md)
