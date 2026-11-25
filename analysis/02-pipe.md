> UNIX V6 Pipe 구현 분석 · [← 이전](./01-overview.md) · [목차](../README.md) · [다음 →](./03-plock-prele.md)

# 2. `pipe()`: 파이프 생성 (7723–7753)

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

---

[← 이전](./01-overview.md) · [목차](../README.md) · [다음 →](./03-plock-prele.md)
