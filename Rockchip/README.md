# OpenWrt on Rockchip SBC — 개발부터 이식까지 완전 정복 가이드

> **목표**: Rockchip SoC 기반 싱글보드 컴퓨터(SBC)를 타겟으로 OpenWrt 펌웨어를 개발 환경 구축부터 소스 수정, 컴파일, 부팅, 그리고 자체 패키지 제작까지 전 과정을 학습한다.

---

## 목차

1. [Rockchip & OpenWrt 개요](#1-rockchip--openwrt-개요)
2. [지원 하드웨어](#2-지원-하드웨어)
3. [개발 환경 구축](#3-개발-환경-구축)
4. [소스 코드 다운로드 및 빌드 설정](#4-소스-코드-다운로드-및-빌드-설정)
5. [펌웨어 컴파일](#5-펌웨어-컴파일)
6. [SD 카드/eMMC에 이미지 쓰기](#6-sd-카드emmc에-이미지-쓰기)
7. [부팅 및 초기 설정](#7-부팅-및-초기-설정)
8. [Rockchip 특화 기능](#8-rockchip-특화-기능)
9. [소스 코드 수정 및 재빌드 실습](#9-소스-코드-수정-및-재빌드-실습)
10. [U-Boot 및 파티션 이해](#10-u-boot-및-파티션-이해)
11. [커스텀 패키지 만들기](#11-커스텀-패키지-만들기)
12. [문제 해결](#12-문제-해결)
13. [참고 자료](#13-참고-자료)

---

## 1. Rockchip & OpenWrt 개요

### 1.1 Rockchip이란?

Rockchip은 **중국 팹리스 반도체 기업**으로, ARM 기반 SoC를 설계합니다. 저렴한 가격 대비 높은 성능으로 싱글보드 컴퓨터(SBC) 시장에서 Raspberry Pi의 대안으로 급부상했습니다.

**OpenWrt에서 Rockchip의 위상:**

- **OpenWrt 21.02+** 부터 공식 `rockchip` 타겟 추가
- `armv8` (aarch64) 서브타겟으로 분류
- Linux 커널 6.1/6.6 LTS 완벽 지원
- 활발한 커뮤니티 포팅 (특히 FriendlyElec/NanoPi)

### 1.2 Raspberry Pi vs Rockchip SBC 비교

| 항목 | Raspberry Pi 4 | Orange Pi 5 | NanoPi R5C |
|---|---|---|---|
| **SoC** | BCM2711 | **RK3588S** | **RK3568** |
| **CPU** | 4x Cortex-A72 @ 1.8GHz | 4x A76 + 4x A55 @ 2.4GHz | 4x A55 @ 2.0GHz |
| **GPU** | VideoCore VI | Mali-G610 MP4 | Mali-G52 |
| **NPU** | 없음 | **6 TOPS NPU** | 0.8 TOPS NPU |
| **RAM** | 4GB LPDDR4 | 8/16GB LPDDR4X | 1/2/4GB LPDDR4 |
| **NIC** | 1x GbE | 1x GbE (RTL8211F) | **2x 2.5GbE (RTL8125)** |
| **PCIe** | PCIe 2.0 x1 | PCIe 3.0 x4 | PCIe 3.0 x1 |
| **USB** | USB 3.0 x2 | USB 3.0 x2 | USB 3.0 x1 |
| **SATA** | 없음 | 없음 | 없음 |
| **OpenWrt 지원** | ✅ 공식 | ✅ 공식 (linux 6.6+) | ✅ 공식 |
| **가격** | ~5만원 | ~8만원 | ~6만원 |

### 1.3 Rockchip SoC 라인업

| SoC | 공정 | CPU | 용도 | 대표 보드 |
|---|---|---|---|---|
| **RK3328** | 28nm | 4x A53 @ 1.5GHz | 저가형 | NanoPi R2S, Orange Pi R1 Plus |
| **RK3399** | 28nm | 2x A72 + 4x A53 @ 2.0GHz | 고성능 | Orange Pi 4, NanoPi M4 |
| **RK3568** | 22nm | 4x A55 @ 2.0GHz | 라우터 특화 | **NanoPi R5C/R5S**, Orange Pi 3B |
| **RK3588/S** | 8nm | 4x A76 + 4x A55 @ 2.4GHz | 플래그십 | **Orange Pi 5/5B/5 Plus**, NanoPi R6C/R6S |

> **OpenWrt 라우터용 최적**: RK3568 (NanoPi R5C/R5S) — 전력 효율 + 듀얼 2.5GbE + PCIe 확장
> **범용 고성능**: RK3588S (Orange Pi 5) — 8코어 + NPU + USB3 + PCIe3.0

---

## 2. 지원 하드웨어

### 2.1 OpenWrt 공식 지원 Rockchip 보드

| 보드 | SoC | RAM | NIC | 저장소 | OpenWrt 버전 |
|---|---|---|---|---|---|
| **NanoPi R2S** | RK3328 | 1GB | 2x GbE | MicroSD | 21.02+ |
| **NanoPi R4S** | RK3399 | 4GB | 2x GbE | MicroSD+eMMC | 21.02+ |
| **NanoPi R5C** | **RK3568** | 4GB | **2x 2.5GbE** | MicroSD+eMMC+M.2 | 23.05+ |
| **NanoPi R5S** | RK3568 | 4GB | 3x GbE | eMMC+M.2 | 23.05+ |
| **NanoPi R6C** | RK3588S | 8GB | 2x 2.5GbE | MicroSD+eMMC+M.2 | **SNAPSHOT** |
| **NanoPi R6S** | RK3588S | 8GB | 3x 2.5GbE | eMMC+M.2 | **SNAPSHOT** |
| **Orange Pi 5** | **RK3588S** | 8/16GB | 1x GbE | MicroSD+eMMC+M.2 | 23.05+ |
| **Orange Pi 5 Plus** | RK3588 | 8/16GB | 2x GbE | MicroSD+eMMC+M.2 | **SNAPSHOT** |
| **Orange Pi 3B** | RK3566 | 4/8GB | 1x GbE | MicroSD+eMMC | 23.05+ |
| **FriendlyElec NanoPC T4** | RK3399 | 4GB | 1x GbE | eMMC+SD | 21.02+ |

### 2.2 추천 보드 별 특징

#### 입문용: NanoPi R2S (약 3~4만원)

```
SoC: RK3328 (4x Cortex-A53 @ 1.5GHz)
RAM: 1GB DDR4
NIC: 2x GbE (WAN: RTL8153 USB, LAN: RTL8211E)
저장소: MicroSD
특징: 가장 저렴한 OpenWrt 전용 SBC
            초소형 (손바닥 절반)
            팬리스, 극저전력 (~5W)
성능: ~600Mbps NAT
```

#### 중급: NanoPi R5C (약 6~8만원) ⭐권장

```
SoC: RK3568 (4x Cortex-A55 @ 2.0GHz)
RAM: 4GB LPDDR4
NIC: 2x 2.5GbE (RTL8125BG)
저장소: MicroSD + 32GB eMMC + M.2 NVMe/SATA
확장: M.2 M-Key (NVMe/SATA), M.2 E-Key (Wi-Fi)
특징: 2.5GbE 듀얼 포트
            M.2 NVMe로 고속 저장
            PCIe Wi-Fi 카드 추가 가능
성능: ~2Gbps NAT (2.5GbE 풀 속도)
```

#### 고급: Orange Pi 5 (약 8~12만원)

```
SoC: RK3588S (4x A76 @ 2.4GHz + 4x A55 @ 1.8GHz)
RAM: 8/16GB LPDDR4X
NIC: 1x GbE (RTL8211F)
저장소: MicroSD + eMMC 모듈 + M.2 M-Key NVMe
확장: PCIe 3.0 x4, USB 3.0 x2, 40핀 GPIO
특징: 8코어 압도적 성능
            NPU 6 TOPS (AI 가속)
            8K 비디오 디코딩
            라우터 + NAS + AI 서버 올인원 가능
성능: ~3Gbps NAT (USB NIC 추가 시)
```

---

## 3. 개발 환경 구축

### 3.1 요구 사양

| 항목 | 최소 사양 | 권장 사양 |
|---|---|---|
| CPU | 2코어 | 4코어 이상 |
| RAM | 4GB | 16GB 이상 |
| 디스크 | 20GB 여유 | 100GB 이상 (SSD 권장) |
| OS | Ubuntu 22.04 / 24.04 LTS | 동일 |

### 3.2 패키지 설치

```bash
# Ubuntu 24.04 기준
sudo apt update
sudo apt install -y build-essential gcc g++ gawk git \
  libncurses5-dev libssl-dev python3 python3-distutils \
  python3-setuptools rsync unzip zlib1g-dev file wget curl \
  qemu-utils

# Rockchip 특화 도구
sudo apt install -y \
  device-tree-compiler \
  u-boot-tools \
  parted \
  dosfstools \
  mtools
```

### 3.3 크로스 컴파일 이해

Rockchip SoC는 **ARMv8 (aarch64)** 아키텍처이므로, x86_64 호스트에서 크로스 컴파일이 필요합니다:

```
호스트 PC (x86_64)                 Rockchip SBC (ARMv8)
┌─────────────────┐             ┌──────────────────┐
│  네이티브 gcc     │             │ CPU: ARM Cortex   │
│  (x86_64 용)     │             │      A55/A72/A76  │
│       ↓          │             │ 아키텍처: aarch64  │
│  aarch64-openwrt-│──→ .ipk ──→│                   │
│  linux-musl-gcc  │             │                   │
│  (ARM64 용 생성)  │             │                   │
└─────────────────┘             └──────────────────┘
```

> **팁**: ARM 크로스 컴파일은 MIPS보다는 빠르고 x86 네이티브보다는 느립니다.
> 첫 빌드: 약 40~80분 (CPU 성능에 따라 다름)

---

## 4. 소스 코드 다운로드 및 빌드 설정

### 4.1 소스 클론

```bash
cd ~/openwrt

# 안정 버전 (권장)
git clone https://git.openwrt.org/openwrt/openwrt.git openwrt-23.05
cd openwrt-23.05
git checkout v23.05.5

# 또는 최신 SNAPSHOT (RK3588 계열 필요 시)
# git clone https://git.openwrt.org/openwrt/openwrt.git openwrt-snapshot
# cd openwrt-snapshot

# feeds 설정
./scripts/feeds update -a
./scripts/feeds install -a
```

### 4.2 타겟 설정 (make menuconfig)

```bash
make menuconfig
```

#### NanoPi R5C (RK3568) 기준 설정:

| 메뉴 | 선택값 | 설명 |
|---|---|---|
| **Target System** | `Rockchip` | Rockchip SoC 계열 |
| **Subtarget** | `ARMv8 based boards` | 64비트 ARM |
| **Target Profile** | `FriendlyElec NanoPi R5C` | 정확한 보드 선택 |
| **Target Images** | `squashfs + ext4` | 이미지 형식 |

#### Orange Pi 5 (RK3588S) 기준 설정:

| 메뉴 | 선택값 | 설명 |
|---|---|---|
| **Target System** | `Rockchip` | Rockchip SoC 계열 |
| **Subtarget** | `ARMv8 based boards` | 64비트 ARM |
| **Target Profile** | `Orange Pi 5` | (23.05+ 에서 지원) |

> **참고**: RK3588 지원은 23.05.5부터 정식 포함되었습니다.
> 만약 Profile에 없다면 최신 SNAPSHOT을 사용하거나 직접 DTS 추가가 필요합니다.

### 4.3 Rockchip 필수 패키지 설정

```
Target Images --->
  [*] squashfs
  [*] ext4
  [*] GZip images

Firmware --->
  [*] brcmfmac-firmware-*    # Wi-Fi 펌웨어 (보드에 따라)
  [*] wireless-regdb          # 무선 규제 도메인

Kernel modules --->
  Network Drivers --->
    <*> kmod-r8169            # Realtek RTL8125 (2.5GbE) - R5C/R6C
    <*> kmod-stmmac-core      # Rockchip GMAC (내장 GbE)
    <*> kmod-stmmac-platform
  USB Support --->
    <*> kmod-usb-core
    <*> kmod-usb3
    <*> kmod-usb-storage
    <*> kmod-usb-net
  Filesystems --->
    <*> kmod-fs-ext4
    <*> kmod-fs-vfat
    <*> kmod-fs-btrfs          # BTRFS (M.2 SSD 사용 시)
  M.2/NVMe --->
    <*> kmod-nvme              # NVMe SSD 지원

LuCI --->
  Collections --->
    <*> luci
  Themes --->
    <*> luci-theme-bootstrap

Utilities --->
  <*> htop
```

### 4.4 Profile에 없는 보드 추가

만약 사용하는 보드가 menuconfig의 Profile 목록에 없다면:

```bash
# 1. DTS 파일 확인
ls target/linux/rockchip/dts/rk35*/rk3*.dts | grep -E "nanopi|orangepi"

# 2. Generic ARMv8 Profile 선택
# Target Profile → "Generic" 또는 "Default"

# 3. 직접 DTS를 지정하여 빌드
# target/linux/rockchip/image/armv8.mk 에서 DEVICE_DTS 값 확인 후
# menuconfig에서 "Custom Device Tree" 옵션 사용

# 4. 또는 U-Boot + DTB를 수동으로 추가
```

---

## 5. 펌웨어 컴파일

### 5.1 첫 빌드

```bash
cd ~/openwrt/openwrt-23.05

# 첫 빌드
make -j$(nproc) V=s
```

> **빌드 시간**:
> - 첫 빌드 (ARM64): 약 40~80분 (CPU 성능에 따라 다름)
> - 증분 빌드: 5~15분
> - 크로스 컴파일이므로 x86_64 타겟보다는 느림

### 5.2 빌드 결과 확인

```bash
ls -lh bin/targets/rockchip/armv8/

# 출력 예시:
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c-ext4-sysupgrade.img.gz
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c-squashfs-sysupgrade.img.gz
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c-ext4-factory.img.gz
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c-squashfs-factory.img.gz
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c-kernel.bin
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c-rootfs.tar.gz
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c.idbloader.img
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c.u-boot.img
# openwrt-23.05.5-rockchip-armv8-friendlyelec_nanopi-r5c.bootable-sd.img.gz  ← SD카드용!
# sha256sums
```

### 5.3 Rockchip 특수 이미지 이해

Rockchip은 Raspberry Pi와 달리 **독자적인 부트 체인**을 사용합니다:

| 파일명 | 설명 | 비고 |
|---|---|---|
| `*bootable-sd.img.gz` | **SD 부팅 통합 이미지** | SD 카드에 바로 쓰기만 하면 됨 |
| `*factory.img.gz` | eMMC 플래싱용 | 최초 설치용 |
| `*sysupgrade.img.gz` | OpenWrt 업그레이드용 | 기존 OpenWrt에서 sysupgrade |
| `*.idbloader.img` | 1단계 부트로더 | Rockchip 부트 체인 파트1 |
| `*.u-boot.img` | U-Boot 이미지 | Rockchip 부트 체인 파트2 |
| `*kernel.bin` | FIT 이미지 커널 | 커널 + DTB 통합 |

### 5.4 Rockchip 부트 체인 구조

Rockchip SoC의 부트 과정은 Raspberry Pi와 다릅니다:

```
전원 ON
  │
  ▼
ROM BootROM (칩 내장)
  │  SD 카드/eMMC/SPI 플래시 순서로 부트 시도
  │
  ▼
idbloader.img (DDR 초기화 + U-Boot SPL)
  │  - DRAM 초기화
  │  - 기본 하드웨어 설정
  │
  ▼
u-boot.img (U-Boot 메인)
  │  - 파티션 테이블 읽기
  │  - ext4/fat 파일시스템 지원
  │  - 커널 로드
  │
  ▼
boot.scr (U-Boot 스크립트)
  │  - FIT 이미지 (kernel + DTB) 선택
  │
  ▼
kernel + DTB (FIT 이미지)
  │
  ▼
OpenWrt 부팅
```

---

## 6. SD 카드/eMMC에 이미지 쓰기

### 6.1 SD 카드 준비 (bootable-sd 사용)

가장 간단한 방법입니다:

```bash
# Linux (또는 WSL2)
cd ~/openwrt/openwrt-23.05/bin/targets/rockchip/armv8/

# SD 카드 확인
lsblk | grep -E "sd[b-z]|mmc"

# bootable-sd 이미지 사용 (권장)
# 이 이미지 하나에 부트로더 + 파티션 + 커널 + 루트FS 모두 포함
gunzip -k openwrt-*-nanopi-r5c-squashfs-bootable-sd.img.gz

# SD 카드 쓰기
sudo dd if=openwrt-*-nanopi-r5c-squashfs-bootable-sd.img \
  of=/dev/sdb bs=4M status=progress conv=fsync
```

> **참고**: `bootable-sd`는 **Rockchip 전용** 이미지입니다. 부트로더(idbloader + U-Boot)가 이미지 시작 부분에 포함되어 있어, 일반 `dd`만으로도 부팅 가능한 SD 카드가 완성됩니다.

### 6.2 Windows에서 SD 카드 쓰기

```powershell
# Balena Etcher 사용 (권장)
# 1. https://www.balena.io/etcher/ 다운로드
# 2. bootable-sd.img.gz 파일 선택 (압축 상태 그대로 가능)
# 3. SD 카드 선택
# 4. Flash!

# 또는 Win32DiskImager
# https://sourceforge.net/projects/win32diskimager/
```

### 6.3 macOS에서 SD 카드 쓰기

```bash
diskutil list
diskutil unmountDisk /dev/disk2
gunzip -k openwrt-*-bootable-sd.img.gz
sudo dd if=openwrt-*-bootable-sd.img of=/dev/rdisk2 bs=4M status=progress conv=fsync
diskutil eject /dev/disk2
```

### 6.4 eMMC에 설치 (Rockchip 전용)

대부분의 Rockchip SBC는 eMMC가 내장되어 있습니다:

```bash
# 방법 1: SD 카드로 부팅 후 eMMC에 설치 (권장)

# 1. SD 카드로 OpenWrt 부팅
# 2. SSH 접속
ssh root@192.168.1.1

# 3. eMMC 확인
ls /dev/mmcblk*
# /dev/mmcblk0 = SD 카드
# /dev/mmcblk1 = eMMC (또는 mmcblk2)

# 4. sysupgrade 이미지를 eMMC에 적용
scp openwrt-*-nanopi-r5c-squashfs-sysupgrade.img.gz \
  root@192.168.1.1:/tmp/

ssh root@192.168.1.1
# eMMC에 직접 쓰기
gunzip -c /tmp/openwrt-*-squashfs-sysupgrade.img.gz | \
  dd of=/dev/mmcblk1 bs=4M status=progress

# 5. 전원 OFF → SD 카드 제거 → 전원 ON → eMMC 부팅

# 방법 2: factory.img를 eMMC에 직접 쓰기
gunzip -k openwrt-*-factory.img.gz
sudo dd if=openwrt-*-factory.img of=/dev/mmcblk1 bs=4M status=progress conv=fsync
```

### 6.5 M.2 NVMe SSD에 설치

NanoPi R5C/R6C, Orange Pi 5 등 M.2 슬롯이 있는 보드:

```bash
# SD 카드로 부팅 후
ssh root@192.168.1.1

# NVMe 디스크 확인
ls /dev/nvme*
# /dev/nvme0n1

# NVMe에 sysupgrade 이미지 쓰기
gunzip -c /tmp/openwrt-*-squashfs-sysupgrade.img.gz | \
  dd of=/dev/nvme0n1 bs=4M status=progress

# 재부팅
reboot
```

---

## 7. 부팅 및 초기 설정

### 7.1 첫 부팅

1. SD 카드 (또는 eMMC) 삽입
2. USB-C 전원 연결
3. LAN 포트에 PC 연결
4. PC IP를 자동(DHCP)으로 설정
5. SSH 접속: `ssh root@192.168.1.1`

### 7.2 NanoPi R5C 네트워크 인터페이스

```bash
# NanoPi R5C의 NIC 구성 확인
ip link

# 일반적인 구성:
# eth0: LAN (내장 GMAC, LAN 포트)
# eth1: WAN (2.5GbE, RTL8125, WAN 포트)

# MAC 주소 확인
ip link show eth0
ip link show eth1
```

### 7.3 2.5GbE 성능 확인

```bash
# R5C의 2.5GbE NIC 링크 속도 확인
ethtool eth1
# Speed: 2500Mb/s 확인

# iperf3 성능 테스트
# 서버 측 (OpenWrt)
iperf3 -s

# 클라이언트 측 (PC)
iperf3 -c 192.168.1.1
```

### 7.4 Wi-Fi 설정 (보드에 Wi-Fi 모듈이 있는 경우)

NanoPi R5C는 M.2 E-Key에 Wi-Fi 모듈을 추가할 수 있습니다:

```bash
# Wi-Fi 모듈 확인
lspci | grep -i network
lsusb | grep -i wireless

# MT7921K (MediaTek Wi-Fi 6E) 모듈 예시
# kmod-mt7921e 필요
opkg update
opkg install kmod-mt7921e

# Wi-Fi 설정
uci set wireless.radio0=wifi-device
uci set wireless.radio0.type='mac80211'
uci set wireless.radio0.channel='auto'
uci set wireless.radio0.country='KR'

uci set wireless.default_radio0=wifi-iface
uci set wireless.default_radio0.device='radio0'
uci set wireless.default_radio0.network='lan'
uci set wireless.default_radio0.mode='ap'
uci set wireless.default_radio0.ssid='OpenWrt-Rockchip'
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='YourPassword123'

uci commit wireless
wifi reload
```

### 7.5 성능 최적화 (Rockchip)

```bash
# CPU 성능 모드 설정 (성능 우선)
echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo performance > /sys/devices/system/cpu/cpu1/cpufreq/scaling_governor
echo performance > /sys/devices/system/cpu/cpu2/cpufreq/scaling_governor
echo performance > /sys/devices/system/cpu/cpu3/cpufreq/scaling_governor

# RK3588 (빅코어 4개 추가)
# echo performance > /sys/devices/system/cpu/cpu4/cpufreq/scaling_governor
# ...

# 네트워크 인터럽트 친화도 설정 (CPU 2번에 집중)
echo 2 > /proc/irq/$(grep eth1 /proc/interrupts | cut -d: -f1 | head -1 | tr -d ' ')/smp_affinity

# RPS (Receive Packet Steering) - 다중 코어에 부하 분산
echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus
echo f > /sys/class/net/eth1/queues/rx-0/rps_cpus
```

---

## 8. Rockchip 특화 기능

### 8.1 Hardware Watchdog

Rockchip SoC에는 하드웨어 워치독 타이머가 내장되어 있습니다:

```bash
# 워치독 확인
ls /dev/watchdog*

# OpenWrt에서 워치독 설정
uci set system.@system[0].watchdog='1'
uci commit system

# /etc/config/system:
# config system
#   option watchdog '1'
```

### 8.2 온도 모니터링

```bash
# Rockchip 내장 온도 센서
cat /sys/class/thermal/thermal_zone0/temp
# 55000 → 55.0°C

# 지속 모니터링
watch -n 2 'cat /sys/class/thermal/thermal_zone*/temp'

# CPU 온도에 따른 throttling 확인
cat /sys/class/thermal/thermal_zone0/trip_point_*_temp
```

### 8.3 GPIO 및 LED 제어

```bash
# Rockchip GPIO 제어 (sysfs)
ls /sys/class/gpio/
echo 92 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio92/direction
echo 1 > /sys/class/gpio/gpio92/value
echo 0 > /sys/class/gpio/gpio92/value

# NanoPi R5C LED 제어
ls /sys/class/leds/
# red:status, green:status 등
echo 1 > /sys/class/leds/red:status/brightness
echo 0 > /sys/class/leds/red:status/brightness

# heartbeat 트리거 설정
echo heartbeat > /sys/class/leds/green:status/trigger
```

### 8.4 NPU 활용 (RK3588S)

Orange Pi 5의 6 TOPS NPU를 활용한 AI 기능:

```bash
# NPU 드라이버 확인
ls /dev/dri/
# renderD128 (NPU)

# rknpu 드라이버 로드
lsmod | grep rknpu

# NPU 사용 패키지 (별도 컴파일 필요)
# rockchip-npu
# rknpu2
```

### 8.5 USB-C DisplayPort Alt Mode (Orange Pi 5)

```bash
# USB-C를 통한 디스플레이 출력 (콘솔/디버그)
# DRM 드라이버 확인
cat /sys/class/drm/*/status
```

---

## 9. 소스 코드 수정 및 재빌드 실습

### 9.1 LED 트리거 변경

NanoPi R5C의 LED 동작 방식을 변경합니다:

```bash
cd ~/openwrt/openwrt-23.05

# R5C DTS 파일 찾기
find target/linux/rockchip/dts -name "*nanopi*r5c*"
# target/linux/rockchip/dts/rk3568-nanopi-r5c.dts

# DTS 내용 확인
cat target/linux/rockchip/dts/rk3568-nanopi-r5c.dts
```

**DTS LED 섹션 수정:**

```dts
// target/linux/rockchip/dts/rk3568-nanopi-r5c.dts (LED 부분)

/ {
    leds {
        compatible = "gpio-leds";

        led_status_red: led-0 {
            label = "red:status";
            gpios = <&gpio0 RK_PB7 GPIO_ACTIVE_HIGH>;
            linux,default-trigger = "default-on";  // ← 수정: "heartbeat"로 변경
        };

        led_status_green: led-1 {
            label = "green:status";
            gpios = <&gpio0 RK_PB5 GPIO_ACTIVE_HIGH>;
            linux,default-trigger = "none";         // ← 수정: "netdev"로 변경 (네트워크 활동 표시)
        };
    };
};
```

### 9.2 2.5GbE LED 커스터마이징

```bash
# RTL8125 LED 설정 변경 (NanoPi R5C)
# 드라이버 파라미터로 LED 동작 제어 가능

# /etc/modules.d/r8169:
# options r8169 led_mode=1
# led_mode: 0=속도별, 1=활동별, 2=링크상태
```

### 9.3 MAC 주소 고정

```bash
# Rockchip 보드의 MAC 주소는 일반적으로 eMMC의 특정 오프셋에 저장
# OpenWrt에서 U-Boot 환경변수로 설정 가능

# /etc/config/network:
# config device
#     option name 'eth0'
#     option macaddr 'XX:XX:XX:XX:XX:XX'

# 또는 DTS에서 직접 설정:
# &gmac0 {
#     nvmem-cells = <&mac_address>;
#     nvmem-cell-names = "mac-address";
# };
```

### 9.4 재빌드

```bash
# DTS 수정 후
make target/linux/compile V=s
make -j$(nproc) V=s

# bootable-sd 이미지 재생성
ls -lh bin/targets/rockchip/armv8/*bootable*
```

---

## 10. U-Boot 및 파티션 이해

### 10.1 Rockchip 부트 미디어 파티션 구조

```
SD/eMMC 파티션 레이아웃 (bootable-sd 이미지):
┌─────────────────────────────────────┐
│ MBR / GPT 헤더                      │  0x00000000
├─────────────────────────────────────┤
│ idbloader.img (SPL + DDR init)      │  0x00000040 (64s)
├─────────────────────────────────────┤
│ U-Boot 이미지                       │  0x00004000 (16384s)
├─────────────────────────────────────┤
│ U-Boot 환경변수                     │  0x00008000 (32768s)
├─────────────────────────────────────┤
│ 커널 + DTB (FIT 이미지)             │  0x0000c000 (49152s)
├─────────────────────────────────────┤
│ ext4/squashfs 루트 파일시스템        │  ~0x00100000
└─────────────────────────────────────┘
```

### 10.2 U-Boot 빌드 및 커스터마이징

```bash
# OpenWrt 빌드 시스템 내에서 U-Boot 소스 위치
ls package/boot/uboot-rockchip/

# U-Boot 설정 확인
ls package/boot/uboot-rockchip/src/configs/ | grep -E "r5c|rk35"

# 특정 보드용 U-Boot 설정:
# nanopi-r5c-rk3568_defconfig

# U-Boot만 따로 빌드
make package/boot/uboot-rockchip/compile V=s
```

### 10.3 U-Boot 콘솔 사용

시리얼 케이블로 연결하여 U-Boot 콘솔에 접속:

```
연결: GPIO UART 핀 사용
설정: 1500000 baud (1.5Mbps) — 대부분의 Rockchip 보드
       (NanoPi R5C: 1500000, Orange Pi 5: 1500000)
```

```bash
# 시리얼 접속 (올바른 baud rate 확인 필수)
sudo screen /dev/ttyUSB0 1500000
# 또는
sudo picocom -b 1500000 /dev/ttyUSB0

# 보드 전원 켜면 U-Boot 로그 확인
# U-Boot 2023.xx ... (xxxxxx xxxx:xx:xx +0000)
# 
# Model: FriendlyElec NanoPi R5C
# DRAM:  4 GiB
# Core:  4 cores, 2 pages
# MMC:   mmc@fe2b0000: 1, mmc@fe2c0000: 0
# Loading Boot0001 from 'MMC 0:1' ...

# 아무 키 눌러서 U-Boot 프롬프트 진입
# =>

# 도움말
help

# 환경변수 확인
printenv

# 부트 디바이스 변경
setenv boot_targets "mmc1 mmc0 nvme usb"
saveenv
boot
```

### 10.4 Rockchip 부트로더 업데이트

```bash
# 기존 부트로더 백업
dd if=/dev/mmcblk1 of=/tmp/uboot-backup.bin bs=1K count=16384

# 새 부트로더 플래싱 (SD 카드에서)
dd if=/tmp/u-boot-rockchip.bin of=/dev/mmcblk0 seek=64 bs=512

# eMMC에 쓰기
dd if=/tmp/u-boot-rockchip.bin of=/dev/mmcblk1 seek=64 bs=512
```

---

## 11. 커스텀 패키지 만들기

### 11.1 Rockchip 하드웨어 정보 패키지

```bash
cd ~/openwrt/openwrt-23.05
mkdir -p package/rockchip-info/files
```

**`package/rockchip-info/Makefile`:**

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=rockchip-info
PKG_VERSION:=1.0
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

define Package/rockchip-info
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=Rockchip hardware information tool
  DEPENDS:=
  PKGARCH:=all
endef

define Package/rockchip-info/description
  Display Rockchip SBC hardware information
  SoC, temperature, NPU, NIC status and more.
endef

define Build/Prepare
endef

define Build/Configure
endef

define Build/Compile
endef

define Package/rockchip-info/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) ./files/rockchip-info.sh $(1)/usr/bin/rockchip-info
endef

$(eval $(call BuildPackage,rockchip-info))
```

**`package/rockchip-info/files/rockchip-info.sh`:**

```bash
#!/bin/sh

echo "================================================"
echo "  Rockchip OpenWrt Hardware Information"
echo "================================================"
echo ""

# SoC 정보
echo "[SoC Info]"
cat /proc/device-tree/compatible 2>/dev/null | tr '\0' ' '
echo ""
cat /proc/cpuinfo | grep "Hardware\|Revision" | head -2

# CPU
echo ""
echo "[CPU]"
grep "model name\|CPU part" /proc/cpuinfo | head -1 | cut -d: -f2
echo "  Cores: $(grep -c processor /proc/cpuinfo)"
echo "  Arch : $(uname -m)"

# CPU 온도
echo ""
echo "[Temperature]"
for zone in /sys/class/thermal/thermal_zone*; do
    [ -f "$zone/temp" ] || continue
    type=$(cat $zone/type 2>/dev/null)
    temp=$(cat $zone/temp 2>/dev/null)
    printf "  %s: %d.%d°C\n" "$type" $((temp/1000)) $((temp%1000))
done

# 메모리
echo ""
echo "[Memory]"
total=$(free | grep Mem | awk '{print $2}')
used=$(free | grep Mem | awk '{print $3}')
free=$(free | grep Mem | awk '{print $4}')
echo "  Total: $((total / 1024)) MB"
echo "  Used:  $((used / 1024)) MB"
echo "  Free:  $((free / 1024)) MB"

# NIC 정보
echo ""
echo "[Network Interfaces]"
for iface in /sys/class/net/eth* /sys/class/net/wlan*; do
    [ -d "$iface" ] || continue
    name=$(basename $iface)
    speed=$(cat $iface/speed 2>/dev/null)
    duplex=$(cat $iface/duplex 2>/dev/null)
    link=$(cat $iface/carrier 2>/dev/null)
    if [ "$link" = "1" ]; then
        echo "  $name: ${speed}Mbit/s ${duplex} (LINK UP)"
    else
        echo "  $name: (LINK DOWN)"
    fi
done

# NVMe 디스크
echo ""
echo "[Storage]"
if ls /dev/nvme*n1 2>/dev/null; then
    nvme list 2>/dev/null | tail -n +2 | while read line; do
        echo "  NVMe: $line"
    done
fi
df -h / /overlay 2>/dev/null | grep -v "^Filesystem"

# NPU (RK3588)
echo ""
echo "[NPU]"
if [ -c /dev/dri/renderD128 ]; then
    echo "  NPU device: /dev/dri/renderD128"
    lsmod | grep rknpu | awk '{print "  Driver: " $1 " (" $2 ")"}'
else
    echo "  (Not available on this SoC)"
fi

echo ""
echo "================================================"
echo "  Uptime: $(uptime | sed 's/.*up //' | sed 's/,.*//')"
echo "  Load  : $(cat /proc/loadavg | cut -d' ' -f1-3)"
echo "================================================"
```

```bash
chmod +x package/rockchip-info/files/rockchip-info.sh

# 빌드
make package/rockchip-info/compile V=s

# 확인
ls -lh bin/packages/aarch64_cortex-a55/base/rockchip-info*.ipk
# (CPU 코어에 따라 디렉토리명: aarch64_cortex-a55, aarch64_cortex-a72 등)

# 설치
scp bin/packages/aarch64_cortex-a55/base/rockchip-info*.ipk \
  root@192.168.1.1:/tmp/
ssh root@192.168.1.1
opkg install /tmp/rockchip-info*.ipk
rockchip-info
```

### 11.2 2.5GbE 네트워크 테스트 패키지

**`package/netspeed-test/files/netspeed-test.sh`:**

```bash
#!/bin/sh

echo "============================================"
echo "  Network Speed Test (2.5GbE Ready)"
echo "============================================"

# NIC 속도 확인
for iface in eth0 eth1; do
    if [ -d "/sys/class/net/$iface" ]; then
        speed=$(cat /sys/class/net/$iface/speed 2>/dev/null)
        echo "  $iface: ${speed}Mbps"
    fi
done

# iperf3 테스트 (서버가 실행 중인 경우)
echo ""
echo "  Run server: iperf3 -s"
echo "  Run client: iperf3 -c <server-ip>"
echo ""

# NAT 성능 측정
if [ -f /proc/net/nf_conntrack ]; then
    count=$(wc -l < /proc/net/nf_conntrack 2>/dev/null)
    max=$(cat /proc/sys/net/nf_conntrack_max 2>/dev/null)
    echo "  Connections: $count / $max"
fi

# CPU 부하 확인 (NAT 처리율 간접 측정)
echo ""
echo "  CPU load at test time:"
mpstat 1 3 | tail -1
```

---

## 12. 문제 해결

### 12.1 부팅 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| **전원 LED만 켜지고 부팅 안 됨** | SD 카드 부트로더 문제 | bootable-sd 이미지 다시 쓰기 |
| **UART에 로그 없음** | 시리얼 baud rate 불일치 | Rockchip은 1500000 baud 확인 |
| **커널 패닉 (DTC 오류)** | 잘못된 DTS | DTS 문법 확인 후 재빌드 |
| **eMMC 부팅 실패** | 부트로더가 eMMC에 없음 | SD 부팅 후 eMMC에 부트로더 설치 |
| **Wi-Fi 모듈 미인식** | M.2 모듈 전원/펌웨어 | `dmesg | grep wifi` 로그 확인 |
| **2.5GbE 링크 안 잡힘** | 케이블/협상 문제 | `ethtool eth1`, 케이블 Cat6 이상 권장 |

### 12.2 시리얼 콘솔 연결 (Rockchip USB-UART)

Rockchip 보드는 대부분 3.3V UART를 사용하며 **baud rate 1500000**이 기본입니다:

```
NanoPi R5C 시리얼 핀맵:
┌──────────────────────────────┐
│ Pin 1: GND                   │
│ Pin 2: VDD_IO (3.3V) — 연결X │
│ Pin 3: UART2_TX (→ 어댑터 RX) │
│ Pin 4: UART2_RX (→ 어댑터 TX) │
└──────────────────────────────┘
```

```bash
# Linux
sudo screen /dev/ttyUSB0 1500000

# Windows (PuTTY)
# Serial → Speed: 1500000 → Open

# macOS
screen /dev/tty.usbserial-* 1500000
```

> **참고**: 일반 USB-UART 어댑터(CP2102, CH340G)는 1.5Mbps를 지원해야 합니다.
> CH340G는 일부 클론에서 1.5Mbps가 안 될 수 있으므로 CP2102 또는 FT232 권장.

### 12.3 전원 문제

```bash
# Rockchip SBC는 전원에 민감합니다
# Orange Pi 5: 5V/4A USB-C PD 권장
# NanoPi R5C:   5V/2A USB-C

# 전압 부족 시:
# - 예기치 않은 재부팅
# - USB/PCIe 불안정
# - eMMC 쓰기 오류

# 전압 확인
cat /sys/class/power_supply/*/voltage_now 2>/dev/null
cat /sys/devices/platform/ff3e0000.i2c/i2c-*/../regulator*/microvolts 2>/dev/null
```

### 12.4 RK3588 전력 관리

```bash
# 빅코어 (A76)만 끄기 (저전력 모드)
echo 0 > /sys/devices/system/cpu/cpu4/online
echo 0 > /sys/devices/system/cpu/cpu5/online
echo 0 > /sys/devices/system/cpu/cpu6/online
echo 0 > /sys/devices/system/cpu/cpu7/online

# 다시 켜기
echo 1 > /sys/devices/system/cpu/cpu4/online
```

### 12.5 NAT 성능 향상 팁

```bash
# 소프트웨어 오프로드 활성화
# Rockchip GMAC는 HW 오프로드 지원
ethtool -K eth0 rx-checksumming on
ethtool -K eth0 tx-checksumming on
ethtool -K eth0 tcp-segmentation-offload on

# GRO (Generic Receive Offload)
ethtool -K eth0 gro on

# 인터럽트 조정
# RPS: 여러 CPU 코어에 인터럽트 분산
echo "f" > /sys/class/net/eth0/queues/rx-0/rps_cpus
echo "f" > /sys/class/net/eth1/queues/rx-0/rps_cpus

# XPS: 전송 큐 분산
echo "f" > /sys/class/net/eth0/queues/tx-0/xps_cpus
```

---

## 13. 참고 자료

### 13.1 공식 문서

| 자료 | URL |
|---|---|
| OpenWrt Rockchip | https://openwrt.org/docs/techref/hardware/soc/soc/rockchip |
| NanoPi R5C Wiki | https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R5C |
| Orange Pi 5 Wiki | http://www.orangepi.org/orangepiwiki/ |
| Rockchip 공식 문서 | https://opensource.rock-chips.com/ |
| U-Boot Rockchip | https://u-boot.readthedocs.io/en/latest/board/rockchip/rockchip.html |

### 13.2 커뮤니티

| 자료 | URL |
|---|---|
| FriendlyElec 포럼 | https://forum.friendlyelec.com/ |
| Orange Pi 포럼 | http://www.orangepi.org/orangepibbsen/ |
| OpenWrt 포럼 (Rockchip) | https://forum.openwrt.org/c/hardware/rockchip/ |
| Armbian (Rockchip 일반) | https://forum.armbian.com/ |

### 13.3 추천 보드별 OpenWrt 이미지 다운로드

| 보드 | 펌웨어 선택기 |
|---|---|
| NanoPi R2S | https://firmware-selector.openwrt.org/?target=rockchip%2Farmv8&id= friendlyelec_nanopi-r2s |
| NanoPi R4S | https://firmware-selector.openwrt.org/?target=rockchip%2Farmv8&id= friendlyelec_nanopi-r4s |
| NanoPi R5C | https://firmware-selector.openwrt.org/?target=rockchip%2Farmv8&id= friendlyelec_nanopi-r5c |
| Orange Pi 5 | https://firmware-selector.openwrt.org/?target=rockchip%2Farmv8&id= orangepi_5 |

### 13.4 교육 실습 커리큘럼 (제안)

| 단계 | 내용 | 예상 시간 | 준비물 |
|---|---|---|---|
| 1단계 | Rockchip SBC 선택 및 개봉 | 30분 | NanoPi R5C 또는 Orange Pi 5 |
| 2단계 | 시리얼 케이블 연결 및 U-Boot 로그 확인 | 30분 | USB-UART (1500000 baud) |
| 3단계 | 빌드 환경 구축 및 첫 빌드 | 2시간 | PC |
| 4단계 | SD 카드 부팅 이미지 만들기 | 20분 | MicroSD 카드 |
| 5단계 | 첫 부팅 및 네트워크 설정 | 30분 | SBC |
| 6단계 | 2.5GbE 성능 테스트 (iperf3) | 20분 | PC + Cat6 케이블 |
| 7단계 | eMMC에 설치 및 sysupgrade | 30분 | SBC |
| 8단계 | DTS LED 수정 및 재빌드 | 1시간 | PC |
| 9단계 | rockchip-info 패키지 제작 | 1시간 | PC |
| 10단계 | M.2 NVMe SSD 설치 (확장) | 30분 | NVMe SSD |

### 13.5 보드별 시리얼 Baud Rate

| 보드 | Baud Rate | 시리얼 핀 위치 |
|---|---|---|
| NanoPi R2S | **1500000** | 4핀 헤더 (GND/TX/RX/3.3V) |
| NanoPi R4S | **1500000** | 4핀 헤더 |
| **NanoPi R5C** | **1500000** | 4핀 헤더 (2.0mm 피치) |
| NanoPi R5S | **1500000** | 4핀 헤더 |
| NanoPi R6C | **1500000** | 4핀 헤더 |
| **Orange Pi 5** | **1500000** | 3핀 헤더 (GND/TX/RX) |
| Orange Pi 3B | **1500000** | 4핀 헤더 |

---

> **참고사항**:
> - Raspberry Pi와 달리 Rockchip은 **부트로더가 SD 카드의 첫 번째 섹터에 기록**되어야 함
> - `bootable-sd` 이미지는 이 부트로더를 포함하고 있어 일반 `dd`만으로 부팅 가능
> - 2.5GbE NIC의 성능을 내려면 Cat6 이상의 케이블과 상대방도 2.5GbE 지원 필요
> - Rockchip SBC 대부분은 **5V 전원**이 필요하며 전류 요구량이 Raspberry Pi보다 높음

---

> **다른 가이드 보기**:
> - RaspberryPi: `C:\OpenWrt\RaspberryPi\RaspberryPi_OpenWrt_개발_가이드.md`
> - ipTIME: `C:\OpenWrt\Iptime\Iptime_OpenWrt_개발_가이드.md`
> - x86_64: `C:\OpenWrt\x86_64\OpenWrt_x86_64_개발_가이드.md`
