# OpenWrt Normalized Build Cache - Complete Guide
## One-Stop Reference for Everything

버전: 1.2  
날짜: 2025-11-26  
대상: m2m, sdaemon-telit package solo build  
플랫폼: SDX35 (arm_cortex-a7_neon-vfpv4)  
업데이트: ccache sloppiness 설정 가이드 (환경 변수, rules.mk, ccache.conf)

---

**목차**
- [Section 1: 프로젝트 개요](#section-1-프로젝트-개요)
- [Section 2: 문제 정의](#section-2-문제-정의)
- [Section 3: 핵심 개념](#section-3-핵심-개념)
- [Section 4: Quick Start](#section-4-quick-start)
- [Section 5: 스크립트 상세 설명](#section-5-스크립트-상세-설명)
- [Section 6: Native 환경 가이드](#section-6-native-환경-가이드)
- [Section 7: Docker 환경 가이드](#section-7-docker-환경-가이드)
- [Section 8: ccache 최적화](#section-8-ccache-최적화)
  - [8.3 자동 설정](#83-자동-설정)
  - [8.4 중요: ccache vs IPK 재사용](#84-중요-ccache-vs-ipk-재사용)
  - [8.7 sloppiness 설정 방법: 환경 변수 vs ccache.conf](#87-sloppiness-설정-방법-환경-변수-vs-ccacheconf)
- [Section 9: 문제 해결](#section-9-문제-해결)
  - [9.8 ccache.conf 설정 오류](#98-ccacheconf-설정-오류)
  - [9.9 "No rule to make target" 에러](#99-no-rule-to-make-target-에러)
  - [9.10 feeds 구조 이해](#910-feeds-구조-이해)
  - [9.16 ccache Hit Rate가 낮음](#916-ccache-hit-rate가-낮음)
- [Section 10: FAQ](#section-10-faq)
  - [10.11 OWRT_ROOT가 정확히 어디인가요?](#1011-owrt_root가-정확히-어디인가요)

---

# Section 1: 프로젝트 개요

## 1.1 목적

**m2m 및 sdaemon-telit 패키지의 solo build 시간을 60-70% 단축**

기존 방식:
```
전체 빌드: 60분
Package solo: 15-20분 (m2m)
```

개선 후:
```
전체 빌드: 60분 (동일)
Package solo: 3-5분 (m2m) ← 70% 단축!
```

## 1.2 대상 환경

- **Platform:** SDX35 (arm_cortex-a7_neon-vfpv4)
- **Packages:** m2m, sdaemon-telit
- **Build System:** OpenWrt
- **Deployment:** Docker container (native 서버 빌드 불가)

### 중요: 경로 구조

```
/home/seungtaena/
└── SDX35/                                    ← 전체 프로젝트 루트
    ├── boot_images/
    ├── common/
    ├── build_config/
    └── apps_proc/                            ← Application Processor
        └── owrt/                             ← OWRT_ROOT (OpenWrt 소스)
            ├── Makefile
            ├── feeds.conf
            ├── package/
            ├── dl/
            ├── staging_dir/
            └── ...

Cache Repository:
/home/seungtaena/SDX35/owrt-cache/            ← OWRT_CACHE_ROOT
```

**핵심:**
- `OWRT_ROOT` = `/home/seungtaena/SDX35/apps_proc/owrt`
- `OWRT_CACHE_ROOT` = `/home/seungtaena/SDX35/owrt-cache`

## 1.3 핵심 특징

✅ **Yocto-like Normalized Caching**
- dl/ (소스 다운로드) ← Yocto DL_DIR
- staging_dir/ (전체 동기화) ← IPK 패키지 + toolchain 포함

✅ **경로 독립적**
- 다른 서버/경로에서도 작동
- ccache sloppiness로 최적화

✅ **Docker 완벽 지원**
- 이미지에 cache 포함
- 자동 복원 (symlink 모드)
- Jenkins CI/CD 통합

✅ **간단하고 안전한 사용**
- 스크립트 3개: setup, sync, restore
- 환경 변수로 제어
- 기본 copy 모드로 안전성 강화

## 1.4 v1.1 주요 변경사항 (2025-11-25)

⚠️ **사용자 맞춤 변경:**
1. **기본 복원 모드:** `copy` (이전: symlink)
   - 더 안전한 기본값
   - symlink는 명시적으로 `--mode symlink` 지정 필요

2. **IPK packages sync/restore:** 비활성화
   - staging_dir 전체 동기화로 대체
   - 더 간단하고 포괄적

3. **Toolchain restore:** 비활성화
   - staging_dir 전체 복원에 포함

4. **Configuration files:** `.config_for_package_solo` 추가
   - Package solo build용 별도 설정 파일 지원

5. **Docker 환경:** symlink 모드 유지
   - 컨테이너 격리로 안전함
   - 빠른 복원 (~30초)

## 1.4 프로젝트 파일 구조

```
/home/seungtaena/SDX35/apps_proc/owrt/
│
├── MEGA-GUIDE.md                      ← 이 파일 (모든 것!)
│
├── Native Scripts (6개)
│   ├── setup-repo-structure.sh       # 1. Repository 생성
│   ├── analyze-package-deps.sh       # 2. 의존성 분석
│   ├── build-cache-sync.sh           # 3. Cache 동기화
│   ├── build-cache-restore.sh        # 4. Cache 복원
│   ├── test-cache-solo-build.sh      # 5. 성능 테스트
│   └── build_host_tools.sh           # 6. Host tools 빌드 (레거시)
│
├── Docker Scripts (4개)
│   ├── Dockerfile                     # 이미지 정의
│   ├── docker-entrypoint.sh           # 자동 초기화
│   ├── docker-build-wrapper.sh        # 빌드 헬퍼
│   └── build-docker-image.sh          # 이미지 빌드
│
├── CI/CD
│   └── Jenkinsfile                    # Jenkins 파이프라인
│
└── Cache Repository
    /home/seungtaena/SDX35/owrt-cache/
    ├── dl/                            # 소스 다운로드
    ├── packages/sdx35/                # IPK 패키지
    ├── ccache/                        # 컴파일 캐시
    ├── config/                        # 설정 백업
    └── cache-metadata.json            # 메타데이터
```

---

# Section 2: 문제 정의

## 2.1 현재 방식 (rsync 전체 복사)

### 백업
```bash
rsync -avh owrt/{
    bin,
    build_dir,
    .ccache,
    .config,
    .config.old,
    dl,
    feeds,
    feeds.conf,
    .kernel_built,
    package,
    key-build*,
    staging_dir
} /backup/
```

### 복원
```bash
rsync -avh /backup/{...} owrt/
```

### 빌드
```bash
make package/m2m/compile
```

## 2.2 현재 방식의 문제점

### 문제 1: 용량 과다 (18-20GB)

```
복사되는 내용:
├── bin/          1GB    (최종 결과물, 불필요)
├── build_dir/    14GB   (중간 파일, 대부분 재사용 불가)
├── staging_dir/  2.1GB  (일부만 재사용 가능)
├── dl/           336MB  (재사용 가능! ✓)
├── .ccache/      603MB  (재사용 가능! ✓)
└── 기타          수백MB

실제 재사용 가능: 3-4GB
낭비: 14-16GB (75%)
```

### 문제 2: 경로 의존성

```c
// build_dir 내부 파일들
/home/alice/owrt/build_dir/...  ← Server A
/home/bob/owrt/build_dir/...    ← Server B (다른 경로!)

→ Makefile, .la 파일에 절대 경로 hard-coded
→ 경로 변경 시 일부 재컴파일 필요
```

### 문제 3: ccache 비효율

```c
// 소스 코드
printf("File: %s\n", __FILE__);        // 절대 경로!
printf("Built: %s\n", __TIME__);       // 빌드 시간!

Server A: /home/alice/owrt/main.c, 10:00
        → ccache key = hash(alice, 10:00, ...)
        
Server B: /home/bob/owrt/main.c, 14:00
        → ccache key = hash(bob, 14:00, ...)  (다름!)
        
→ cache MISS! (30-50% hit rate)
```

### 문제 4: 비표준화

- 무엇을 백업해야 하는지 불명확
- 버전 관리 어려움
- 재현성 낮음
- Docker 미지원

## 2.3 비교 요약

| 항목 | 기존 rsync | 제안 방식 |
|------|-----------|----------|
| 백업 용량 | 18-20GB | 3-5GB |
| 백업 시간 | 10-15분 | 2-3분 |
| 복원 시간 | 10-15분 | 30초-3분 |
| 재사용률 | 50-70% | 80-100% |
| 경로 독립성 | ❌ | ✅ |
| ccache hit | 30-50% | 70-90% |
| Docker 지원 | ❌ | ✅ |

---

# Section 3: 핵심 개념

## 3.1 OpenWrt vs Yocto: 중요한 차이점

### Yocto Build System

```
Yocto의 sstate-cache:
├── do_fetch         → 소스 다운로드
├── do_unpack        → 압축 해제
├── do_patch         → 패치 적용
├── do_configure     → configure 실행
├── do_compile       → 컴파일
├── do_install       → 설치
└── do_package       → 패키지 생성

각 task마다 intermediate cache 존재!
```

### OpenWrt Build System

```
OpenWrt에는 sstate-cache가 없음!

대신:
├── dl/              → 소스 다운로드 (Yocto DL_DIR와 동일)
├── build_dir/       → 중간 빌드 파일 (경로 의존적, 재사용 어려움)
├── staging_dir/packages/*.ipk  → 최종 결과물 (재사용 가능!)
└── .ccache/         → 컴파일 캐시 (경로 의존적)
```

### 비교표

| Feature | Yocto | OpenWrt |
|---------|-------|---------|
| Download Cache | DL_DIR ✅ | dl/ ✅ |
| **Binary Cache** | **sstate-cache ✅** | **없음 ❌** |
| Package Cache | N/A | .ipk files ✅ |
| Compile Cache | sstate ✅ | ccache only |
| Task-level Cache | ✅ | ❌ |

### 결론

**OpenWrt는 최종 빌드 산출물(.ipk 패키지)을 재사용하는 것이 핵심!**

## 3.2 Normalized Caching 전략

### 경로 독립적 항목 (핵심!)

```
1. dl/ (336MB)
   - 소스 tarball
   - 어디에 두든 동일하게 작동
   - 경로 무관 ✓
   
2. staging_dir/packages/sdx35/*.ipk (2GB)
   - 빌드된 패키지
   - 어디에 두든 동일하게 작동
   - 경로 무관 ✓
```

### 경로 의존적 항목 (주의 필요)

```
3. .ccache/ (603MB)
   - CCACHE_BASEDIR로 normalized 가능
   - sloppiness 설정 필수
   - 조건부 경로 독립 △
   
4. staging_dir/ (2.1GB)
   - 일부 파일은 경로 의존적
   - 선택적 사용 권장
   - 부분적 경로 의존 △
```

### 제외하는 항목

```
X bin/              - 최종 결과물 (불필요)
X build_dir/        - 중간 파일 (경로 의존, 재사용 불가)
X package/          - 소스 (feeds에 있음)
```

## 3.3 디렉토리 구조 이해

### OpenWrt 빌드 디렉토리

```
/home/seungtaena/SDX35/apps_proc/owrt/
├── dl/                                    ← 소스 다운로드
│   ├── curl-8.0.1.tar.xz
│   ├── glib-2.70.5.tar.xz
│   └── ... (142개 파일)
│
├── staging_dir/
│   ├── packages/
│   │   └── sdx35/                         ← IPK 패키지 (핵심!)
│   │       ├── m2m_1.0-1_*.ipk
│   │       ├── sdaemon-telit_1.0-1_*.ipk
│   │       └── ... (492개 IPK)
│   │
│   ├── host/                              ← 호스트 도구
│   ├── hostpkg/                           ← 호스트 패키지
│   ├── toolchain-*/                       ← 툴체인
│   └── target-*/                          ← 타겟 파일
│
├── .ccache/                               ← ccache 캐시
├── build_dir/                             ← 중간 빌드 (14GB, 제외)
├── bin/                                   ← 최종 결과 (제외)
└── feeds/                                 ← Feed 소스
```

### Cache Repository 구조

```
/home/seungtaena/SDX35/owrt-cache/
├── dl/                                    ← owrt/dl/ 복사
│   └── (소스 tarball들)
│
├── packages/                              ← owrt/staging_dir/packages/ 복사
│   └── sdx35/
│       ├── *.ipk
│       ├── Packages                       ← 패키지 인덱스
│       └── Packages.gz                    ← 압축된 인덱스
│
├── ccache/                                ← owrt/.ccache/ 복사
│
├── staging_dir/                           ← 선택적 복사
│   ├── host/
│   ├── hostpkg/
│   └── toolchain-*/
│
├── config/                                ← 설정 백업
│   ├── .config
│   ├── feeds.conf
│   └── diffconfig
│
└── cache-metadata.json                    ← 메타데이터
```

### 중요: "packages" 디렉토리

많은 혼동이 있는 부분:

```
질문: "packages 디렉토리가 어디 있나요?"

답변:
원본: owrt/staging_dir/packages/sdx35/
캐시: owrt-cache/packages/sdx35/

스크립트가 자동으로 처리하므로 신경 쓸 필요 없음!
```

### 중요: "ccache" vs ".ccache" 디렉토리 이름

**질문:** "왜 소스는 `.ccache`인데 캐시는 `ccache`인가요?"

**답변:** 의도적인 디자인입니다!

```
소스 디렉토리 (OpenWrt):
  owrt/.ccache/          ← 점(.) 포함 (숨김 디렉토리)
  - ccache의 표준 위치
  - 일반적으로 숨김 파일로 관리

캐시 저장소 (Cache Repo):
  owrt-cache/ccache/     ← 점(.) 없음 (일반 디렉토리)
  - 가시성 향상
  - 관리 편의성
  - 명확한 목적 표시
```

**동작:**
```bash
# 백업 (sync)
owrt/.ccache/ → owrt-cache/ccache/
(숨김)          (가시적)

# 복원 (restore)
owrt-cache/ccache/ → owrt/.ccache/
(가시적)            (숨김, ccache 표준)
```

**이점:**
- ✅ Cache repository에서 쉽게 식별
- ✅ 버전 관리 시 명확함
- ✅ ccache는 여전히 표준 위치(`.ccache`) 사용

**스크립트가 자동 처리하므로 신경 쓸 필요 없음!**

---

## 3.5 Symlink 처리 전략 (중요!)

### 문제: 심볼릭 링크와 경로 의존성

OpenWrt 빌드 시스템은 다양한 심볼릭 링크를 사용하며, 이들은 **용도에 따라 다르게 처리**해야 합니다.

### 전략 개요

```
staging_dir/
├── host/           ← 시스템 도구 링크 (symlink 유지!)
├── hostpkg/        ← OpenWrt 빌드 산출물 (dereference!)
├── toolchain-*/    ← 크로스 컴파일러 (symlink 유지!)
├── target-*/       ← 타겟 파일들 (dereference!)
└── packages/       ← IPK 패키지들 (dereference!)
```

### 유형별 상세 설명

#### 1. `staging_dir/host/` - 시스템 도구 (심볼릭 링크 유지)

```bash
# 예시:
host/bin/cmake -> /usr/bin/cmake
host/bin/gcc -> /usr/bin/gcc-11
host/bin/python3 -> /usr/bin/python3.10

# 특징:
- 호스트 시스템의 도구를 가리킴
- GLIBC 버전 의존성 있음
- 환경마다 다른 경로/버전

# rsync 옵션: -avh (링크 유지)
rsync -avh staging_dir/host/ cache/staging_dir/host/

# 이유:
✅ 각 환경의 시스템 도구 사용
✅ GLIBC 호환성 문제 없음
✅ 경로 독립성 유지

❌ -L 사용 시: GLIBC_2.38 not found 에러!
```

#### 2. `staging_dir/hostpkg/` - OpenWrt 빌드 도구 (실제 파일로 변환)

```bash
# 예시:
hostpkg/bin/bzcmp -> /abs/path/owrt/staging_dir/hostpkg/bin/bzdiff
hostpkg/bin/bzless -> /abs/path/owrt/staging_dir/hostpkg/bin/bzmore

# 특징:
- OpenWrt가 빌드한 호스트용 도구
- 절대 경로 사용 (Docker에서 깨짐!)
- 이식 가능한 바이너리

# rsync 옵션: -avhL (링크 해제)
rsync -avhL staging_dir/hostpkg/ cache/staging_dir/hostpkg/

# 이유:
✅ 절대 경로 제거 (Docker 호환)
✅ 자체 포함 바이너리 (GLIBC 독립적)
✅ 완전한 이식성

❌ 링크 유지 시: Docker에서 경로 깨짐!
```

#### 3. `staging_dir/toolchain-*/` - 크로스 컴파일러 (심볼릭 링크 유지)

```bash
# 예시:
toolchain-arm_cortex-a7_neon-vfpv4_gcc-11.2.0_musl_eabi/
├── bin/
│   ├── arm-openwrt-linux-gcc -> arm-openwrt-linux-gcc-11.2.0
│   └── arm-openwrt-linux-g++ -> arm-openwrt-linux-g++-11.2.0
└── lib/lib -> lib  (순환 링크 - 무시됨)

# rsync 옵션: -avh (링크 유지)
rsync -avh staging_dir/toolchain-*/ cache/staging_dir/toolchain-*/

# 이유:
✅ 상대 링크 유지 (상대 경로라 문제없음)
✅ 툴체인 구조 보존
✅ 순환 링크 자동 처리

⚠️ -L 사용 시: 순환 링크 에러 (무해하지만 불필요)
```

#### 4. `staging_dir/target-*/` - 타겟 파일 (실제 파일로 변환)

```bash
# 예시:
target-arm_cortex-a7+neon-vfpv4_musl_eabi/
├── packages/sdx35/m2m_*.ipk -> /abs/path/owrt/bin/packages/.../m2m_*.ipk
└── usr/include/ncursesw -> .  (순환 링크 - 스킵됨)

# rsync 옵션: -avhL (링크 해제)
rsync -avhL staging_dir/target-*/ cache/staging_dir/target-*/

# 이유:
✅ IPK 절대 경로 제거 (Docker 호환)
✅ 헤더/라이브러리 실제 파일화
✅ 완전한 독립성

❌ 링크 유지 시: Docker에서 IPK 찾을 수 없음!
```

#### 5. `staging_dir/packages/` - IPK 패키지 (실제 파일로 변환)

```bash
# 예시:
packages/sdx35/*.ipk -> /abs/path/owrt/bin/packages/.../

# rsync 옵션: -avhL (링크 해제)
rsync -avhL staging_dir/packages/ cache/staging_dir/packages/

# 이유:
✅ 절대 경로 제거
✅ Docker/다른 환경 호환
✅ 완전한 이식성
```

### 전략 요약표

| 디렉토리 | rsync 옵션 | 이유 | Docker 호환 |
|---------|-----------|------|------------|
| `host/` | `-avh` | 시스템 도구, GLIBC 의존성 | ✅ (각 환경의 도구 사용) |
| `hostpkg/` | `-avhL` | OpenWrt 빌드 산출물, 절대 경로 | ✅ (경로 독립적) |
| `toolchain-*/` | `-avh` | 상대 링크, 순환 링크 | ✅ (상대 경로) |
| `target-*/` | `-avhL` | 타겟 파일, IPK 링크 | ✅ (절대 경로 제거) |
| `packages/` | `-avhL` | IPK 패키지 링크 | ✅ (절대 경로 제거) |

### 실제 적용 (스크립트 자동 처리)

```bash
# build-cache-sync.sh가 자동으로:

# 1. host - 링크 유지
rsync -avh staging_dir/host/ cache/staging_dir/host/

# 2. hostpkg - 링크 해제
rsync -avhL staging_dir/hostpkg/ cache/staging_dir/hostpkg/

# 3. toolchain - 링크 유지
rsync -avh staging_dir/toolchain-*/ cache/staging_dir/toolchain-*/

# 4. target - 링크 해제
rsync -avhL staging_dir/target-*/ cache/staging_dir/target-*/

# 5. packages - 링크 해제
rsync -avhL staging_dir/packages/ cache/staging_dir/packages/
```

### 검증 방법

```bash
# Cache 생성 후 확인:

# 1. host: 심볼릭 링크 유지 확인
file owrt-cache/staging_dir/host/bin/cmake
# → symbolic link to /usr/bin/cmake ✅

# 2. hostpkg: 실제 파일 확인
file owrt-cache/staging_dir/hostpkg/bin/bzcmp
# → Bourne-Again shell script 또는 실제 파일 ✅

# 3. packages: 실제 파일 확인
file owrt-cache/staging_dir/packages/sdx35/m2m_*.ipk
# → gzip compressed data ✅

# 4. 절대 경로 링크 확인 (없어야 함)
find owrt-cache/staging_dir -type l -exec readlink {} \; | grep "/home/seungtaena"
# → 빈 출력 (또는 host만 나옴) ✅
```

### 왜 이렇게 복잡한가?

OpenWrt는 3가지 환경을 혼합:

```
1. 호스트 시스템 도구 (host/)
   → 시스템 의존적, 링크 유지

2. OpenWrt 빌드 도구 (hostpkg/)
   → 이식 가능, 링크 해제

3. 타겟 파일 (target-*, packages/)
   → 배포용, 링크 해제
```

**각각 다른 목적과 의존성 → 다른 처리 방식 필요!**

---

# Section 4: Quick Start

## 4.1 전제 조건

```bash
# 1. OpenWrt 소스 존재 확인
# 중요: OWRT_ROOT는 /home/seungtaena/SDX35/apps_proc/owrt 입니다!
cd /home/seungtaena/SDX35/apps_proc/owrt
ls -la Makefile  # 있어야 함

# 2. 한 번 이상 전체 빌드 완료 확인
find staging_dir/packages/sdx35 -name "*.ipk" | wc -l
# → 492개 정도 있어야 함

# 3. Docker 설치 (Docker 사용 시)
docker --version
```

## 4.2 옵션 A: Native 환경 (5분)

```bash
cd /home/seungtaena/SDX35/apps_proc/owrt

# Step 1: Repository 생성
./setup-repo-structure.sh

# Step 2: Cache 생성
./build-cache-sync.sh

# Step 3: 새 빌드에서 복원 (기본값: copy 모드)
./build-cache-restore.sh
# 또는 빠른 symlink 모드:
# ./build-cache-restore.sh --mode symlink

# Step 4: 빌드
make package/m2m/clean
time make package/m2m/compile -j$(nproc)

# 결과: 3-5분 (기존 15-20분)
```

**⚠️ v1.1 변경사항:**
- 기본 복원 모드가 `copy`로 변경되었습니다
- 더 안전하지만 symlink보다 느립니다 (~3분 vs ~30초)

## 4.3 옵션 B: Docker 환경 (15분, 권장)

```bash
cd /home/seungtaena/SDX35/apps_proc/owrt

# Step 1: Cache 준비
./setup-repo-structure.sh
./build-cache-sync.sh

# Step 2: Docker 이미지 빌드 (10-15분)
./build-docker-image.sh
# 결과: owrt-builder:sdx35-cached (8-10GB)

# Step 3: 배포 (선택)
docker tag owrt-builder:sdx35-cached registry.com/owrt-builder:sdx35
docker push registry.com/owrt-builder:sdx35

# Step 4: 사용
docker pull registry.com/owrt-builder:sdx35
docker run -it --rm \
  -v $(pwd):/home/builder/workspace/owrt \
  owrt-builder:sdx35

# 컨테이너 내부 (자동 cache 복원!)
make package/m2m/compile -j$(nproc)
```

## 4.4 옵션 C: Jenkins (자동화)

```bash
# Step 1: Jenkinsfile을 repo에 추가
cp Jenkinsfile /path/to/repo/

# Step 2: Jenkins Job 생성
# - Pipeline job
# - Pipeline script from SCM
# - Jenkinsfile 경로 지정

# Step 3: Build with Parameters
# Package: m2m
# USE_CACHE: true
# Build!
```

---

# Section 5: 스크립트 상세 설명

## 5.1 setup-repo-structure.sh

### 용도
Cache repository 디렉토리 구조를 초기 생성

### 사용법
```bash
./setup-repo-structure.sh

# 환경 변수로 커스터마이징
OWRT_CACHE_ROOT=/custom/path ./setup-repo-structure.sh
```

### 하는 일
```
1. 디렉토리 구조 생성
   /home/seungtaena/SDX35/owrt-cache/
   ├── dl/
   ├── packages/sdx35/{base,telit,updates}
   ├── feeds/
   ├── staging_dir/{host,hostpkg,packages,toolchain,target}
   ├── ccache/
   └── config/

2. 메타데이터 파일 생성
   - cache-metadata.json
   - README.md
   - STRUCTURE.txt

3. .gitignore 생성 (버전 관리용)

4. .gitkeep 배치 (빈 디렉토리 유지)
```

### 출력 예시
```
=========================================
OpenWrt Local Package Repository Setup
=========================================

Repository Root: /home/seungtaena/SDX35/owrt-cache
Target: sdx35
Architecture: arm_cortex-a7_neon-vfpv4

[1/5] Creating repository directory structure...
  ✓ Directory structure created
[2/5] Creating repository metadata...
  ✓ Metadata created
[3/5] Creating .gitignore...
  ✓ .gitignore created
[4/5] Creating convenience structure...
  ✓ Documentation created
[5/5] Repository setup complete!

Repository is ready for use!
```

### 환경 변수
- `OWRT_CACHE_ROOT`: Cache repository 위치 (기본: /home/seungtaena/SDX35/owrt-cache)
- `OWRT_ROOT`: OpenWrt 소스 위치 (기본: 현재 디렉토리)
- `OWRT_TARGET`: 타겟 (기본: sdx35)
- `OWRT_ARCH`: 아키텍처 (기본: arm_cortex-a7_neon-vfpv4)

### 주의사항
- 한 번만 실행하면 됨
- 기존 디렉토리가 있으면 덮어쓰지 않음
- 안전하게 여러 번 실행 가능

---

## 5.2 analyze-package-deps.sh

### 용도
m2m 및 sdaemon-telit 패키지의 의존성 분석

### 사용법
```bash
./analyze-package-deps.sh

# 결과 확인
cat package-deps-analysis.txt
```

### 하는 일
```
1. Makefile에서 DEPENDS 추출
   - m2m 패키지 의존성
   - sdaemon-telit 패키지 의존성

2. staging_dir/packages/에서 IPK 파일 검색

3. 의존성 매핑
   - 각 의존 패키지 → IPK 파일

4. 통계 생성
   - 전체 IPK 개수
   - 전체 크기
```

### 출력 예시
```
OpenWrt Package Dependency Analysis
Generated: Wed Nov 19 2025
========================================

=== M2M Package Analysis ===

## Package: m2m
Makefile: .../feeds/telit/packages/m2m/Makefile

### Direct Dependencies:
+common
+curl
+dsutils
+libopenssl
+libqcmap_client
+libwolfssl
+loc-api-v02
+log-telit
+paho-mqtt-c
+paho-mqtt-sn
+qmi
+qmi-framework
(+glib2, +loc-pla-hdr, +location-client-api, ...)

### IPK Files:
  m2m_1.0-1_arm_cortex-a7_neon-vfpv4.ipk - 4.0K

=== SDAEMON-TELIT Package Analysis ===
...

=== Summary ===
Total IPK files: 492
Total size: 2.0M
```

### 파일 생성
- `package-deps-analysis.txt`: 분석 결과

---

## 5.3 build-cache-sync.sh

### 용도
빌드 완료 후 artifacts를 cache repository로 동기화

### 사용법
```bash
# 기본 동기화
./build-cache-sync.sh

# Dry-run (변경사항 미리보기)
./build-cache-sync.sh --dry-run

# 조용한 모드
./build-cache-sync.sh --quiet

# 다른 경로 지정
./build-cache-sync.sh --repo-root /custom/cache --owrt-root /custom/owrt
```

### 하는 일
```
[1/7] Syncing download cache (dl/)
      rsync -avh --delete owrt/dl/ → cache/dl/

[2/7] Syncing binary packages (⚠️ 현재 비활성화)
      ※ 주석 처리됨 (line 202-215) - 필요시 스크립트에서 활성화

[3/7] Syncing ccache
      rsync -avh owrt/.ccache/ → cache/.ccache/
      ※ 디렉토리 이름: .ccache (숨김) 그대로 유지

[4/7] Syncing configuration files
      cp .config, feeds.conf, .config.old, .config_for_package_solo

[5/6] Syncing feeds metadata
      rsync owrt/feeds/ → cache/feeds/
      rsync owrt/package/feeds/ → cache/package/feeds/

[6/6] Syncing staging_dir components
      rsync owrt/staging_dir/ → cache/staging_dir/ (전체)

[7/7] Updating cache metadata
      cache-metadata.json 업데이트
```

**⚠️ 중요 변경사항:**
- IPK packages sync: 주석 처리 (line 202-215)
- staging_dir 전체 동기화로 변경 (line 281-287)
- diffconfig 생성 비활성화 (line 248-257)

### 옵션
- `--dry-run`: 실제 변경 없이 미리보기
- `--quiet`, `-q`: 출력 최소화
- `--repo-root PATH`: Cache 위치 지정
- `--owrt-root PATH`: OpenWrt 위치 지정
- `--help`, `-h`: 도움말

### 성공 조건 확인
```bash
# 동기화 후 확인
du -sh /home/seungtaena/SDX35/owrt-cache/*

# 예상 결과:
# 321M    dl/          ✓
# 2.0G    packages/    ✓ (중요!)
# 603M    ccache/      ✓
# 24K     config/      ✓
# 2.1G    staging_dir/ ✓

# metadata 확인
cat /home/seungtaena/SDX35/owrt-cache/cache-metadata.json
```

---

## 5.4 build-cache-restore.sh

### 용도
빌드 전 cache repository에서 artifacts 복원

### 사용법
```bash
# Copy 모드 (기본값, 가장 안전)
./build-cache-restore.sh

# Symlink 모드 (빠름)
./build-cache-restore.sh --mode symlink

# Selective 모드 (dl/ + IPK만)
./build-cache-restore.sh --mode selective

# Dry-run
./build-cache-restore.sh --dry-run
```

**⚠️ 중요 변경사항:**
- 기본 모드가 `copy`로 변경되었습니다 (이전: symlink)
- 더 안전한 복원을 위한 설정입니다

### 하는 일
```
[1/7] Restoring download cache (dl/)
      모드에 따라:
      - copy: rsync cache/dl/ → owrt/dl/ (기본값)
      - symlink: ln -sf cache/dl/ → owrt/dl/

[2/7] Restoring binary packages (⚠️ 현재 비활성화)
      ※ 주석 처리됨 - 필요시 스크립트에서 활성화

[3/7] Restoring ccache (선택적)
      rsync cache/.ccache/ → owrt/.ccache/
      + ccache.conf 자동 생성 (sloppiness 포함!)
      ※ symlink 모드에서 활성화

[4/7] Configuration files
      rsync cache/.config, feeds.conf 등

[5/7] Restoring feeds
      rsync cache/feeds/ → owrt/feeds/
      rsync cache/package/feeds/ → owrt/package/feeds/

[6/7] Restoring toolchain (⚠️ 현재 비활성화)
      ※ 주석 처리됨 - 필요시 스크립트에서 활성화

[7/7] Restoring staging_dir/host tools
      copy 또는 symlink 모드에서만 실행
```

**⚠️ 중요 변경사항:**
- IPK packages restore: 주석 처리 (line 272-302)
- Toolchain restore: 주석 처리 (line 396-404)
- staging_dir 전체 복원으로 변경 (선택적)

### 복원 모드 비교

| 모드 | dl/ | packages/ | ccache/ | 속도 | 안전성 | 권장 용도 | 기본값 |
|------|-----|-----------|---------|------|--------|-----------|--------|
| **copy** | 복사 | (비활성화) | 복사 | ⚡ | ⭐⭐⭐ | 프로덕션 | ✅ |
| **symlink** | 심볼릭링크 | (비활성화) | 심볼릭링크 | ⚡⚡⚡ | ⭐⭐ | 개발/테스트 | |
| **selective** | 심볼릭링크 | (비활성화) | 생략 | ⚡⚡ | ⭐⭐⭐ | CI/CD | |

**⚠️ 참고:**
- packages/ (IPK) 복원은 현재 스크립트에서 주석 처리되어 있습니다
- staging_dir 전체가 복원되므로 IPK는 포함됩니다

### ccache 자동 설정

`.ccache/ccache.conf` 파일 자동 생성:
```ini
base_dir = /home/seungtaena/SDX35/apps_proc/owrt
hash_dir = true
max_size = 20G
compression = true
compression_level = 6

# 경로 독립성을 위한 sloppiness (핵심!)
sloppiness = file_macro,time_macros,include_file_mtime,include_file_ctime
```

### 옵션
- `--mode [symlink|copy|selective]`: 복원 모드
- `--dry-run`: 미리보기
- `--quiet`, `-q`: 조용한 모드
- `--repo-root PATH`: Cache 위치
- `--owrt-root PATH`: OpenWrt 위치

### 확인
```bash
# 복원 후 확인
ls -la dl/  # 심볼릭 링크 또는 파일들
ls -la staging_dir/packages/sdx35/*.ipk | wc -l  # IPK 개수
cat .ccache/ccache.conf  # ccache 설정
```

---

## 5.5 test-cache-solo-build.sh

### 용도
Cache를 활용한 package solo build 성능 측정

### 사용법
```bash
# m2m 패키지 테스트
./test-cache-solo-build.sh --package m2m

# sdaemon-telit 패키지 테스트
./test-cache-solo-build.sh --package sdaemon-telit

# Dry-run
./test-cache-solo-build.sh --dry-run
```

### 하는 일
```
[Step 1] Verifying cache repository
         - Cache 통계 확인
         - 필요시 자동 sync

[Step 2] Cleaning package build artifacts
         - make package/<name>/clean
         - 시간 측정

[Step 3] Restoring from cache
         - ./build-cache-restore.sh 실행
         - 시간 측정

[Step 4] Testing package solo build
         Phase 4a: Download (cache hit 확인)
         Phase 4b: Prepare
         Phase 4c: Compile (주요 측정!)
         Phase 4d: Install
         각 단계 시간 측정

[Step 5] Results Summary
         - 단계별 시간
         - ccache 통계
         - 총 시간
```

### 결과 파일
`cache-test-results-YYYYMMDD-HHMMSS.txt`:
```
OpenWrt Package Solo Build Cache Test
Generated: Wed Nov 19 2025
Package: m2m
========================================

Cache Statistics:
  Download files: 142
  IPK packages: 492

Phase Breakdown:
  Clean:    2s
  Restore:  30s
  Download: 0s      ← cache hit!
  Prepare:  45s
  Compile:  180s    ← 주요 측정
  Install:  15s
  -------------------------
  TOTAL:    272s (4.5분)

ccache Statistics:
  cache hit (direct):     1234
  cache hit (preprocessed): 456
  cache miss:              123
  cache hit rate:         93.2%

Build Status: SUCCESS
```

### 환경 변수
- `TEST_PACKAGE`: 테스트할 패키지
- `OWRT_CACHE_ROOT`: Cache 위치
- `OWRT_ROOT`: OpenWrt 위치

---

## 5.6 build-docker-image.sh

### 용도
Cache 포함 Docker 이미지 빌드

### 사용법
```bash
# 기본 빌드
./build-docker-image.sh

# 커스텀 이름/태그
./build-docker-image.sh --name my-owrt --tag v1.0

# Docker 캐시 없이
./build-docker-image.sh --no-cache
```

### 하는 일
```
[1/6] Verifying prerequisites
      - Docker 설치 확인
      - Cache 존재 확인
      - Dockerfile 확인

[2/6] Checking cache size
      - Cache 통계 표시
      - 완전성 검증

[3/6] Preparing build context (시간 소요)
      - Cache 복사: cache/ → docker-build-context/owrt-cache/
      - Scripts 복사
      - Documentation 복사
      - Dockerfile 복사

[4/6] Building Docker image (가장 오래 걸림)
      docker build -t owrt-builder:sdx35-cached .
      - Base image (Ubuntu 22.04)
      - Build tools 설치
      - Cache 추가
      - Scripts 추가

[5/6] Verifying image
      - 이미지 크기 확인
      - Quick test 실행

[6/6] Cleaning up
      - Build context 삭제
```

### 옵션
- `--name NAME`: 이미지 이름
- `--tag TAG`: 이미지 태그
- `--no-cache`: Docker 캐시 사용 안 함
- `--help`, `-h`: 도움말

### 빌드 시간
- 첫 빌드: 10-15분
- 재빌드 (Docker cache): 2-5분

### 결과 이미지
```
REPOSITORY          TAG              SIZE
owrt-builder        sdx35-cached     8-10GB

포함 내용:
- Ubuntu 22.04 base       ~500MB
- Build tools             ~2GB
- OpenWrt cache           ~5GB
- Scripts & docs          ~10MB
```

### 배포
```bash
# Registry에 push
docker tag owrt-builder:sdx35-cached registry.com/owrt:sdx35
docker push registry.com/owrt:sdx35

# 파일로 저장
docker save owrt-builder:sdx35-cached | gzip > owrt-builder.tar.gz

# 다른 서버에서 로드
docker load < owrt-builder.tar.gz
```

---

## 5.7 docker-entrypoint.sh

### 용도
Docker 컨테이너 시작 시 자동 초기화

### 동작
```
컨테이너 시작 시:

1. 환경 정보 표시
   - Image info
   - Cache 통계
   - USE_BUILD_CACHE 상태

2. OpenWrt 소스 확인
   - Volume mount 확인
   - Makefile 존재 확인

3. Cache 자동 복원 (USE_BUILD_CACHE=1)
   - ./build-cache-restore.sh --mode symlink
   - ccache 환경 설정

4. 준비 완료 메시지
   - 빌드 명령어 안내
   - 환경 변수 표시

5. Command 실행
   - bash (대화형)
   - 또는 사용자 지정 명령
```

### 환경 변수
- `USE_BUILD_CACHE`: 0=사용 안 함, 1=사용 (기본: 1)
- `OWRT_ROOT`: OpenWrt 위치
- `OWRT_CACHE_ROOT`: Cache 위치

### 출력 예시
```
=========================================
OpenWrt Build Environment
=========================================

OpenWrt Build Environment with Cache
Cache location: /home/builder/owrt-cache
Workspace: /home/builder/workspace/owrt
Build date: 2025-11-19T10:00:00+09:00

Cache Statistics:
  Download files: 142
  IPK packages: 492
  Total size: 5GB

Usage:
  USE_BUILD_CACHE=1 (default) - Use cache
  USE_BUILD_CACHE=0 - Skip cache

[INFO] Build cache is ENABLED
[INFO] OpenWrt source detected, preparing...
[INFO] Restoring build cache...
[SUCCESS] Cache restored successfully

[READY] Build environment is ready!

Quick commands:
  make package/<name>/{clean,download,prepare}
  time make package/<name>/compile -j$(nproc)

Test cache performance:
  ~/test-cache-solo-build.sh --package m2m

=========================================
```

---

## 5.8 build_host_tools.sh (레거시)

### 용도
/tmp에서 host tools를 별도로 빌드하는 레거시 스크립트

### 사용법
```bash
./build_host_tools.sh <HOST> <CHIPSET> <PROFILE> <TARGET_DIR>

# 예제:
./build_host_tools.sh FE990 sdx75 mbb OWRT.PRODUCT.2.0.r1-07600-SDX75.0-1
```

### 동작 방식
```
1. /tmp/<CHIPSET>-host-tools/<TARGET_DIR>/ 생성
2. 필요한 소스 파일 복사
3. .config 생성 (disable_kernel)
4. make tools/compile 실행
5. make tools/mark_built 실행
6. 불필요한 파일 제거
7. staging_dir과 scripts만 유지
```

### 파라미터
- `HOST`: 호스트 이름 (예: FE990)
- `CHIPSET`: 칩셋 (예: sdx75, sdx35)
- `PROFILE`: 프로파일 (예: mbb)
- `TARGET_DIR`: 타겟 디렉토리 이름

### 출력
```
/tmp/<CHIPSET>-host-tools/<TARGET_DIR>/
├── staging_dir/    ← 빌드된 host tools
├── scripts/        ← 빌드 스크립트
└── .build_done     ← 완료 마커
```

### 타임스탬프 확인
- 같은 날 빌드된 경우 skip
- 다른 날 빌드 시 재실행

### 참고
- 이 스크립트는 레거시 방식입니다
- 새로운 빌드 시스템에서는 `build-cache-restore.sh` 사용 권장
- Host tools도 cache에 포함되어 있습니다

---

## 5.9 docker-build-wrapper.sh

### 용도
Docker 컨테이너 내에서 편리한 빌드

### 사용법
```bash
# 컨테이너 내에서
~/docker-build-wrapper.sh m2m
~/docker-build-wrapper.sh sdaemon-telit

# 병렬 작업 수 지정
BUILD_JOBS=16 ~/docker-build-wrapper.sh m2m
```

### 하는 일
```
[1/4] Cleaning...
      make package/<name>/clean

[2/4] Downloading...
      make package/<name>/download -j$(nproc)

[3/4] Preparing...
      make package/<name>/prepare -j$(nproc)

[4/4] Compiling...
      time make package/<name>/compile -j$(nproc)

결과:
      - 빌드 시간 표시
      - IPK 파일 위치 표시
```

### 환경 변수
- `BUILD_JOBS`: 병렬 작업 수 (기본: $(nproc))

---

# Section 6: Native 환경 가이드

## 6.1 초기 설정

### 1회만: Repository 생성

```bash
cd /home/seungtaena/SDX35/apps_proc/owrt

# Step 1: 구조 생성
./setup-repo-structure.sh

# Step 2: 확인
ls -la /home/seungtaena/SDX35/owrt-cache/
```

### 1회만: 전체 빌드 후 Cache 생성

```bash
# 전체 빌드 (이미 했다면 skip)
make -j$(nproc)

# Cache 동기화
./build-cache-sync.sh

# 확인
du -sh /home/seungtaena/SDX35/owrt-cache/*
```

## 6.2 일상 사용 (Package Solo Build)

### 기본 Workflow

```bash
cd /home/seungtaena/SDX35/apps_proc/owrt

# 1. Cache 복원 (기본값: copy 모드)
./build-cache-restore.sh
# 또는 명시적으로:
# ./build-cache-restore.sh --mode copy

# 2. Package 빌드
make package/m2m/clean
make package/m2m/download      # ~0초 (cache hit!)
make package/m2m/prepare        # ~1분
time make package/m2m/compile -j$(nproc)  # ~3-5분

# 3. 결과 확인
find staging_dir/packages -name "m2m*.ipk"
```

**⚠️ 기본 모드 변경:**
- 기존: `--mode symlink` (기본값)
- 현재: `--mode copy` (기본값) - 더 안전

### 다른 패키지 빌드

```bash
# sdaemon-telit
make package/sdaemon-telit/{clean,download,prepare}
time make package/sdaemon-telit/compile -j$(nproc)  # ~2-4분
```

### 여러 패키지 순차 빌드

```bash
# Cache 복원 (1회)
./build-cache-restore.sh

# 여러 패키지 빌드
for pkg in m2m sdaemon-telit afs-telit; do
    echo "Building $pkg..."
    make package/$pkg/{clean,compile} -j$(nproc)
done

# Cache 업데이트 (선택)
./build-cache-sync.sh
```

## 6.3 복원 모드 선택

### Copy 모드 (기본값 - 프로덕션)

```bash
./build-cache-restore.sh
# 또는 명시적으로:
./build-cache-restore.sh --mode copy

장점:
- 가장 안전 ⭐⭐⭐
- cache 원본 불필요
- 수정 가능
- 기본값으로 설정됨

단점:
- 느림 (~3분)
- 디스크 공간 2배
```

**⚠️ 변경사항:** 기본 모드가 copy로 변경되었습니다!

### Symlink 모드 (빠름 - 개발)

```bash
./build-cache-restore.sh --mode symlink

장점:
- 가장 빠름 (~30초) ⚡⚡⚡
- 디스크 공간 절약
- dl/ 은 읽기 전용으로 안전

주의:
- cache 원본이 필요
- cache 삭제하면 안 됨
- 명시적으로 지정 필요
```

### Selective 모드 (균형 - CI/CD)

```bash
./build-cache-restore.sh --mode selective

포함:
- dl/ (symlink)
- packages/ (copy)

제외:
- ccache
- staging_dir

장점:
- 빠르고 안전
- 디스크 절약
```

## 6.4 성능 측정

```bash
# 자동 테스트
./test-cache-solo-build.sh --package m2m

# 수동 측정
time (
    make package/m2m/clean
    ./build-cache-restore.sh --mode symlink
    make package/m2m/{download,prepare,compile} -j$(nproc)
)

# ccache 통계
ccache -s
```

## 6.5 Cache 업데이트

### 언제?
- 전체 빌드 후
- 새 패키지 추가 후
- 소스 업데이트 후

### 방법

```bash
# 전체 빌드
make -j$(nproc)

# Cache 업데이트
./build-cache-sync.sh

# 확인
cat /home/seungtaena/SDX35/owrt-cache/cache-metadata.json
```

---

# Section 7: Docker 환경 가이드

## 7.1 Docker를 사용하는 이유

### 문제 상황
> "native 서버에서는 package 빌드를 할 수 없고
>  docker image로 들어가서 빌드를 해야한다"

### Docker의 장점

✅ **완전한 격리**
- 호스트 시스템과 독립
- 다른 빌드와 충돌 없음

✅ **재현 가능성**
- 동일한 빌드 환경 보장
- "Works on my machine" 해결

✅ **Cache 포함**
- 이미지에 cache 미리 포함
- Pull만 하면 즉시 사용

✅ **배포 편의성**
- Registry로 간편 배포
- 버전 관리 용이

✅ **CI/CD 친화적**
- Jenkins 완벽 통합
- 병렬 빌드 가능

## 7.2 Docker 이미지 빌드 (관리자)

### 전제 조건

```bash
# 1. Cache 준비
./setup-repo-structure.sh
./build-cache-sync.sh

# 2. Docker 확인
docker --version
docker info
```

### 이미지 빌드

```bash
cd /home/seungtaena/SDX35/apps_proc/owrt

# 기본 빌드 (10-15분)
./build-docker-image.sh

# 과정:
# [1/6] Verifying prerequisites
# [2/6] Checking cache size
# [3/6] Preparing build context (cache 복사, 시간 소요)
# [4/6] Building Docker image (가장 오래)
# [5/6] Verifying image
# [6/6] Cleaning up

# 결과
docker images owrt-builder
# REPOSITORY      TAG           SIZE
# owrt-builder    sdx35-cached  8-10GB
```

### 이미지 배포

```bash
# Registry에 push
docker tag owrt-builder:sdx35-cached registry.com/owrt:sdx35
docker login registry.com
docker push registry.com/owrt:sdx35

# 또는 파일로 저장
docker save owrt-builder:sdx35-cached | gzip > owrt-builder.tar.gz
# → 다른 서버로 전송
# → docker load < owrt-builder.tar.gz
```

## 7.3 Docker 컨테이너 사용 (엔지니어)

### 기본 사용

```bash
# 1. 이미지 pull (첫 번째만)
docker pull registry.com/owrt:sdx35

# 2. 컨테이너 실행
docker run -it --rm \
  -v /path/to/owrt:/home/builder/workspace/owrt \
  registry.com/owrt:sdx35

# 컨테이너 시작 시:
# - IMAGE_INFO 표시
# - Cache 통계 표시
# - 자동 cache 복원! (USE_BUILD_CACHE=1)

# 3. 컨테이너 내부에서 빌드
make package/m2m/{clean,download,prepare}
time make package/m2m/compile -j$(nproc)
# → 3-5분!

# 4. 결과 확인
find staging_dir/packages -name "m2m*.ipk"

# 5. 종료
exit
```

### Cache 제어

```bash
# Cache 사용 (기본)
docker run -it --rm \
  -e USE_BUILD_CACHE=1 \
  -v /path/to/owrt:/home/builder/workspace/owrt \
  owrt-builder:sdx35

# Cache 사용 안 함
docker run -it --rm \
  -e USE_BUILD_CACHE=0 \
  -v /path/to/owrt:/home/builder/workspace/owrt \
  owrt-builder:sdx35
```

### 원샷 빌드

```bash
# 컨테이너 진입 없이 바로 빌드
docker run --rm \
  -v $(pwd):/home/builder/workspace/owrt \
  owrt-builder:sdx35 \
  bash -c 'make package/m2m/compile -j$(nproc)'

# 또는 wrapper 사용
docker run --rm \
  -v $(pwd):/home/builder/workspace/owrt \
  owrt-builder:sdx35 \
  ~/docker-build-wrapper.sh m2m
```

### 리소스 제어

```bash
# CPU/메모리 제한
docker run -it --rm \
  --cpus="16" \
  --memory="32g" \
  -v /path/to/owrt:/home/builder/workspace/owrt \
  owrt-builder:sdx35
```

## 7.4 Docker Compose 사용

### docker-compose.yml

```yaml
version: '3.8'

services:
  owrt-builder:
    image: registry.com/owrt:sdx35
    container_name: owrt-build
    volumes:
      - /path/to/owrt:/home/builder/workspace/owrt
      - build-output:/home/builder/workspace/owrt/bin
    environment:
      - USE_BUILD_CACHE=1
      - BUILD_JOBS=16
    command: bash
    tty: true
    stdin_open: true

volumes:
  build-output:
```

### 사용법

```bash
# 시작
docker-compose up -d

# 접속
docker-compose exec owrt-builder bash

# 빌드
make package/m2m/compile -j$(nproc)

# 종료
docker-compose down
```

## 7.5 Jenkins 통합

### Jenkinsfile 사용

프로젝트에 `Jenkinsfile` 포함:

```groovy
pipeline {
    agent {
        docker {
            image 'registry.com/owrt:sdx35'
            args '-v $WORKSPACE:/home/builder/workspace/owrt -e USE_BUILD_CACHE=1'
        }
    }
    
    parameters {
        choice(name: 'PACKAGE', choices: ['m2m', 'sdaemon-telit'])
        booleanParam(name: 'USE_CACHE', defaultValue: true)
        string(name: 'BUILD_JOBS', defaultValue: '8')
    }
    
    stages {
        stage('Build') {
            steps {
                sh """
                    cd /home/builder/workspace/owrt
                    make package/${params.PACKAGE}/compile -j${params.BUILD_JOBS}
                """
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'staging_dir/packages/**/*.ipk', allowEmptyArchive: true
        }
    }
}
```

### Jenkins Job 설정

```
1. New Item → Pipeline

2. Pipeline 설정:
   - Definition: Pipeline script from SCM
   - SCM: Git
   - Repository URL: <your-repo>
   - Script Path: Jenkinsfile

3. Build with Parameters:
   - Package: m2m
   - USE_CACHE: true
   - BUILD_JOBS: 8

4. Build Now!
```

### 결과

```
빌드 로그:
[Pipeline] stage (Build)
[Pipeline] sh
+ make package/m2m/compile -j8
...
Build completed in 3m 45s

Artifacts:
m2m_1.0-1_arm_cortex-a7_neon-vfpv4.ipk (4.0K)
```

---

# Section 8: ccache 최적화

## 8.1 ccache Sloppiness - 필수 설정!

### 문제: 경로 의존성

```c
// OpenWrt 소스 코드에서
void log_error(const char* msg) {
    fprintf(stderr, "[%s:%d] %s\n", __FILE__, __LINE__, msg);
}

void print_build_info() {
    printf("Built on %s at %s\n", __DATE__, __TIME__);
}
```

**문제:**
```
Server A: /home/alice/owrt/main.c, 10:00
         → __FILE__ = "/home/alice/owrt/main.c"
         → __TIME__ = "10:00:00"
         → ccache key = hash(alice, owrt, 10:00, ...)

Server B: /home/bob/work/owrt/main.c, 14:00
         → __FILE__ = "/home/bob/work/owrt/main.c"
         → __TIME__ = "14:00:00"
         → ccache key = hash(bob, work, 14:00, ...)  (다름!)

결과: cache MISS! 재컴파일!
Cache hit rate: 30-50% (나쁨)
```

### 해결: sloppiness 설정

```ini
# .ccache/ccache.conf
sloppiness = file_macro,time_macros,include_file_mtime,include_file_ctime
```

**효과:**
```
Server A: main.c → ccache key (normalized)
Server B: main.c → ccache key (동일!)

결과: cache HIT! ✅
Cache hit rate: 70-90% (좋음!)
```

## 8.2 sloppiness 옵션 설명

### file_macro (가장 중요!)

**역할:** `__FILE__` 매크로를 슬로피하게 처리

```
없을 때:
  /home/alice/owrt/main.c → hash(alice, owrt, main.c)
  /home/bob/owrt/main.c   → hash(bob, owrt, main.c)  (다름!)

있을 때:
  /home/alice/owrt/main.c → hash(main.c)
  /home/bob/owrt/main.c   → hash(main.c)  (동일!)
```

### time_macros (가장 중요!)

**역할:** `__DATE__`, `__TIME__` 매크로 무시

```
없을 때:
  오전 빌드: __TIME__ = "10:00:00" → hash(10:00)
  오후 빌드: __TIME__ = "14:00:00" → hash(14:00)  (다름!)

있을 때:
  오전 빌드: __TIME__ 무시 → hash()
  오후 빌드: __TIME__ 무시 → hash()  (동일!)
```

### include_file_mtime (중요)

**역할:** include 파일의 수정 시간 무시

```
없을 때:
  rsync 전: config.h (2025-11-19 10:00)
  rsync 후: config.h (2025-11-19 14:00)  ← mtime 변경
  → 재컴파일!

있을 때:
  내용 동일 → cache HIT!
```

### include_file_ctime (선택적)

**역할:** include 파일의 변경 시간(ctime) 무시

- 파일 복사/이동 시에도 cache hit

## 8.3 자동 설정

`build-cache-restore.sh`가 자동으로 설정:

```ini
# .ccache/ccache.conf (자동 생성)
base_dir = /home/seungtaena/SDX35/apps_proc/owrt
hash_dir = true
max_size = 20G
compression = true
compression_level = 6

# Sloppiness for path-independent caching (중요!)
sloppiness = file_macro,time_macros,include_file_mtime,include_file_ctime
```

### 중요: 이것만 있으면 됩니다!

**자주 묻는 질문:**

#### Q1: "ccache.conf 파일만 있으면 되나요?"
**A: 네! 이 파일만 있으면 됩니다.** ✅

```bash
# 이것만 하면 됨!
cat > .ccache/ccache.conf <<EOF
base_dir = /home/seungtaena/SDX35/apps_proc/owrt
hash_dir = true
max_size = 20G
compression = true
compression_level = 6
sloppiness = file_macro,time_macros,include_file_mtime,include_file_ctime
EOF

# 다음 빌드부터 자동 적용!
make package/m2m/compile
```

#### Q2: "sloppiness 적용을 위해 소스를 재컴파일해야 하나요?"
**A: 아니요! 재컴파일 불필요합니다.** ✅

**이유:**
- sloppiness는 **ccache의 동작 방식**을 제어하는 설정
- **컴파일러 옵션이 아님** (CFLAGS가 아님!)
- 기존 ccache 결과물도 계속 사용 가능
- 설정 파일만 생성하면 다음 빌드부터 적용

**동작 방식:**
```bash
# 1단계: 설정 파일 생성 (즉시)
echo "sloppiness = file_macro,..." > .ccache/ccache.conf

# 2단계: 끝! 
# 다음 빌드부터 ccache가 자동으로 읽고 적용

# 3단계: 확인
make package/m2m/compile
ccache -s  # hit rate 확인
```

#### Q3: "기존 ccache는 어떻게 되나요?"
**A: 그대로 사용됩니다!** ✅

```bash
기존 캐시: .ccache/ 디렉토리 (그대로 유지)
         ├── a/1/23456789...  (기존 .o 파일들)
         ├── b/2/34567890...
         └── ...

새 설정:  .ccache/ccache.conf (추가)
         → 기존 캐시 + 새 동작 방식

결과: 기존 캐시도 사용하되, 
      새 빌드는 sloppiness 규칙 적용
```

#### Q4: "언제 적용되나요?"
**A: 다음 gcc 호출부터 즉시!** ✅

```bash
# 설정 전
gcc -c main.c  
→ ccache miss (경로 다르면)

# ccache.conf 생성
cat > .ccache/ccache.conf <<EOF
sloppiness = file_macro,time_macros,...
EOF

# 설정 후 (바로 다음 빌드부터)
gcc -c main.c
→ ccache HIT! (경로 무시하고 hit)
```

**요약:**
- ✅ ccache.conf 파일만 생성
- ✅ 재컴파일 불필요
- ✅ 기존 캐시 유지
- ✅ 다음 빌드부터 자동 적용
- ❌ 컴파일 옵션 변경 필요 없음
- ❌ 소스 코드 수정 필요 없음

---

## 8.4 중요: ccache vs IPK 재사용

### 질문: "ccache만으로는 왜 부족한가?"

많은 분들이 궁금해하시는 부분입니다:
> "ccache + sloppiness로 Docker에 포함시키면 되는 거 아닌가?"

**답변: ccache는 중간 단계 캐시이지만, 범위가 제한적입니다.**

### OpenWrt 빌드 프로세스 전체

```bash
make package/m2m/compile  # 실제로는 6단계 실행

[1/6] Download   (소스 다운로드)      ← ccache 무관 ❌
[2/6] Prepare    (압축 해제, 패치)    ← ccache 무관 ❌
[3/6] Configure  (./configure)       ← ccache 무관 ❌
[4/6] Compile    (gcc 컴파일)        ← ccache 작동! ✅
      ├─ gcc -c main.c → main.o     (ccache 캐시!)
      ├─ gcc -c util.c → util.o     (ccache 캐시!)
      └─ gcc -o binary ...          (linking, ccache 불가)
[5/6] Install    (staging_dir 복사)  ← ccache 무관 ❌
[6/6] Package    (.ipk 생성)         ← ccache 무관 ❌
```

**ccache는 [4/6] Compile 단계의 .o 파일(중간 산출물)만 캐시합니다!**

### 시간 비교

#### A. ccache only (sloppiness 포함)
```
Download:   2-3분  ← 여전히 실행
Prepare:    2-3분  ← 여전히 실행
Configure:  2-3분  ← 여전히 실행
Compile:    1-2분  ← ccache 효과! (원래 10분)
Install:    1분    ← 여전히 실행
Package:    1분    ← 여전히 실행
---------------------------
Total:      9-15분 (50% 절약)
```

#### B. IPK 재사용
```
Download:   0초    ← dl/ 캐시
Prepare:    skip   ← IPK 있음!
Configure:  skip   ← IPK 있음!
Compile:    skip   ← IPK 있음!
Install:    0.5분  ← IPK unpack
Package:    skip   ← IPK 있음!
---------------------------
Total:      0.5-2분 (90% 절약!)
```

### ccache의 3가지 한계

#### 1. 의존성 패키지 문제
```bash
# m2m 빌드에 필요한 것들:
- libcurl  (5분)
- glib2    (10분)
- openssl  (15분)
- qmi      (8분)
...

ccache only:  각 의존성을 빌드 (30-40분!)
IPK 재사용:   모든 의존성 skip (0분!)
```

#### 2. 다른 5개 단계는 매번 실행
```bash
./configure --prefix=/usr ...
checking for gcc... yes
checking for library... yes
...
```
**2-3분 소요, ccache는 도움 안 됨**

#### 3. Linking은 ccache 불가
```bash
gcc -c main.c -o main.o      ← ccache OK ✅
gcc -shared -o lib.so ...    ← ccache 불가 ❌
strip lib.so                 ← ccache 불가 ❌
```

### OpenWrt vs Yocto

| 시스템 | 포괄적 중간 캐시 | 대안 |
|--------|----------------|------|
| Yocto | sstate-cache (모든 task) | - |
| OpenWrt | ❌ 없음 | IPK 재사용 ✅ |

**OpenWrt는 Yocto의 sstate-cache가 없습니다!**
- ccache: Compile 단계만 (제한적 중간 캐시)
- IPK: 전체 프로세스 결과 (유일한 "전체 skip" 방법)

### 최적 전략: 둘 다 사용

```
우선순위:

1차 방어: IPK 재사용 (90% 케이스)
   ↓
   IPK 있음? → Yes → Skip! (0.5-2분) ✅
   ↓ No (소스 변경)
   
2차 방어: ccache (10% 케이스)
   ↓
   빌드 필요, ccache로 최적화
   ↓
   9-15분 (ccache 없으면 18-23분)
```

**결론:**
- ccache는 중간 캐시이지만 범위가 좁음 (Compile만)
- IPK는 전체 빌드 skip 가능 (OpenWrt의 sstate 대체)
- 둘 다 포함하는 것이 최선!

---

## 8.5 효과 측정

### ccache 통계 보기

```bash
$ ccache -s

cache hit (direct)       :  1234  ← 직접 히트
cache hit (preprocessed) :   456  ← 전처리 후 히트
cache miss               :   123  ← 미스
cache hit rate           :  93.2% ← 이것 확인!

files in cache           :  3456
cache size               :   1.2 GB
max cache size           :  20.0 GB
```

### 좋은 cache hit rate

- **90% 이상:** 매우 좋음 ✅
- **70-90%:** 좋음 ✓
- **50-70%:** 보통 (개선 필요)
- **50% 이하:** 나쁨 (설정 확인!)

### 비교 테스트

```bash
# sloppiness 없이 빌드
export CCACHE_SLOPPINESS=""
ccache -z  # 통계 초기화
time make package/m2m/compile -j$(nproc)
ccache -s  # hit rate 확인 → 30-50%

# sloppiness 있게 빌드
export CCACHE_SLOPPINESS="file_macro,time_macros,include_file_mtime,include_file_ctime"
ccache -z
time make package/m2m/compile -j$(nproc)
ccache -s  # hit rate 확인 → 70-90%
```

## 8.6 주의사항

### 디버깅 시

```c
// __FILE__ 매크로가 이상하게 보일 수 있음
printf("Error in %s\n", __FILE__);
// 출력: "Error in <built-in>"
```

**해결:**
```bash
# 디버그 빌드 시 ccache 비활성화
CCACHE_DISABLE=1 make package/m2m/compile V=s
```

### 타임스탬프 필요 시

```bash
# 별도 스크립트로 생성
echo "const char* BUILD_TIME = \"$(date)\";" > build_time.h
```

---

## 8.7 sloppiness 설정 방법: 환경 변수 vs ccache.conf

### 핵심 이해: 캐시 생성과 조회

**sloppiness는 캐시 키 계산에 영향을 줍니다:**

```
캐시 저장 시: 소스 + sloppiness 규칙 → 키 abc123 → 저장
캐시 조회 시: 소스 + sloppiness 규칙 → 키 abc123 → 찾음! ✅

캐시 저장 시: 소스 + sloppiness 규칙 → 키 abc123 → 저장
캐시 조회 시: 소스 + NO sloppiness → 키 def456 → 못 찾음! ❌
```

**결론: 캐시 생성할 때와 조회할 때 모두 동일한 sloppiness 설정 필요!**

### 방법 1: rules.mk에 환경 변수 추가 (추천! ⭐)

OpenWrt의 `include/rules.mk`에 추가:

```makefile
ifneq ($(CONFIG_CCACHE),)
  TARGET_CC:= ccache_cc
  TARGET_CXX:= ccache_cxx
  HOSTCC:= ccache $(HOSTCC)
  HOSTCXX:= ccache $(HOSTCXX)
  export CCACHE_BASEDIR:=$(TOPDIR)
  export CCACHE_DIR:=$(if $(call qstrip,$(CONFIG_CCACHE_DIR)),$(call qstrip,$(CONFIG_CCACHE_DIR)),$(TOPDIR)/.ccache)
  export CCACHE_COMPILERCHECK:=%compiler% -dumpmachine; %compiler% -dumpversion
  
  # ★ Path-independent caching (for build cache portability)
  export CCACHE_SLOPPINESS:=file_macro,time_macros,include_file_mtime,include_file_ctime
  export CCACHE_MAXSIZE:=20G
  export CCACHE_COMPRESS:=1
  export CCACHE_COMPRESSLEVEL:=6
endif
```

**⚠️ 주의: Makefile 문법!**
```makefile
# 잘못된 방법 (따옴표가 값에 포함됨!):
export CCACHE_SLOPPINESS="file_macro,..."
# → 실제 값: "file_macro,..." (따옴표 포함!)

# 올바른 방법:
export CCACHE_SLOPPINESS:=file_macro,...
# → 실제 값: file_macro,... (따옴표 없음) ✅
```

**장점:**
- .ccache 디렉토리 없어도 작동
- 첫 빌드부터 sloppiness 적용
- 원본과 복사 환경 모두에서 자동 적용 (rules.mk가 같으면)

### 방법 2: ccache.conf 파일 (보조)

```bash
mkdir -p .ccache
cat > .ccache/ccache.conf << 'EOF'
base_dir = /path/to/owrt
sloppiness = file_macro,time_macros,include_file_mtime,include_file_ctime
max_size = 20G
compression = true
compression_level = 6
EOF
```

**사용 시점:**
- rules.mk가 다른 환경에서 캐시 사용 시
- 환경 변수 설정이 안 되는 특수 환경

### 방법 3: 쉘 환경 변수 (.build-env)

```bash
# .build-env
export CCACHE_DIR=$PWD/.ccache
export CCACHE_BASEDIR=$PWD
export CCACHE_SLOPPINESS="file_macro,time_macros,include_file_mtime,include_file_ctime"
export CCACHE_MAXSIZE="20G"
```

**사용:**
```bash
source .build-env
make package/m2m/compile
```

### 우선순위

ccache는 다음 순서로 설정을 읽습니다:

```
1. 환경 변수 (CCACHE_SLOPPINESS 등) ← 최우선
2. $CCACHE_DIR/ccache.conf
3. 기본값
```

**환경 변수가 있으면 ccache.conf보다 우선!**

### 시나리오별 필요 설정

| 시나리오 | rules.mk 동일 | 필요한 설정 |
|---------|-------------|------------|
| 원본에서 캐시 생성 | - | rules.mk에 환경 변수 |
| 같은 환경에서 조회 | ✅ | 추가 설정 불필요 |
| 다른 환경 (rules.mk 동일) | ✅ | 추가 설정 불필요 |
| 다른 환경 (rules.mk 다름) | ❌ | ccache.conf 또는 .build-env |
| Docker (rules.mk 포함) | ✅ | 추가 설정 불필요 |

### 확인 방법

```bash
# 1. 환경 변수 확인
echo $CCACHE_SLOPPINESS

# 2. ccache가 읽는 설정 확인
ccache -p | grep sloppiness
# (environment) sloppiness = file_macro,...  ← 환경 변수에서
# ($CCACHE_DIR/ccache.conf) sloppiness = ... ← 설정 파일에서

# 3. 설정 적용 테스트
ccache -z  # 통계 초기화
make package/m2m/compile -j$(nproc)
ccache -s | grep "Hits:"
```

### .ccache 디렉토리 생성 시점

**질문: ".ccache 디렉토리가 언제 생성되나요?"**

```
1. ccache가 처음 실행될 때 자동 생성
2. 또는 CCACHE_DIR로 지정된 경로에 생성

시점:
make 시작 → ccache 실행 → .ccache 생성 → 캐시 저장
```

**문제:**
```
첫 빌드 전에 ccache.conf가 없음
→ .ccache 디렉토리도 없음
→ sloppiness 없이 캐시 생성됨 ❌
```

**해결:**
```bash
# 방법 A: rules.mk에 환경 변수 (추천!)
# → .ccache 없어도 환경 변수로 sloppiness 적용

# 방법 B: 미리 디렉토리와 설정 생성
mkdir -p .ccache
cat > .ccache/ccache.conf << 'EOF'
sloppiness = file_macro,time_macros,...
EOF
# → 첫 빌드부터 적용
```

### 요약: 권장 설정 방법

**원본 소스에서 (캐시 생성용):**
```makefile
# include/rules.mk에 추가
export CCACHE_SLOPPINESS:=file_macro,time_macros,include_file_mtime,include_file_ctime
```

**복원 환경에서 (캐시 사용용):**
```bash
# rules.mk가 같으면: 추가 설정 불필요!
# rules.mk가 다르면: .build-env 또는 ccache.conf 사용
```

---

# Section 9: 문제 해결

## 9.1 owrt-cache가 비어있음

### 증상

```bash
$ du -sh /home/seungtaena/SDX35/owrt-cache/*
4.0K    ccache/       ← 거의 비어있음
4.0K    config/       ← 거의 비어있음
321M    dl/           ← OK
4.0K    feeds/        ← 거의 비어있음
20K     packages/     ← 문제! 2GB 있어야 함
24K     staging_dir/  ← 거의 비어있음
```

### 원인

`build-cache-sync.sh`를 실행하지 않았거나 IPK 파일이 없음

### 해결

```bash
# 1. IPK 파일 확인
find staging_dir/packages/sdx35 -name "*.ipk" | wc -l
# → 492개 정도 있어야 함

# 없으면: 전체 빌드 필요
make -j$(nproc)

# 2. Cache 동기화
./build-cache-sync.sh

# 3. 확인
du -sh /home/seungtaena/SDX35/owrt-cache/packages/
# → 2GB 정도 나와야 성공
```

## 9.2 "packages 디렉토리를 찾을 수 없다"

### 증상

스크립트에서 packages 디렉토리 관련 오류

### 원인

경로 혼동

### 설명

```
"packages" 디렉토리는:
  원본: owrt/staging_dir/packages/sdx35/
  캐시: owrt-cache/packages/sdx35/

스크립트가 자동으로 처리하므로 신경 쓸 필요 없음!
```

## 9.3 ccache가 작동하지 않음

### 증상

```bash
$ ccache -s
cache hit rate: 10%  ← 너무 낮음
```

### 원인

1. sloppiness 설정 누락
2. CCACHE_BASEDIR 미설정
3. 경로 변경

### 해결

```bash
# 1. ccache 설정 확인
cat .ccache/ccache.conf
# sloppiness 있는지 확인

# 2. 환경 변수 확인
echo $CCACHE_DIR
echo $CCACHE_BASEDIR

# 3. 재설정
./build-cache-restore.sh --mode copy

# 4. 통계 초기화 후 테스트
ccache -z
make package/m2m/compile -j$(nproc)
ccache -s
# → hit rate 70% 이상 나와야 함
```

## 9.4 Docker: Permission denied

### 증상

```
docker run ...
Permission denied: /home/builder/workspace/owrt
```

### 원인

Volume mount 권한 문제

### 해결

```bash
# 방법 1: UID/GID 매칭
docker run -it --rm \
  -u $(id -u):$(id -g) \
  -v /path/to/owrt:/home/builder/workspace/owrt \
  owrt-builder:sdx35

# 방법 2: 권한 변경
sudo chown -R $USER:$USER /path/to/owrt

# 방법 3: Docker에서 실행
docker run -it --rm \
  -v /path/to/owrt:/home/builder/workspace/owrt \
  owrt-builder:sdx35 \
  bash -c 'sudo chown -R builder:builder /home/builder/workspace/owrt'
```

## 9.5 심볼릭 링크 문제

### 증상

```
./build-cache-restore.sh --mode symlink
Error: dl/ already exists as directory
```

### 원인

dl/이 이미 일반 디렉토리로 존재

### 해결

```bash
# 방법 1: 백업 후 제거
mv dl dl.backup
./build-cache-restore.sh --mode symlink

# 방법 2: copy 모드 사용
./build-cache-restore.sh --mode copy
```

## 9.6 빌드 실패: 의존성 누락

### 증상

```
make package/m2m/compile
ERROR: package/m2m/compile failed
```

### 원인

의존 패키지가 staging_dir에 없음

### 해결

```bash
# 방법 1: copy 모드로 전체 복원
./build-cache-restore.sh --mode copy

# 방법 2: 의존성 패키지 먼저 빌드
make package/common/compile
make package/qmi-framework/compile
make package/m2m/compile

# 방법 3: 의존성 분석
./analyze-package-deps.sh
cat package-deps-analysis.txt
# → 누락된 패키지 확인 후 빌드
```

## 9.7 Docker: No space left on device

### 증상

```
docker build ...
no space left on device
```

### 원인

Docker 이미지가 크거나 디스크 부족

### 해결

```bash
# 1. Docker 정리
docker system prune -a
docker volume prune

# 2. 빌드 context 크기 확인
du -sh docker-build-context/

# 3. staging_dir 제외하고 빌드
# build-docker-image.sh 수정 필요

# 4. 다른 디스크 사용
docker build --no-cache \
  --storage-opt dm.basesize=50G \
  -t owrt-builder:sdx35 .
```

## 9.8 ccache.conf 설정 오류

### 증상

```bash
# ccache 경고 또는 낮은 hit rate
ccache: WARNING: Unknown sloppiness option: file_macro
cache hit rate: 30%
```

### 원인

ccache 버전에 따라 sloppiness 옵션이 다름

### 설명

**중요:** `file_macro` 옵션은 **ccache 버전에 따라 다릅니다!**

| ccache 버전 | file_macro 지원 | 대체 방법 |
|------------|----------------|----------|
| **3.x** | ✅ 지원 | 그대로 사용 |
| **4.x** | ❌ 제거됨 | base_dir로 해결 |

### ccache 버전 확인

```bash
ccache --version
# ccache version 3.7.7  → file_macro 사용 가능 ✅
# ccache version 4.8    → file_macro 제거됨 ❌
```

### 해결 방법

#### 옵션 1: 버전별 분기 (완벽)

```bash
# build-cache-restore.sh 수정
CCACHE_VERSION=$(ccache --version 2>/dev/null | head -1 | grep -oP '\d+' | head -1)

if [ "$CCACHE_VERSION" = "3" ]; then
    # ccache 3.x용
    SLOPPINESS="file_macro,time_macros,include_file_mtime,include_file_ctime"
else
    # ccache 4.x용 (file_macro 제거)
    SLOPPINESS="time_macros,include_file_mtime,include_file_ctime"
fi

cat > "$OWRT_ROOT/.ccache/ccache.conf" <<EOF
base_dir = $OWRT_ROOT
hash_dir = true
max_size = 20G
compression = true
compression_level = 6
sloppiness = $SLOPPINESS
EOF
```

#### 옵션 2: 안전한 호환 설정 (권장)

**대부분의 OpenWrt 환경은 ccache 3.x 사용**

```ini
# 현재 설정 그대로 유지
# ccache 3.x: 모든 옵션 작동 ✅
# ccache 4.x: file_macro 무시, 나머지 작동 ✅
# (경고 나오지만 치명적이지 않음)
sloppiness = file_macro,time_macros,include_file_mtime,include_file_ctime
```

#### 옵션 3: 최대 호환성 (보수적)

```ini
# file_macro 제거, 양쪽 버전 모두 작동
base_dir = /home/seungtaena/SDX35/apps_proc/owrt
hash_dir = true
max_size = 20G
compression = true
compression_level = 6
sloppiness = time_macros,include_file_mtime,include_file_ctime
```

**Trade-off:**
- ccache 3.x: 약간의 hit rate 감소 (5-10%)
- ccache 4.x: 완벽 작동
- `base_dir`가 대부분의 경로 정규화 담당

### 왜 file_macro가 제거되었나?

ccache 4.x부터는 `base_dir` 설정만으로도 `__FILE__` 매크로 경로가 자동으로 정규화됩니다.

```bash
# ccache 3.x: file_macro 필요
sloppiness = file_macro  # __FILE__ 무시

# ccache 4.x: base_dir로 자동 해결
base_dir = /path/to/owrt  # __FILE__ 자동 정규화
```

### 확인

```bash
# 설정 후 테스트
ccache -z  # 통계 초기화
make package/m2m/compile -j$(nproc)
ccache -s  # hit rate 확인

# 목표:
# ccache 3.x: 70-90% hit rate
# ccache 4.x: 70-90% hit rate (base_dir 효과)
```

## 9.9 "No rule to make target" 에러

### 증상

```bash
make package/m2m/clean
make[1]: *** No rule to make target 'package/m2m/clean'.  Stop.
make: *** [/home/seungtaena/test/SDX35/apps_proc/owrt/include/toplevel.mk:230: package/m2m/clean] Error 2
```

### 원인

**OpenWrt 빌드 시스템이 m2m 패키지를 인식하지 못함**

```
feeds/telit/packages/m2m/        ✅ 존재 (실제 소스)
package/feeds/telit/m2m/         ❌ 없음! (심볼릭 링크 없음)
→ make가 못 찾음 → "No rule to make target"
```

### OpenWrt Feeds 메커니즘

```
정상 동작 시:
1. feeds/telit/packages/m2m/              ← 실제 패키지 소스
2. package/feeds/telit/m2m/               ← 심볼릭 링크 (빌드 시스템 연결)
   → ../../../../feeds/telit/packages/m2m

make는 package/feeds/를 스캔:
package/feeds/telit/m2m/Makefile ✅ 발견
→ make package/m2m/clean 타겟 생성 ✅
```

### Cache 복원 시 문제

```bash
# rsync/cp로 cache 복원 시:
rsync -avh cache/staging_dir/ owrt/staging_dir/
# → 실제 파일만 복사됨
# → package/feeds/의 심볼릭 링크는 복원 안 됨! ❌

# 결과:
ls -la package/feeds/telit/m2m
# → No such file or directory
```

### 해결 방법

#### 즉시 해결

```bash
cd /home/seungtaena/test/SDX35/apps_proc/owrt

# feeds 업데이트 및 설치 (10초 소요)
./scripts/feeds update telit
./scripts/feeds install m2m sdaemon-telit

# 확인
ls -la package/feeds/telit/m2m
# → ../../../../feeds/telit/packages/m2m (심볼릭 링크)

# 이제 작동함:
make package/m2m/clean        # ✅
make package/m2m/compile      # ✅
```

#### 근본 해결: build-cache-restore.sh 개선

Cache 복원 스크립트에 feeds 복원 단계 추가:

```bash
# build-cache-restore.sh 수정 제안

# [6/6] Restoring feeds links (NEW!)
restore_feeds_links() {
    log_info "[6/6] Restoring feeds links..."

    cd "$OWRT_ROOT" || return 1

    # feeds가 이미 존재하면 (cache에서 복원됨)
    if [ -d "feeds/telit" ]; then
        # index 재생성
        ./scripts/feeds update telit

        # 필수 패키지만 install (심볼릭 링크 생성)
        ./scripts/feeds install m2m sdaemon-telit

        log_success "Feeds links created"
    else
        log_warning "feeds/telit not found, skipping"
    fi
}
```

### 왜 이런 일이 발생하나?

OpenWrt는 **2단계 구조**를 사용:

1. **feeds/** = 패키지 저장소 (실제 소스)
2. **package/feeds/** = 빌드 시스템 연결 (심볼릭 링크)

Cache는 빌드 산출물만 저장하므로, **빌드 시스템 연결**은 동적으로 재생성 필요!

### 확인

```bash
# 정상 상태 확인
ls -la package/feeds/telit/m2m
# → lrwxrwxrwx ... ../../../../feeds/telit/packages/m2m

# 빌드 가능 확인
make package/m2m/clean
# → Cleaning package/m2m... OK
```

## 9.10 feeds 구조 이해

많은 사용자가 혼동하는 부분을 명확히 정리합니다.

### feeds/ vs package/feeds/ 차이

| 항목 | `feeds/` | `package/feeds/` |
|------|----------|------------------|
| **본질** | 실제 저장소 | 심볼릭 링크 계층 |
| **내용** | 소스 코드, Makefile | 링크만 (데이터 없음) |
| **생성** | `feeds update` | `feeds install` |
| **용도** | 패키지 보관 | 빌드 시스템 연결 |
| **빌드 인식** | ❌ 직접 인식 못함 | ✅ 여기를 스캔함 |
| **크기** | 수 GB | 수 KB (링크만) |
| **예제** | `feeds/telit/packages/m2m/` | `package/feeds/telit/m2m/` |

### 실제 구조 예시

#### feeds/ (실제 저장소)

```
feeds/telit/packages/m2m/
├── Makefile              ← 실제 파일 (3KB)
├── src/
│   ├── main.c            ← 실제 파일 (50KB)
│   ├── config.h
│   └── utils.c
├── patches/
│   └── 001-fix.patch
└── README.md

Total: ~500KB (실제 데이터)
```

#### package/feeds/ (심볼릭 링크)

```
package/feeds/telit/m2m/
└── → ../../../../feeds/telit/packages/m2m/

Total: ~100 bytes (링크 정보만)
```

### 동작 메커니즘

```bash
# Step 1: feeds 다운로드
./scripts/feeds update -a
# 결과: feeds/ 디렉토리 채워짐
# 메타데이터 생성: feeds/telit.index

# Step 2: 패키지 설치 (심볼릭 링크 생성)
./scripts/feeds install m2m
# 결과: package/feeds/telit/m2m → feeds/telit/packages/m2m
# 빌드 시스템이 이제 인식 가능!

# Step 3: Make가 패키지 인식
make package/m2m/compile
# OpenWrt Makefile이 package/feeds/ 스캔
# → m2m 발견 → 타겟 생성 ✅
```

### 왜 이렇게 복잡한가?

#### 이유 1: 모듈화 및 독립성

```
OpenWrt Core (작고 안정적)
    +
외부 Feeds (크고 다양함, 자주 업데이트)
    =
유연한 빌드 시스템
```

#### 이유 2: 선택적 설치

```bash
# 3000개 패키지 중 필요한 것만:
./scripts/feeds install curl htop m2m
# → package/feeds/에 3개만 링크됨
# → 빌드 시스템 부하 감소
```

#### 이유 3: 버전 독립성

```bash
# OpenWrt 21.02 + 최신 packages feed
feeds/packages/ → branch: master (최신)
owrt/           → branch: openwrt-21.02

# 각자의 업데이트 주기 유지 가능
```

### Cache 전략 관련 시사점

```bash
# feeds/ 포함 여부:
Option 1: feeds/를 cache에 포함 (권장)
  ✅ 소스 코드 보존
  ✅ 버전 일관성
  ❌ Cache 크기 증가 (수백 MB)

Option 2: feeds/를 제외, feeds update로 재생성
  ✅ Cache 크기 작음
  ❌ 버전 drift 가능성

# package/feeds/ 포함 여부:
❌ 포함 불필요!
  → 심볼릭 링크는 동적 생성이 더 안전
  → `./scripts/feeds install`로 재생성 (1초)
```

### 권장 Cache 구조

```
owrt-cache/
├── dl/                   ✅ 포함
├── packages/             ✅ 포함 (.ipk)
├── staging_dir/          ✅ 포함
├── ccache/               ✅ 포함
├── feeds/                ✅ 포함 (소스 일관성)
└── config/
    └── feeds-installed.txt   ✅ 포함 (메타데이터)
        # m2m
        # sdaemon-telit

# package/feeds/는 복원 후 재생성:
./scripts/feeds install $(cat feeds-installed.txt)
```

### 핵심 정리

1. **feeds/** = 패키지 **저장소** (실제 코드)
2. **package/feeds/** = 빌드 시스템 **연결 계층** (심볼릭 링크)
3. **연결 과정:**
   ```
   feeds update → feeds/ 채움
   feeds install → package/feeds/ 링크 생성
   make → package/feeds/ 스캔
   ```
4. **Cache 복원 시 feeds install 재실행 필수!**

---

## 9.11 Prebuilt 패키지 빌드 실패: "Cannot open: No such file or directory" (중요!)

### 증상

```bash
make package/m2m/compile

# 빌드 로그:
tar: ../prebuilt_/libvmmem_prebuilt.tar.gz: Cannot open: No such file or directory
tar: Error is not recoverable: exiting now
make[2]: *** [Makefile:82: /home/.../libvmmem-1.0.0/.built] Error 2
make[1]: *** [package/Makefile:120: package/feeds/qtibspprop/libvmmem/compile] Error 2

# 또는:
tar: ../prebuilt_/xmllib_prebuilt.tar.gz: Cannot open: No such file or directory
make[2]: *** [Makefile:76: /home/.../xmllib-1/.built] Error 2
```

**로그 예시 (m2m_8.log):**
```
Line 312: tar: ../prebuilt_/libvmmem_prebuilt.tar.gz: Cannot open
Line 327: make[2]: *** [Makefile:82: .../libvmmem-1.0.0/.built] Error 2
Line 418: tar: ../prebuilt_/xmllib_prebuilt.tar.gz: Cannot open
Line 427: make[2]: *** [Makefile:76: .../xmllib-1/.built] Error 2
Line 1211: make: *** [include/toplevel.mk:230: package/m2m/compile] Error 2
```

### 원인

**핵심: `EXTERNAL_VARIANT` 환경 변수 누락!**

OpenWrt의 일부 패키지는 **prebuilt 바이너리**를 사용합니다:
- `libvmmem` (Qualcomm VMM memory library)
- `xmllib` (XML parsing library)
- 기타 vendor-specific 패키지

이러한 패키지의 Makefile:

```makefile
PKG_NAME:=libvmmem
PKG_SOURCE:=$(PKG_NAME)_prebuilt.tar.gz
PKG_SOURCE_URL:=../prebuilt_$(EXTERNAL_VARIANT)/

# EXTERNAL_VARIANT가 비어있으면:
# ../prebuilt_/libvmmem_prebuilt.tar.gz  ← 잘못된 경로!

# EXTERNAL_VARIANT=HY11이면:
# ../prebuilt_HY11/libvmmem_prebuilt.tar.gz  ← 올바른 경로! ✅
```

### 진단 과정 (실제 해결 과정)

```bash
# 1. 에러 로그 확인
grep "Cannot open" m2m_8.log
# → tar: ../prebuilt_/libvmmem_prebuilt.tar.gz: Cannot open

# 2. 실제 prebuilt 디렉토리 확인
ls -la /home/seungtaena/SDX35/apps_proc/
# → prebuilt_HY11/  존재함!

# 3. 성공한 "old fashioned" 빌드 로그 비교
grep "tar.*prebuilt" old_fashioend_m2m.log
# Line 708407: tar -zxvf .../prebuilt_HY11/xmllib_prebuilt.tar.gz
# Line 708442: tar -zxvf .../prebuilt_HY11/libvmmem_prebuilt.tar.gz
#             ^^^^^^^^^ HY11이 포함!

# 4. 환경 변수 확인
echo $EXTERNAL_VARIANT
# → (비어있음) ← 문제 발견!

# 5. 환경 변수 설정 후 재시도
export EXTERNAL_VARIANT=HY11
make package/m2m/clean
make package/m2m/compile -j$(nproc)
# → 성공! ✅

# 6. 성공 로그 확인 (m2m_9.log)
grep "tar.*prebuilt" m2m_9.log
# Line 303: tar -zxvf /home/.../prebuilt_HY11/libvmmem_prebuilt.tar.gz ✅
# Line 352: tar -zxvf /home/.../prebuilt_HY11/xmllib_prebuilt.tar.gz ✅
```

### 왜 이 문제가 발생했나?

#### Old Fashioned Way (성공):
```bash
# 전체 빌드 환경 설정 스크립트 실행
source owrt-qti-conf/set_openwrt_env.sh
configure sdx35 mbb perf disable_kernel
# → 이 과정에서 EXTERNAL_VARIANT 자동 설정됨

make -j$(nproc)  # 전체 빌드
# → 모든 환경 변수가 이미 설정된 상태
```

#### New Way (Cache Restore 후 실패):
```bash
# Cache만 복원하고 바로 빌드
./build-cache-restore.sh
make package/m2m/compile
# → 환경 변수가 설정되지 않음! ❌
# → prebuilt_ 경로가 잘못됨
```

### 해결 방법

#### 방법 1: 즉시 해결 (임시)

```bash
# 터미널에서 환경 변수 설정
export EXTERNAL_VARIANT=HY11

# 빌드
make package/m2m/clean
make package/m2m/compile -j$(nproc)
# → 성공! ✅
```

#### 방법 2: 환경 파일 생성 (권장)

```bash
# .build-env 파일 생성
cat > /home/seungtaena/test/SDX35/apps_proc/owrt/.build-env << 'EOF'
# Prebuilt 패키지 variant 설정
export EXTERNAL_VARIANT=HY11
EOF

# 빌드 전에 source
cd /home/seungtaena/test/SDX35/apps_proc/owrt
source .build-env
make package/m2m/compile
```

#### 방법 3: build-cache-restore.sh에 통합 (최선)

```bash
# build-cache-restore.sh 수정
echo "[5/7] 환경 변수 설정..."
cat >> "$OWRT_ROOT/.build-env" << 'EOF'
# Prebuilt 패키지 variant 설정
export EXTERNAL_VARIANT=HY11
EOF

log_success "환경 변수 설정 완료"
echo ""
echo "빌드 시작 전에 실행하세요:"
echo "  source $OWRT_ROOT/.build-env"
```

### 확인 방법

```bash
# 1. 환경 변수 확인
echo $EXTERNAL_VARIANT
# → HY11

# 2. 빌드 로그에서 확인
make package/m2m/compile V=s 2>&1 | tee build.log
grep "tar.*prebuilt" build.log
# → tar -zxvf .../prebuilt_HY11/libvmmem_prebuilt.tar.gz ✅

# 3. .built 마커 생성 확인
ls -la build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/libvmmem-1.0.0/.built
# → 파일 존재 ✅

# 4. .installed 스탬프 확인 (빌드 완료 후)
ls -la staging_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/stamp/.libvmmem_installed
# → 파일 존재 ✅
```

### 관련 패키지

EXTERNAL_VARIANT가 필요한 패키지들:

```bash
# Prebuilt를 사용하는 주요 패키지:
- libvmmem         (VMM memory library)
- xmllib           (XML parser)
- (기타 Qualcomm vendor 패키지)

# 확인 방법:
cd /home/seungtaena/SDX35/apps_proc/owrt
grep -r "prebuilt_\$(EXTERNAL_VARIANT)" feeds/*/packages/*/Makefile

# 출력 예시:
# feeds/qtibspprop/packages/libvmmem/Makefile
# feeds/qtibspprop/packages/xmllib/Makefile
```

### OpenWrt의 .built와 .installed 마커

```bash
# .built 마커 (build_dir):
build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/libvmmem-1.0.0/.built
# → 컴파일 완료를 표시
# → make package/xxx/clean이 삭제함

# .installed 스탬프 (staging_dir):
staging_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/stamp/.libvmmem_installed
# → 설치(staging) 완료를 표시
# → OpenWrt가 자동 관리 (빌드 시작 시 삭제, 완료 시 생성)

# 로그 예시 (m2m_9.log):
# Line 322: rm -f .../stamp/.libvmmem_installed  (빌드 시작)
# Line 1601: touch .../stamp/.libvmmem_installed  (빌드 완료)
```

### 핵심 교훈

**Cache는 빌드 산출물만 복원합니다.**  
**빌드 환경 설정(환경 변수)은 별도로 필요합니다!**

```
Cache Restore = 파일 복사
Environment Setup = 환경 변수, PATH, 설정 등

둘 다 필요합니다! ✅
```

---

## 9.12 Kernel Module 빌드 실패: "autoconf.h missing" / ".ko: No such file"

### 증상

```bash
make package/m2m/compile
# 또는:
make package/location-client-api/compile

# 에러 1:
ERROR: Kernel configuration is invalid.
       include/generated/autoconf.h or include/config/auto.conf are missing.
       Run 'make oldconfig && make prepare' on kernel src to fix it.

# 에러 2:
LD_LIBRARY_PATH=.../kernel-build-tools/linux-x86/lib64/ \
  .../scripts/sign-file sha512 ... smcinvoke_dlkm.ko
sign-file: .../smcinvoke_dlkm.ko: No such file or directory

# 에러 3:
make[1]: *** [Makefile:135: .../securemsm-dlkm-kernel-1.0/.stamp_built] Error 1
ERROR: package/feeds/qtisecurity/securemsm_dlkm_kernel failed to build.
```

**로그 예시 (m2m_new_1.log):**
```
Line 191: ERROR: Kernel configuration is invalid.
Line 192:        include/generated/autoconf.h or include/config/auto.conf are missing.
Line 193:        Run 'make oldconfig && make prepare' on kernel src to fix it.
Line 213: sign-file: .../smcinvoke_dlkm.ko: No such file or directory
Line 217: ERROR: package/feeds/qtisecurity/securemsm_dlkm_kernel failed to build.
```

### 원인

Kernel module을 포함한 패키지 빌드 시 다음이 필요합니다:

```
✅ 1. Kernel 빌드 트리 (linux-5.15/)
✅ 2. Kernel out 디렉토리 (out/)
✅ 3. Kernel build tools (prebuilts/)
❌ 4. temp_out_dir/ (누락!)
❌ 5. 사전 컴파일된 .ko 파일들 (누락!)
```

1-3번은 이미 복사했지만, **4-5번이 누락**되어 실패!

### 진단 과정 (실제 해결 과정)

```bash
# 상황: 1-5 단계만 수행하고 빌드 시도
# 1. 커널 빌드 트리 복사 ✅
# 2. Kernel out 복사 ✅
# 3. Kernel build-tools 복사 ✅
# 4. TARGET_VARIANT 변경 ✅
# 5. EXTERNAL_VARIANT 설정 ✅

make package/m2m/compile
# → 실패! (m2m_new_1.log)

# 진단 1: autoconf.h 확인
ls -la src/kernel-5.15/kernel_platform/temp_out_dir/include/generated/autoconf.h
# → No such file or directory ← 문제!

# 원본에서 확인
ls -la /home/seungtaena/SDX35/apps_proc/owrt/src/kernel-5.15/kernel_platform/temp_out_dir/
# → 존재함!
# → 복사하지 않았음을 발견

# 진단 2: .ko 파일 확인
ls -la build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/
# → 디렉토리 비어있음

# 원본에서 확인
ls -la /home/seungtaena/SDX35/apps_proc/owrt/build_dir/.../securemsm-dlkm-kernel-1.0/*.ko
# → 여러 .ko 파일 존재!
# → 이것도 복사하지 않았음을 발견
```

### temp_out_dir의 역할

```bash
src/kernel-5.15/kernel_platform/temp_out_dir/
├── include/
│   ├── generated/
│   │   ├── autoconf.h       ← Kernel config 헤더 (필수!)
│   │   └── uapi/
│   └── config/
│       └── auto.conf         ← Kernel config 메타데이터 (필수!)
├── .config
└── Module.symvers

# 이 파일들이 없으면:
# → Kernel module 컴파일이 아예 불가능
# → "Kernel configuration is invalid" 에러
```

### .ko 파일의 역할

```bash
# 일부 패키지는 이미 컴파일된 .ko를 사용:
build_dir/.../securemsm-dlkm-kernel-1.0/
├── smcinvoke_dlkm.ko        ← 사전 컴파일됨
├── tz_log_dlkm.ko           ← 사전 컴파일됨
├── qseecom_dlkm.ko
└── ...

# 빌드 프로세스:
# 1. .ko 파일들이 이미 존재 (사전 컴파일)
# 2. sign-file 스크립트가 서명만 수행:
#    sign-file sha512 kernel.key cert.pem smcinvoke_dlkm.ko
# 3. 서명된 .ko를 패키징

# .ko 파일이 없으면:
# → sign-file이 실패
# → "No such file or directory" 에러
```

### 해결 방법

#### 추가 단계 6-7 (필수)

```bash
ORIGINAL_OWRT="/home/seungtaena/SDX35/apps_proc/owrt"
TARGET_OWRT="/home/seungtaena/test/SDX35/apps_proc/owrt"

# 6. Kernel temp_out_dir 복사 (중요!)
echo "[6/7] Kernel temp_out_dir 복사..."
rsync -avh --progress \
  ${ORIGINAL_OWRT}/src/kernel-5.15/kernel_platform/temp_out_dir/ \
  ${TARGET_OWRT}/src/kernel-5.15/kernel_platform/temp_out_dir/

# 7. Kernel 모듈(.ko) 파일 복사 (중요!)
echo "[7/7] Kernel 모듈(.ko) 파일 복사..."
mkdir -p ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/
rsync -avh \
  ${ORIGINAL_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/*.ko \
  ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/
```

### 확인 방법

```bash
# 1. temp_out_dir 확인
ls -la src/kernel-5.15/kernel_platform/temp_out_dir/include/generated/autoconf.h
# → 파일 존재 ✅

ls -la src/kernel-5.15/kernel_platform/temp_out_dir/include/config/auto.conf
# → 파일 존재 ✅

# 2. .ko 파일 확인
ls -la build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/
# → smcinvoke_dlkm.ko, tz_log_dlkm.ko 등 존재 ✅

# 3. 빌드 테스트
source .build-env
make package/m2m/clean
make package/m2m/compile -j$(nproc)
# → 성공! ✅
```

### 언제 필요한가?

| 패키지 유형 | temp_out_dir | .ko 파일 | 예시 |
|-------------|--------------|----------|------|
| 순수 userspace | ❌ 불필요 | ❌ 불필요 | curl, glib2, openssl |
| Prebuilt 패키지 | ❌ 불필요 | ❌ 불필요 | libvmmem, xmllib |
| **Kernel module 포함** | **✅ 필수** | **✅ 필수** | **m2m, location-client-api** |
| Kernel 전용 패키지 | ✅ 필수 | ✅ 필수 | securemsm_dlkm_kernel |

### 다른 패키지에서는 왜 에러가 안 났나?

```bash
# location-client-api 빌드 시:
make package/location-client-api/compile
# → 성공 (같은 에러가 안 남)

# 이유:
# 1. location-client-api는 kernel module을 직접 빌드하지 않음
# 2. 이미 빌드된 kernel module을 참조만 함
# 3. m2m은 securemsm_dlkm_kernel에 직접 의존
#    → securemsm_dlkm_kernel이 빌드되어야 함
#    → 이 단계에서 .ko 파일과 config가 필요
```

### 핵심 교훈

Kernel module 빌드는 단순한 소스 컴파일이 아닙니다:

```
Kernel Module Build =
  ✅ Kernel 소스 트리
  ✅ Build tools
  ✅ Kernel configuration (temp_out_dir/)
  ✅ 사전 컴파일된 모듈 (.ko files)

모든 요소가 갖춰져야 합니다! ✅
```

---

## 9.13 Cache Restore 후 완전한 빌드 환경 구성 (종합)

### 배경

우리가 해결한 문제들을 종합하면:

1. **Prebuilt 패키지 실패** → `EXTERNAL_VARIANT` 누락
2. **Kernel module 실패** → `temp_out_dir`, `.ko` 파일 누락
3. **"No rule to make target"** → `feeds install` 누락

이 모든 것을 해결하는 **완전한 복원 절차**가 필요합니다.

### 완전한 build-cache-restore.sh (최종 버전)

```bash
#!/bin/bash
# 최종 검증된 Cache Restore 스크립트
# 모든 문제 해결 내용 통합

ORIGINAL_OWRT="/home/seungtaena/SDX35/apps_proc/owrt"
TARGET_OWRT="/home/seungtaena/test/SDX35/apps_proc/owrt"

echo "========================================="
echo "Build Cache Restore (Complete Version)"
echo "========================================="
echo ""

# [1/10] 커널 빌드 트리 복사
echo "[1/10] 커널 빌드 트리 복사..."
mkdir -p ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/linux-5.15/
rsync -avh --progress \
  ${ORIGINAL_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/linux-5.15/ \
  ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/linux-5.15/

# [2/10] Kernel out 디렉토리 복사
echo "[2/10] Kernel out 디렉토리 복사..."
mkdir -p ${TARGET_OWRT}/src/kernel-5.15/
rsync -avh --progress \
  ${ORIGINAL_OWRT}/src/kernel-5.15/out/ \
  ${TARGET_OWRT}/src/kernel-5.15/out/

# [3/10] Kernel build tools 복사
echo "[3/10] Kernel build tools 복사..."
mkdir -p ${TARGET_OWRT}/src/kernel-5.15/kernel_platform/prebuilts/
rsync -avh --progress \
  ${ORIGINAL_OWRT}/src/kernel-5.15/kernel_platform/prebuilts/kernel-build-tools/ \
  ${TARGET_OWRT}/src/kernel-5.15/kernel_platform/prebuilts/kernel-build-tools/

# [4/10] Variant 설정 (debug -> perf)
echo "[4/10] Variant 설정 (debug -> perf)..."
sed -i 's/^TARGET_VARIANT:=debug$/TARGET_VARIANT:=perf/' \
  ${TARGET_OWRT}/target/linux/sdx35/Makefile

# [5/10] 환경 변수 설정
echo "[5/10] 환경 변수 설정..."
cat > ${TARGET_OWRT}/.build-env << 'EOF'
# Prebuilt 패키지 variant 설정
export EXTERNAL_VARIANT=HY11
EOF

# [6/10] Kernel temp_out_dir 복사 (중요!)
echo "[6/10] Kernel temp_out_dir 복사..."
rsync -avh --progress \
  ${ORIGINAL_OWRT}/src/kernel-5.15/kernel_platform/temp_out_dir/ \
  ${TARGET_OWRT}/src/kernel-5.15/kernel_platform/temp_out_dir/

# [7/10] Kernel 모듈(.ko) 파일 복사 (중요!)
echo "[7/10] Kernel 모듈(.ko) 파일 복사..."
mkdir -p ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/
rsync -avh \
  ${ORIGINAL_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/*.ko \
  ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/ 2>/dev/null || true

# [8/10] location-client-api build_dir 복사 (Race condition 방지!)
echo "[8/10] location-client-api build_dir 복사..."
echo "  → .built 마커 포함, 재컴파일 방지"
mkdir -p ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/
rsync -avh \
  ${ORIGINAL_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/location-client-api-1.0/ \
  ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/location-client-api-1.0/ 2>/dev/null || true

# [9/10] DL cache 복원
echo "[9/10] DL cache 복원..."
cd ${TARGET_OWRT}
./build-cache-restore.sh --mode symlink --quiet 2>/dev/null || \
echo "  (기존 build-cache-restore.sh 실행)"

# [10/10] Feeds 링크 재생성
echo "[10/10] Feeds 링크 재생성..."
cd ${TARGET_OWRT}
./scripts/feeds update telit 2>/dev/null || echo "  (feeds update 스킵)"
./scripts/feeds install m2m sdaemon-telit 2>/dev/null || echo "  (feeds install 스킵)"

echo ""
echo "✅ Build cache restore 완료!"
echo ""
echo "========================================="
echo "다음 단계:"
echo "========================================="
echo "1. 환경 변수 활성화:"
echo "   source ${TARGET_OWRT}/.build-env"
echo ""
echo "2. 빌드 실행:"
echo "   make package/m2m/clean"
echo "   make package/m2m/compile -j\$(nproc)"
echo ""
echo "========================================="
```

### 단계별 설명

| 단계 | 내용 | 필수 여부 | 해결하는 문제 |
|------|------|-----------|---------------|
| 1-3 | Kernel 환경 | ✅ 필수 (kernel module) | Basic kernel build |
| 4 | Variant 설정 | ✅ 필수 | Target variant mismatch |
| 5 | 환경 변수 | ✅ 필수 | **Prebuilt 패키지 실패** |
| 6 | temp_out_dir | ✅ 필수 (kernel module) | **autoconf.h missing** |
| 7 | .ko 파일 | ✅ 필수 (kernel module) | **.ko: No such file** |
| 8 | location-client-api build_dir | ✅ 필수 | **Race condition** |
| 9 | DL cache | ⭐ 권장 | Download 시간 단축 |
| 10 | Feeds 링크 | ✅ 필수 | **No rule to make target** |

### 패키지별 필요 단계

```bash
# 순수 userspace 패키지 (curl, openssl):
단계 4, 5, 9, 10

# Prebuilt 패키지 (libvmmem, xmllib):
단계 4, 5, 9, 10

# Kernel module 포함 패키지 (m2m, location-client-api):
모든 단계 1-10 ✅
```

### 사용 방법

```bash
# 1. 스크립트 저장
cat > complete-cache-restore.sh << 'EOF'
#!/bin/bash
# (위의 스크립트 내용)
EOF

chmod +x complete-cache-restore.sh

# 2. 실행
./complete-cache-restore.sh

# 3. 환경 변수 활성화
source /home/seungtaena/test/SDX35/apps_proc/owrt/.build-env

# 4. 빌드
cd /home/seungtaena/test/SDX35/apps_proc/owrt
make package/m2m/clean
make package/m2m/compile -j$(nproc)

# 5. 성공! ✅
```

### 검증 체크리스트

복원 후 반드시 확인:

```bash
# ✅ 환경 변수
echo $EXTERNAL_VARIANT
# → HY11

# ✅ Kernel configuration
ls src/kernel-5.15/kernel_platform/temp_out_dir/include/generated/autoconf.h
# → 존재

# ✅ .ko 파일
ls build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/*.ko
# → 여러 .ko 파일 존재

# ✅ Feeds 링크
ls -la package/feeds/telit/m2m
# → 심볼릭 링크 존재

# ✅ Variant
grep "TARGET_VARIANT" target/linux/sdx35/Makefile
# → TARGET_VARIANT:=perf

# 모두 확인되면 빌드 시작! ✅
```

### 시간 비교

```bash
# 불완전한 복원 (1-7 단계만, 8번 누락):
- 복원 시간: 12분
- 빌드 시도: 실패 ❌ (Racing compilation)
- 문제 진단: 1시간+
- 재시도: 성공
총 시간: 70분+ (불확실, 좌절감)

# 완전한 복원 (1-10 단계):
- 복원 시간: 15분
- 빌드: 3-5분
총 시간: 18-20분 (확실) ✅
```

### 왜 이렇게 복잡한가?

OpenWrt 빌드 시스템은 여러 계층:

```
Layer 1: 순수 소스 빌드
  └─ 소스 코드만 필요

Layer 2: Prebuilt 패키지
  └─ 소스 + Prebuilt tarball + EXTERNAL_VARIANT
      ↓
      해결: Step 5

Layer 3: Kernel module
  └─ 소스 + Kernel 환경 + Configuration + 사전 컴파일 .ko
      ↓
      해결: Steps 1-3, 6-7

Layer 4: 빌드 시스템 연결
  └─ Feeds 링크 (package/feeds/)
      ↓
      해결: Step 9

각 계층마다 다른 요구사항! ✅
```

### 핵심 교훈

**Cache = 파일 복사**  
**Environment = 설정 + 링크 + 환경 변수**

```
완전한 빌드 환경 =
  ✅ Cache (파일들)
  ✅ Configuration (설정들)
  ✅ Environment (환경 변수들)
  ✅ Links (심볼릭 링크들)

모두 필요합니다! ✅
```

---

## 9.14 Racing Compilation Issue: location-client-api 빌드 실패

### 증상

```bash
# 상황: build-cache-restore.sh 실행 + 1-6 단계 수행
# (Kernel 환경 복사, EXTERNAL_VARIANT 설정 등)

make package/m2m/clean
make package/m2m/compile -j$(nproc)

# 에러:
In file included from src/LocationIntegrationApiImpl.h:73,
                 from src/LocationIntegrationApiImpl.cpp:70:
inc/LocationIntegrationApi.h:67:10: fatal error: LocationClientApi.h: No such file or directory
   67 | #include <LocationClientApi.h>
      |          ^~~~~~~~~~~~~~~~~~~~~
compilation terminated.

make[4]: *** [Makefile:581: src/liblocation_integration_api_la-LocationIntegrationApiImpl.lo] Error 1
make[3]: *** [Makefile:439: all] Error 2
make[2]: *** [Makefile:48: .../location-integration-api-1.0/.built] Error 2
```

**로그 예시 (m2m_new_4.log):**
```
Line 24861: inc/LocationIntegrationApi.h:67:10: fatal error: LocationClientApi.h: No such file or directory
Line 24867: make[3]: *** [Makefile:439: all] Error 2
Line 24869: make[2]: *** [Makefile:48: .../location-integration-api-1.0/.built] Error 2
```

### 혼란스러운 점

```bash
# 의문 1: staging_dir에는 파일이 있음
ls -la staging_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/usr/include/location-client-api/
# → LocationClientApi.h 존재! ✅

# 의문 2: build-cache-restore.sh로 staging_dir 복사함
./build-cache-restore.sh --mode copy
# → staging_dir 전체 복사 완료! ✅

# 의문 3: .installed stamp도 있음
ls -la staging_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/stamp/.location-client-api_installed
# → 존재! ✅

# 그런데 왜 컴파일 실패? ❓
```

### 원인 분석: 타임라인 추적

로그를 분석하면 놀라운 사실이 드러납니다:

```bash
# m2m_new_4.log 타임라인:

# Line 23868: location-client-api .installed stamp 삭제!
rm -f .../stamp/.location-client-api_installed

# Line 23876: location-client-api autoreconf 시작
(cd .../location-client-api-1.0; ... autoreconf ...)

# Line 24861: location-integration-api 컴파일 시도
# → LocationClientApi.h 찾지 못함! ❌
#    (location-client-api가 아직 install 안 됨!)

# Line 24886: location-client-api pkgdir 정리
rm -rf .../location-client-api-1.0/.pkgdir/
```

**핵심 발견: location-client-api가 재컴파일되고 있음!**

```bash
# 추가 확인:
grep -E "(rm -f.*location-client-api.*\.built|touch.*location-client-api.*\.built)" m2m_new_4.log

# 결과:
rm -f .../location-client-api-1.0/.built     ← 삭제!
touch .../location-client-api-1.0/.built_check
touch .../location-client-api-1.0/.built     ← 재생성
```

### 진짜 원인: .built 마커 누락

```
┌─────────────────────────────────────────────────────┐
│ build-cache-restore.sh 복사 범위:                   │
├─────────────────────────────────────────────────────┤
│ ✅ staging_dir/ → 전체 복사 (헤더, 라이브러리, stamp)│
│ ❌ build_dir/   → 복사하지 않음! (.built 마커 없음) │
└─────────────────────────────────────────────────────┘

결과:
staging_dir/stamp/.location-client-api_installed      ✅ 존재
staging_dir/usr/include/location-client-api/*.h       ✅ 존재
build_dir/.../location-client-api-1.0/.built          ❌ 없음!

OpenWrt의 판단:
1. .installed stamp 있음 → "이전에 설치됨"
2. .built 마커 없음 → "빌드 안 됨" 🤔
3. 의존성 체크 → "다시 빌드 필요!"
4. make package/m2m/clean → 의존성 체인 clean
5. 재컴파일 시도 → 병렬 빌드로 인한 race condition
6. location-integration-api가 먼저 컴파일 시도
7. 실패! ❌
```

### OpenWrt 빌드 메커니즘

```bash
# OpenWrt가 패키지 빌드 여부를 판단하는 로직:

if [ ! -f "build_dir/.../package-1.0/.built" ]; then
    # .built 마커 없음 → 빌드 필요
    echo "Building package..."
    
    # 의존성이 있으면 먼저 clean
    make package/<dependency>/clean
    
    # 그리고 빌드
    make package/<package>/compile
fi

# 문제:
# 1. staging_dir에 파일이 있어도
# 2. build_dir에 .built가 없으면
# 3. 재컴파일 시도!
# 4. 병렬 빌드 (-j) 시 race condition
```

### Race Condition 상세

```
시간 →

T1: make package/m2m/clean
    └─ m2m 의존성 체크
       └─ location-integration-api 필요
          └─ location-client-api 필요
             └─ .built 없음! → clean 필요

T2: rm -f .../location-client-api-1.0/.built        ← 삭제
    rm -f .../stamp/.location-client-api_installed   ← 삭제

T3: make package/m2m/compile -j$(nproc)
    ├─ [Thread 1] location-client-api 재컴파일 시작
    │  └─ autoreconf... (느림)
    │
    └─ [Thread 2] location-integration-api 컴파일 시작
       └─ LocationClientApi.h 필요
       └─ 아직 install 안 됨! ❌
       └─ 실패!

T4: location-client-api 컴파일 완료 (늦음)
    touch .../location-client-api-1.0/.built
    touch .../stamp/.location-client-api_installed
```

### 해결 방법: build_dir 복사

**문제의 핵심:** `build_dir`에 `.built` 마커가 없음

**해결책:** `.built` 마커를 포함한 `build_dir` 복사

```bash
ORIGINAL_OWRT="/home/seungtaena/SDX35/apps_proc/owrt"
TARGET_OWRT="/home/seungtaena/test/SDX35/apps_proc/owrt"

# 추가 단계: location-client-api build_dir 복사
echo "[추가] location-client-api build_dir 복사..."
mkdir -p ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/
rsync -avh \
  ${ORIGINAL_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/location-client-api-1.0/ \
  ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/location-client-api-1.0/
```

### .built 마커의 역할

```bash
# 원본 환경 확인:
ls -la /home/seungtaena/SDX35/apps_proc/owrt/build_dir/.../location-client-api-1.0/
# -rw-r--r-- 1 user group 0 Nov 18 12:37 .built  ← 중요!

# .built 마커가 있으면:
# → OpenWrt: "이미 빌드됨, skip"
# → 재컴파일 하지 않음
# → Race condition 발생 안 함 ✅

# .built 마커가 없으면:
# → OpenWrt: "빌드 안 됨, 빌드 필요"
# → 재컴파일 시도
# → 병렬 빌드 시 race condition
# → 실패 ❌
```

### 복사해야 하는 build_dir 구성

```bash
build_dir/.../location-client-api-1.0/
├── .built                    ← 가장 중요!
├── .configured_*
├── .prepared_*
├── src/                      ← 소스 파일
├── Makefile
└── ...

# 전체를 복사하는 이유:
# 1. .built 마커
# 2. Configuration artifacts
# 3. Prepare artifacts
# → OpenWrt가 "이미 완료됨"으로 인식
```

### 확인 방법

```bash
# 1. build_dir 복사 확인
ls -la build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/location-client-api-1.0/.built
# → 존재해야 함 ✅

# 2. staging_dir 확인
ls -la staging_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/usr/include/location-client-api/
# → LocationClientApi.h 존재 ✅

# 3. .installed stamp 확인
ls -la staging_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/stamp/.location-client-api_installed
# → 존재 ✅

# 4. 빌드 테스트
make package/m2m/clean
make package/m2m/compile -j$(nproc) 2>&1 | tee build.log

# 5. location-client-api 재컴파일 여부 확인
grep "location-client-api" build.log | grep -E "(autoreconf|configure)"
# → 아무것도 안 나와야 함 (재컴파일 안 함) ✅
```

### 언제 필요한가?

| 패키지 | staging_dir 복사 | build_dir 복사 | 이유 |
|--------|------------------|----------------|------|
| 순수 userspace | ✅ | ❌ | 의존성 체인 짧음 |
| Prebuilt | ✅ | ❌ | 소스 빌드 안 함 |
| **복잡한 의존성** | **✅** | **✅** | **Race condition 방지** |
| location-client-api | ✅ | ✅ | m2m이 의존 |
| m2m | ✅ | N/A | 빌드 대상 |

### 다른 패키지도 필요한가?

```bash
# 원칙: 재컴파일되는 모든 의존성 패키지

# 확인 방법:
make package/m2m/clean
make package/m2m/compile V=s 2>&1 | tee build.log

# 로그에서 확인:
grep "autoreconf\|Entering directory" build.log | grep -v "Leaving"

# 재컴파일되는 패키지 리스트:
# - location-client-api     ← 복사 필요!
# - location-integration-api ← m2m에 가까움, 괜찮을 수도
# - (기타 의존성)

# 안전하게:
# 모든 location 관련 패키지의 build_dir 복사
for pkg in location-client-api location-integration-api location-client-api-hdr; do
    rsync -avh \
      ${ORIGINAL_OWRT}/build_dir/.../
/${pkg}-*/ \
      ${TARGET_OWRT}/build_dir/...//${pkg}-*/
done
```

### 왜 이전 분석에서 놓쳤나?

```bash
# 이전 확인:
ls staging_dir/.../usr/include/location-client-api/LocationClientApi.h
# → 존재! ✅ "복사 불필요"로 판단

# 놓친 점:
ls build_dir/.../location-client-api-1.0/.built
# → 확인 안 함! ❌

# 교훈:
# staging_dir 파일 존재 ≠ 빌드 skip
# build_dir/.built 존재 = 빌드 skip ✅

# OpenWrt는 .built 마커를 우선 확인!
```

### 완전한 복원 스크립트 (수정)

```bash
#!/bin/bash
# Build Cache Restore - 완전판 (v9.14)

ORIGINAL_OWRT="/home/seungtaena/SDX35/apps_proc/owrt"
TARGET_OWRT="/home/seungtaena/test/SDX35/apps_proc/owrt"

# ... (이전 1-6 단계)

# [7/10] Kernel 모듈(.ko) 파일 복사
echo "[7/10] Kernel 모듈(.ko) 파일 복사..."
mkdir -p ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/
rsync -avh \
  ${ORIGINAL_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/*.ko \
  ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/linux-sdx35/securemsm-dlkm-kernel-1.0/

# [8/10] location-client-api build_dir 복사 (NEW!)
echo "[8/10] location-client-api build_dir 복사..."
echo "  → Race condition 방지를 위한 .built 마커 복사"
rsync -avh \
  ${ORIGINAL_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/location-client-api-1.0/ \
  ${TARGET_OWRT}/build_dir/target-arm_cortex-a7+neon-vfpv4_musl_eabi/location-client-api-1.0/

# [9/10] DL cache 복원
echo "[9/10] DL cache 복원..."
# (이전과 동일)

# [10/10] Feeds 링크 재생성
echo "[10/10] Feeds 링크 재생성..."
# (이전과 동일)
```

### 핵심 교훈

**1. OpenWrt의 빌드 판단 기준**

```
staging_dir → Install 여부 확인
build_dir/.built → Build 여부 확인

둘 다 있어야 skip됨! ✅
```

**2. Cache Restore의 완전성**

```
Cache = staging_dir + build_dir
    
staging_dir만 복사 → 불완전 ❌
staging_dir + build_dir 복사 → 완전 ✅
```

**3. 병렬 빌드의 위험성**

```
-j$(nproc) + 불완전한 cache = Race condition
-j$(nproc) + 완전한 cache = 안전 ✅
```

**4. 진단 방법**

```bash
# 실패 시 로그 확인:
grep -E "(autoreconf|configure|rm -f.*\.built)" build.log

# 재컴파일되는 패키지 발견:
# → 해당 패키지의 build_dir 복사 필요!
```

---

## 9.15 GLIBC 버전 충돌: "GLIBC_2.38 not found"

### 증상

```bash
# build-cache-restore.sh 실행 후
# Kernel 환경 복사 (1-6단계) 후
make package/m2m/clean
make package/m2m/compile

# 에러:
grep: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found (required by grep)
bash: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found (required by bash)
/bin/sh: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

**로그 예시 (m2m_new_5.log):**
```
Line 15: grep: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
Line 35: bash: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

### 원인 분석

**잘못된 rsync 사용:**

```bash
# 초기 시도 (잘못됨):
rsync -avhL staging_dir/ cache/staging_dir/
#         ↑ -L: 모든 심볼릭 링크를 실제 파일로 복사

# 결과:
staging_dir/host/bin/grep (Ubuntu 24.04, GLIBC 2.38)
  → cache에 실제 바이너리로 복사됨
  → 테스트 환경 (Ubuntu 22.04, GLIBC 2.35)에서 실행 불가! ❌
```

**문제 상황:**

```
원본 환경:
├── Ubuntu 24.04
├── GLIBC 2.38
└── staging_dir/host/bin/grep (GLIBC 2.38 요구)

rsync -avhL (모든 링크 해제)
    ↓
    
Cache:
└── staging_dir/host/bin/grep (GLIBC 2.38 바이너리)

rsync 복원
    ↓
    
테스트 환경:
├── Ubuntu 22.04
├── GLIBC 2.35
└── staging_dir/host/bin/grep (GLIBC 2.38 요구)
    → 실행 실패! ❌
```

### `staging_dir/host/` 의 역할

```bash
# staging_dir/host/ 구조:
staging_dir/host/
├── bin/
│   ├── grep -> /usr/bin/grep          ← 호스트 시스템 도구
│   ├── bash -> /bin/bash              ← 호스트 시스템 쉘
│   ├── python3 -> /usr/bin/python3    ← 호스트 Python
│   └── cmake -> /usr/bin/cmake        ← 호스트 CMake
└── ...

# 특징:
- OpenWrt가 빌드한 것이 아님!
- 호스트 시스템의 도구를 가리키는 심볼릭 링크
- GLIBC 버전에 의존적
- 환경마다 경로와 버전이 다름
```

### `staging_dir/hostpkg/` 와의 차이

```bash
# staging_dir/hostpkg/ 구조:
staging_dir/hostpkg/
├── bin/
│   ├── bzip2                          ← OpenWrt가 빌드
│   ├── bzcmp -> /abs/path/.../bzdiff  ← 절대 경로 링크 (문제!)
│   └── libtool                        ← OpenWrt가 빌드
└── ...

# 특징:
- OpenWrt가 직접 빌드한 호스트용 도구
- 이식 가능한 바이너리
- 하지만 절대 경로 심볼릭 링크 사용
- Docker에서 경로 깨짐 → 링크 해제 필요!
```

### 올바른 해결책: 선택적 `-L` 적용

**수정된 `build-cache-sync.sh`:**

```bash
# [6a] host - 링크 유지 (-avh)
rsync -avh staging_dir/host/ cache/staging_dir/host/
# → 각 환경의 시스템 도구 사용 ✅

# [6b] hostpkg - 링크 해제 (-avhL)
rsync -avhL staging_dir/hostpkg/ cache/staging_dir/hostpkg/
# → 절대 경로 제거, Docker 호환 ✅

# [6c] toolchain - 링크 유지 (-avh)
rsync -avh staging_dir/toolchain-*/ cache/staging_dir/toolchain-*/
# → 상대 링크는 문제 없음 ✅

# [6d] target - 링크 해제 (-avhL)
rsync -avhL staging_dir/target-*/ cache/staging_dir/target-*/
# → IPK 절대 경로 제거 ✅

# [6e] packages - 링크 해제 (-avhL)
rsync -avhL staging_dir/packages/ cache/staging_dir/packages/
# → 완전한 이식성 ✅
```

### 진단 과정

```bash
# 1. 에러 발견
make package/m2m/compile
# → GLIBC_2.38 not found

# 2. grep 위치 확인
which grep
# → staging_dir/host/bin/grep

# 3. grep 타입 확인
file staging_dir/host/bin/grep
# → ELF 64-bit executable  ← 실제 바이너리! (문제!)

# 4. 원본 확인
file /home/seungtaena/SDX35/.../staging_dir/host/bin/grep
# → symbolic link to /usr/bin/grep  ← 심볼릭 링크여야 함!

# 5. Cache 확인
file owrt-cache/staging_dir/host/bin/grep
# → ELF 64-bit executable  ← -L로 복사됨 (문제 발견!)

# 6. 해결: host만 -avh로 변경
```

### 확인 및 검증

```bash
# 1. Cache 재생성
./build-cache-sync.sh

# 2. 타입 확인
file owrt-cache/staging_dir/host/bin/grep
# → symbolic link ✅

file owrt-cache/staging_dir/hostpkg/bin/bzcmp
# → script or executable ✅

file owrt-cache/staging_dir/packages/sdx35/m2m_*.ipk
# → gzip compressed data ✅

# 3. 테스트 환경 재구성
rm -rf test/SDX35/apps_proc/owrt/staging_dir
./build-cache-restore.sh --owrt-root test/SDX35/apps_proc/owrt

# 4. 빌드 성공 확인
cd test/SDX35/apps_proc/owrt
make package/m2m/compile
# → 성공! ✅
```

### 핵심 교훈

**1. `staging_dir/` 하위는 용도가 다르다**

```
host/      → 시스템 도구 링크 (GLIBC 의존적)
hostpkg/   → OpenWrt 빌드 산출물 (이식 가능)
toolchain/ → 크로스 컴파일러 (상대 링크)
target-*/  → 타겟 파일 (절대 경로 제거 필요)
packages/  → IPK 패키지 (절대 경로 제거 필요)
```

**2. rsync -L은 선택적으로**

```
모든 곳에 -L 적용 ❌
→ GLIBC 충돌 발생

선택적 -L 적용 ✅
→ 이식성과 호환성 모두 확보
```

**3. 심볼릭 링크의 목적 이해**

```
시스템 도구 링크: 환경 적응용 → 유지
절대 경로 링크: 빌드 편의용 → 해제
상대 경로 링크: 구조 유지용 → 유지
```

### 관련 섹션

- **Section 3.5**: Symlink 처리 전략 (상세)
- **Section 9.14**: Racing compilation issue
- **Appendix**: 스크립트 변경 이력

---

## 9.16 ccache Hit Rate가 낮음: sloppiness 설정 문제

### 증상

```bash
# 캐시 복원 후 빌드
ccache -z
make package/m2m/compile -j$(nproc)
ccache -s

# 결과:
cache hit (direct)       :    12
cache hit (preprocessed) :     5
cache miss               :  1234  ← 대부분 miss!
cache hit rate           :  1.4%  ← 매우 낮음! ❌
```

### 원인 1: sloppiness 없이 생성된 캐시

```
원본 환경에서 캐시 생성:
- sloppiness 설정 없음
- 캐시 키: hash(소스 + 절대 경로 + 시간 + ...)
- 예: abc123

다른 환경에서 캐시 조회:
- sloppiness 설정 있음
- 캐시 키: hash(소스 + 경로 무시 + 시간 무시 + ...)
- 예: def456 ← 다름!

결과: 캐시 miss ❌
```

### 원인 2: 조회 시 sloppiness 설정 누락

```
원본 환경에서 캐시 생성:
- sloppiness 설정 있음 (rules.mk)
- 캐시 키: abc123

다른 환경에서 캐시 조회:
- rules.mk가 다름 (sloppiness 없음)
- 캐시 키: def456 ← 다름!

결과: 캐시 miss ❌
```

### 핵심 이해

**sloppiness는 캐시 생성과 조회 모두에 필요합니다!**

```
┌─────────────────────────────────────────────────────┐
│ 캐시 저장 시: 소스 + sloppiness → 키 abc123 → 저장   │
│ 캐시 조회 시: 소스 + sloppiness → 키 abc123 → hit ✅ │
├─────────────────────────────────────────────────────┤
│ 캐시 저장 시: 소스 + sloppiness → 키 abc123 → 저장   │
│ 캐시 조회 시: 소스 + NO sloppi → 키 def456 → miss ❌ │
└─────────────────────────────────────────────────────┘
```

### 해결 방법 1: rules.mk에 환경 변수 추가 (권장!)

원본 소스의 `include/rules.mk` 수정:

```makefile
ifneq ($(CONFIG_CCACHE),)
  TARGET_CC:= ccache_cc
  TARGET_CXX:= ccache_cxx
  HOSTCC:= ccache $(HOSTCC)
  HOSTCXX:= ccache $(HOSTCXX)
  export CCACHE_BASEDIR:=$(TOPDIR)
  export CCACHE_DIR:=$(if $(call qstrip,$(CONFIG_CCACHE_DIR)),$(call qstrip,$(CONFIG_CCACHE_DIR)),$(TOPDIR)/.ccache)
  export CCACHE_COMPILERCHECK:=%compiler% -dumpmachine; %compiler% -dumpversion
  
  # ★ 추가: Path-independent caching
  export CCACHE_SLOPPINESS:=file_macro,time_macros,include_file_mtime,include_file_ctime
  export CCACHE_MAXSIZE:=20G
  export CCACHE_COMPRESS:=1
  export CCACHE_COMPRESSLEVEL:=6
endif
```

**⚠️ Makefile 문법 주의!**
```makefile
# ❌ 잘못됨 (따옴표가 값에 포함됨):
export CCACHE_SLOPPINESS="file_macro,..."

# ✅ 올바름:
export CCACHE_SLOPPINESS:=file_macro,...
```

### 해결 방법 2: .build-env에 환경 변수

```bash
# .build-env
export CCACHE_DIR=$PWD/.ccache
export CCACHE_BASEDIR=$PWD
export CCACHE_SLOPPINESS="file_macro,time_macros,include_file_mtime,include_file_ctime"
export CCACHE_MAXSIZE="20G"

# 사용:
source .build-env
make package/m2m/compile
```

### 해결 방법 3: ccache.conf 파일

```bash
mkdir -p .ccache
cat > .ccache/ccache.conf << 'EOF'
base_dir = /path/to/owrt
sloppiness = file_macro,time_macros,include_file_mtime,include_file_ctime
max_size = 20G
compression = true
compression_level = 6
EOF
```

### 기존 캐시 재생성이 필요한가?

**상황에 따라 다릅니다:**

| 상황 | 재생성 필요 | 이유 |
|-----|-----------|------|
| 같은 경로에서 빌드 | ❌ | 키가 같음 |
| 다른 경로 + 원본에 sloppiness 있었음 | ❌ | sloppiness로 생성된 캐시 hit |
| 다른 경로 + 원본에 sloppiness 없었음 | ✅ | 키 계산 방식이 다름 |

**재생성 방법:**
```bash
# 1. sloppiness 설정 (rules.mk 또는 환경 변수)
export CCACHE_SLOPPINESS="file_macro,time_macros,include_file_mtime,include_file_ctime"

# 2. 기존 캐시 삭제
ccache -C

# 3. 전체 재빌드
make clean
make -j$(nproc)

# 4. 이제 경로 독립적 캐시 생성됨!
```

### 확인 방법

```bash
# 1. 설정 확인
ccache -p | grep sloppiness
# (environment) sloppiness = file_macro,...  ← 환경 변수
# ($CCACHE_DIR/ccache.conf) sloppiness = ... ← 설정 파일

# 2. 통계 초기화 후 테스트
ccache -z
make package/m2m/compile -j$(nproc)
ccache -s | grep -E "Hits:|hit rate"

# 3. 기대 결과:
# cache hit rate: 70-90% ✅
```

### 진단 체크리스트

```bash
# 1. sloppiness 환경 변수 확인
echo $CCACHE_SLOPPINESS
# → file_macro,time_macros,... 출력되어야 함

# 2. ccache가 설정을 인식하는지 확인
ccache -p | grep sloppiness
# → sloppiness = file_macro,... 출력되어야 함

# 3. rules.mk에 설정 있는지 확인
grep "CCACHE_SLOPPINESS" include/rules.mk
# → export CCACHE_SLOPPINESS 있어야 함

# 4. 빌드 중 환경 변수 전달 확인
make package/m2m/compile V=s 2>&1 | grep CCACHE_SLOPPINESS
# → 환경 변수가 전달되는지 확인
```

### 핵심 교훈

**1. sloppiness는 양쪽에 필요**
```
생성 환경: sloppiness 있어야 경로 독립적 캐시 생성
조회 환경: sloppiness 있어야 같은 키로 조회
```

**2. rules.mk 수정이 가장 확실**
```
rules.mk에 환경 변수 추가
→ 원본/복사 환경 모두 자동 적용
→ .ccache 디렉토리 없어도 작동
```

**3. 첫 빌드 전에 설정**
```
.ccache 디렉토리 생성 후 ccache.conf? ❌ 복잡
rules.mk에 환경 변수? ✅ 항상 적용
```

### 관련 섹션

- **Section 8.7**: sloppiness 설정 방법 상세
- **Section 8.1-8.2**: sloppiness 옵션 설명

---

# Section 10: FAQ

## 10.1 기존 rsync 방식과 뭐가 다른가요?

**비교:**

| 항목 | 기존 rsync | 제안 방식 | 개선 |
|------|-----------|----------|------|
| 용량 | 18-20GB | 3-5GB | **75% 감소** |
| 시간 | 17-28분 | 3.5-5.5분 | **70% 단축** |
| 재사용률 | 50-70% | 80-100% | **30% 향상** |
| 경로 독립성 | ❌ | ✅ | **완전** |
| ccache hit | 30-50% | 70-90% | **2배** |

**핵심 차이:**
- 필요한 것만 캐싱 (dl/ + IPK)
- 경로 normalized (ccache sloppiness)
- 표준화된 스크립트

## 10.2 OpenWrt sstate-cache가 없다는데?

**맞습니다!**

Yocto와 달리 OpenWrt는 task별 intermediate cache가 없습니다.

**따라서:**
- 최종 .ipk 패키지 재사용이 핵심
- dl/ (소스 다운로드) 재사용
- ccache (컴파일 캐시)로 보완

**이것이 최선의 전략입니다.**

## 10.3 Docker가 필수인가요?

**아니요, 선택사항입니다.**

**Native만으로도 충분합니다:**
```bash
./setup-repo-structure.sh
./build-cache-sync.sh
./build-cache-restore.sh
make package/m2m/compile
```

**하지만 Docker 권장:**
- 문제 정의: "native 서버 빌드 불가"
- 재현성 보장
- CI/CD 통합 쉬움
- 배포 편리

## 10.4 기존 빌드가 망가지나요?

**아니요, 완전히 분리되어 있습니다.**

```
Cache repository: /home/seungtaena/SDX35/owrt-cache/
기존 빌드:        /home/seungtaena/SDX35/apps_proc/owrt/

서로 독립적입니다.
```

symlink 모드로 dl/을 연결하더라도 읽기 전용으로 안전합니다.

## 10.5 ccache sloppiness가 왜 필수인가요?

**경로 독립성 때문입니다.**

```c
// 소스 코드에서
printf("File: %s\n", __FILE__);  // 절대 경로!
printf("Time: %s\n", __TIME__);  // 빌드 시간!
```

**sloppiness 없으면:**
- 경로 변경 → cache miss
- 시간 변경 → cache miss
- Hit rate: 30-50%

**sloppiness 있으면:**
- 경로 무시 → cache hit
- 시간 무시 → cache hit
- Hit rate: 70-90%

**필수입니다!**

## 10.6 다른 서버에서 사용하려면?

**방법 1: Docker (권장)**

```bash
# Server A
./build-docker-image.sh
docker push registry.com/owrt:sdx35

# Server B
docker pull registry.com/owrt:sdx35
docker run -it owrt:sdx35
```

**방법 2: Cache 복사**

```bash
# Server A → Server B
rsync -avh /home/user/owrt-cache/ serverB:/path/owrt-cache/

# Server B
./build-cache-restore.sh --repo-root /path/owrt-cache
```

## 10.7 시간이 얼마나 걸리나요?

**초기 설정: 1-2시간**
- 문서 읽기: 30분
- Native 테스트: 30분
- Docker 빌드: 15분
- 검증: 15분

**이후 사용: 3-5분**
- Cache 복원: 30초
- Package 빌드: 3-5분

**ROI:** 두 번째 빌드부터 이미 시간 절약!

## 10.8 업데이트는 어떻게 하나요?

**Cache 업데이트:**

```bash
# 전체 빌드 후
make -j$(nproc)
./build-cache-sync.sh
```

**Docker 이미지 재빌드:**

```bash
./build-docker-image.sh --tag sdx35-$(date +%Y%m%d)
docker push registry.com/owrt:sdx35-$(date +%Y%m%d)
```

**주기:** 주 1회 or 월 1회 권장

## 10.9 문제가 생기면?

**체크리스트:**

1. Dry-run으로 테스트
   ```bash
   ./build-cache-restore.sh --dry-run
   ```

2. 로그 확인
   ```bash
   cat cache-test-results-*.txt
   ```

3. ccache 통계 확인
   ```bash
   ccache -s
   ```

4. 이 가이드의 Section 9 참조

5. 스크립트 --help 실행
   ```bash
   ./build-cache-sync.sh --help
   ```

## 10.10 검증은 어떻게 하나요?

**단계별 검증:**

```bash
# Phase 1: 구조 확인
./setup-repo-structure.sh
ls -la /home/seungtaena/SDX35/owrt-cache/

# Phase 2: 동기화 확인
./build-cache-sync.sh
du -sh /home/seungtaena/SDX35/owrt-cache/*
# packages/ 가 2GB 정도 있어야 함

# Phase 3: 복원 확인
./build-cache-restore.sh --mode symlink
ls -la dl/  # 심볼릭 링크 확인

# Phase 4: 빌드 확인
time make package/m2m/compile -j$(nproc)
# 3-5분이면 성공!

# Phase 5: 성능 확인
./test-cache-solo-build.sh --package m2m
cat cache-test-results-*.txt
```

## 10.11 OWRT_ROOT가 정확히 어디인가요?

**질문:** OWRT_ROOT는 SDX35까지인가요? apps_proc/owrt까지인가요?

**답변:**

```
✅ OWRT_ROOT = /home/seungtaena/SDX35/apps_proc/owrt

경로 구조:
/home/seungtaena/
└── SDX35/                    ← 프로젝트 루트
    ├── boot_images/
    ├── build_config/
    └── apps_proc/
        └── owrt/             ← OWRT_ROOT (여기!)
            ├── Makefile      ← OpenWrt 빌드 시스템
            ├── feeds.conf
            ├── dl/
            └── staging_dir/
```

**확인 방법:**
```bash
cd /home/seungtaena/SDX35/apps_proc/owrt
ls -la Makefile feeds.conf  # OpenWrt 소스 확인
```

**SDX35까지가 아니라 apps_proc/owrt 까지입니다!**

---

## 10.12 v1.1에서 무엇이 바뀌었나요?

**질문:** 이전 버전(v1.0)과 v1.1의 차이점이 무엇인가요?

**답변:**

### 주요 변경사항

1. **기본 복원 모드 변경**
   ```bash
   # v1.0: symlink가 기본값
   ./build-cache-restore.sh  # symlink 모드
   
   # v1.1: copy가 기본값
   ./build-cache-restore.sh  # copy 모드 ✅
   ./build-cache-restore.sh --mode symlink  # 명시적 지정 필요
   ```

2. **IPK packages sync/restore 비활성화**
   - `build-cache-sync.sh`: Line 202-215 주석 처리
   - `build-cache-restore.sh`: Line 272-302 주석 처리
   - 이유: staging_dir 전체 동기화로 충분

3. **Toolchain restore 비활성화**
   - `build-cache-restore.sh`: Line 396-404 주석 처리
   - 이유: staging_dir 전체 복원에 포함됨

4. **Configuration files 확장**
   - `.config_for_package_solo` 추가
   - Package solo build용 별도 설정 지원

5. **diffconfig 생성 비활성화**
   - `build-cache-sync.sh`: Line 248-257 주석 처리
   - 에러 발생으로 인한 비활성화

### 호환성

| 항목 | v1.0 | v1.1 | 호환성 |
|------|------|------|--------|
| 스크립트 호출 | ✅ | ✅ | 완전 호환 |
| Cache 구조 | ✅ | ✅ | 동일 |
| 기본 동작 | symlink | copy | ⚠️ 주의 |
| --mode 옵션 | ✅ | ✅ | 동일 |

### 마이그레이션

v1.0 → v1.1로 업그레이드 시:

```bash
# 변경 없음: Cache 재생성 불필요
# 스크립트만 업데이트하면 됨

# 단, 기본 동작이 변경되었으니 주의:
# 이전: ./build-cache-restore.sh → symlink
# 이후: ./build-cache-restore.sh → copy

# symlink 계속 사용하려면:
./build-cache-restore.sh --mode symlink
```

### 왜 이렇게 변경했나요?

**안전성 우선:**
- copy 모드가 더 안전 (cache 원본 불필요)
- 초보자에게 더 적합한 기본값
- 명시적으로 symlink 선택 가능

**간소화:**
- staging_dir 전체 동기화로 로직 단순화
- packages, toolchain 별도 관리 불필요

---

## 10.13 symlink와 copy 모드, 어떤 것을 써야 하나요?

**질문:** 기본값이 copy인데, symlink는 언제 쓰나요?

**답변:**

### 상황별 권장 모드

| 상황 | 권장 모드 | 이유 |
|------|----------|------|
| 처음 사용 | **copy** | 가장 안전, 실수 방지 |
| 프로덕션 빌드 | **copy** | Cache 원본 보호 |
| 빠른 테스트 | **symlink** | 복원 시간 ~30초 |
| CI/CD | **selective** | 균형잡힌 성능 |
| Docker | **symlink** | 컨테이너 격리로 안전 |
| Cache 실험 | **copy** | 원본 보존 |

### 실행 시간 비교

```bash
# Copy 모드 (~3분)
time ./build-cache-restore.sh --mode copy
# real    2m 45s

# Symlink 모드 (~30초)
time ./build-cache-restore.sh --mode symlink
# real    0m 28s
```

### 디스크 사용량 비교

| 모드 | dl/ | staging_dir/ | 합계 |
|------|-----|--------------|------|
| copy | 336MB 복사 | 2.1GB 복사 | ~2.5GB 추가 |
| symlink | 0 (링크) | 2.1GB 복사 | ~2.1GB 추가 |

### 추천

```bash
# 일반적인 경우: copy (기본값)
./build-cache-restore.sh

# 빠른 빌드가 필요한 경우: symlink
./build-cache-restore.sh --mode symlink

# CI/CD: selective
./build-cache-restore.sh --mode selective
```

---

# 부록 A: 환경 변수 레퍼런스

## A.1 공통 환경 변수

```bash
# OpenWrt 소스 위치 (중요!)
# SDX35까지가 아니라 apps_proc/owrt 까지입니다!
OWRT_ROOT="/home/seungtaena/SDX35/apps_proc/owrt"

# Cache repository 위치
# owrt 디렉토리와 같은 레벨의 상위 디렉토리에 위치
OWRT_CACHE_ROOT="/home/seungtaena/SDX35/owrt-cache"

# 타겟 플랫폼
OWRT_TARGET="sdx35"

# 아키텍처
OWRT_ARCH="arm_cortex-a7_neon-vfpv4"
```

### 경로 구조 다이어그램

```
/home/seungtaena/
└── SDX35/                           ← 프로젝트 루트
    ├── apps_proc/
    │   └── owrt/                    ← OWRT_ROOT (여기!)
    │       ├── Makefile
    │       ├── dl/
    │       ├── staging_dir/
    │       └── ...
    │
    └── owrt-cache/                  ← OWRT_CACHE_ROOT (여기!)
        ├── dl/
        ├── packages/
        └── ...
```

## A.2 ccache 환경 변수

```bash
# ccache 디렉토리
CCACHE_DIR="$OWRT_ROOT/.ccache"

# Base directory (경로 normalized)
CCACHE_BASEDIR="$OWRT_ROOT"

# Sloppiness (경로 독립성)
CCACHE_SLOPPINESS="file_macro,time_macros,include_file_mtime,include_file_ctime"

# 최대 크기
CCACHE_MAXSIZE="20G"

# 압축
CCACHE_COMPRESS="1"
CCACHE_COMPRESSLEVEL="6"

# 비활성화 (디버그 시)
CCACHE_DISABLE="1"
```

## A.3 Docker 환경 변수

```bash
# Cache 사용 여부
USE_BUILD_CACHE="1"  # 0=사용 안 함, 1=사용

# 빌드 작업 수
BUILD_JOBS="$(nproc)"

# Docker 이미지 이름
DOCKER_IMAGE_NAME="owrt-builder"
DOCKER_IMAGE_TAG="sdx35-cached"
```

---

# 부록 B: 디렉토리 크기 참고

## B.1 OpenWrt 빌드 디렉토리

```
전체 빌드 후:

├── dl/           336MB   (142 files)
├── build_dir/    14GB    (중간 파일)
├── staging_dir/  2.1GB   (492 IPK + toolchain)
│   └── packages/
│       └── sdx35/  2.0GB (492 IPK files)
├── .ccache/      603MB
├── bin/          1.1GB   (최종 결과)
└── 기타          수백MB

총: ~18-20GB
```

## B.2 Cache Repository

```
동기화 후:

├── dl/           336MB
├── packages/     2.0GB   (492 IPK)
├── ccache/       603MB
├── staging_dir/  2.1GB   (선택적)
├── config/       수백KB
└── 기타          수십MB

총: ~5GB (staging_dir 포함 시)
   ~3GB (staging_dir 제외 시)
```

## B.3 Docker 이미지

```
owrt-builder:sdx35-cached

├── Base OS       500MB   (Ubuntu 22.04)
├── Build tools   2GB     (gcc, ccache, etc.)
├── Cache         5GB     (위 cache repository)
└── Scripts       10MB

총: 8-10GB (압축 시 ~4GB)
```

---

# 부록 C: 성능 벤치마크

## C.1 Package Solo Build 시간

### m2m 패키지

| 환경 | 시간 | 비고 |
|------|------|------|
| 처음 전체 빌드 | 60분 | Baseline |
| Cache 없는 solo | 15-20분 | 기존 방식 |
| **Cache 있는 solo** | **3-5분** | **70% 단축!** |

### sdaemon-telit 패키지

| 환경 | 시간 | 비고 |
|------|------|------|
| 처음 전체 빌드 | 60분 | Baseline |
| Cache 없는 solo | 10-15분 | 기존 방식 |
| **Cache 있는 solo** | **2-4분** | **70% 단축!** |

## C.2 단계별 시간 분석 (m2m)

### Cache 없음

```
Download:   2-3분  (네트워크 다운로드)
Prepare:    2-3분  (압축 해제, 패치)
Compile:    10-12분 (컴파일)
Install:    1-2분
---------------------------------
Total:      15-20분
```

### Cache 있음

```
Download:   ~0초    (cache hit!)
Prepare:    1분     (일부 준비)
Compile:    2-3분   (ccache hit 70-90%)
Install:    1분
---------------------------------
Total:      3-5분   (70% 단축!)
```

## C.3 ccache 효율

| 시나리오 | sloppiness 없음 | sloppiness 있음 |
|---------|----------------|----------------|
| 동일 서버/경로 | 80-90% | 90-95% |
| 다른 경로 | 30-50% | 70-90% |
| 다른 서버 | 10-30% | 70-90% |

**결론:** sloppiness 필수!

---

# 부록 D: 체크리스트

## D.1 초기 설정 체크리스트

- [ ] OpenWrt 소스 존재 확인
- [ ] 한 번 이상 전체 빌드 완료
- [ ] IPK 파일 492개 확인
- [ ] `./setup-repo-structure.sh` 실행
- [ ] `./build-cache-sync.sh` 실행
- [ ] Cache 크기 확인 (packages/ = 2GB)

## D.2 Native 빌드 체크리스트

- [ ] `./build-cache-restore.sh` 실행
- [ ] dl/ 심볼릭 링크 확인
- [ ] staging_dir/packages/에 IPK 확인
- [ ] `.ccache/ccache.conf` 확인 (sloppiness!)
- [ ] `make package/m2m/compile` 실행
- [ ] 빌드 시간 3-5분 확인
- [ ] ccache hit rate 70% 이상 확인

## D.3 Docker 빌드 체크리스트

- [ ] Cache 준비 완료
- [ ] Docker 설치 확인
- [ ] `./build-docker-image.sh` 실행
- [ ] 이미지 크기 8-10GB 확인
- [ ] `docker run` 테스트
- [ ] 컨테이너 내 빌드 테스트
- [ ] Registry에 push (선택)

## D.4 검증 체크리스트

- [ ] Cache 복원 성공
- [ ] Download 시간 ~0초
- [ ] Compile 시간 3-5분
- [ ] ccache hit rate 70% 이상
- [ ] IPK 파일 생성 확인
- [ ] 전체 프로세스 문제 없음

---

# 마무리

## 핵심 요약

### 3가지 핵심

1. **OpenWrt는 sstate-cache가 없음**
   → IPK 패키지 재사용이 핵심

2. **경로 독립적 캐싱**
   → dl/ + packages/ + ccache (sloppiness!)

3. **Docker 환경 최적화**
   → Cache 포함 이미지로 즉시 사용

### 사용 방법

```bash
# Native (v1.1 - copy 모드 기본)
./setup-repo-structure.sh
./build-cache-sync.sh
./build-cache-restore.sh  # copy 모드 (기본값)
make package/m2m/compile

# Native (빠른 symlink 모드)
./build-cache-restore.sh --mode symlink
make package/m2m/compile

# Docker (symlink 모드 자동 사용)
./build-docker-image.sh
docker run -it owrt-builder:sdx35
make package/m2m/compile
```

### 예상 효과

- **용량:** 18GB → 5GB (75% 감소)
- **시간:** 15-20분 → 3-5분 (70% 단축)
- **ccache:** 30-50% → 70-90% (2배 개선)

---

**이 가이드가 OpenWrt 빌드 시간 단축에 도움이 되기를 바랍니다!**

문의사항은 각 스크립트의 `--help` 옵션을 참조하세요.

---

# 변경 이력

## v1.2 (2025-11-26)
**주요 변경사항:**
- **Section 8.7 추가**: sloppiness 설정 방법 상세 가이드
  - rules.mk에 환경 변수 추가 방법 (권장!)
  - Makefile 문법 주의: `:=` 사용, 따옴표 제거
  - 캐시 생성과 조회 시 동일한 sloppiness 필요성 설명
  - 환경 변수 우선순위 (환경 변수 > ccache.conf > 기본값)
  - .ccache 디렉토리 생성 시점 설명
- **Section 9.16 추가**: ccache Hit Rate 낮음 문제 해결
  - sloppiness 없이 생성된 캐시 문제
  - 조회 시 sloppiness 누락 문제
  - 진단 체크리스트
  - 캐시 재생성 필요 여부 판단 기준

---

## v1.1 (2025-11-26)
**주요 변경사항:**
- 기본 복원 모드: `symlink` → `copy` 변경
- IPK packages sync/restore: 주석 처리 (staging_dir 전체 동기화로 대체)
- Toolchain restore: 주석 처리
- **선택적 symlink dereference 전략 도입** ← CRITICAL!
  - `staging_dir/host/`: 심볼릭 링크 유지 (GLIBC 호환성)
  - `staging_dir/hostpkg/`: `-L` 적용 (절대 경로 제거)
  - `staging_dir/toolchain-*/`: 심볼릭 링크 유지
  - `staging_dir/target-*/`, `packages/`: `-L` 적용 (Docker 이식성)
- ccache 디렉토리 이름: cache 저장소에서도 `.ccache` 유지
- Configuration files: `.config_for_package_solo` 추가
- diffconfig 생성: 비활성화
- **Section 3.5 추가**: Symlink 처리 전략 상세 설명 ← NEW!
- **Section 8.7 추가**: sloppiness 설정 방법 (환경 변수 vs ccache.conf) ← NEW!
  - rules.mk에 환경 변수 추가 방법
  - 캐시 생성/조회 시 동일한 sloppiness 필요
  - Makefile 문법 주의사항 (따옴표 제거)
- **Section 9: 문제 해결 가이드 대폭 확장**
  - Section 9.11: Prebuilt 패키지 실패 (EXTERNAL_VARIANT)
  - Section 9.12: Kernel module 빌드 실패 (autoconf.h, .ko 파일)
  - Section 9.13: Cache restore 완전판 (9→10 단계로 확장)
  - Section 9.14: Racing compilation issue (location-client-api)
  - Section 9.15: GLIBC 버전 충돌 (선택적 -L 전략)
  - **Section 9.16: ccache Hit Rate 낮음 (sloppiness 설정 문제)** ← NEW!

**스크립트 변경:**
- `build-cache-sync.sh`: **선택적 -L 전략 구현**
  - Line 271-290: `staging_dir/` 하위를 5개 섹션으로 분리
  - [6a] host: `-avh` (링크 유지)
  - [6b] hostpkg: `-avhL` (링크 해제)
  - [6c] toolchain: `-avh` (링크 유지)
  - [6d] target: `-avhL` (링크 해제)
  - [6e] packages: `-avhL` (링크 해제)
- `build-cache-restore.sh`: 동일한 선택적 전략으로 복원
  - Line 454-495: 5개 섹션으로 분리 복원
- 이전 일괄 `-avhL` 적용으로 인한 GLIBC 충돌 문제 해결

**문서 변경:**
- **Section 3.5 추가**: Symlink 처리 전략 (165줄)
  - 5가지 디렉토리별 처리 전략 상세 설명
  - rsync 옵션 비교 및 이유 설명
  - 검증 방법 및 예시
- **Section 9.15 추가**: GLIBC 버전 충돌 (145줄)
  - 문제 원인 (rsync -L의 맹점)
  - `host/`와 `hostpkg/`의 차이
  - 진단 과정 및 해결 방법
- Section 9.14: Racing compilation issue 진단 및 해결
- Section 9.13 업데이트: 9단계 → 10단계 (location-client-api build_dir 복사 추가)
- 실제 문제 해결 과정을 타임라인 분석과 함께 상세히 문서화

## v1.0 (2025-11-19)
- 초기 릴리스
- 모든 스크립트 구현
- Docker 통합
- Jenkins CI/CD 지원

---
**현재 버전:** 1.1  
**날짜:** 2025-11-25  
**문서 크기:** Complete reference guide

