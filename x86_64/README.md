# OpenWrt on x86_64 (PC/미니 PC/씬 클라이언트) — 개발부터 이식까지 완전 정복 가이드

> **목표**: x86_64 아키텍처 (일반 PC, 미니 PC, 씬 클라이언트, 노트북, VM 등) 를 타겟으로 OpenWrt 펌웨어를 개발 환경 구축부터 소스 수정, 컴파일, 부팅, 그리고 자체 패키지 제작까지 전 과정을 학습한다.

---

## 목차

1. [OpenWrt x86_64 개요](#1-openwrt-x86_64-개요)
2. [지원 하드웨어](#2-지원-하드웨어)
3. [개발 환경 구축](#3-개발-환경-구축)
4. [소스 코드 다운로드 및 빌드 설정](#4-소스-코드-다운로드-및-빌드-설정)
5. [펌웨어 컴파일](#5-펌웨어-컴파일)
6. [부팅 미디어 준비](#6-부팅-미디어-준비)
7. [부팅 및 초기 설정](#7-부팅-및-초기-설정)
8. [NIC 및 Wi-Fi 드라이버 추가](#8-nic-및-wi-fi-드라이버-추가)
9. [소스 코드 수정 및 재빌드 실습](#9-소스-코드-수정-및-재빌드-실습)
10. [가상화 환경에서 실행](#10-가상화-환경에서-실행)
11. [고급 활용 (ESXi/PVE/Proxmox)](#11-고급-활용-esxipveproxmox)
12. [커스텀 패키지 만들기](#12-커스텀-패키지-만들기)
13. [문제 해결](#13-문제-해결)
14. [참고 자료](#14-참고-자료)

---

## 1. OpenWrt x86_64 개요

### 1.1 왜 x86_64인가?

x86_64는 OpenWrt의 가장 **범용적인 타겟** 중 하나입니다. 일반 PC, 미니 PC, 씬 클라이언트, 노트북, 가상 머신 등 **어떤 x86_64 하드웨어에서도 동일한 펌웨어 이미지**로 부팅할 수 있습니다.

**x86_64 OpenWrt의 장점:**

| 항목 | 내용 |
|---|---|
| **하드웨어 자유도** | CPU/칩셋 제약 없이 표준 x86_64면 모두 가능 |
| **성능** | ARM/MIPS 대비 압도적인 CPU 성능 (멀티코어, 고클럭) |
| **메모리 확장** | RAM/Disk 자유롭게 증설 가능 (수 GB ~ 수십 GB) |
| **NIC 확장** | PCIe NIC, USB NIC 등 자유로운 네트워크 카드 추가 |
| **가상화** | VM, Docker, LXC 등 고급 기능 네이티브 지원 |
| **저렴한 진입장벽** | 중고 씬 클라이언트 2~5만원, 기존 PC 재활용 가능 |
| **범용성** | 방화벽, NAS, VPN 서버, DNS 서버 등 올인원 구성 가능 |

### 1.2 Raspberry Pi vs x86_64 비교

| 항목 | Raspberry Pi 4 | x86_64 씬 클라이언트 (예: Dell Wyse 5070) |
|---|---|---|
| CPU | ARM Cortex-A72 4코어 @ 1.8GHz | Intel J5005 4코어 @ 2.8GHz |
| RAM | 4GB (공유) | 8GB+ (확장 가능) |
| 네트워크 | 1GbE (USB 추가 필요) | 1GbE + PCIe 슬롯 추가 가능 |
| 디스크 | SD 카드 (느림) | mSATA / M.2 / SATA SSD (빠름) |
| 전력 | ~15W | ~10~25W |
| 가격 (중고) | 5~8만원 | 3~7만원 |
| **OpenWrt 성능** | ~600Mbps NAT | **~2~10Gbps NAT** |

> **결론**: NAT 성능이 중요하거나 (1Gbps 이상), 여러 서비스를 동시에 운영해야 한다면 x86_64가 월등히 유리합니다.

### 1.3 빌드 특징

x86_64 OpenWrt의 가장 큰 특징은 **크로스 컴파일이 필요 없다**는 점입니다:

```
타겟도 x86_64, 호스트도 x86_64
→ 네이티브 컴파일
→ 크로스 툴체인 불필요
→ 매우 빠른 빌드 속도
```

```bash
# x86_64 호스트에서 x86_64 OpenWrt를 빌드하면:
# gcc 가 호스트 gcc 와 동일
# 툴체인 빌드 생략 가능
# 빌드 시간이 ARM/MIPS 대비 1/3 수준
```

---

## 2. 지원 하드웨어

### 2.1 실행 가능한 모든 하드웨어

OpenWrt x86_64는 **표준 x86_64 UEFI/BIOS**만 있으면 모든 하드웨어에서 동작합니다:

| 분류 | 예시 하드웨어 | 특징 |
|---|---|---|
| **씬 클라이언트** | Dell Wyse 3040/5070, HP t430/t530/t740, Lenovo M715q | 저전력, 팬리스, 저렴 (중고 3~8만원) |
| **미니 PC** | Intel NUC, ASUS PN 시리즈, Beelink, Minisforum | 콤팩트, 적당한 성능 |
| **구형 PC/노트북** | 보유 중인 오래된 x86 PC | 재활용, 전력 소모 다소 높음 |
| **서버/워크스테이션** | 일반 데스크탑, 서버 | 고성능, 확장성 우수 |
| **가상 머신** | VMware, VirtualBox, Proxmox, Hyper-V | 테스트/개발에 최적 |
| **임베디드 x86** | PC Engine APU, Protectli, Netgate | 라우터 전용 팬리스 x86 |

### 2.2 추천 하드웨어 (가성비)

#### 입문용: Dell Wyse 3040 (약 3~5만원)

```
CPU: Intel Atom x5-Z8350 (4코어 @ 1.44GHz)
RAM: 2GB (온보드, 확장 불가)
스토리지: 8GB eMMC + MicroSD
NIC: 1x GbE (Realtek RTL8111)
전력: ~6W (팬리스)
장점: 매우 저렴, 극저전력, 매우 조용함
단점: RAM 업그레이드 불가, 100Mbps NAT 한계
```

#### 중급: Dell Wyse 5070 (약 5~8만원)

```
CPU: Intel Pentium J5005 (4코어 @ 2.8GHz)
RAM: 4~8GB (확장 가능, 최대 32GB)
스토리지: M.2 SATA SSD + eMMC
NIC: 1x GbE (Intel I210)
전력: ~10W (팬리스)
장점: 가격 대비 성능 우수, M.2 확장, 1Gbps NAT 가능
```

#### 고급: HP t740 (약 15~20만원)

```
CPU: AMD Ryzen V1807B (4코어 8스레드 @ 3.8GHz)
RAM: 8~32GB (DDR4)
스토리지: M.2 NVMe + 2.5" SATA
NIC: 2x GbE (Intel)
전력: ~25W
장점: 2.5Gbps/10Gbps NIC 추가 가능, VPN 고성능
```

#### 라우터 전용: Protectli VP2420 (약 20만원+)

```
CPU: Intel Celeron J4125 (4코어 @ 2.7GHz)
RAM: 8GB DDR4
NIC: 4x GbE (Intel I210)
케이스: 팬리스 금속 케이스
장점: 4포트, OpenWrt 최적화 설계
```

---

## 3. 개발 환경 구축

### 3.1 요구 사양

| 항목 | 최소 사양 | 권장 사양 |
|---|---|---|
| CPU | 2코어 | 4코어 이상 |
| RAM | 4GB | 8GB 이상 |
| 디스크 | 20GB 여유 | 50GB 이상 (SSD 권장) |
| OS | Ubuntu 22.04 / 24.04 LTS | 동일 |

> **x86_64 타겟의 장점**: 호스트와 타겟 아키텍처가 같아 크로스 컴파일이 필요 없어 빌드 속도가 매우 빠릅니다.

### 3.2 패키지 설치

```bash
# Ubuntu 24.04 기준
sudo apt update
sudo apt install -y build-essential gcc g++ gawk git \
  libncurses5-dev libssl-dev python3 python3-distutils \
  python3-setuptools rsync unzip zlib1g-dev file wget curl \
  qemu-utils qemu-system-x86 qemu-kvm \
  dosfstools mtools

# ext4 이미지 생성을 위한 도구
sudo apt install -y parted e2fsprogs
```

### 3.3 디렉토리 준비

```bash
mkdir -p ~/openwrt
cd ~/openwrt
```

---

## 4. 소스 코드 다운로드 및 빌드 설정

### 4.1 소스 클론

```bash
cd ~/openwrt

# 안정 버전 (권장)
git clone https://git.openwrt.org/openwrt/openwrt.git openwrt-23.05
cd openwrt-23.05
git checkout v23.05.5

# feeds 설정
./scripts/feeds update -a
./scripts/feeds install -a
```

### 4.2 타겟 설정 (make menuconfig)

```bash
make menuconfig
```

**x86_64 기준 권장 설정:**

| 메뉴 | 선택값 | 설명 |
|---|---|---|
| **Target System** | `x86` | x86 계열 타겟 |
| **Subtarget** | `x86_64` | 64비트 x86 |
| **Target Profile** | `Generic` | 모든 x86_64 호환 (범용) |
| **Target Images** | `squashfs + ext4 + ISO` | USB용 + VM용 |

**Target Images 상세 설정:**

```
Target Images --->
  [*] squashfs                   # 임베디드 스타일 (overlay)
  [*] ext4                       # 일반 리눅스 스타일
  [*] ISO image                  # CD/DVD/VM 부팅용
  [*] VirtualBox VDI             # VirtualBox 전용 이미지
  [*] QEMU qcow2                 # QEMU/Proxmox 전용 이미지
  [*] Root filesystem images     # 루트 파일시스템만
  [*] Build GRUB images          # GRUB 부트로더 포함
  [*] Build UEFI images          # UEFI 부팅 지원
      (4) Number of sectors for ext4 image  # 크기 (MB)
      (512) Root partition size (MB)        # 루트 파티션 크기
```

**x86_64 필수 패키지 설정:**

```
Base system --->
  <*> busybox                     # 기본 유틸리티
  <*> dnsmasq-full               # DNS/DHCP (full 버전)
  <*> dropbear                    # SSH 서버
  <*> firewall4                   # nftables 기반 방화벽

Kernel modules --->
  Network Drivers --->
    <*> kmod-e1000               # Intel PRO/1000 (가상 NIC 포함)
    <*> kmod-e1000e              # Intel PRO/1000e
    <*> kmod-igb                 # Intel I210/I211 기가비트
    <*> kmod-igc                 # Intel 2.5GbE I225/I226
    <*> kmod-r8169               # Realtek 기가비트
    <*> kmod-tg3                 # Broadcom 기가비트
    <*> kmod-forcedeth           # NVIDIA nForce
    <*> kmod-mlx4-core           # Mellanox ConnectX-3
    <*> kmod-mlx5-core           # Mellanox ConnectX-4/5/6
  USB Support --->
    <*> kmod-usb-core
    <*> kmod-usb-ohci
    <*> kmod-usb-uhci
    <*> kmod-usb2
    <*> kmod-usb3
    <*> kmod-usb-net             # USB 이더넷
    <*> kmod-usb-net-asix
    <*> kmod-usb-net-rtl8152
    <*> kmod-usb-storage
  Filesystems --->
    <*> kmod-fs-ext4
    <*> kmod-fs-vfat
    <*> kmod-fs-ntfs3            # NTFS 읽기/쓰기
  Hardware Monitoring --->
    <*> lm-sensors               # 온도/전압 센서

LuCI --->
  Collections --->
    <*> luci
  Themes --->
    <*> luci-theme-bootstrap
  Applications --->
    <*> luci-app-firewall
    <*> luci-app-opkg
    <*> luci-app-statistics      # 시스템 모니터링

Network --->
  <*> iperf3                    # 네트워크 성능 측정
  <*> tcpdump                   # 패킷 캡처
  <*> ethtool                   # NIC 설정 도구
  <*> lldpd                     # LLDP (네트워크 토폴로지)

Utilities --->
  <*> htop                      # 프로세스 모니터링
  <*> lm-sensors                # 센서 모니터링
  <*> smartmontools             # 디스크 S.M.A.R.T. 모니터링
```

---

## 5. 펌웨어 컴파일

### 5.1 첫 빌드

```bash
cd ~/openwrt/openwrt-23.05

make -j$(nproc) V=s
```

> **참고**: x86_64 타겟은 크로스 컴파일이 필요 없어 ARM/MIPS 대비 빌드 시간이 1/3 수준입니다.
> - 첫 빌드: 약 20~40분
> - 증분 빌드: 3~10분

### 5.2 빌드 결과 확인

```bash
ls -lh bin/targets/x86/64/

# 출력 예시:
# openwrt-23.05.5-x86-64-generic-ext4-combined.img.gz
# openwrt-23.05.5-x86-64-generic-ext4-combined-efi.img.gz
# openwrt-23.05.5-x86-64-generic-squashfs-combined.img.gz
# openwrt-23.05.5-x86-64-generic-squashfs-combined-efi.img.gz
# openwrt-23.05.5-x86-64-generic-squashfs-rootfs.img.gz
# openwrt-23.05.5-x86-64-generic-ext4-rootfs.img.gz
# openwrt-23.05.5-x86-64-generic.iso                  # ISO 이미지
# openwrt-23.05.5-x86-64-generic.vdi                  # VirtualBox
# openwrt-23.05.5-x86-64-generic.qcow2                # QEMU/Proxmox
# openwrt-23.05.5-x86-64-generic.vmdk                 # VMware
# openwrt-23.05.5-x86-64-generic-kernel.bin
# config.buildinfo
# sha256sums
```

### 5.3 이미지 파일 종류 이해

| 파일명 | 용도 | 설명 |
|---|---|---|
| `*combined.img.gz` | **HDD/USB 부팅용** | HDD/SSD/USB에 직접写入 |
| `*combined-efi.img.gz` | **UEFI 부팅용** | UEFI 모드 전용 (최신 PC) |
| `*squashfs-*` | squashfs | 압축 읽기전용 + overlay (초기화 쉬움) |
| `*ext4-*` | ext4 | 일반 리눅스 방식 |
| `*.iso` | **CD/DVD/VM 부팅** | 가상 CD-ROM 부팅용 |
| `*.vdi` | VirtualBox 전용 | VirtualBox에서 직접 import |
| `*.qcow2` | **QEMU/Proxmox 전용** | KVM/Proxmox에서 사용 |
| `*.vmdk` | VMware 전용 | VMware에서 사용 |

> **x86_64의 특장점**: 단일 빌드로 ISO, VDI, qcow2, VMDK 등 **모든 이미지 포맷을 한 번에 생성**합니다.

### 5.4 증분 빌드

```bash
# 커널만 재빌드
make target/linux/compile V=s

# 이미지만 재생성
make target/linux/install V=s

# 특정 패키지 재빌드
make package/dnsmasq/compile V=s

# 전체 클린
make clean
```

---

## 6. 부팅 미디어 준비

### 6.1 USB 메모리에 이미지 쓰기 (가장 보편적)

#### Linux 호스트에서:

```bash
# 이미지 압축 해제
cd ~/openwrt/openwrt-23.05/bin/targets/x86/64
gunzip -k openwrt-23.05.5-x86-64-generic-ext4-combined.img.gz

# USB 디바이스 확인
lsblk | grep -E "sd[b-z]"
# 예: /dev/sdb (USB 메모리)

# 이미지 쓰기 (BIOS/Legacy 모드)
sudo dd if=openwrt-23.05.5-x86-64-generic-ext4-combined.img \
  of=/dev/sdb bs=4M status=progress conv=fsync

# UEFI 모드 필요 시:
# sudo dd if=openwrt-23.05.5-x86-64-generic-ext4-combined-efi.img \
#   of=/dev/sdb bs=4M status=progress conv=fsync
```

#### Windows 호스트에서:

```powershell
# Rufus 사용 (권장)
# 1. https://rufus.ie 에서 Rufus 다운로드
# 2. USB 메모리 선택
# 3. combined.img.gz (또는 combined.img) 선택
# 4. "DD 이미지 모드"로 쓰기 (중요!)
# 5. 시작 → 확인

# 또는 Balena Etcher 사용
# https://www.balena.io/etcher/
```

#### macOS 호스트에서:

```bash
# USB 확인
diskutil list

# 언마운트
diskutil unmountDisk /dev/disk2

# 이미지 쓰기
gunzip -k openwrt-*-ext4-combined.img.gz
sudo dd if=openwrt-*-ext4-combined.img of=/dev/rdisk2 bs=4M status=progress conv=fsync
```

### 6.2 디스크(SSD/HDD)에 직접 설치

```bash
# Linux (대상 디스크가 /dev/sda라고 가정)
sudo dd if=openwrt-*-ext4-combined.img of=/dev/sda bs=4M status=progress conv=fsync

# 부트로더 확인
sudo fdisk -l /dev/sda
```

### 6.3 PXE 네트워크 부팅 (고급)

```bash
# TFTP 서버에 커널과 rootfs 배포
sudo cp bin/targets/x86/64/openwrt-*-generic-kernel.bin /srv/tftp/openwrt-kernel
sudo cp bin/targets/x86/64/openwrt-*-generic-squashfs-rootfs.img.gz /srv/tftp/openwrt-rootfs

# DHCP 서버에 PXE 부팅 옵션 설정
# (dnsmasq.conf)
# enable-tftp
# tftp-root=/srv/tftp
# dhcp-boot=openwrt-kernel
```

### 6.4 부팅 모드 선택 (BIOS vs UEFI)

| 부팅 모드 | 이미지 선택 | 설명 |
|---|---|---|
| **BIOS/Legacy** (구형) | `*combined.img` | 레거시 BIOS, 구형 PC |
| **UEFI** (최신) | `*combined-efi.img` | UEFI 전용, Secure Boot (비활성화 필요) |
| **UEFI + CSM** | 둘 다 가능 | CSM(호환성 모듈) 지원 시 |

---

## 7. 부팅 및 초기 설정

### 7.1 USB 부팅

1. USB 메모리를 PC에 연결
2. BIOS/UEFI에서 USB 부팅 우선순위 설정
   - 부팅 시 F2/Del/ESC 키 연타 → Boot Menu
   - USB 장치를 첫 번째 부팅 순서로 설정
3. 저장 후 재부팅

### 7.2 콘솔 화면 확인

OpenWrt x86_64가 정상 부팅되면 다음과 같은 콘솔 화면이 나타납니다:

```
OpenWrt 23.05.5, ...

[    0.000000] Linux version 6.1.xx
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz root=PARTUUID=...
...
[    3.123456] e1000e: Intel(R) PRO/1000 Network Driver
[    3.654321] r8169: Realtek RTL8169 Gigabit Ethernet driver
...
Press Enter to activate this console.

BusyBox v1.36.0 built-in shell (ash)

 -----------------------------------------------------
  OpenWrt 23.05.5, x86_64
 -----------------------------------------------------
  root@OpenWrt:/#
```

### 7.3 초기 네트워크 설정

```bash
# 기본 IP 확인
ip addr show

# 기본 설정: LAN IP는 192.168.1.1
# eth0이 LAN, eth1이 WAN (NIC 인식 순서에 따라 다름)

# SSH 접속 (PC IP를 192.168.1.x 로 설정)
ssh root@192.168.1.1
# 패스워드 설정
passwd
```

### 7.4 NIC 이름 매핑 확인

```bash
# 실제 하드웨어에 설치 시 NIC 이름 확인
ip link
# 출력 예시:
# 1: lo: ...
# 2: eth0: <BROADCAST,MULTICAST,UP> ...    # LAN
# 3: eth1: <BROADCAST,MULTICAST> ...        # WAN

# PCI 슬롯 위치로 NIC 식별
lspci | grep Ethernet

# MAC 주소로 NIC 식별
ip link show eth0
ip link show eth1
```

### 7.5 WAN 및 LAN 설정

```bash
# WAN 인터페이스 설정 (DHCP)
uci set network.wan=interface
uci set network.wan.device='eth1'
uci set network.wan.proto='dhcp'
uci commit network
/etc/init.d/network reload

# 또는 PPPoE (한국 ISP, FTTH)
# uci set network.wan.proto='pppoe'
# uci set network.wan.username='your-id@your-isp'
# uci set network.wan.password='your-password'
```

### 7.6 LuCI 웹 UI 접속

```bash
# 브라우저에서 http://192.168.1.1 접속
```

### 7.7 디스크 확장 (ext4 이미지)

ext4 이미지로 부팅한 경우, 전체 디스크 용량을 사용하려면:

```bash
# 파티션 테이블 확인
fdisk -l /dev/sda

# 파티션 확장
# 파티션 2 (root)를 전체 디스크로 확장
parted /dev/sda resizepart 2 100%

# 파일시스템 확장
resize2fs /dev/sda2

# 확인
df -h
```

---

## 8. NIC 및 Wi-Fi 드라이버 추가

### 8.1 NIC 드라이버 선택

x86_64의 가장 큰 장점은 다양한 NIC 드라이버 지원입니다:

```bash
# menuconfig에서 NIC 드라이버 선택
Kernel modules --->
  Network Drivers --->
    # 인텔
    <*> kmod-e1000       # 82540EM/82545EM (구형, VM 기본)
    <*> kmod-e1000e      # I218/I219 등 (소비자용)
    <*> kmod-igb         # I210/I211 (서버용 기가비트)
    <*> kmod-igc         # I225-V/I226-V (2.5GbE)
    <*> kmod-ixgbe       # 82599/ X520 (10GbE)
    <*> kmod-i40e        # X710/XL710 (40GbE)
    
    # Realtek
    <*> kmod-r8169       # RTL8111/8168/8125 (가장 보편적)
    <*> kmod-r8125       # RTL8125 (2.5GbE)
    
    # Broadcom
    <*> kmod-tg3         # BCM57xx (서버용)
    <*> kmod-bnx2        # BCM5706/5708
    
    # Mellanox
    <*> kmod-mlx4-core   # ConnectX-3 (10/40GbE)
    <*> kmod-mlx5-core   # ConnectX-4/5/6 (25/50/100GbE)
    
    # USB NIC
    <*> kmod-usb-net
    <*> kmod-usb-net-asix        # ASIX AX88179
    <*> kmod-usb-net-rtl8152     # Realtek RTL8152/8153
    <*> kmod-usb-net-cdc-eem     # CDC EEM
    <*> kmod-usb-net-cdc-ether   # CDC Ethernet
```

### 8.2 Wi-Fi 드라이버

x86_64에서 Wi-Fi를 AP 모드로 사용하려면 호환되는 칩셋이 필요합니다:

```bash
Kernel modules --->
  Network Drivers --->
    <*> kmod-ath9k-htc        # Atheros USB (AR9271)
    <*> kmod-ath10k           # Qualcomm QCA6174/QCA9377
    <*> kmod-ath10k-ct        # ath10k CT 버전
    <*> kmod-mt7921e          # MediaTek MT7921 (PCIe)
    <*> kmod-mt7921u          # MediaTek MT7921 (USB)
    <*> kmod-rt2800-usb       # Ralink RT28xx (USB)
    <*> kmod-rtlwifi          # Realtek RTL8821CE 등
    <*> kmod-brcmfmac         # Broadcom BCM43455 등
```

> **x86 Wi-Fi 추천 USB 어댑터**: 
> - Comfast CF-WU782AC (MT7620, AP 모드 가능)
> - Alfa AWUS036ACH (QCA9377, AP 모드 가능)
> - TP-Link TL-WN722N v2 (AR9271, AP 모드 가능)

### 8.3 빌드 후 적용

```bash
# 드라이버 추가 후 재빌드
make -j$(nproc) V=s

# 재부팅 후 드라이버 로드 확인
lsmod | grep -E "e1000|r8169|igb|ath"
```

---

## 9. 소스 코드 수정 및 재빌드 실습

### 9.1 GRUB 부트 메뉴 커스터마이징

x86_64의 부트로더는 GRUB2를 사용합니다:

```bash
cd ~/openwrt/openwrt-23.05

# GRUB 설정 파일 찾기
find target/linux/x86 -name "*grub*"

# GRUB 설정 확인
cat target/linux/x86/base-files/boot/grub/grub.cfg
# 출력 예시:
# set default="0"
# set timeout="5"
# 
# menuentry "OpenWrt" {
#   linux /boot/vmlinuz root=PARTUUID=... console=tty0 console=ttyS0,115200n8
# }
# 
# menuentry "OpenWrt (failsafe)" {
#   linux /boot/vmlinuz root=PARTUUID=... failsafe
# }
```

### 9.2 GRUB 타임아웃 및 커널 파라미터 변경

```bash
# GRUB 설정 변경
nano target/linux/x86/base-files/boot/grub/grub.cfg

# 예: 부트 타임아웃 3초로 변경, quiet 부팅
# set timeout="3"
# menuentry "OpenWrt" {
#   linux /boot/boot/vmlinuz root=PARTUUID=... console=tty0 quiet
# }
```

### 9.3 GRUB 배경화면 추가

```bash
# GRUB 배경 이미지 추가
mkdir -p target/linux/x86/base-files/boot/grub/themes/openwrt
# 640x480 PNG 이미지 준비
# grub.cfg에 테마 설정 추가
# set theme=($root)/boot/grub/themes/openwrt/theme.txt
```

### 9.4 재빌드 및 확인

```bash
# GRUB 변경사항 적용
make target/linux/install V=s

# 새 이미지로 부팅하여 GRUB 메뉴 확인
```

### 9.5 기본 패키지 세트 변경

펌웨어에 포함될 기본 패키지를 변경하려면:

```bash
# .config 파일에서 패키지 설정 직접 편집
# PACKAGE_ 접두사로 검색
grep "PACKAGE_" .config | grep "=y" | head -20

# 또는 make menuconfig에서 변경 후 저장
make menuconfig
# Base system, Kernel modules, Network 등에서 필요한 패키지 선택/해제
```

---

## 10. 가상화 환경에서 실행

### 10.1 QEMU/KVM (Linux)

가장 빠른 테스트 방법:

```bash
# QEMU가 이미 설치되어 있어야 함
cd ~/openwrt/openwrt-23.05/bin/targets/x86/64

# 1. 이미지 압축 해제
gunzip -k openwrt-*-ext4-combined.img.gz

# 2. QEMU로 부팅 (가장 빠름)
qemu-system-x86_64 \
  -machine q35,accel=kvm \
  -cpu host \
  -smp 2 \
  -m 512M \
  -drive file=openwrt-*-ext4-combined.img,format=raw \
  -netdev user,id=lan,net=192.168.100.0/24,dhcpstart=192.168.100.10 \
  -device virtio-net-pci,netdev=lan \
  -netdev user,id=wan,restrict=off \
  -device virtio-net-pci,netdev=wan \
  -nographic
```

### 10.2 QEMU qcow2 이미지 사용

```bash
# qcow2가 있으면 바로 사용 가능
qemu-system-x86_64 \
  -machine q35,accel=kvm \
  -cpu host \
  -m 512M \
  -drive file=openwrt-*-generic.qcow2,format=qcow2 \
  -device virtio-net-pci \
  -nographic
```

### 10.3 VirtualBox (Windows/Linux/macOS)

```bash
# VirtualBox VDI 이미지 사용
# openwrt-*-generic.vdi 파일을 VirtualBox에 직접 연결
# 
# 1. VirtualBox → 새 가상 머신 (Linux, Other Linux 64-bit)
# 2. 기존 가상 하드 디스크 사용 → VDI 파일 선택
# 3. 네트워크: 어댑터 1 (NAT 또는 호스트 전용)
# 4. 네트워크: 어댑터 2 (NAT) - WAN 테스트용
# 5. 시작

# 또는 ISO로 부팅 후 디스크에 설치
# 1. 가상 머신 생성 (디스크 없이)
# 2. ISO 이미지를 광학 드라이브에 연결
# 3. 부팅 → OpenWrt 라이브 환경 진입
# 4. 설치 방법:
#    dd if=/dev/urandom of=/dev/sda bs=1M count=1
#    gunzip -c /tmp/openwrt-*-ext4-combined.img.gz | dd of=/dev/sda
```

### 10.4 VMware

```bash
# VMDK 이미지 사용
# openwrt-*-generic.vmdk 파일을 VMware 가상 머신에 연결
```

---

## 11. 고급 활용 (ESXi/PVE/Proxmox)

### 11.1 Proxmox VE에 OpenWrt 설치

Proxmox VE에서 OpenWrt를 VM으로 실행:

```bash
# Proxmox 호스트에서
# qcow2 이미지를 VM 디스크로 변환
qemu-img convert -f qcow2 -O raw \
  openwrt-*-generic.qcow2 \
  /var/lib/vz/images/100/vm-100-disk-0.raw

# 또는 ISO 이미지로 VM 생성 후 설치
# Proxmox Web UI → Create VM:
#   OS: Linux Kernel 6.x
#   Disk: 0.5~1GB (충분)
#   CPU: 1~2 cores
#   Memory: 256~512MB
#   Network: virtio (LAN)
#   Network: virtio (WAN)
```

### 11.2 Proxmox OpenWrt 템플릿

```bash
# Proxmox 호스트에 OpenWrt VM 생성 스크립트
cat << 'SCRIPT' > /root/create-openwrt-vm.sh
#!/bin/bash
VM_ID=$1
BRIDGE_LAN=vmbr0
BRIDGE_WAN=vmbr1

qm create $VM_ID \
  --name "OpenWrt" \
  --memory 512 \
  --cores 2 \
  --net0 virtio,bridge=$BRIDGE_LAN \
  --net1 virtio,bridge=$BRIDGE_WAN \
  --ostype l26
  
qm importdisk $VM_ID openwrt-*-generic.qcow2 local-lvm
qm set $VM_ID --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-$VM_ID-disk-0
qm set $VM_ID --boot c --bootdisk scsi0
qm start $VM_ID
SCRIPT
chmod +x /root/create-openwrt-vm.sh
```

### 11.3 Docker에서 OpenWrt 실행 (실험적)

```bash
# OpenWrt를 Docker 컨테이너로 실행 (macvlan 네트워크 필요)
docker run -d \
  --name openwrt \
  --network host \
  --privileged \
  --restart unless-stopped \
  openwrt-x86_64:latest

# 단, Docker는 호스트 네트워크 스택을 공유하므로
# 완전한 라우터 기능보다는 특정 서비스 테스트용
```

---

## 12. 커스텀 패키지 만들기

### 12.1 x86_64 네이티브 바이너리 패키지

RaspberryPi 가이드의 패키지 구조와 동일하지만, **크로스 컴파일이 아닌 네이티브 컴파일**이라는 점이 다릅니다:

```bash
cd ~/openwrt/openwrt-23.05
mkdir -p package/x86-info/files
```

**`package/x86-info/Makefile`:**

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=x86-info
PKG_VERSION:=1.0
PKG_RELEASE:=1

include $(INCLUDE_DIR)/package.mk

define Package/x86-info
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=x86_64 hardware information tool
  DEPENDS:=
  PKGARCH:=all
endef

define Package/x86-info/description
  Display x86_64 hardware information: CPU, RAM, disk, NIC, sensors
endef

define Build/Prepare
endef

define Build/Configure
endef

define Build/Compile
endef

define Package/x86-info/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) ./files/x86-info.sh $(1)/usr/bin/x86-info
endef

$(eval $(call BuildPackage,x86-info))
```

**`package/x86-info/files/x86-info.sh`:**

```bash
#!/bin/sh

echo "================================================"
echo "  x86_64 OpenWrt Hardware Information"
echo "================================================"
echo ""

# CPU 정보
echo "[CPU]"
grep "model name" /proc/cpuinfo | head -1 | cut -d: -f2
echo "  Cores: $(grep -c processor /proc/cpuinfo)"
echo "  Arch : $(uname -m)"

# 메모리
echo ""
echo "[Memory]"
free -h | grep Mem | awk '{print "  Total: " $2 " / Free: " $4}'

# 디스크
echo ""
echo "[Disk]"
df -h / | tail -1 | awk '{print "  Root: " $2 " total, " $4 " free"}'
smartctl -a /dev/sda 2>/dev/null | grep -E "Model Family|Device Model|User Capacity"

# NIC
echo ""
echo "[Network Interfaces]"
lspci 2>/dev/null | grep -i ethernet | while read line; do
  echo "  $line"
done

# 센서
echo ""
echo "[Sensors]"
if command -v sensors >/dev/null 2>&1; then
  sensors | grep -E "Core|Package|temp" | head -5 | while read line; do
    echo "  $line"
  done
else
  echo "  (lm-sensors not installed)"
fi

# DMI 정보
echo ""
echo "[System]"
cat /sys/devices/virtual/dmi/id/product_name 2>/dev/null || echo "  (DMI not available)"
cat /sys/devices/virtual/dmi/id/sys_vendor 2>/dev/null

echo ""
echo "================================================"
echo "  Uptime: $(uptime | sed 's/.*up //' | sed 's/,.*//')"
echo "  Load  : $(cat /proc/loadavg | cut -d' ' -f1-3)"
echo "================================================"
```

```bash
chmod +x package/x86-info/files/x86-info.sh

# 패키지 빌드
make package/x86-info/compile V=s

# 빌드된 IPK 확인
ls -lh bin/packages/x86_64/base/x86-info*.ipk
```

### 12.2 네이티브 C 패키지 (크로스 컴파일 불필요)

x86_64의 장점: 호스트에서 테스트한 바이너리를 그대로 타겟에 사용할 수 있습니다:

```bash
mkdir -p package/hello-x86/src
```

**`package/hello-x86/src/main.c`:**

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/sysinfo.h>
#include <string.h>

int main() {
    char hostname[256];
    gethostname(hostname, sizeof(hostname));

    struct sysinfo info;
    sysinfo(&info);

    printf("========================================\n");
    printf("  Hello from x86_64 OpenWrt!\n");
    printf("========================================\n");
    printf("  Hostname: %s\n", hostname);
    printf("  Architecture: x86_64\n");
    printf("  Uptime: %ld days %02ld:%02ld:%02ld\n",
        info.uptime / 86400,
        (info.uptime % 86400) / 3600,
        (info.uptime % 3600) / 60,
        info.uptime % 60);
    printf("  Total RAM: %ld MB\n", info.totalram / (1024 * 1024));
    printf("  Free RAM:  %ld MB\n", info.freeram / (1024 * 1024));
    printf("  Processes: %d\n", info.procs);
    printf("  Load: %.2f %.2f %.2f\n",
        info.loads[0] / 65536.0,
        info.loads[1] / 65536.0,
        info.loads[2] / 65536.0);
    printf("========================================\n");
    return 0;
}
```

**`package/hello-x86/Makefile`:**

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=hello-x86
PKG_VERSION:=1.0
PKG_RELEASE:=1

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk

define Package/hello-x86
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=Hello World for x86_64
  DEPENDS:=
endef

define Build/Prepare
  mkdir -p $(PKG_BUILD_DIR)
  $(CP) ./src/* $(PKG_BUILD_DIR)/
endef

define Build/Configure
endef

define Build/Compile
  $(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/hello-x86 \
    $(PKG_BUILD_DIR)/main.c
endef

define Package/hello-x86/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) $(PKG_BUILD_DIR)/hello-x86 $(1)/usr/bin/
endef

$(eval $(call BuildPackage,hello-x86))
```

### 12.3 빌드 없이 직접 바이너리 배포 (x86_64만 가능)

```bash
# 호스트에서 직접 컴파일
gcc -static -o hello-x86 hello.c

# 그대로 타겟에서 실행 가능 (x86_64 동일 아키텍처)
scp hello-x86 root@192.168.1.1:/tmp/
ssh root@192.168.1.1 /tmp/hello-x86
```

---

## 13. 문제 해결

### 13.1 부팅 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| **Grub Rescue 프롬프트** | GRUB 설정 손상 | 라이브 CD로 부팅 → GRUB 재설치 |
| **No bootable device** | 잘못된 이미지 쓰기 | `dd` 이미지 다시 쓰기 (DD 모드) |
| **커널 패닉 (Kernel panic)** | 잘못된 root 파티션 | GRUB에서 `root=` 파라미터 확인 |
| **UEFI 부팅 실패** | Secure Boot 활성화 | BIOS에서 Secure Boot OFF |
| **NIC 미인식** | 드라이버 미포함 | menuconfig에서 NIC 드라이버 추가 후 재빌드 |
| **화면 출력 없음** | 잘못된 그래픽 모드 | `console=tty0` GRUB 파라미터 확인 |

### 13.2 GRUB 복구

```bash
# 라이브 Linux USB로 부팅 후

# 디스크 확인
lsblk

# OpenWrt 파티션 마운트
mount /dev/sda2 /mnt

# GRUB 재설치 (BIOS 모드)
grub-install --target=i386-pc --boot-directory=/mnt/boot /dev/sda

# GRUB 재설치 (UEFI 모드)
grub-install --target=x86_64-efi --efi-directory=/mnt/boot --boot-directory=/mnt/boot

# 재부팅
umount /mnt
reboot
```

### 13.3 NIC 이름 변경 (eth0 → eth1 등)

```bash
# udev 규칙으로 NIC 이름 고정
cat /etc/udev/rules.d/10-network.rules
# SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="XX:XX:XX:XX:XX:XX", NAME="wan"
# SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="YY:YY:YY:YY:YY:YY", NAME="lan"

# 재부팅 또는
udevadm trigger
```

### 13.4 NAT 성능 테스트

```bash
# iperf3로 NAT 성능 측정
# WAN 인터페이스에서 iperf3 서버 실행
iperf3 -s -B <WAN_IP>

# LAN 측 클라이언트에서 연결
iperf3 -c <WAN_IP>

# NAT 처리량 확인 예시:
# [ ID] Interval           Transfer     Bandwidth
# [  5]   0.00-30.00  sec  3.45 GBytes  987 Mbits/sec  → ~1Gbps
```

### 13.5 전력 소모 측정

```bash
# powertop 설치
opkg update
opkg install powertop

# 전력 소모 분석
powertop

# 또는 센서로 확인
sensors
cat /sys/class/power_supply/*/power_now 2>/dev/null
```

---

## 14. 참고 자료

### 14.1 공식 문서

| 자료 | URL |
|---|---|
| OpenWrt x86_64 | https://openwrt.org/docs/guide-user/installation/openwrt_x86 |
| x86 빌드 가이드 | https://openwrt.org/docs/guide-developer/build-system/start |
| GRUB 설정 | https://openwrt.org/docs/techref/bootloader/grub |
| 하드웨어 호환 목록 | https://openwrt.org/toh/start?dataflt%5B0%5D=**architecture**%3Dx86_64 |

### 14.2 성능 비교

| 플랫폼 | NAT 처리량 (단일 코어) | OpenWrt 빌드 시간 | 전력 소모 |
|---|---|---|---|
| RPi 4 (BCM2711) | ~600 Mbps | ~90분 | ~15W |
| Wyse 5070 (J5005) | ~2.5 Gbps | ~30분 | ~10W |
| HP t740 (V1807B) | ~9 Gbps | ~20분 | ~25W |
| 일반 데스크탑 (i5-12400) | ~40 Gbps | ~15분 | ~60W |

### 14.3 교육 실습 커리큘럼 (제안)

| 단계 | 내용 | 예상 시간 |
|---|---|---|
| 1단계 | 개발 환경 구축 및 첫 빌드 | 1시간 |
| 2단계 | USB 부팅 디스크 만들기 | 30분 |
| 3단계 | 실제 PC/NUC/씬클라에서 부팅 | 30분 |
| 4단계 | WAN/LAN 설정, NAT 테스트 | 30분 |
| 5단계 | QEMU 가상화 실행 | 20분 |
| 6단계 | NIC 드라이버 추가 및 재빌드 | 1시간 |
| 7단계 | GRUB 커스터마이징 | 30분 |
| 8단계 | x86-info 패키지 제작 | 1시간 |
| 9단계 | Proxmox VM 설치 (고급) | 1시간 |

---

> **다음 학습 주제**: Rockchip SBC OpenWrt 개발 가이드를 참고하세요 (`C:\OpenWrt\Rockchip\Rockchip_OpenWrt_개발_가이드.md`)
