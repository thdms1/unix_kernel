# source/: 원본 자료

> 이 폴더의 코드 파일은 **원본 그대로**입니다. 내용을 고치지 않습니다.
> 분석, 메모, 설명은 모두 `analysis/`와 `study-log/`에 따로 씁니다.

## 1. 출처와 판본

| 구분 | 내용 |
|---|---|
| 운영체제 | UNIX Sixth Edition (V6), Bell Labs, 1975 |
| 원본 저장소 (A) | GitHub `dspinellis/unix-history-repo`, 브랜치 `Research-V6` |
| 파일 경로 (A) | `usr/sys/ken/pipe.c` |
| 원본 URL (A) | https://raw.githubusercontent.com/dspinellis/unix-history-repo/Research-V6/usr/sys/ken/pipe.c |
| pipe.c 마지막 커밋 (A) | `1a45f6c8f91014cf66aecce09faa682ece2973a7` (작성일 1975-07-17, 메시지 "Research V6 development") |
| 줄 번호 판본 (B) | Lions 판 UNIX V6 소스 (John Lions, *A Commentary on the UNIX Operating System*, 1977의 소스 목록) |
| 줄 번호 판본 URL (B) | https://warsus.github.io/lions-/all.html (저장소: https://github.com/warsus/lions-) |
| 해설서 | 같은 사이트의 Lions 해설서 HTML 판, 21장 "Pipes": https://warsus.github.io/lions-/lionc/sect0024.html |
| 자료를 받은 날짜 | 2026-11-24 |

**아직 확인하지 못한 것**
- 수업에서 언급한 `v6.pdf`(소스)와 `unix6.pdf`(해설서) 파일은 직접 보지 못했습니다.
- 위 HTML 판과 같은 Lions 자료로 보이지만, PDF의 페이지 번호는 아직 대조하지 않았습니다.

## 2. 파일 목록

| 파일 | 출처 | 범위 | 설명 |
|---|---|---|---|
| `pipe.c` | (A) | 파일 전체, 222줄 | 원본 파일을 바이트 단위 그대로 복사 |
| `pipe_lions.txt` | (B) | Lions 7700–7894 (195줄) | Lions 줄 번호가 붙은 pipe.c. **분석과 보고서는 이 줄 번호를 기준으로 합니다.** |
| `related/main.txt` | (B) | 1532–1638 | `main()`과 바로 위 주석 (main.c) |
| `related/sleep_wakeup.txt` | (B) | 2055–2127 | `sleep()`, `wakeup()` (slp.c) |
| `related/trap_syscall.txt` | (B) | 2751–2771, 2950–2956 | `trap()`의 시스템 콜 분기, `sysent[]`의 42번 pipe 항목 주변 |
| `related/user_struct.txt` | (B) | 0413–0458 | `struct user` (user.h): `u_ofile`, `u_ar0`, `u_count` 등 |
| `related/file_struct.txt` | (B) | 5500–5519 | `struct file`과 FREAD/FWRITE/FPIPE (file.h) |
| `related/inode_struct.txt` | (B) | 5650–5698 | 메모리에 있는 `struct inode`와 플래그 (inode.h) |
| `related/rdwr.txt` | (B) | 5711–5759 | `read()`, `write()`, `rdwr()` (sys2.c): readp/writep를 호출하는 곳 |
| `related/closef.txt` | (B) | 6635–6658 | `closef()` (fio.c): 파이프를 닫을 때 처리 |
| `related/ufalloc_falloc.txt` | (B) | 6821–6865 | `ufalloc()`, `falloc()` (fio.c): pipe()가 쓰는 할당 함수 |
| `related/getf.txt` | (B) | 6611–6632 | `getf()` (fio.c): fd 번호로 file 구조체를 찾는 함수 (2026-11-24 추가) |

**참고 사항**
- `pipe_lions.txt`와 `related/*.txt`는 (B)의 HTML에서 태그만 걷어내고, 위 줄 범위를 그대로 잘라낸 것입니다.
- 코드나 주석의 내용은 바꾸지 않았습니다.
- `trap_syscall.txt`는 서로 떨어진 두 구간을 빈 줄 하나로 구분해 붙였습니다.

## 3. 검증 기록 (2026-11-24)

**1) 두 번 받아서 비교했습니다.** 처음 받은 파일과 다시 받은 파일의 SHA-256 값이 같았습니다.
- `pipe.c` (A): `a7c29aec94676158b00089cc05621919f4d2781f938aac65433e06942b5c0ae0`
- `all.html` (B): `978ccb444e4ea35bd284fc435c0b8ce441eb83b8b1d1d8e4472bb62e6db47e06`

**2) (A)와 (B)의 코드가 같은지 확인했습니다.**
- 비교 방법: 두 파일에서 주석과 빈 줄을 지우고, 공백을 하나로 통일한 뒤 비교했습니다.
- 결과: 남은 코드가 양쪽 모두 126줄이고, 모두 같습니다.
- 차이는 주석 모양(`/*` 줄바꿈 위치), 빈 줄, 구분선(`/* ---- */`)뿐입니다. Lions 판은 인쇄용으로 정리된 것입니다.

**3) 코드 분량 (1차 집계, Lions 줄 번호 기준)**

| 범위 | 전체 | 빈 줄 | 주석 | 실제 코드 |
|---|---|---|---|---|
| pipe.c 전체 7700–7894 | 195 | 12 | 57 | 126 |
| `pipe()` 7723–7753 | 31 | 1 | 0 | 30 |
| `readp()` 7758–7800 | 43 | 2 | 11 | 30 |
| `writep()` 7805–7855 | 51 | 1 | 13 | 37 |
| `plock()` 7862–7873 | 12 | 1 | 0 | 11 |
| `prele()` 7882–7893 | 12 | 1 | 0 | 11 |
| **5개 함수 합계** | 149 | 6 | 24 | **119** |

이 표는 1차 집계입니다. 최종 보고서를 쓰기 전에 다시 세어 보고 확정합니다.
