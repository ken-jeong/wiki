---
aliases: []
type: Lecture
tags:
  - 3-2/임베디드시스템
draft: false
date: 2026-10-13
---
## 1. BusyBox로 작은 사용자 공간 만들기
### 1.1. BusyBox의 구조
**BusyBox**
- 여러 UNIX 명령을 **하나의 실행 파일**로 묶는다.
- 각 명령 기능을 applet이라고 하며, 명령 이름의 심볼릭 링크를 통해 같은 BusyBox 실행 파일을 호출할 수 있다.
```text
/bin/ls    -> /bin/busybox
/bin/cp    -> /bin/busybox
/bin/sh    -> /bin/busybox
```
따라서 모든 명령이 각각 별도 실행 파일일 필요가 없다.
선택한 applet과 빌드 방식에 따라 크기가 달라진다.

### 1.2. 빌드 과정
아래는 강의의 BusyBox 1.37.x 실습 기준이다.
```sh
# 호스트에서 실행
cd
git clone https://git.busybox.net/busybox

cd busybox
git checkout 1_37_stable

# 일반적인 기본 설정으로 시작
make defconfig
```

| 설정                 | 목적                       |
| :----------------- | :----------------------- |
| `make allnoconfig` | 대부분 비활성화한 상태에서 필요한 기능 선택 |
| Destination path   | `make install`의 설치 대상 경로 |

```sh
# applet 및 빌드 옵션 조정
make menuconfig
```

다음 항목을 설정한다.
- 공유 라이브러리 없이 실행하도록 정적 링크
- 타겟용 컴파일러 접두어 지정
```
Settings -> Build Options
[*] Build static binary (no shared libs)
(arm-linux-) Cross compiler prefix

Settings -> Library Tuning
[ ] SHA1: Use hardware accelerated instructions if possible
[ ] SHA256: Use hardware accelerated instructions if possible
```

크로스 컴파일러의 실행 파일 디렉터리를 호스트 `PATH`에 추가하고 빌드한다.
접두어와 경로는 실제 툴체인과 일치해야 한다.
```sh
export PATH="$HOME/tools/arm-unknown-linux-uclibcgnueabi/bin:$PATH"
make
make install
```
강의의 기본 설치 결과는 `./_install`이다.
이를 NFS 루트에 복사한다.
```sh
cp -a _install/. ~/nfsroot/
mkdir -p ~/nfsroot/dev ~/nfsroot/proc ~/nfsroot/sys ~/nfsroot/etc/init.d
```

> `cp -a` 사용과 `mkdir -p`는 실습 정리용으로 보완한 명령이다.
> 심볼릭 링크 등 설치 결과를 보존하고 이미 있는 디렉터리도 처리한다.

## 2. BusyBox init으로 부팅 자동화하기
**BusyBox init**
- SysV init이나 systemd보다 간단하다.
- 핵심 설정 파일은 `/etc/inittab`이다.
- *서비스 시작, 종료 처리, 프로그램 유지 등을 구성할 수 있다.*

### 2.1. `/etc/inittab`
강의에서 사용하는 줄 형식:
```text
<id>::<action>:<process>
```

타겟 기준 파일 내용:
```
~ # cat << 'EOF' > /etc/inittab
::sysinit:/etc/init.d/rcS
ttyAMA0::askfirst:/bin/sh
EOF

~ # cat /etc/inittab
```

| 항목 | 동작 |
|---|---|
| `sysinit` | 시스템 초기화 과정에서 지정 프로그램 실행 |
| `/etc/init.d/rcS` | proc·sysfs 등 기본 환경을 준비할 스크립트 |
| `ttyAMA0` | 강의 타겟의 시리얼 콘솔 |
| `askfirst` | 사용자 입력을 기다린 뒤 셸 실행 |
| `/bin/sh` | 실행할 셸 |

### 2.2. `/etc/init.d/rcS`
```
can't run '/etc/init.d/rcS': No such file or directory
```

1. `rcS` 스크립트 파일을 작성한다.
```
cat << 'EOF' > /etc/init.d/rcS
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo "Basic filesystems mounted."
EOF
```

2. 실행 권한을 부여한다.
```
chmod a+x /etc/init.d/rcS
```
호스트에서 NFS 루트 파일을 편집했다면 다음과 같이 실행한다.
```sh
chmod a+x ~/nfsroot/etc/init.d/rcS
```

3. 재부팅 후 타겟에서 `mount`와 `ps`를 확인한다.
```
reboot
mount
```

동작 순서는 **커널 → BusyBox init → inittab의 sysinit → rcS → 콘솔 셸**이다.
proc와 sysfs를 수동으로 마운트하던 작업을 rcS로 옮긴 셈이다.

## 3. 정적 링크와 동적 링크
### 3.1. 차이
| 구분                | 정적 링크                      | 동적 링크               |
| ----------------- | -------------------------- | ------------------- |
| 라이브러리 코드          | 필요한 코드가 실행 파일에 포함          | 실행 시 외부 공유 라이브러리 사용 |
| 실행 파일 크기          | 일반적으로 큼                    | 일반적으로 작음            |
| rootfs에 추가로 필요한 것 | 해당 프로그램의 공유 라이브러리 의존성이 줄어듦 | 동적 로더와 필요한 공유 라이브러리 |
| 실습상의 의미           | 처음 사용자 공간을 구성하기 쉬움         | 로더·라이브러리를 함께 배치해야 함 |

### 3.2. 파일이 있는데 왜 `not found`인가?
강의의 `hello`는 다음 인터프리터를 요구한다.
```text
interpreter /lib/ld-musl-armhf.so.1
```

