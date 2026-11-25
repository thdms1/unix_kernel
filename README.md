# UNIX V6 커널의 Pipe 구현 분석

**C언어 기반 프로세스 간 통신(IPC)의 구현 원리와 커널 자원 관리**

운영체제 과목 "UNIX 커널 분석" 과제입니다. UNIX 6판(V6) 커널의 `pipe.c`를 Lions 판 줄 번호 기준으로 분석했습니다.

- 📄 보고서 PDF: [docs/report.pdf](docs/report.pdf)
- 📝 보고서 원고(Markdown): [docs/report.md](docs/report.md)

## 함수별 분석 문서

| 문서 | 내용 |
|---|---|
| [01-overview.md](analysis/01-overview.md) | 서론, 배경 지식(fd → file → inode, `main()`에서 `pipe()`까지), `PIPSIZ`와 옛날 C 문법 |
| [02-pipe.md](analysis/02-pipe.md) | `pipe()` 파이프 생성 (7723–7753) |
| [03-plock-prele.md](analysis/03-plock-prele.md) | `plock()` / `prele()` 잠금, `sleep` / `wakeup` (7858–7893) |
| [04-readp.md](analysis/04-readp.md) | `readp()` 파이프에서 읽기 (7756–7800) |
| [05-writep.md](analysis/05-writep.md) | `writep()` 파이프에 쓰기 (7803–7855) |
| [06-flow-conclusion.md](analysis/06-flow-conclusion.md) | 운영체제 개념 연결, 실행 흐름 예제, 결론, 참고 자료 |

분석 문서는 `docs/report.md`를 장별로 나눈 것으로, 내용은 보고서와 같습니다.

## 분석 범위

| 구분 | Lions 줄 번호 | 내용 | 전체 줄 | 실제 코드 줄 |
|---|---|---|---|---|
| `pipe.c` 전체 | 7700–7894 | include, `PIPSIZ`, 함수 5개 | 195 | **126** |
| `pipe()` | 7723–7753 | 파이프 생성 | 31 | 30 |
| `readp()` | 7758–7800 | 파이프에서 읽기 | 43 | 30 |
| `writep()` | 7805–7855 | 파이프에 쓰기 | 51 | 37 |
| `plock()` | 7862–7873 | 파이프(inode) 잠금 | 12 | 11 |
| `prele()` | 7882–7893 | 잠금 해제 | 12 | 11 |
| **함수 5개 합계** | | | 149 | **119** |

"실제 코드 줄"은 빈 줄과 주석만 있는 줄을 뺀 수입니다.

## 보고서 구성

1. **서론**: 과제 소개와 분석 목적, 분석 범위, 원본 자료와 줄 번호
2. **배경 지식**: UNIX V6와 커널, Pipe를 고른 이유, fd → file → inode 3단 구조, `main()`에서 `pipe()`까지의 경로
3. **함수별 코드 분석**: `PIPSIZ`, `pipe()`, `plock()`/`prele()`(+ `sleep`/`wakeup`), `readp()`, `writep()`
4. **운영체제 개념과의 연결**: IPC, 커널 자원 할당, 상호 배제, 생산자-소비자, EOF와 SIGPIPE
5. **전체 실행 흐름**: 5000바이트 쓰기와 3000바이트 읽기를 코드 순서대로 따라간 예제
6. **결론**: 분석 정리, 해설서와 비교하며 확인한 점
7. **참고 자료**, **부록**(`pipe.c` 전체 원본)

## 핵심 정리

- 파이프는 새로운 자료구조가 아니라 **이름 없는 inode 하나와 file 구조체 두 개(읽기용, 쓰기용)** 로 만들어진다.
- 읽기 위치는 읽기용 file의 `f_offset[1]`, 쓰기 위치는 inode의 `i_size1`이다. 다 읽으면 둘 다 0으로 되감고, 크기는 `PIPSIZ`(4096)로 제한된다.
- `plock()`/`prele()`는 inode를 한 번에 한 프로세스만 다루게 하는 **상호 배제**이다. `IREAD`/`IWRITE`와 `sleep`/`wakeup`(채널 `ip+1`, `ip+2`)은 버퍼가 비거나 찼을 때 **기다리고 깨우는** 역할을 한다.
- `main()`은 `pipe()`를 직접 부르지 않지만 `binit()`, `iinit()`으로 버퍼와 루트 장치를 준비한다. `pipe()`는 `trap()` → `sysent[42]`로, `readp()`/`writep()`는 `rdwr()`을 거쳐 실행된다.

## 폴더 구성

```
├── README.md
├── analysis/           함수별 분석 문서 (보고서를 장별로 나눔)
│   ├── 01-overview.md
│   ├── 02-pipe.md
│   ├── 03-plock-prele.md
│   ├── 04-readp.md
│   ├── 05-writep.md
│   └── 06-flow-conclusion.md
├── docs/
│   ├── report.pdf      제출용 보고서
│   └── report.md       보고서 원고
└── source/             보고서에 인용한 원본 코드 (수정하지 않음)
    ├── README.md       출처, 판본, 검증 기록
    ├── pipe.c          Research-V6 원본
    ├── pipe_lions.txt  Lions 줄 번호 판 (7700–7894)
    └── related/        main, trap, rdwr, file/inode 구조체, sleep/wakeup 등 관련 코드
```

## 참고 자료

1. John Lions, *A Commentary on the UNIX Operating System*, University of New South Wales, 1977. (21장 Pipes)
2. Lions' Commentary 소스·해설 열람 사이트, https://warsus.github.io/lions-/
3. Diomidis Spinellis, *unix-history-repo*, Research-V6 브랜치, https://github.com/dspinellis/unix-history-repo
