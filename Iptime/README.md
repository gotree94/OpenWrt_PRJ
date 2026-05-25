# OpenWrt on ipTIME — 개발부터 이식까지 완전 정복 가이드

> **목표**: ipTIME 공유기를 타겟으로 OpenWrt 펌웨어를 개발 환경 구축부터 소스 수정, 컴파일, 플래싱, 그리고 자체 패키지 제작까지 전 과정을 학습한다.

---

## 목차

1. [OpenWrt 개요](#1-openwrt-개요)
2. [OpenWrt 지원 ipTIME 모델](#2-openwrt-지원-iptime-모델)
3. [하드웨어 스펙 분석](#3-하드웨어-스펙-분석)
4. [개발 환경 구축](#4-개발-환경-구축)
5. [OpenWrt 빌드 시스템 이해](#5-openwrt-빌드-시스템-이해)
6. [소스 코드 다운로드](#6-소스-코드-다운로드)
7. [빌드 환경 설정 (feeds & menuconfig)](#7-빌드-환경-설정-feeds--menuconfig)
8. [펌웨어 컴파일](#8-펌웨어-컴파일)
9. [펌웨어 플래싱 (Flashing)](#9-펌웨어-플래싱-flashing)
10. [부팅 및 초기 설정](#10-부팅-및-초기-설정)
11. [소스 코드 수정 및 재빌드 실습](#11-소스-코드-수정-및-재빌드-실습)
12. [커스텀 패키지 만들기](#12-커스텀-패키지-만들기)
13. [커스텀 DTS (Device Tree) 작성](#13-커스텀-dts-device-tree-작성)
14. [문제 해결 (Troubleshooting)](#14-문제-해결-troubleshooting)
15. [참고 자료](#15-참고-자료)

---

## 1. OpenWrt 개요

### 1.1 OpenWrt란?

OpenWrt는 **리눅스 기반 임베디드 운영체제**로, 가정용 라우터/공유기에 최적화된 오픈소스 펌웨어입니다. 
리눅스 커널을 기반으로 하며, 경량화된 libc(musl)와 자체 패키지 관리자(opkg)를 통해 확장성을 제공합니다.

**ipTIME 공유기에 OpenWrt를 이식하면 얻는 이점:**

- **펌웨어 자유도**: 제조사의 기능 제한 없이 원하는 기능 탑재
- **최신 보안 패치**: 제조사 지원이 끊긴 제품도 최신 커널 사용 가능
- **고급 네트워크 기능**: VLAN, QoS, VPN, AdBlock, WireGuard 등
- **경량화**: 불필요한 기능 제거하고 필요한 기능만 탑재
- **커스텀 기능**: 직접 개발한 패키지를 펌웨어에 포함

### 1.2 주요 ipTIME 칩셋 아키텍처

| 칩셋 제조사 | 아키텍처 | 대표 모델 | OpenWrt Target |
|---|---|---|---|
| **MediaTek (MTK)** | MIPS32 24Kc / MIPS32 1004Kc / ARM Cortex-A7 | A2004NS-M, A3004NS-M, AX2004M, AX3004 | `mediatek` / `ramips` |
| **Realtek** | MIPS32 / ARM | A7004NS-M, A8004NS-M | `realtek` |
| **Qualcomm (QCA)** | ARM Cortex-A7 | AX8004BC-M, AX2004BC-M | `qualcommax` / `ipq807x` |

> **교육 목적 권장 모델 (MediaTek MT7621)**: ipTIME A3004NS-M, A2004NS-M, AX2004M 등
> - 커뮤니티 지원이 가장 활발함
> - OpenWrt 공식 지원
> - 880MHz 듀얼코어 MIPS, 128/256MB RAM, 16MB 플래시

---

## 2. OpenWrt 지원 ipTIME 모델

### 2.1 공식 지원 모델 (OpenWrt 메인라인)

| ipTIME 모델 | 칩셋 | 플래시 | RAM | OpenWrt Target | 지원 버전 |
|---|---|---|---|---|---|
| **A1004NS** | MT7620A | 8MB | 64MB | `ramips/mt7620` | 21.02+ |
| **A2004NS-M** | MT7621AT | 16MB | 128MB | `ramips/mt7621` | 19.07+ |
| **A3004NS-M** | MT7621AT | 16MB | 256MB | `ramips/mt7621` | 19.07+ |
| **A6004NS-M** | MT7621AT | 16MB | 512MB | `ramips/mt7621` | 21.02+ |
| **AX2004M** | MT7621AT | 16MB | 256MB | `ramips/mt7621` | 23.05+ |

### 2.2 커뮤니티 포팅 모델 (비공식)

| ipTIME 모델 | 칩셋 | 비고 |
|---|---|---|
| AX3004ITL | MT7981B | MediaTek Filogic 820 |
| AX8004BC-M | IPQ8074 | Qualcomm IPQ8074 4코어 |
| A3008NS-M | MT7621AT | 8포트 스위치 |
| T5008 | MT7621AT | 8포트 기가비트 스위치 |

### 2.3 모델 식별 방법

공유기 하단의 라벨을 확인하거나, SSH/telnet으로 접속하여 칩셋 정보를 확인합니다:

```bash
# ipTIME 공장 펌웨어에서 (관리자 페이지 → 시스템 로그 또는)
# 시리얼 콘솔로 접속한 경우
cat /proc/cpuinfo
cat /proc/mtd
dmesg | grep -i "Memory"
```

---

## 3. 하드웨어 스펙 분석

### 3.1 MT7621A 칩셋 상세

MT7621A는 ipTIME A2004NS-M, A3004NS-M 등에 사용되는 대표 칩셋입니다:

```
MT7621A 블록 다이어그램:
┌──────────────────────────────────────────┐
│  MediaTek MT7621A                        │
│                                          │
│  ┌──────────────┐  ┌──────────────────┐  │
│  │  MIPS1004Kc  │  │  MIPS1004Kc      │  │
│  │  (CPU 코어 0) │  │  (CPU 코어 1)    │  │
│  │  880MHz      │  │  880MHz          │  │
│  └──────┬───────┘  └──────┬───────────┘  │
│         │                 │               │
│         └────────┬────────┘               │
│                  │                        │
│  ┌───────────────┴──────────────┐         │
│  │        L2 Cache 256KB        │         │
│  └───────────────┬──────────────┘         │
│                  │                        │
│  ┌───────────────┴──────────────┐         │
│  │      시스템 버스 (AXI)        │         │
│  └───────┬───────┬──────┬──────┘         │
│          │       │      │                │
│  ┌───────┴──┐ ┌──┴───┐ ┌─┴────────┐     │
│  │  DDR3   │ │ PCIe │ │  Gigabit │     │
│  │ 컨트롤러 │ │ 2.0  │ │ Ethernet │     │
│  │         │ │      │ │ (5 ports)│     │
│  └─────────┘ └──────┘ └──────────┘     │
│          │       │                      │
│  ┌───────┴──┐ ┌──┴──────┐              │
│  │ 128/256MB │ │ MT7612E │              │
│  │   DDR3   │ │ (5GHz)  │              │
│  └──────────┘ └─────────┘              │
└──────────────────────────────────────────┘
```

**주요 스펙:**

| 항목 | 사양 |
|---|---|
| CPU | MIPS1004Kc (MIPS32 interAptiv) 듀얼코어 @ 880MHz |
| RAM | 128MB (A2004NS-M) / 256MB (A3004NS-M) DDR3 |
| 플래시 | 16MB SPI NOR (Winbond W25Q128) |
| 스위치 | 내장 기가비트 스위치 (5포트, MT7530) |
| 2.4GHz Wi-Fi | MT7603E (b/g/n, 2T2R) |
| 5GHz Wi-Fi | MT7612E (a/n/ac, 2T2R) |
| USB | 1x USB 3.0 (A3004NS-M) |
| PCIe | 2레인 (Wi-Fi 카드 연결) |

### 3.2 MT7620A 칩셋 상세

A1004NS 등에 사용되는 보급형 칩셋:

| 항목 | 사양 |
|---|---|
| CPU | MIPS24Kc @ 580MHz |
| RAM | 64MB DDR2 |
| 플래시 | 8MB SPI NOR |
| 스위치 | 내장 5포트 100Mbps 스위치 |
| Wi-Fi | 내장 2.4GHz b/g/n (MT7620 내장) |
| USB | 1x USB 2.0 |

### 3.3 시리얼 핀맵 찾기

ipTIME 공유기에는 대부분 4핀 시리얼 포트(UART)가 있습니다:

```
PCB 시리얼 포트 패턴 (일반적):
┌─────────────────┐
│ [ ][ ][ ][ ]    │  ← 4핀 헤더 (2.54mm间距)
│ VCC TX RX GND  │
└─────────────────┘

핀 배열 (ipTIME A2004NS-M/A3004NS-M 기준):
Pin 1: VCC (3.3V)  ─── 사용 안 함 (전원 공급용)
Pin 2: TXD         ─── USB-UART RX에 연결
Pin 3: RXD         ─── USB-UART TX에 연결
Pin 4: GND         ─── USB-UART GND에 연결

설정: 57600 baud, 8N1 (MT7621 기준)
      (MT7620은 115200 baud)
```

> **주의**: VCC 핀은 절대 연결하지 마세요. USB-UART 어댑터의 VCC는 연결하지 않아야 합니다.
> TX ↔ RX는 크로스 연결 (공유기 TX → 어댑터 RX, 공유기 RX → 어댑터 TX)

---

## 4. 개발 환경 구축

### 4.1 요구 사양

| 항목 | 최소 사양 | 권장 사양 |
|---|---|---|
| CPU | 2코어 | 4코어 이상 |
| RAM | 4GB | 16GB 이상 |
| 디스크 | 20GB 여유 | 100GB 이상 (SSD 권장) |
| OS | Ubuntu 22.04 / 24.04 LTS | 동일 (데비안 계열 권장) |
| 추가 장비 | USB-UART 어댑터 (CP2102/CH340G) | 동일 + USB TTL 3.3V |

### 4.2 WSL2 환경 구축 (Windows 사용자)

```powershell
# PowerShell (관리자 권한)
wsl --install -d Ubuntu-24.04
wsl --set-version Ubuntu-24.04 2
```

WSL2 진입 후:

```bash
# 패키지 미러 변경 (한국)
sudo sed -i 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
sudo sed -i 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list

# 필수 패키지 설치
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk \
  gcc-multilib g++-multilib gettext git libncurses5-dev libssl-dev \
  python3 python3-distutils python3-setuptools rsync unzip zlib1g-dev \
  file wget curl

# 추가 패키지 (임베디드 개발)
sudo apt install -y autoconf automake libtool pkg-config \
  u-boot-tools device-tree-compiler binutils

# USB 시리얼 어댑터 사용을 위한 설정 (WSL2)
# WSL2는 USB 디바이스 직접 연결이 기본적으로 안 됨
# usbipd-win을 사용하여 연결 필요
```

### 4.3 WSL2에서 USB 시리얼 사용 설정

```powershell
# Windows PowerShell (관리자)
# usbipd-win 설치
winget install --interactive --exact dorssel.usbipd-win

# USB-UART 어댑터 연결 후
usbipd bind --busid $(usbipd list | Select-String "CP2102\|CH340\|FTDI" | ForEach-Object { $_.ToString().Split()[0] })

# WSL2에서
sudo usbip attach -r $(hostname) -b <busid>
# 또는 usbipd-win WSL2 자동 연결 설정 (usbipd wsl --auto-attach)
```

### 8.4 4.4 네이티브 Ubuntu 환경

```bash
# 필수 패키지 설치
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk \
  gcc-multilib g++-multilib gettext git libncurses5-dev libssl-dev \
  python3 python3-distutils python3-setuptools rsync unzip zlib1g-dev \
  file wget curl

# 디렉토리 준비
mkdir -p ~/openwrt
```

---

## 5. OpenWrt 빌드 시스템 이해

### 5.1 디렉토리 구조

```
openwrt/
├── tools/              # 호스트 PC용 도구
├── toolchain/          # 크로스 컴파일러 (mipsel-openwrt-linux-gcc)
│   ├── gcc/            #   - 크로스 gcc
│   ├── binutils/       #   - 크로스 어셈블러/링커
│   └── musl/           #   - 경량 C 라이브러리
├── target/             # 타겟 플랫폼별 설정
│   └── linux/          #   - 리눅스 커널 패치/설정/DTS
│       └── ramips/     #     - Ralink/MediaTek MIPS 계열
│           └── dts/    #       - Device Tree Source 파일들
├── package/            # 패키지 소스
├── feeds/              # 외부 패키지 피드
├── build_dir/          # [빌드 중] 컴파일 중간 파일
├── staging_dir/        # [빌드 중] 크로스 컴파일된 라이브러리/헤더
├── bin/                # [빌드 결과] 최종 펌웨어 이미지
│   └── targets/
│       └── ramips/
│           └── mt7621/  # MT7621 타겟의 바이너리
└── dl/                 # [빌드 중] 다운로드한 소스 아카이브
```

### 5.2 크로스 컴파일 개념

ipTIME의 MT7621은 **MIPS 아키텍처**를 사용하므로, x86_64인 호스트 PC에서 MIPS용 바이너리를 생성해야 합니다:

```
호스트 PC (x86_64)                ipTIME (MIPS)
┌─────────────────┐            ┌─────────────────┐
│  gcc (x86_64용)  │            │  CPU: MIPS      │
│  ↓ 크로스 컴파일   │            │  1004Kc         │
│  mipsel-openwrt- │            │  아키텍처:       │
│  linux-musl-gcc  │──→ .ipk ──→│  mipsel (리틀    │
│  (MIPS용 바이너리  │            │  엔디안 MIPS)    │
│   생성)           │            │                 │
└─────────────────┘            └─────────────────┘
```

> **MIPS 엔디안**: MT7621은 **little-endian (mipsel)** 입니다. `mipsel-openwrt-linux-musl-gcc` 컴파일러를 사용하며, 일부 구형 MIPS 칩셋은 big-endian을 사용하므로 반드시 확인이 필요합니다.

---

## 6. 소스 코드 다운로드

### 6.1 공식 OpenWrt 소스 클론

```bash
cd ~/openwrt

# 안정 버전 (권장)
git clone https://git.openwrt.org/openwrt/openwrt.git openwrt-23.05
cd openwrt-23.05
git checkout v23.05.5

# 또는 최신 개발 버전
# git clone https://git.openwrt.org/openwrt/openwrt.git openwrt-main
# cd openwrt-main
```

### 6.2 MT7621 지원 확인

```bash
# ramips/mt7621 타겟이 존재하는지 확인
ls target/linux/ramips/
# dts/ 디렉토리에서 MT7621 기기 확인
ls target/linux/ramips/dts/mt7621_*.dts

# ipTIME A3004NS-M DTS 확인
ls target/linux/ramips/dts/mt7621_*iptime* 2>/dev/null || \
  ls target/linux/ramips/dts/mt7621_*
```

> **참고**: 일부 ipTIME 모델은 DTS 파일명이 정확히 매칭되지 않을 수 있습니다. `mt7621_` prefix 아래에서 유사한 스펙의 DTS를 찾거나 직접 DTS를 작성해야 합니다.

---

## 7. 빌드 환경 설정 (feeds & menuconfig)

### 7.1 Feeds 업데이트 및 설치

```bash
cd ~/openwrt/openwrt-23.05

# feeds 업데이트
./scripts/feeds update -a

# feeds 패키지 설치
./scripts/feeds install -a

# 필요한 패키지만 선택적으로 설치
# ./scripts/feeds install luci luci-base luci-app-firewall luci-proto-ipv6
```

### 7.2 타겟 설정 (make menuconfig)

```bash
make menuconfig
```

#### MT7621 기반 ipTIME 기준 설정값:

**ipTIME A2004NS-M / A3004NS-M 공통:**

| 메뉴 경로 | 선택값 | 설명 |
|---|---|---|
| **Target System** | `MediaTek Ralink MIPS` | MIPS 계열 타겟 |
| **Subtarget** | `MT7621 based boards` | MT7621 칩셋 |
| **Target Profile** | `ipTIME A3004NS-M` | 정확한 모델 선택 |
| **Target Images** | `squashfs` | 기본 이미지 형식 |
| **LuCI → Collections** | `luci` ✅ | 웹 인터페이스 |
| **LuCI → Themes** | `luci-theme-bootstrap` ✅ | 테마 |

> **중요**: Target Profile이 중요합니다. 여기서 선택된 DTS(Device Tree Source)가 컴파일되어 올바른 GPIO/스위치/LED 설정이 적용됩니다.

#### Profile에 ipTIME이 없는 경우

```bash
# DTS 파일을 직접 확인하여 유사한 스펙의 Profile 선택
ls target/linux/ramips/dts/mt7621_*.dts | head -30

# 또는 "MT7621_DEVICE_xxxx" 형태로 검색
grep -r "iptime\|iptime" target/linux/ramips/*.mk
```

---

## 8. 펌웨어 컴파일

### 8.1 기본 컴파일

```bash
cd ~/openwrt/openwrt-23.05

# 첫 빌드 (툴체인 + 커널 + 패키지 + 이미지)
make -j$(nproc) V=s
```

> **첫 빌드 소요 시간**: 약 30분~2시간 (인터넷 속도, CPU 성능에 따라 다름)
> - MT7621용 첫 빌드: 보통 40~60분
> - 이후 증분 빌드: 5~15분

### 8.2 빌드 결과 확인

```bash
# 빌드 완료 후 생성된 이미지 확인
ls -lh bin/targets/ramips/mt7621/

# 주요 이미지 파일:
# openwrt-23.05.5-ramips-mt7621-iptime_a3004ns-m-squashfs-sysupgrade.bin
# openwrt-23.05.5-ramips-mt7621-iptime_a3004ns-m-squashfs-factory.bin
# openwrt-23.05.5-ramips-mt7621-iptime_a2004ns-m-squashfs-sysupgrade.bin
# openwrt-23.05.5-ramips-mt7621-iptime_a2004ns-m-squashfs-factory.bin
# openwrt-23.05.5-ramips-mt7621-root.squashfs
# openwrt-23.05.5-ramips-mt7621-vmlinux.bin
# openwrt-23.05.5-ramips-mt7621-vmlinux.elf
# sha256sums
```

### 8.3 이미지 파일 종류 이해

| 파일명 | 용도 | 설명 |
|---|---|---|
| `*factory.bin` | **초기 플래싱용** | ipTIME 공장 펌웨어에서 최초로 OpenWrt를 설치할 때 사용 |
| `*sysupgrade.bin` | **업그레이드용** | 이미 OpenWrt가 설치된 상태에서 업그레이드 시 사용 |
| `*-squashfs-*` | squashfs 파일시스템 | 압축 읽기전용 + overlay (설정 초기화 쉬움) |
| `*-initramfs-*` | initramfs 이미지 | RAM에서만 실행 (테스트/복구용) |
| `*vmlinux.bin` | 압축된 커널 | 커널 이미지만 (uImage 형식) |
| `*vmlinux.elf` | ELF 형식 커널 | 디버깅용 |
| `*root.squashfs` | 루트FS 단독 | squashfs 루트 파일시스템만 |

### 8.4 증분 빌드 명령어

```bash
# 특정 단계만 재실행
make target/linux/compile V=s          # 커널만 재빌드
make package/luci/base/compile V=s     # 특정 패키지만 재빌드
make target/linux/install V=s          # 이미지 다시 생성

# 전체 클린
make clean              # build_dir 삭제
make dirclean           # build_dir + dl 삭제
make distclean          # 모든 것 초기화 (.config 포함)

# ccache로 재빌드 가속
sudo apt install -y ccache
export CCACHE_DIR=~/.ccache
make -j$(nproc) CCACHE=y V=s
```

---

## 9. 펌웨어 플래싱 (Flashing)

### 9.1 방법 개요

ipTIME에 OpenWrt를 설치하는 방법은 크게 3가지입니다:

| 방법 | 난이도 | 위험도 | 설명 |
|---|---|---|---|
| **Web UI 플래싱** | ★★ | ★★ | ipTIME 관리자 페이지를 통한 업데이트 |
| **TFTP 복구 모드** | ★★★ | ★ | U-Boot TFTP를 통한 플래싱 (벽돌 복구) |
| **시리얼 + TFTP** | ★★★★ | ★★ | 시리얼 콘솔로 U-Boot 명령어 직접 실행 |

### 9.2 사전 준비

펌웨어 플래싱 전에 다음을 준비하세요:

```bash
# 1. 호스트 PC의 IP 주소 확인/설정 (Windows)
ipconfig
# 이더넷 어댑터 IP를 192.168.0.x 대역으로 설정

# 2. TFTP 서버 준비 (Windows - Tftp64 또는 SolarWinds TFTP)
#    또는 Linux/macOS:
#    sudo apt install tftpd-hpa
#    sudo systemctl start tftpd-hpa

# 3. 펌웨어 파일을 TFTP 루트 디렉토리에 복사
cp bin/targets/ramips/mt7621/openwrt-*-iptime_*-factory.bin \
  /srv/tftp/firmware.bin
```

### 9.3 방법 1: ipTIME Web UI를 통한 플래싱 (초보자)

> **경고**: ipTIME의 공식 업데이트가 아니므로 웹 UI에서 강제로 업데이트해야 할 수 있습니다.
> 일부 최신 ipTIME 펌웨어에서는 서명 검증으로 인해 실패할 수 있습니다.

```bash
# 1. PC를 ipTIME LAN 포트에 연결
# 2. 브라우저에서 192.168.0.1 접속 (또는 iptime.com)
# 3. 관리자 로그인 (admin / admin)
# 4. 관리 도구 → 시스템 설정 → 펌웨어 업그레이드

# 팁: factory.bin 파일을 선택하고 업데이트 실행
# 만약 "펌웨어 파일이 올바르지 않습니다" 오류 발생 → TFTP 방법 사용
```

### 9.4 방법 2: TFTP 복구 모드 (권장)

대부분의 ipTIME 공유기는 **U-Boot 복구 모드**를 지원합니다:

```
1. 공유기 전원 OFF
2. LAN 포트와 PC를 이더넷 케이블로 연결
3. PC IP를 192.168.0.2 (고정) 로 설정
4. TFTP 서버 실행, firmware.bin 준비
5. 다음 과정 중 하나:
   
   방법 A: 리셋 버튼 방식
   - 공유기 전원 ON
   - 리셋 버튼 (뒷면)을 10초 이상 누름
   - 전면 LED가 빠르게 깜빡이면 복구 모드 진입
   - TFTP 서버가 자동으로 firmware.bin 업로드
   
   방법 B: U-Boot 콘솔 방식 (시리얼 필요)
   - 시리얼 콘솔 연결 후 부팅 로그 확인
   - "Press any key to stop autoboot" 메시지에서 키 입력
   - U-Boot 프롬프트 (#)에서 명령어 실행
```

**U-Boot 콘솔에서 TFTP 플래싱 (시리얼 필요):**

```bash
# U-Boot 프롬프트에서 다음 명령어 입력
# (IP는 환경에 맞게 조정)

# 네트워크 설정
setenv ipaddr 192.168.0.1
setenv serverip 192.168.0.2
tftp 0x82000000 firmware.bin

# 플래시 메모리에 쓰기 (MT7621 기준, 16MB NOR)
# 플래시 칩 확인
mtdparts

# 전체 펌웨어 영역에 쓰기 (일반적)
erase 0xbc000000 +0x1000000     # 16MB
cp.b 0x82000000 0xbc000000 0x1000000

# 또는 OpenWrt 방식
# flash_eraseall /dev/mtd1
# nandwrite /dev/mtd1 firmware.bin

# 재부팅
reset
```

> **참고**: MT7621의 플래시 시작 주소는 `0xbc000000` (SPI NOR 매핑)입니다.
> 정확한 주소는 `bdinfo` 또는 `mtdparts` 명령어로 확인하세요.

### 9.5 방법 3: 시리얼 콘솔 + U-Boot (고급)

전문가용 방법으로, 모든 과정을 수동으로 제어합니다:

```bash
# 시리얼 연결 (USB-UART 어댑터)
# PC에서 시리얼 터미널 실행
sudo screen /dev/ttyUSB0 57600
# 또는
sudo picocom -b 57600 /dev/ttyUSB0

# 공유기 전원 ON
# U-Boot 부팅 로그 확인
# "Press any key to stop autoboot" 메시지에서 아무 키 입력

# U-Boot 명령어로 기존 펌웨어 백업 (선택)
tftp 0x82000000 firmware.bin
md5sum 0x82000000 ${filesize}

# 현재 플래시 내용 읽어서 백업
nand read 0x82000000 0x0 0x1000000  # NAND 플래시인 경우
tftp 0x82000000 backup.bin 0x1000000

# 전체 플래시 지우기
erase 0xbc000000 +0x1000000

# 새 펌웨어 쓰기
tftp 0x82000000 openwrt-*-factory.bin
cp.b 0x82000000 0xbc000000 ${filesize}

# 확인 및 재부팅
md5sum 0x82000000 ${filesize}
cmp.b 0x82000000 0xbc000000 ${filesize}
reset
```

### 9.6 벽돌 복구 방법

플래싱에 실패하여 공유기가 부팅되지 않는 경우:

```bash
# 1. 하드웨어 리셋 (핀/버튼)
#  - 전원 켠 상태에서 리셋 버튼 30초 이상 누름
#  - LED 패턴 변화 확인

# 2. TFTP 복구 모드 재시도
#  - PC IP: 192.168.0.2 고정
#  - TFTP 서버 실행
#  - 공유기 전원 ON 직후 리셋 버튼 연타

# 3. 시리얼 콘솔로 U-Boot 진입
#  - 시리얼 케이블 연결
#  - 전원 ON 후 U-Boot 메뉴에서 복구

# 4. 프로그래머 사용 (최후의 수단)
#  - CH341A 등의 SPI 플래시 프로그래머로 칩 직접 읽기/쓰기
#  - Winbond W25Q128 (16MB) 칩 리더기로 덤프
#  - 정상 펌웨어 바이너리로 덮어쓰기
```

---

## 10. 부팅 및 초기 설정

### 10.1 첫 부팅

```bash
# 공유기 LAN 포트에 PC 연결
# PC IP를 자동(DHCP)으로 설정
# OpenWrt 기본 IP: 192.168.1.1

# SSH 접속
ssh root@192.168.1.1
# 초기에는 패스워드 없음

# 패스워드 설정
passwd
```

### 10.2 첫 부팅 시 확인 사항

```bash
# 시스템 정보
cat /proc/cpuinfo
cat /proc/meminfo
free -m
df -h

# 네트워크 인터페이스
ip link
ifconfig

# 플래시 메모리 파티션
cat /proc/mtd
# 또는
cat /proc/partitions
```

### 10.3 Wi-Fi 설정

MT7621 기반 ipTIME의 Wi-Fi 칩셋 설정:

```bash
# Wi-Fi 칩셋 확인
lspci
lsmod | grep mt

# 2.4GHz Wi-Fi (MT7603E) 설정
uci set wireless.radio0=wifi-device
uci set wireless.radio0.type='mac80211'
uci set wireless.radio0.channel='auto'
uci set wireless.radio0.htmode='HT40'
uci set wireless.radio0.country='KR'

uci set wireless.default_radio0=wifi-iface
uci set wireless.default_radio0.device='radio0'
uci set wireless.default_radio0.network='lan'
uci set wireless.default_radio0.mode='ap'
uci set wireless.default_radio0.ssid='OpenWrt-ipTIME-2G'
uci set wireless.default_radio0.encryption='psk2+ccmp'
uci set wireless.default_radio0.key='YourPassword123'

# 5GHz Wi-Fi (MT7612E) 설정
uci set wireless.radio1=wifi-device
uci set wireless.radio1.type='mac80211'
uci set wireless.radio1.channel='auto'
uci set wireless.radio1.htmode='VHT80'
uci set wireless.radio1.country='KR'
uci set wireless.radio1.disabled='0'

uci set wireless.default_radio1=wifi-iface
uci set wireless.default_radio1.device='radio1'
uci set wireless.default_radio1.network='lan'
uci set wireless.default_radio1.mode='ap'
uci set wireless.default_radio1.ssid='OpenWrt-ipTIME-5G'
uci set wireless.default_radio1.encryption='psk2+ccmp'
uci set wireless.default_radio1.key='YourPassword123'

uci commit wireless
wifi reload
```

### 10.4 WAN 포트 설정

ipTIME 공유기의 WAN 포트(일반적으로 파란색 포트) 설정:

```bash
# WAN 인터페이스가 잘 인식되었는지 확인
ip link

# WAN 인터페이스가 eth0.2 등 VLAN 태그로 설정된 경우
# MT7621의 내장 스위치(MT7530)는 VLAN 설정이 중요

# OpenWrt 기본 VLAN 설정 확인
cat /etc/config/network
# 예시:
# config interface 'lan'
#     option device 'eth0.1'
#     option proto 'static'
#     option ipaddr '192.168.1.1'
#     option netmask '255.255.255.0'
#
# config interface 'wan'
#     option device 'eth0.2'
#     option proto 'dhcp'

# WAN이 DHCP인 경우 자동 IP 할당 확인
ip addr show eth0.2
```

### 10.5 VLAN 설정 (선택)

MT7530 스위치의 VLAN 설정을 변경하려면:

```bash
# 현재 VLAN 설정 확인
cat /etc/config/network

# VLAN 수동 설정 예시 (WAN: 포트 0, LAN: 포트 1-4)
# /etc/config/network
# config device
#     option name 'switch0'
#     option name 'mt7530'
#
# config switch_vlan
#     option device 'switch0'
#     option vlan '1'
#     option ports '1 2 3 4 6t'
#
# config switch_vlan
#     option device 'switch0'
#     option vlan '2'
#     option ports '0 6t'
```

### 10.6 LuCI 웹 인터페이스 확인

```bash
# 브라우저에서 http://192.168.1.1 접속
# 또는 공유기 IP 확인 후 접속
ip route show default
```

---

## 11. 소스 코드 수정 및 재빌드 실습

### 11.1 ipTIME LED 설정 변경 실습

ipTIME 공유기의 LED 동작 방식을 커스터마이징합니다:

```bash
# 호스트 PC에서
cd ~/openwrt/openwrt-23.05

# ipTIME A3004NS-M DTS 파일 찾기
find target/linux/ramips/dts -name "*iptime*" -o -name "*a3004ns*"

# DTS 파일 내용 확인
cat target/linux/ramips/dts/mt7621_iptime_a3004ns-m.dts
```

**DTS 파일 예시 (LED 섹션):**

```dts
// target/linux/ramips/dts/mt7621_iptime_a3004ns-m.dts
// (일부 발췌)

/ {
    compatible = "iptime,a3004ns-m", "mediatek,mt7621-soc";

    leds {
        compatible = "gpio-leds";

        power {
            label = "blue:power";
            gpios = <&gpio 12 GPIO_ACTIVE_LOW>;
        };

        wan {
            label = "blue:wan";
            gpios = <&gpio 13 GPIO_ACTIVE_LOW>;
        };

        wlan2g {
            label = "blue:wlan2g";
            gpios = <&gpio 14 GPIO_ACTIVE_LOW>;
        };

        wlan5g {
            label = "blue:wlan5g";
            gpios = <&gpio 15 GPIO_ACTIVE_LOW>;
        };

        usb {
            label = "blue:usb";
            gpios = <&gpio 16 GPIO_ACTIVE_LOW>;
        };
    };
};
```

### 11.2 LED 동작 방식 변경

```bash
# DTS 파일에서 LED 트리거를 "heartbeat"로 변경
# (예: power LED를 부팅 상태 표시용으로)

# LED 트리거는 /sys/class/leds/에서 제어 가능
# 또는 /etc/config/system에서 LED 설정 추가

# system LED 설정 예시
cat package/base-files/files/etc/config/system
```

**`/etc/config/system` 또는 `/etc/board.d/01_leds`에서 LED 설정:**

```bash
# target/linux/ramips/base-files/etc/board.d/01_leds
# ipTIME A3004NS-M LED 매핑 예시

iptime,a3004ns-m)
    ucidef_set_led_netdev "wan" "WAN" "blue:wan" "eth0.2"
    ucidef_set_led_wlan "wlan2g" "2.4GHz" "blue:wlan2g" "phy0"
    ucidef_set_led_wlan "wlan5g" "5GHz" "blue:wlan5g" "phy1"
    ucidef_set_led_usbport "usb" "USB" "blue:usb" "usb1"
    ucidef_set_led_heartbeat "power" "Power" "blue:power"
    ;;
```

### 11.3 LED 변경 후 재빌드

```bash
# DTS 수정 후 커널/이미지 재빌드
make target/linux/compile V=s
make -j$(nproc) V=s

# 또는 해당 기기만 빌드
# make target/linux/install V=s
```

### 11.4 스위치 포트 라벨 변경

ipTIME 공유기의 LAN 포트 라벨링을 변경하려면:

```bash
# 네트워크 설정 파일 확인
cat package/base-files/files/lib/upgrade/platform.sh

# 02_network 파일에서 포트 매핑 확인
cat target/linux/ramips/base-files/etc/board.d/02_network

# ipTIME A3004NS-M 예시
# "iptime,a3004ns-m")
#     ucidef_set_interfaces_lan_wan "eth0.1" "eth0.2"
#     ;;
```

---

## 12. 커스텀 패키지 만들기

### 12.1 ipTIME 전용 패키지 예제

ipTIME 공유기의 하드웨어 정보를 표시하는 커스텀 패키지를 만듭니다:

```bash
cd ~/openwrt/openwrt-23.05
mkdir -p package/iptime-info/files
```

**`package/iptime-info/Makefile`:**

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=iptime-info
PKG_VERSION:=1.0
PKG_RELEASE:=1

PKG_MAINTAINER:=Your Name <your@email.com>
PKG_LICENSE:=GPL-2.0

include $(INCLUDE_DIR)/package.mk

define Package/iptime-info
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=ipTIME hardware information tool
  DEPENDS:=+ubus +libubus
  PKGARCH:=all
endef

define Package/iptime-info/description
  Display ipTIME router hardware information on OpenWrt
  Shows CPU, RAM, Flash, Wi-Fi chip, and switch info.
endef

define Build/Prepare
endef

define Build/Configure
endef

define Build/Compile
endef

define Package/iptime-info/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) ./files/iptime-info.sh $(1)/usr/bin/iptime-info
endef

$(eval $(call BuildPackage,iptime-info))
```

**`package/iptime-info/files/iptime-info.sh`:**

```bash
#!/bin/sh

echo "============================================"
echo "  ipTIME OpenWrt Hardware Information"
echo "============================================"
echo ""

# 모델명
echo "[Device Info]"
echo "  Model   : $(cat /tmp/sysinfo/model 2>/dev/null || echo 'Unknown')"
echo "  Board   : $(cat /tmp/sysinfo/board 2>/dev/null || echo 'Unknown')"

# CPU 정보
echo ""
echo "[CPU Info]"
if [ -f /proc/cpuinfo ]; then
    echo "  CPU     : $(grep 'system type' /proc/cpuinfo | head -1 | cut -d: -f2)"
    echo "  MHz     : $(grep 'cpu MHz' /proc/cpuinfo | head -1 | cut -d: -f2)"
    echo "  Core    : $(grep 'cpu model' /proc/cpuinfo | wc -l) cores"
fi

# 메모리 정보
echo ""
echo "[Memory Info]"
total=$(free | grep Mem | awk '{print $2}')
free=$(free | grep Mem | awk '{print $4}')
used=$((total - free))
echo "  Total   : $((total / 1024)) MB"
echo "  Used    : $((used / 1024)) MB"
echo "  Free    : $((free / 1024)) MB"

# 플래시 정보
echo ""
echo "[Flash Info]"
cat /proc/mtd 2>/dev/null | while read line; do
    echo "  $line"
done

# Wi-Fi 정보
echo ""
echo "[Wi-Fi Info]"
for phy in /sys/class/ieee80211/*; do
    [ -d "$phy" ] || continue
    phy_name=$(basename $phy)
    hw_name=$(cat $phy/device/uevent 2>/dev/null | grep DRIVER | cut -d= -f2)
    echo "  PHY: $phy_name ($hw_name)"
done

# 스위치 정보
echo ""
echo "[Switch Info]"
if [ -d /sys/class/net/eth0 ]; then
    echo "  Switch : MT7530 (MediaTek Gigabit Switch)"
    swconfig dev switch0 show 2>/dev/null | grep -E "ports|link|VLAN" | while read line; do
        echo "    $line"
    done
fi

echo ""
echo "============================================"
echo "  Uptime: $(uptime | sed 's/.*up //' | sed 's/,.*//')"
echo "  Load  : $(cat /proc/loadavg | cut -d' ' -f1-3)"
echo "============================================"
```

실행 권한 부여:

```bash
chmod +x package/iptime-info/files/iptime-info.sh
```

### 12.2 패키지 빌드 및 설치

```bash
# menuconfig에서 패키지 선택
make menuconfig
# Utilities → iptime-info 선택

# 패키지 빌드
make package/iptime-info/compile V=s

# 빌드된 IPK 확인
ls -lh bin/packages/mipsel_24kc/base/iptime-info*.ipk

# 공유기에 설치
scp bin/packages/mipsel_24kc/base/iptime-info*.ipk \
  root@192.168.1.1:/tmp/

ssh root@192.168.1.1
opkg install /tmp/iptime-info*.ipk
iptime-info
```

### 12.3 MIPS 크로스 컴파일 C 패키지

**`package/hello-iptime/Makefile`:**

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=hello-iptime
PKG_VERSION:=1.0
PKG_RELEASE:=1

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk

define Package/hello-iptime
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=Hello from ipTIME OpenWrt
  DEPENDS:=
endef

define Build/Prepare
  mkdir -p $(PKG_BUILD_DIR)
  $(CP) ./src/* $(PKG_BUILD_DIR)/
endef

define Build/Configure
endef

define Build/Compile
  $(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/hello-iptime \
    $(PKG_BUILD_DIR)/main.c
endef

define Package/hello-iptime/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) $(PKG_BUILD_DIR)/hello-iptime $(1)/usr/bin/
endef

$(eval $(call BuildPackage,hello-iptime))
```

**`package/hello-iptime/src/main.c`:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    FILE *fp;
    char buffer[128];

    printf("========================================\n");
    printf("  Hello from ipTIME OpenWrt!\n");
    printf("  Compiled for: MIPSEL 24Kc\n");
    printf("========================================\n\n");

    /* CPU 정보 읽기 */
    fp = fopen("/proc/cpuinfo", "r");
    if (fp) {
        printf("[CPU Information]\n");
        while (fgets(buffer, sizeof(buffer), fp)) {
            if (strstr(buffer, "system type") || strstr(buffer, "cpu MHz"))
                printf("  %s", buffer);
        }
        fclose(fp);
    }

    /* 메모리 정보 읽기 */
    fp = fopen("/proc/meminfo", "r");
    if (fp) {
        printf("\n[Memory Information]\n");
        fgets(buffer, sizeof(buffer), fp);
        printf("  %s", buffer);
        fclose(fp);
    }

    /* 실행 시간 확인 */
    fp = popen("uptime", "r");
    if (fp) {
        printf("\n[System Uptime]\n");
        fgets(buffer, sizeof(buffer), fp);
        printf("  %s\n", buffer);
        pclose(fp);
    }

    printf("\n========================================\n");
    printf("  Built with: %s\n", __VERSION__);
    printf("========================================\n");

    return 0;
}
```

---

## 13. 커스텀 DTS (Device Tree) 작성

### 13.1 DTS가 필요한 경우

OpenWrt에서 공식 지원하지 않는 ipTIME 모델에 포팅할 때는 **Device Tree Source (DTS)** 파일을 직접 작성해야 합니다.

### 13.2 DTS 기본 구조

```dts
// target/linux/ramips/dts/mt7621_my_iptime_model.dts

/dts-v1/;

#include "mt7621.dtsi"

/ {
    compatible = "iptime,my-model", "mediatek,mt7621-soc";
    model = "ipTIME My Model";

    aliases {
        led-boot = &led_power;
        led-failsafe = &led_power;
        led-running = &led_power;
        led-upgrade = &led_power;
        label-mac-device = &gmac0;
    };

    chosen {
        bootargs = "console=ttyS0,57600";
    };

    leds {
        compatible = "gpio-leds";

        led_power: power {
            label = "blue:power";
            gpios = <&gpio 12 GPIO_ACTIVE_LOW>;
        };
    };

    keys {
        compatible = "gpio-keys";

        reset {
            label = "reset";
            gpios = <&gpio 18 GPIO_ACTIVE_LOW>;
            linux,code = <KEY_RESTART>;
        };
    };
};

&spi0 {
    status = "okay";

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;

        partitions {
            compatible = "fixed-partitions";

            partition@0 {
                label = "u-boot";
                reg = <0x0 0x30000>;
                read-only;
            };

            partition@30000 {
                label = "u-boot-env";
                reg = <0x30000 0x10000>;
                read-only;
            };

            factory: partition@40000 {
                label = "factory";
                reg = <0x40000 0x10000>;
                read-only;
            };

            partition@50000 {
                label = "firmware";
                reg = <0x50000 0x1fb0000>;
                compatible = "denx,uimage";
            };
        };
    };
};

&pcie {
    status = "okay";
};

&pcie0 {
    wifi0: mt76@0,0 {
        reg = <0x0000 0 0 0 0>;
        mediatek,mtd-eeprom = <&factory 0x0>;
        ieee80211-freq-limit = <2400000 2500000>;
    };
};

&pcie1 {
    wifi1: mt76@0,0 {
        reg = <0x0000 0 0 0 0>;
        mediatek,mtd-eeprom = <&factory 0x8000>;
        ieee80211-freq-limit = <5000000 6000000>;
    };
};

&gmac0 {
    mtd-mac-address = <&factory 0xe000>;
};

&switch0 {
    ports = <0 1 2 3 4 5>;
};

&ethphy0 {
    /delete-property/ interrupts;
};
```

### 13.3 DTS GPIO 찾는 방법

ipTIME 공유기의 GPIO를 확인하려면 시리얼 콘솔로 공장 펌웨어에서 확인합니다:

```bash
# 공장 펌웨어 시리얼 콘솔에서
# GPIO 직접 제어해보기
ls /sys/class/gpio/
cd /sys/class/gpio/
echo 12 > export
echo "out" > gpio12/direction
echo 0 > gpio12/value   # LED ON (ACTIVE_LOW인 경우)
echo 1 > gpio12/value   # LED OFF

# 또는 OpenWrt에서 커스텀 DTS로 부팅 후
cat /sys/kernel/debug/gpio
```

### 13.4 DTS 컴파일 및 적용

```bash
# DTS → DTB 컴파일
make target/linux/compile V=s

# 생성된 DTB 확인
ls -lh build_dir/target-mipsel_24kc/linux-ramips_mt7621/*.dtb

# 또는 수동 컴파일
./scripts/dtc -I dts -O dtb -o mt7621_my_iptime_model.dtb \
  target/linux/ramips/dts/mt7621_my_iptime_model.dts
```

### 13.5 새 DTS를 빌드 시스템에 등록

```bash
# 1. DTS 파일 저장
# target/linux/ramips/dts/mt7621_my_model.dts

# 2. image/mt7621.mk에 디바이스 추가
# target/linux/ramips/image/mt7621.mk 편집
nano target/linux/ramips/image/mt7621.mk

# 다음 블록 추가:
# define Device/iptime_my-model
#   DEVICE_VENDOR := ipTIME
#   DEVICE_MODEL := My Model
#   DEVICE_PACKAGES := kmod-mt7603 kmod-mt7615e
#   BOARDNAME := MT7621_MT7530
#   DEVICE_DTS := mt7621_my_model
# endef
# TARGET_DEVICES += iptime_my-model

# 3. base-files에 LED/네트워크 설정 추가
# target/linux/ramips/base-files/etc/board.d/01_leds
# target/linux/ramips/base-files/etc/board.d/02_network
```

---

## 14. 문제 해결 (Troubleshooting)

### 14.1 빌드 오류

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| `Package foo is missing` | feeds 미설치 | `./scripts/feeds install foo` |
| `No rule to make target` | 설정 불일치 | `make defconfig` 후 `make menuconfig` 재설정 |
| `mipsel-openwrt-linux-musl-gcc: not found` | toolchain 미빌드 | `make toolchain/install V=s` |
| `Kernel build failed` | DTS 오류 | DTS 문법 확인: `make target/linux/compile V=s 2>&1 | grep error` |
| `Image too big` | 플래시 크기 초과 | 불필요한 패키지 제거 후 재빌드 |

### 14.2 플래싱 오류

| 증상 | 원인 | 해결 |
|---|---|---|
| **공유기가 켜지지 않음** | 잘못된 펌웨어 | TFTP 복구 모드 또는 프로그래머 |
| **Wi-Fi가 안 잡힘** | EEPROM 데이터 문제 | `factory` 파티션 확인, 올바른 칩셋 드라이버 확인 |
| **LED가 모두 꺼짐** | 잘못된 GPIO 설정 | DTS의 GPIO 번호 확인 |
| **WAN 포트 인식 안 됨** | 잘못된 VLAN 설정 | `/etc/config/network` VLAN 태그 확인 |
| **부팅 루프** | 커널 패닉 | 시리얼 로그로 원인 파악 후 DTS/커널 설정 수정 |

### 14.3 시리얼 콘솔 로그 분석

```bash
# 정상 부팅 로그 예시 (MT7621):
#
# U-Boot 1.1.3 (Oct 20 2021 - 10:00:00)
# Board: MT7621_MT7530
# DRAM:  256 MB
# SPI:   W25Q128 (16 MB)
# ...
# ## Booting kernel at 0xbc050000...
# ...
# [    0.000000] Linux version 5.15.150
# [    0.000000] CPU0 revision is: 0001992f (MIPS 1004Kc)
# [    0.000000] CPU1 revision is: 0001992f (MIPS 1004Kc)
# ...
# [    3.141592] mt7530 mdio-bus:1f: MT7530 adaptor configured
# [    3.200000] mt7603e 0000:00:00.0: WM Firmware Version: 4.4.2
# [    3.250000] mt7615e 0000:01:00.0: WM Firmware Version: 4.4.2
# ...
# [   10.000000] procd: - init -
```

### 14.4 디버깅 명령어 모음

```bash
# 부팅 중 커널 메시지 확인
logread

# 시스템 상태 진단
top                     # 프로세스/CPU 사용률
free                    # 메모리 상태
df -h                   # 디스크 사용량
cat /proc/mtd           # MTD 파티션 (플래시)
swconfig dev switch0 show  # 스위치 설정

# Wi-Fi 진단
iwinfo                  # Wi-Fi 상태
iw dev wlan0 info       # 무선 인터페이스 정보
iw list                 # 무선 하드웨어 기능

# 네트워크 진단
tcpdump -i eth0.2       # WAN 포트 패킷 캡처
ping -c 5 8.8.8.8       # 인터넷 연결 확인
nslookup google.com     # DNS 확인

# 패키지 관리
opkg list-installed     # 설치된 패키지 목록
opkg list               # 사용 가능한 패키지 목록
opkg update             # 패키지 목록 업데이트

# GPIO 확인
cat /sys/kernel/debug/gpio  # GPIO 상태 (DTS 디버깅)
```

---

## 15. 참고 자료

### 15.1 하드웨어 데이터시트

| 자료 | URL |
|---|---|
| MT7621A Datasheet | https://wiki.openwrt.org/media/mtk/mt7621a_datasheet.pdf |
| MT7603E Datasheet | https://www.mediatek.com/products/wifi/mt7603e |
| MT7612E Datasheet | https://www.mediatek.com/products/wifi/mt7612e |
| Winbond W25Q128 | https://www.winbond.com/resource-files/w25q128jv%20revf%2003272018.pdf |

### 15.2 공식 문서

| 자료 | URL |
|---|---|
| OpenWrt 공식 문서 | https://openwrt.org/docs/start |
| 빌드 시스템 가이드 | https://openwrt.org/docs/guide-developer/build-system/start |
| OpenWrt on MT7621 | https://openwrt.org/toh/hwdata/mediatek/mt7621 |
| Device Tree 가이드 | https://openwrt.org/docs/techref/hardware/device-tree |
| 패키지 개발 가이드 | https://openwrt.org/docs/guide-developer/package-development |
| 이미지 생성 | https://openwrt.org/docs/guide-user/installation/generic.flashing |

### 15.3 커뮤니티

- **OpenWrt 포럼**: https://forum.openwrt.org/
- **OpenWrt 한국 커뮤니티 (Klut):** https://openwrthub.com/
- **ipTIME 포팅 정보 (DCInside):** https://gall.dcinside.com/mgallery/board/lists?id=openwrt
- **GitHub**: https://github.com/openwrt/openwrt

### 15.4 교육 실습 커리큘럼 (제안)

| 단계 | 내용 | 예상 시간 | 준비물 |
|---|---|---|---|
| 1단계 | ipTIME 공유기 분해 및 시리얼 핀 찾기 | 30분 | 드라이버, USB-UART |
| 2단계 | 시리얼 콘솔 연결 및 U-Boot 로그 분석 | 30분 | USB-UART, PuTTY |
| 3단계 | 개발 환경 구축 및 소스 다운로드 | 30분 | PC |
| 4단계 | menuconfig 설정 및 첫 빌드 | 2시간 | PC |
| 5단계 | 공장 펌웨어 백업 | 20분 | TFTP 서버 |
| 6단계 | OpenWrt 플래싱 (TFTP/시리얼) | 30분 | 공유기 |
| 7단계 | 초기 설정 (Wi-Fi, WAN, LuCI) | 30분 | 공유기 |
| 8단계 | DTS 수정 및 LED 동작 변경 | 1시간 | PC |
| 9단계 | 커스텀 패키지 제작 및 설치 | 1시간 | PC |
| 10단계 | 전체 재빌드 및 sysupgrade | 30분 | PC + 공유기 |

### 15.5 주요 ipTIME 모델별 시리얼 속도

| 모델 | 칩셋 | Baud Rate | 시리얼 전압 |
|---|---|---|---|
| A1004NS | MT7620A | **115200** | 3.3V |
| A2004NS-M | MT7621AT | **57600** | 3.3V |
| A3004NS-M | MT7621AT | **57600** | 3.3V |
| A6004NS-M | MT7621AT | **57600** | 3.3V |
| AX2004M | MT7621AT | **57600** | 3.3V |
| AX3004ITL | MT7981B | **115200** | 3.3V |
| A8004NS-M | RTL8197D | **115200** | 3.3V |

---

> **주의사항**: 
> - ipTIME 공유기에 OpenWrt를 설치하면 제조사 보증이 상실됩니다.
> - 플래싱 과정에서 실수하면 공유기가 벽돌(Brick) 상태가 될 수 있습니다.
> - 반드시 시리얼 케이블과 TFTP 환경을 먼저 준비하세요.
> - 교육 목적이라면 저렴한 중고 모델(A2004NS-M 또는 A3004NS-M)로 시작하는 것을 권장합니다.
> - OpenWrt 포팅 과정은 하드웨어 지식과 리눅스 커널에 대한 이해가 필요합니다.

---

> **다음 학습 주제**: 라즈베리 파이 OpenWrt 개발 가이드를 참고하세요 (`C:\OpenWrt\RaspberryPi\RaspberryPi_OpenWrt_개발_가이드.md`)