타겟에 `hello`만 복사하고 이 파일을 준비하지 않으면 다음 오류가 발생한다.
```text
/bin/sh: ./hello: not found
```

**이 사례의 원인은 `hello` 자체의 부재가 아니라 필요한 동적 로더의 부재다.**

해결 과정:
1. 호스트에서 `file hello`로 실행 파일 형식과 인터프리터 경로를 확인한다.
2. 사용한 툴체인의 sysroot에서 해당 로더를 찾는다.
3. 타겟 rootfs의 동일한 경로인 `/lib/ld-musl-armhf.so.1`에 배치한다.
4. 타겟에서 다시 실행한다.

강의는 musl에서 동적 로더에 C 라이브러리 기능이 포함된다는 점을 설명한다. 다른 라이브러리 구성에서도 로더 하나만 복사하면 된다고 일반화하지 않는다.

### 3.3. BusyBox 크기 비교
강의의 측정 결과:

| 빌드 방식 | BusyBox 실행 파일 크기 |
|---|---:|
| 정적 링크 | 1,266,860 bytes |
| 동적 링크 | 1,096,400 bytes |
| 차이 | 170,460 bytes 감소, 약 13.5% |

동적 링크 실험은 `CONFIG_STATIC`을 끄고 다시 빌드한다.
이 수치는 **BusyBox 파일 하나의 크기**다.
rootfs 전체를 비교할 때는 추가된 로더와 공유 라이브러리도 포함해야 한다.

> 2026-10-15

## 4. 웹 서비스 추가하기
1. 강의에서 제공하는 `data/www` 내용을 타겟의 `/www`에 배치한다.
```sh
~ ❯ scp ~/Downloads/www.zip user@192.168.64.2:~/
```

```sh
user@ubuntu:~$ sudo unzip ~/www.zip -d ~/nfsroot
```

2. 타겟에서 BusyBox HTTP 서버를 실행한다.
```sh
/usr/sbin/httpd -h /www/
```

3. 호스트 브라우저에서 `http://192.168.76.100/index.html`에 접속한다.
```sh
# macOS http://localhost:8080에서 확인
~ ❯ ssh -N -L 8080:192.168.76.100:80 user@192.168.64.2
```

4. 부팅 시 자동 실행하려면 `/etc/init.d/rcS`에 추가한다.
```
~ # cat << 'EOF' >> /etc/init.d/rcS

busybox httpd -h /www/
echo "Web service started."
EOF

~ # cat /etc/init.d/rcS
```

여기서 `-h /www/`는 웹 문서를 제공할 디렉터리를 지정한다.
이 실습은 **rootfs 구성 → init 초기화 → 서비스 실행**이 연결되는 예다.

트러블슈팅
```
~ # chmod +x /www/cgi-bin/cpuinfo /www/cgi-bin/uptime /www/cgi-bin/list /www/cgi-bin/reboot /www/cgi-bin/upload
```

## 5. 완성된 rootfs를 initramfs로 묶기
### 5.1. 구조와 사용 목적
**initramfs**
- CPIO 아카이브를 부팅 시 메모리에 풀어 초기 사용자 공간을 제공한다.

제공 방식
1. **커널 내장**: `CONFIG_INITRAMFS_SOURCE`로 포함할 내용을 지정한다.
2. **별도 파일**: 부트로더가 커널·DTB와 함께 아카이브를 적재한다.

주요 용도
1. 작은 시스템을 빠르게 부팅한다.
2. 실제 루트 파일 시스템을 준비하는 중간 단계로 사용한다.

### 5.2. `/init` 준비
BusyBox의 `/sbin/init`을 초기 진입점으로 쓰는 강의 예:
```sh
# 호스트에서 실행: 아카이브를 만들기 전에 수행
cd ~/nfsroot
ln -s /sbin/init init
```
이 명령은 `/init`이 없는 상태를 전제로 한다.
완성된 타겟 파일 계층에서 `/init`이 `/sbin/init`을 가리키도록 만든다.

**아카이브를 만든 뒤 원본 디렉터리에 `/init`을 추가하면 기존 아카이브에는 반영되지 않는다. 다시 패키징해야 한다.**

### 5.3. CPIO 아카이브 생성과 압축
```sh
cd ~/nfsroot

# rootfs 안의 경로 목록 생성
# | newc 형식의 CPIO 아카이브 생성
# > 출력 파일 initramfs.cpio.gz
find . | cpio -H newc -o > ../initramfs.cpio

cd ..

# 생성한 아카이브 압축
gzip initramfs.cpio
```

### 5.4. U-Boot 컨테이너 생성
컨테이너 방식:
```sh
mkimage \
	-n 'Ramdisk Image' \
	-A arm \
	-O linux \
	-T ramdisk \
	-C gzip \
	-d initramfs.cpio.gz uInitramfs
```

- 입력: 압축된 CPIO 아카이브 `initramfs.cpio.gz`
- 출력: U-Boot용 헤더로 감싼 `uInitramfs`
- `bootz`의 인자 순서: **커널 주소 → initramfs 주소 → DTB 주소**.
```text
bootz <kernel-addr> <initramfs-addr> <dtb-addr>
```

주소 배치 예:

| 적재 대상     | RAM 주소       |
| --------- | ------------ |
| `zImage`  | `0x61000000` |
| DTB       | `0x62000000` |
| initramfs | `0x63000000` |

주소는 해당 실습 보드의 예이며, 다른 타겟에 그대로 적용하는 값은 아니다.
강의 15쪽의 압축 CPIO 직접 전달 예제와 6쪽의 컨테이너 방식은 구분해서 읽어야 한다.
