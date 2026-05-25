# OpenWrt on Raspberry Pi — 개발부터 이식까지 완전 정복 가이드

> **목표**: 라즈베리 파이를 타겟으로 OpenWrt 펌웨어를 개발 환경 구축부터 소스 수정, 컴파일, SD 카드 이식, 그리고 자체 패키지 제작까지 전 과정을 학습한다.

---

## 목차

1. [OpenWrt 개요](#1-openwrt-개요)
2. [지원되는 라즈베리 파이 모델](#2-지원되는-라즈베리-파이-모델)
3. [개발 환경 구축](#3-개발-환경-구축)
4. [OpenWrt 빌드 시스템 이해](#4-openwrt-빌드-시스템-이해)
5. [소스 코드 다운로드](#5-소스-코드-다운로드)
6. [빌드 환경 설정 (feeds & menuconfig)](#6-빌드-환경-설정-feeds--menuconfig)
7. [펌웨어 컴파일](#7-펌웨어-컴파일)
8. [SD 카드에 이미지 쓰기](#8-sd-카드에-이미지-쓰기)
9. [부팅 및 초기 설정](#9-부팅-및-초기-설정)
10. [소스 코드 수정 및 재빌드 실습](#10-소스-코드-수정-및-재빌드-실습)
11. [커스텀 패키지 만들기](#11-커스텀-패키지-만들기)
12. [펌웨어 업데이트 (sysupgrade)](#12-펌웨어-업데이트-sysupgrade)
13. [문제 해결 (Troubleshooting)](#13-문제-해결-troubleshooting)
14. [참고 자료](#14-참고-자료)

---

## 1. OpenWrt 개요

### 1.1 OpenWrt란?

OpenWrt는 **리눅스 기반 임베디드 운영체제**로, 가정용 라우터/공유기에 최적화된 오픈소스 펌웨어입니다. 리눅스 커널을 기반으로 하며, 경량화된 libc 구현체(musl)와 자체 패키지 관리자(opkg)를 통해 확장성을 제공합니다.

**주요 특징:**

- **완전한 맞춤형**: 리눅스 커널부터 모든 패키지를 빌드 타임에 선택/구성 가능
- **경량화**: 4MB 플래시, 32MB RAM에서도 동작 가능
- **패키지 시스템**: 8000+ 패키지를 opkg로 설치 가능
- **루트 파일 시스템 오버레이**: squashfs + overlayfs 조합으로 초기화 상태 유지
- **엔터프라이즈급 네트워킹**: VLAN, QoS, VPN, 방화벽 등 완벽 지원

### 1.2 OpenWrt 아키텍처

```
┌──────────────────────────────────────────┐
│              LuCI (Web UI)                │  ← 웹 관리 인터페이스
├──────────────────────────────────────────┤
│              uHTTPd / nginx               │  ← 웹 서버
├──────────────────────────────────────────┤
│           procd (init system)             │  ← 프로세스 관리/부트
├──────────────────────────────────────────┤
│    netifd / hostapd / dnsmasq / ...       │  ← 네트워크 데몬들
├──────────────────────────────────────────┤
│              opkg (패키지 관리자)           │  ← 패키지 설치/삭제
├──────────────────────────────────────────┤
│              musl libc (C 라이브러리)       │  ← 경량 libc
├──────────────────────────────────────────┤
│         Linux Kernel (커스텀)              │  ← 5.15 / 6.1 / 6.6 LTS
├──────────────────────────────────────────┤
│         Bootloader (U-Boot)               │  ← 부트로더
└──────────────────────────────────────────┘
```

### 1.3 빌드 시스템 개요 (OpenWrt Buildroot)

OpenWrt는 **Buildroot** 기반의 크로스 컴파일 빌드 시스템을 사용합니다:

- 호스트 PC(x86_64)에서 ARM/MIPS 등 타겟 CPU용 바이너리 생성
- `make menuconfig` → `make` 의 단순한 2단계로 전체 펌웨어 생성
- **toolchain** → **kernel** → **packages** 순차 빌드
- 병렬 빌드 지원 (`make -j$(nproc)`)

---

## 2. 지원되는 라즈베리 파이 모델

### 2.1 타겟/서브타겟 매핑

| 라즈베리 파이 모델 | SoC | OpenWrt Target | OpenWrt Subtarget | 커널 아키텍처 |
|---|---|---|---|---|
| **RPi Zero W / Zero 2 W** | BCM2835 / RP3A0 | `bcm27xx` | `bcm2708` (Zero W) / `bcm2710` (Zero 2 W) | armv6l / armv8a |
| **RPi 2 (v1.2)** | BCM2837 | `bcm27xx` | `bcm2709` | armv7l |
| **RPi 3B / 3B+** | BCM2837B0 | `bcm27xx` | `bcm2710` | armv8a (aarch64) |
| **RPi 4B / 400** | BCM2711 | `bcm27xx` | `bcm2711` | aarch64 |
| **RPi 5** | BCM2712 | `bcm27xx` | `bcm2712` | aarch64 |

> **권장**: RPi 4B (BCM2711)는 성능/가격/지원 측면에서 최적의 선택입니다. USB 3.0 + 기가비트 이더넷으로 OpenWrt 라우터에 적합합니다.

### 2.2 라즈베리 파이를 라우터로 사용 시 고려사항

| 항목 | RPi 4B | RPi 5 |
|---|---|---|
| 이더넷 | 기가비트 1포트 (USB 이더넷 추가 필요) | 기가비트 1포트 |
| USB | 2x USB 3.0, 2x USB 2.0 | 2x USB 3.0, 2x USB 2.0 |
| Wi-Fi | BCM43455 (5GHz a/n/ac) | BCM43455 (동일) |
| RAM | 1/2/4/8GB | 4/8GB |
| 전력 | 5V/3A ~15W | 5V/5A ~25W |
| **라우터 구성** | USB 이더넷 어댑터로 WAN 포트 추가 필요 | 동일 |

> **팁**: WAN용 USB3 기가비트 이더넷 어댑터(ASIX AX88179, Realtek RTL8153)를 사용하여 2포트 라우터로 구성하세요.

---

## 3. 개발 환경 구축

### 3.1 요구 사양

| 항목 | 최소 사양 | 권장 사양 |
|---|---|---|
| CPU | 2코어 | 4코어 이상 |
| RAM | 4GB | 16GB 이상 |
| 디스크 | 20GB 여유 | 100GB 이상 (SSD 권장) |
| OS | Ubuntu 22.04 / 24.04 LTS | 동일 (데비안 계열 권장) |
| 인터넷 | 필요 | 필요 |

### 3.2 WSL2 환경 구축 (Windows 사용자)

Windows에서 개발 시 WSL2(Ubuntu)를 권장합니다:

```powershell
# PowerShell (관리자 권한)
wsl --install -d Ubuntu-24.04
wsl --set-version Ubuntu-24.04 2
```

WSL2 진입 후:

```bash
# 패키지 미러 업데이트
sudo sed -i 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
sudo sed -i 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list

# 필수 패키지 설치
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk \
  gcc-multilib g++-multilib gettext git libncurses5-dev libssl-dev \
  python3 python3-distutils python3-setuptools rsync unzip zlib1g-dev \
  file wget curl qemu-utils

# 추가 패키지
sudo apt install -y autoconf automake libtool pkg-config \
  u-boot-tools device-tree-compiler
```

### 3.3 네이티브 Ubuntu 환경 구축

```bash
# 필수 패키지 설치
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk \
  gcc-multilib g++-multilib gettext git libncurses5-dev libssl-dev \
  python3 python3-distutils python3-setuptools rsync unzip zlib1g-dev \
  file wget curl qemu-utils

# 디렉토리 준비
mkdir -p ~/openwrt
```

### 3.4 macOS 환경 구축

```bash
# Homebrew 설치 (없는 경우)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 필수 패키지
brew install coreutils diffutils findutils gawk gnu-getopt gnu-tar \
  grep make ncurses python3 libtool util-linux wget qemu
```

> **주의**: macOS는 대소문자 구분 없는 파일 시스템(APFS 기본)이므로, 대소문자 구분 볼륨을 생성하는 것이 좋습니다:
> ```bash
> diskutil apfs addVolume disk1 "Case-sensitive APFS" OpenWrt 50g
> ```

---

## 4. OpenWrt 빌드 시스템 이해

### 4.1 디렉토리 구조

```
openwrt/
├── tools/              # 호스트 PC용 도구 (빌드에 필요)
├── toolchain/          # 크로스 컴파일러 (gcc, binutils, musl)
├── target/             # 타겟 플랫폼별 커널 및 이미지 설정
│   ├── linux/          #   - 리눅스 커널 패치/설정
│   └── imagebuilder/   #   - 이미지 생성 도구
├── include/            # 빌드 시스템 Makefile include
├── package/            # 패키지 소스 (feeds에서 다운로드)
├── feeds/              # 패키지 피드 설정
├── config/             # 빌드 설정 관련
├── scripts/            # 빌드 스크립트
├── build_dir/          # [빌드 중 생성] 컴파일 중간 파일
├── staging_dir/        # [빌드 중 생성] 임시 설치 파일
├── bin/                # [빌드 결과] 최종 펌웨어 이미지
│   └── targets/        #   - 타겟별 바이너리
└── dl/                 # [빌드 중 생성] 다운로드한 소스 아카이브
```

### 4.2 빌드 과정 흐름도

```
make menuconfig
     │
     ▼
  target selected ──────→ tools/ 빌드
     │                        │
     │                        ▼
     │                  toolchain 빌드
     │                  (gcc, musl 등)
     │                        │
     │                        ▼
     ├── feeds update ──→ 패키지 소스 다운로드
     ├── feeds install
     │                        │
     │                        ▼
     │                  커널 빌드 (리눅스 + DTB)
     │                        │
     │                        ▼
     │                  패키지 빌드 (선택한 패키지들)
     │                        │
     │                        ▼
     │                  루트 파일 시스템 생성
     │                  (squashfs + overlayfs)
     │                        │
     │                        ▼
     └──→         최종 펌웨어 이미지 생성
                   (kernel + rootfs 통합)
```

### 4.3 디렉토리 별 빌드 순서

make 빌드 시스템은 의존성에 따라 자동으로 다음 순서로 빌드합니다:

1. **tools** → host에서 실행될 도구들 (sed, grep, mkimage 등)
2. **toolchain** → 크로스 컴파일러 (gcc, binutils, musl/glibc)
3. **target/linux** → 리눅스 커널
4. **package** → 선택한 패키지들
5. **이미지 생성** → 커널 + 루트 파일 시스템 통합

---

## 5. 소스 코드 다운로드

### 5.1 공식 OpenWrt 소스 클론

```bash
cd ~/openwrt

# 안정 버전 (권장) - 23.05 시리즈
git clone https://git.openwrt.org/openwrt/openwrt.git openwrt-23.05
cd openwrt-23.05
git checkout v23.05.5

# 또는 최신 개발 버전 (main)
git clone https://git.openwrt.org/openwrt/openwrt.git openwrt-main
cd openwrt-main
```

> **팁**: OpenWrt는 약 6개월마다 안정 버전을 릴리스합니다. 교육 목적이라면 안정 버전을 사용하세요.

### 5.2 특정 태그 확인

```bash
# 사용 가능한 태그 확인
git tag | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$'

# 특정 버전 체크아웃
git checkout v23.05.5
```

### 5.3 소스 트리 구조 확인

```bash
cd ~/openwrt/openwrt-23.05

# Makefile 확인
head -10 Makefile
# 출력 예시:
# VERSION:=$(strip $(subst ",, \
#   $(shell git log --oneline -1 2>/dev/null | grep -o '[0-9]\+\.[0-9]\+\.[0-9]' 2>/dev/null)))
# ...

# 주요 디렉토리 구조
ls -la
```

---

## 6. 빌드 환경 설정 (feeds & menuconfig)

### 6.1 Feeds 업데이트 및 설치

Feeds는 OpenWrt의 추가 패키지 저장소입니다:

```bash
cd ~/openwrt/openwrt-23.05

# 기본 피드 설정 확인
cat feeds.conf.default
# 출력:
# src-git packages https://git.openwrt.org/feed/packages.git^...
# src-git luci https://git.openwrt.org/project/luci.git^...
# src-git routing https://git.openwrt.org/feed/routing.git^...
# src-git telephony https://git.openwrt.org/feed/telephony.git^...

# feeds 업데이트 (원격 저장소에서 패키지 소스 다운로드)
./scripts/feeds update -a

# feeds 설치 (Makefile 생성)
./scripts/feeds install -a

# 특정 패키지만 설치할 경우:
# ./scripts/feeds install luci luci-base luci-app-firewall luci-proto-ipv6
```

### 6.2 타겟 설정 (make menuconfig)

```bash
cd ~/openwrt/openwrt-23.05

# 설정 진입
make menuconfig
```

**라즈베리 파이 4B 기준 설정값:**

| 메뉴 | 선택값 | 설명 |
|---|---|---|
| **Target System** | `Broadcom BCM27xx` | BCM27xx SoC 계열 |
| **Subtarget** | `BCM2711 boards` | RPi 4B 전용 |
| **Target Profile** | `Raspberry Pi 4B` | 정확한 보드 선택 |
| **Target Images** | `squashfs + ext4` | 기본 이미지 형식 |
| **LuCI → Collections** | `luci` ✅ | 웹 UI 포함 |
| **LuCI → Modules** | `luci-mod-admin-full` ✅ | 관리 모듈 |
| **LuCI → Themes** | `luci-theme-bootstrap` ✅ | 테마 |

> **핵심 개념**: Target Profile을 정확히 선택해야 올바른 커널 모듈, DT(Device Tree)가 포함됩니다.

### 6.3 필수 선택 옵션 상세

```bash
# Target Images 설정 예시
Target Images --->
  [*] squashfs          # 압축된 읽기전용 루트파일시스템 (기본)
  [*] ext4              # 쓰기 가능한 ext4 파일시스템 (SD 카드)
  [*] GZip images       # 이미지 압축

# LuCI 웹 인터페이스 (필수)
LuCI --->
  Collections --->
    <*> luci            # LuCI 메타 패키지 (전체 설치)
  Modules --->
    <*> luci-mod-admin-full  # 전체 관리 모듈
    <*> luci-mod-status      # 상태 정보
    <*> luci-mod-system      # 시스템 설정
  Themes --->
    <*> luci-theme-bootstrap # 기본 테마
  Applications --->
    <*> luci-app-opkg        # 패키지 관리 UI

# 기본 네트워크 패키지
Network --->
  [*] WirelessAPD
  [*] hostapd-wolfssl   # Wi-Fi AP 모드 지원

# 기본 유틸리티
Utilities --->
  Editors --->
    <*> nano            # 텍스트 에디터
  Shells --->
    <*> bash            # bash 셸
```

### 6.4 커널 모듈 선택

네트워크 기능에 필요한 커널 모듈을 추가할 수 있습니다:

```
Kernel modules --->
  Network Support --->
    <*> kmod-ipt-nat                    # NAT (MASQUERADE)
    <*> kmod-ipt-conntrack              # 연결 추적
    <*> kmod-nft-core                   # nftables 코어
    <*> kmod-nft-nat                    # nftables NAT
    <*> kmod-usb-net                    # USB 이더넷 지원
    <*> kmod-usb-net-asix              # ASIX USB 이더넷
    <*> kmod-usb-net-rtl8152           # Realtek RTL8152/8153
    <*> kmod-brcmfmac                  # BCM Wi-Fi (RPi 내장)
    <*> kmod-brcmutil                  # Broadcom 유틸리티
  USB Support --->
    <*> kmod-usb-core                   # USB 코어
    <*> kmod-usb-ohci                   # OHCI (USB 1.1)
    <*> kmod-usb-uhci                   # UHCI (USB 1.1)
    <*> kmod-usb2                       # USB 2.0
    <*> kmod-usb3                       # USB 3.0 (RPi 4)
    <*> kmod-usb-storage                # USB 저장장치
  Filesystems --->
    <*> kmod-fs-ext4                   # ext4 파일시스템
    <*> kmod-fs-vfat                   # vfat 파일시스템
```

### 6.5 .config 파일 직접 편집

menuconfig 외에도 `.config` 파일을 직접 편집할 수 있습니다:

```bash
# .config 파일의 주요 항목 예시
cat .config | grep -E "^CONFIG_TARGET|^CONFIG_PACKAGE"
```

---

## 7. 펌웨어 컴파일

### 7.1 기본 컴파일

```bash
cd ~/openwrt/openwrt-23.05

# 첫 빌드: 모든 도구 및 툴체인도 함께 빌드
# -j$(nproc) : CPU 코어 수만큼 병렬 빌드
# V=s         : 자세한 로그 출력 (오류 추적 용이)
make -j$(nproc) V=s
```

> **첫 빌드 소요 시간**: 약 30분~2시간 (인터넷 속도, CPU 성능에 따라 다름)
> - RPi 4용 첫 빌드: 보통 60~90분
> - 이후 증분 빌드: 5~15분

### 7.2 빌드 단계별 실행

첫 빌드 이후 특정 단계만 다시 빌드할 수 있습니다:

```bash
# 모든 단계 재실행
make -j$(nproc) V=s

# 툴체인만 재빌드 (거의 안 함)
make toolchain/install V=s

# 커널만 재빌드
make target/linux/compile V=s

# 특정 패키지만 빌드
make package/luci/base/compile V=s

# 이미지만 다시 생성 (패키지 빌드 생략)
make target/linux/install V=s

# 전체 클린 (build_dir 삭제, 소스는 유지)
make clean

# 완전 클린 (dl 디렉토리까지 삭제)
make dirclean

# 설정까지 초기화
make distclean
```

### 7.3 빌드 결과 확인

```bash
# 빌드 완료 후 생성된 이미지 확인
ls -la bin/targets/bcm27xx/bcm2711/

# 예시 출력:
# openwrt-23.05.5-bcm27xx-bcm2711-rpi-4-ext4-factory.img.gz
# openwrt-23.05.5-bcm27xx-bcm2711-rpi-4-ext4-sysupgrade.img.gz
# openwrt-23.05.5-bcm27xx-bcm2711-rpi-4-squashfs-factory.img.gz
# openwrt-23.05.5-bcm27xx-bcm2711-rpi-4-squashfs-sysupgrade.img.gz
# openwrt-23.05.5-bcm27xx-bcm2711-rpi-4-kernel.bin
# openwrt-23.05.5-bcm27xx-bcm2711-rpi-4-rootfs.tar.gz
# config.buildinfo
# sha256sums
```

### 7.4 이미지 파일 종류 이해

| 파일명 | 용도 | 설명 |
|---|---|---|
| `*factory.img.gz` | **초기 설치용** | SD 카드에 처음 쓸 때 사용 |
| `*sysupgrade.img.gz` | **업그레이드용** | 기존 OpenWrt에서 업그레이드 시 사용 |
| `*-squashfs-*` | squashfs (기본) | 읽기전용 압축 루트파일시스템 + overlay |
| `*-ext4-*` | ext4 | 일반 리눅스 방식의 읽기/쓰기 파일시스템 |
| `*kernel.bin` | 커널 단독 | 커널 이미지만 (디버깅용) |
| `*rootfs.tar.gz` | 루트FS 아카이브 | 루트 파일시스템만 (디버깅용) |

> **추천**: 처음은 `squashfs-factory` 이미지를 사용하세요. squashfs + overlayfs 조합으로 설정 초기화가 쉽습니다.

---

## 8. SD 카드에 이미지 쓰기

### 8.1 이미지 준비

```bash
# 이미지 압축 해제
cd ~/openwrt/openwrt-23.05/bin/targets/bcm27xx/bcm2711/
gunzip -k openwrt-23.05.5-bcm27xx-bcm2711-rpi-4-squashfs-factory.img.gz

# 파일 확인
ls -lh *.img
```

### 8.2 Linux (또는 WSL2)에서 SD 카드 쓰기

> **경고**: 반드시 올바른 디바이스를 선택하세요. 잘못 지정하면 시스템 디스크가 손상됩니다!

```bash
# SD 카드 디바이스 확인 (연결 후)
lsblk | grep -E "sd[b-z]|mmc"

# 예: /dev/sdb 또는 /dev/mmcblk0

# 이미지 쓰기 (sudo 필요)
sudo dd if=openwrt-23.05.5-bcm27xx-bcm2711-rpi-4-squashfs-factory.img \
  of=/dev/sdb bs=4M status=progress conv=fsync

# 또는 Balena Etcher CLI 사용
# sudo apt install -y balena-etcher-electron
# balena-etcher-electron openwrt-*.img
```

### 8.3 Windows에서 SD 카드 쓰기

```powershell
# Windows에서는 Win32DiskImager 또는 Rufus 사용
# WSL2에서는 Windows 디스크를 직접 마운트할 수 없으므로
# 다음 도구 중 하나를 사용하세요:

# 옵션 1: Balena Etcher (GUI, 추천)
# https://www.balena.io/etcher/ 에서 다운로드

# 옵션 2: Win32 Disk Imager
# https://sourceforge.net/projects/win32diskimager/

# 옵션 3: dd for Windows
# https://github.com/slavoutich/dd for Windows
```

### 8.4 macOS에서 SD 카드 쓰기

```bash
# SD 카드 확인
diskutil list

# 언마운트 (예: /dev/disk2)
diskutil unmountDisk /dev/disk2

# 이미지 쓰기
sudo dd if=openwrt-*.img of=/dev/rdisk2 bs=4M status=progress conv=fsync

# 완료 후
diskutil eject /dev/disk2
```

### 8.5 ext4 이미지 크기 조정 (선택사항)

ext4 이미지를 사용한 경우, SD 카드 전체 용량을 사용하려면 부팅 후 파티션을 확장해야 합니다:

```bash
# RPi 부팅 후 SSH 접속
ssh root@192.168.1.1

# 루트 파티션 확인
block info

# 파티션 확장 (RPi 4는 /dev/mmcblk0p2)
parted /dev/mmcblk0 resizepart 2 100%

# 파일시스템 확장
resize2fs /dev/mmcblk0p2

# 확인
df -h
```

---

## 9. 부팅 및 초기 설정

### 9.1 부팅 준비

1. OpenWrt 이미지를 쓴 SD 카드를 RPi에 삽입
2. 이더넷 케이블 연결 (LAN 포트 → PC)
3. USB-C 전원 연결
4. 초록색 LED가 깜빡이면 부팅 중

### 9.2 초기 접속

```bash
# RPi 기본 IP: 192.168.1.1 (고정)
# PC의 IP를 192.168.1.x 대역으로 설정

# SSH 접속 (password: 비밀번호 없음 → 처음 설정 시 입력)
ssh root@192.168.1.1

# 처음 접속 시 암호 설정 프롬프트가 나타남 (또는 직접 설정)
passwd
```

### 9.3 웹 인터페이스 접속 (LuCI)

브라우저에서 `http://192.168.1.1` 접속 → LuCI 웹 UI 확인

### 9.4 Wi-Fi 설정

라즈베리 파이 4의 내장 Wi-Fi를 AP 모드로 설정:

```bash
# SSH 접속
ssh root@192.168.1.1

# 무선 인터페이스 확인
iw dev

# Wi-Fi 설정 (OpenWrt 23.05 기준)
uci set wireless.@wifi-device[0].disabled='0'
uci set wireless.@wifi-iface[0].mode='ap'
uci set wireless.@wifi-iface[0].ssid='OpenWrt-RPi'
uci set wireless.@wifi-iface[0].encryption='psk2'
uci set wireless.@wifi-iface[0].key='YourPassword123'

uci commit wireless
wifi reload
```

### 9.5 WAN 포트 설정 (USB 이더넷)

USB 이더넷 어댑터를 WAN 포트로 설정:

```bash
# USB 이더넷 어댑터 연결 후 인터페이스 확인
ip link

# 새 인터페이스 확인 (eth1 또는 usb0 등)
# WAN 인터페이스 설정
uci set network.wan=interface
uci set network.wan.proto='dhcp'   # 또는 'pppoe' (ISP가 PPPoE인 경우)
uci set network.wan.device='eth1'  # USB 이더넷 어댑터

uci commit network
/etc/init.d/network reload
```

### 9.6 기본 방화벽 설정 확인

```bash
# 방화벽 설정 확인
uci show firewall

# LAN → WAN 트래픽 허용 (기본값)
uci show firewall.@zone[1]

# SSH 접속 확인 (WAN에서 SSH 차단은 기본)
uci show firewall.@rule[0]
```

---

## 10. 소스 코드 수정 및 재빌드 실습

### 10.1 배너 메시지 변경 실습

OpenWrt 부팅 시 표시되는 배너 메시지를 변경해봅니다:

```bash
# 호스트 PC에서
cd ~/openwrt/openwrt-23.05

# 배너 파일 위치 확인
find package/base-files -name "banner"

# 파일 내용 보기
cat package/base-files/files/etc/banner
# 출력 예시:
#  _______                     ________        __
# |       |.-----.-----.-----.|  |  |  |.----.|  |_
# |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
# |_______||   __|_____|__|__||________||__|  |____|
#          |__| W I R E L E S S   F R E E D O M
# -----------------------------------------------------
#  %D %V %C
# -----------------------------------------------------
```

### 10.2 배너 파일 수정

```bash
# 배너 파일 편집
nano package/base-files/files/etc/banner
```

다음과 같이 수정합니다:

```
  __          __   _               _____       _    ___
  \ \        / /  | |             |  __ \     | |  |__ \
   \ \  /\  / /_ _| |_ __ ___   __| |__) |_ _ | |_   ) |
    \ \/  \/ / _` | | '__/ _ \ / _|  ___/ _` | __| / /
     \  /\  / (_| | | | | (_) | (_| |  | (_| | |_ / /_
      \/  \/ \__,_|_|_|  \___/ \__|_|   \__,_|\__|____|

   ___  _ __   ___ _ __   ___ _ __ ___   __ _ _ __
  / _ \| '_ \ / _ \ '_ \ / _ \ '_ ` _ \ / _` | '_ \
 | (_) | | | |  __/ | | |  __/ | | | | | (_| | | | |
  \___/|_| |_|\___|_| |_|\___|_| |_| |_|\__,_|_| |_|

  My Custom OpenWrt Build for Raspberry Pi 4
  Version: %D %V
  Build Date: $(date)
```

### 10.3 루트 패스워드 기본값 설정

```bash
# shadow 파일 수정 (기본 패스워드 해시 설정)
cat package/base-files/files/etc/shadow
# root::10933:0:99999:7:::
# → root:!:10933:0:99999:7:::  ('!'는 잠김, 비워두면 패스워드 없음)
```

### 10.4 기본 네트워크 설정 변경

기본 IP 주소를 변경하고 싶다면:

```bash
# 네트워크 설정 파일
cat package/base-files/files/bin/config_generate

# 이 파일이 첫 부팅 시 /etc/config/network를 생성
# 192.168.1.1 대신 다른 IP로 수정 가능
```

### 10.5 수정 후 재빌드

```bash
# 수정한 패키지만 다시 빌드
make package/base-files/compile V=s

# 이미지 다시 생성
make -j$(nproc) V=s

# 또는 전체 재빌드 (의존성 문제가 있다면)
# make clean && make -j$(nproc) V=s
```

### 10.6 기존 설정 유지하며 업그레이드 (sysupgrade)

```bash
# 빌드된 sysupgrade 이미지 확인
ls -lh bin/targets/bcm27xx/bcm2711/*sysupgrade*

# RPi에서 실행 (SSH 접속 후)
scp bin/targets/bcm27xx/bcm2711/openwrt-*-rpi-4-squashfs-sysupgrade.img.gz \
  root@192.168.1.1:/tmp/

ssh root@192.168.1.1
sysupgrade -v /tmp/openwrt-*-rpi-4-squashfs-sysupgrade.img.gz
```

> **참고**: `sysupgrade -n` 옵션을 추가하면 설정을 유지하지 않고 초기화합니다.

---

## 11. 커스텀 패키지 만들기

### 11.1 패키지 구조 이해

OpenWrt 패키지는 Makefile 기반의 빌드 정의로 구성됩니다:

```
package/my-package/
├── Makefile              # 패키지 빌드 정의 (핵심)
├── src/                  # (선택) C 소스 코드
│   └── my-program.c
└── files/                # (선택) 설정 파일 등
    └── etc/
        └── config/
            └── my-package
```

### 11.2 간단한 셸 스크립트 패키지 예제

```bash
cd ~/openwrt/openwrt-23.05

# 패키지 디렉토리 생성
mkdir -p package/hello-world/files
```

**`package/hello-world/Makefile`:**

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=hello-world
PKG_VERSION:=1.0
PKG_RELEASE:=1

PKG_MAINTAINER:=Your Name <your@email.com>
PKG_LICENSE:=GPL-2.0

include $(INCLUDE_DIR)/package.mk

define Package/hello-world
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=Hello World - My first OpenWrt package
  DEPENDS:=
  PKGARCH:=all
endef

define Package/hello-world/description
  A simple Hello World package for OpenWrt
  This is an educational example.
endef

define Build/Prepare
endef

define Build/Configure
endef

define Build/Compile
endef

define Package/hello-world/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) ./files/hello-world.sh $(1)/usr/bin/hello-world
endef

$(eval $(call BuildPackage,hello-world))
```

**`package/hello-world/files/hello-world.sh`:**

```bash
#!/bin/sh

echo "==================================="
echo "  Hello from OpenWrt!"
echo "  This is my custom package!"
echo "==================================="
echo "  System Information:"
echo "  Hostname: $(uci get system.@system[0].hostname)"
echo "  Kernel: $(uname -a)"
echo "  Uptime: $(uptime)"
echo "  Memory: $(free -m | grep Mem | awk '{print $3 " / " $2 " MB"}')"
echo "==================================="
```

실행 권한 부여:

```bash
chmod +x package/hello-world/files/hello-world.sh
```

### 11.3 패키지 메뉴에 등록하고 빌드

```bash
# menuconfig에서 확인
make menuconfig
# Utilities → hello-world 찾아서 선택 (M=모듈, *=내장)

# 패키지 빌드
make package/hello-world/compile V=s

# 빌드된 IPK 확인
ls -lh bin/packages/aarch64_cortex-a72/base/hello-world*.ipk
```

### 11.4 C 언어 기반 패키지 예제

```bash
mkdir -p package/hello-c/src
```

**`package/hello-c/Makefile`:**

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=hello-c
PKG_VERSION:=1.0
PKG_RELEASE:=1

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk

define Package/hello-c
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=Hello World C program
  DEPENDS:=
endef

define Package/hello-c/description
  A Hello World program written in C for OpenWrt
endef

define Build/Prepare
  mkdir -p $(PKG_BUILD_DIR)
  $(CP) ./src/* $(PKG_BUILD_DIR)/
endef

define Build/Configure
endef

define Build/Compile
  $(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/hello-c \
    $(PKG_BUILD_DIR)/main.c
endef

define Package/hello-c/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) $(PKG_BUILD_DIR)/hello-c $(1)/usr/bin/
endef

$(eval $(call BuildPackage,hello-c))
```

**`package/hello-c/src/main.c`:**

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/sysinfo.h>

int main() {
    struct sysinfo info;
    sysinfo(&info);

    printf("========================================\n");
    printf("  Hello from OpenWrt C Package!\n");
    printf("========================================\n");
    printf("  Uptime: %ld days %ld hours %ld mins\n",
        info.uptime / 86400,
        (info.uptime % 86400) / 3600,
        (info.uptime % 3600) / 60);
    printf("  Total RAM: %ld MB\n", info.totalram / (1024 * 1024));
    printf("  Free RAM:  %ld MB\n", info.freeram / (1024 * 1024));
    printf("  Processes: %d\n", info.procs);
    printf("========================================\n");

    return 0;
}
```

### 11.5 패키지의존성 설정

```makefile
# 다른 패키지에 의존하는 예제
define Package/my-app
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=My application
  DEPENDS:=+libcurl +libubus +libblobmsg-json
  # +libcurl : libcurl에 의존 (필수)
  # @BUSYBOX_CONFIG_DHCP : busybox의 특정 기능 필요
endef
```

### 11.6 빌드된 IPK를 RPi에 설치

```bash
# 호스트에서 RPi로 전송
scp bin/packages/aarch64_cortex-a72/base/hello-world*.ipk \
  root@192.168.1.1:/tmp/

# RPi에서 직접 설치
ssh root@192.168.1.1
opkg install /tmp/hello-world*.ipk

# 실행
hello-world

# 또는 LuCI 웹 UI → System → Software 에서 업로드 설치 가능
```

---

## 12. 펌웨어 업데이트 (sysupgrade)

### 12.1 설정을 유지하는 업그레이드

```bash
# 현재 설정 백업 (선택사항)
ssh root@192.168.1.1
sysupgrade -b /tmp/backup.tar.gz
scp root@192.168.1.1:/tmp/backup.tar.gz ./

# sysupgrade 이미지 전송
scp bin/targets/bcm27xx/bcm2711/openwrt-*-rpi-4-squashfs-sysupgrade.img.gz \
  root@192.168.1.1:/tmp/

# 설정 유지 업그레이드
ssh root@192.168.1.1
sysupgrade -v /tmp/openwrt-*-rpi-4-squashfs-sysupgrade.img.gz
```

### 12.2 설정 초기화 업그레이드

```bash
# 설정을 완전 초기화하고 업그레이드
ssh root@192.168.1.1
sysupgrade -v -n /tmp/openwrt-*-rpi-4-squashfs-sysupgrade.img.gz
```

### 12.3 설정만 백업/복원

```bash
# 백업
ssh root@192.168.1.1
sysupgrade -b /tmp/backup.tar.gz

# 복원
sysupgrade -r /tmp/backup.tar.gz
```

---

## 13. 문제 해결 (Troubleshooting)

### 13.1 빌드 오류

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| `make: command not found` | build-essential 미설치 | `sudo apt install build-essential` |
| `Kernel panic - not syncing` | 잘못된 SD 카드 쓰기 | SD 카드 다시 쓰기 (dd 재실행) |
| `No space left on device` | 디스크 부족 | `make clean` 후 재빌드 |
| `download failed` | 네트워크 문제 | `make dirclean` 후 재시도 |
| `Package foo is missing` | feeds 미설치 | `./scripts/feeds install foo` |
| `build_dir/.../compile failed` | 커널 모듈 오류 | `make target/linux/clean` 후 재빌드 |

### 13.2 첫 빌드 실패 시 빠른 해결

```bash
# 가장 흔한 문제: 패키지 다운로드 실패
# dl/ 디렉토리를 확인하고 재시도
make dirclean
./scripts/feeds update -a
./scripts/feeds install -a
make -j$(nproc) V=s

# 특정 패키지 빌드 실패 시, 해당 패키지만 제외하고 진행
# menuconfig에서 해당 패키지 선택 해제 후 재시작
```

### 13.3 부팅 문제

| 증상 | 원인 및 해결 |
|---|---|
| 빨간 LED만 켜짐 | SD 카드 인식 불가 → 이미지 다시 쓰기 |
| 초록 LED 깜빡임 후 꺼짐 | 커널 패닉 → 시리얼 콘솔 로그 확인 |
| IP 할당 안 됨 | DHCP 서버 문제 → PC를 192.168.1.x로 고정 설정 |
| SSH 접속 거부 | 드롭베어 미설치 또는 방화벽 → 시리얼 콘솔로 확인 |

### 13.4 시리얼 콘솔 사용 (GPIO)

라즈베리 파이의 UART 핀을 통해 시리얼 콘솔로 접속할 수 있습니다:

```
RPi GPIO Header:
Pin 6 (GND)  ─── GND
Pin 8 (TX)   ─── USB-UART RX
Pin 10 (RX)  ─── USB-UART TX

설정: 115200 baud, 8N1
```

```bash
# 호스트 PC에서 시리얼 접속
sudo screen /dev/ttyUSB0 115200
# 또는
sudo minicom -D /dev/ttyUSB0 -b 115200
```

### 13.5 Wi-Fi 문제

```bash
# Wi-Fi 칩셋 인식 확인
lspci | grep -i broadcom
lsmod | grep brcm

# Wi-Fi 상태 확인
iwinfo
ubus call iwinfo scan '{"device":"wlan0"}'

# 채널 문제 (규제 도메인)
uci set wireless.radio0.country='US'
uci commit wireless
wifi reload
```

---

## 14. 참고 자료

### 14.1 공식 문서

| 자료 | URL |
|---|---|
| OpenWrt 공식 문서 | https://openwrt.org/docs/start |
| 빌드 시스템 가이드 | https://openwrt.org/docs/guide-developer/build-system/start |
| 라즈베리 파이 지원 | https://openwrt.org/toh/raspberry_pi_foundation/raspberry_pi |
| 패키지 작성 가이드 | https://openwrt.org/docs/guide-developer/package-development |
| Feeds 소스 | https://github.com/openwrt/ |
| 커스텀 펌웨어 빌더 | https://firmware-selector.openwrt.org/ |

### 14.2 커뮤니티

- **포럼**: https://forum.openwrt.org/
- **Slack**: https://openwrt-slack.herokuapp.com/
- **IRC**: #openwrt on irc.libera.chat
- **GitHub**: https://github.com/openwrt/openwrt

### 14.3 유용한 명령어 모음

```bash
# 개발 환경 정보 확인
uname -a
gcc --version
python3 --version

# OpenWrt 소스 정보
cd ~/openwrt/openwrt-23.05
git log --oneline -5
git describe --tags

# 빌드 설정 확인
grep CONFIG_TARGET_BOARD .config
grep CONFIG_TARGET_SUBTARGET .config
grep CONFIG_TARGET_PROFILE .config

# 빌드 시간 단축 팁
# 1차 빌드 후 .config에서 불필요한 패키지 제거
# feeds에서 필요한 패키지만 설치
./scripts/feeds install luci base-files busybox dnsmasq dropbear

# ccache 사용 (재빌드 속도 향상)
sudo apt install ccache
export CCACHE_DIR=~/.ccache
make -j$(nproc) CCACHE=y V=s
```

### 14.4 교육 실습 커리큘럼 (제안)

| 단계 | 내용 | 예상 시간 |
|---|---|---|
| 1단계 | 개발 환경 구축 및 소스 다운로드 | 30분 |
| 2단계 | menuconfig 설정 및 첫 빌드 | 2시간 |
| 3단계 | SD 카드 쓰기 및 부팅 | 30분 |
| 4단계 | 초기 설정 (Wi-Fi, WAN, LuCI) | 30분 |
| 5단계 | 배너/설정 파일 수정 및 재빌드 | 1시간 |
| 6단계 | sysupgrade로 업데이트 | 30분 |
| 7단계 | hello-world 패키지 제작 | 1시간 |
| 8단계 | C 패키지 제작 및 크로스 컴파일 | 2시간 |

---

> **다음 학습 주제**: ipTIME 공유기 OpenWrt 포팅 가이드를 참고하세요 (`C:\OpenWrt\Iptime\Iptime_OpenWrt_개발_가이드.md`)
