# Airgap Autoinstall 개요

[autoinstall-user-data](/Users/joon/github/airgap-autoinstall/autoinstall-user-data)는 오프라인 부트스트랩 노드를 위한 Ubuntu Server 자동 설치 정의 파일이다. 이 파일 하나에 다음 내용이 함께 들어 있다.

- `#cloud-config` 및 `autoinstall` 기반 Ubuntu 설치
- `THAKI_DATA` 라벨을 가진 USB 볼륨을 이용한 오프라인 APT 부트스트랩
- `cloud-init user-data` 기반 첫 부팅 후 프로비저닝
- Docker, PXE/TFTP, Harbor 구성을 위한 단계형 post-install 스크립트

설치는 먼저 수행되고, OS가 첫 부팅한 뒤 `/usr/local/sbin/init-cp.sh`가 순서화된 stage 스크립트를 한 번씩 실행하며 완료 상태를 기록한다.

## 실행 흐름

1. `early-commands`가 USB 미디어를 `/mnt/usbdata`에 마운트한다.
2. 같은 USB 내용을 `python3 -m http.server`로 `127.0.0.1:18080`에 노출한다.
3. APT는 이 로컬 HTTP 엔드포인트와 USB 기반 저장소를 사용하도록 재설정된다.
4. Ubuntu는 지정된 계정, 스토리지, locale, timezone, SSH, 네트워크 설정으로 설치된다.
5. 첫 부팅 시 `user-data.write_files`가 다음 파일들을 생성한다.
   - `/usr/local/bin/cp.env`
   - `/usr/local/lib/cp/lib.sh`
   - `/usr/local/sbin/init-cp.sh`
   - `/usr/local/lib/cp/stages/*.sh`
6. `runcmd`가 `init-cp.sh`를 실행한다.
7. `init-cp.sh`는 `cp.env`를 로드하고 필수 변수를 검증한 뒤 stage를 파일명 순서대로 실행한다.
8. 완료된 stage는 `/var/lib/cp/<stage>.done` 파일을 남기므로, 재실행 시 이미 끝난 작업은 건너뛴다.

## 설치 시점 설정

### 기본 설치 패키지

- `apt-transport-https`
- `ca-certificates`
- `curl`
- `gnupg`
- `python3-venv`
- `git`
- `dnsmasq`
- `syslinux-common`

### APT 동작

- `backports` 비활성화
- GeoIP 기반 미러 탐색 비활성화
- 기본 미러를 `http://127.0.0.1:18080`로 고정
- `amd64`, `i386`용 security 미러도 `http://127.0.0.1:18080` 사용
- 설정한 미러 사용이 실패하면 중단

### 시스템 식별 정보

- hostname: `bootstrap-test1`
- username: `tkadmin`
- real name: `tkadmin`
- password: 파일 내부에 해시값으로 저장

### OS 기본값

- kernel: `linux-generic`
- keyboard layout: `us`
- locale: `en_US.UTF-8`
- timezone: `Asia/Seoul`
- OEM install mode: `auto`
- installation source: `ubuntu-server`
- driver 검색 비활성화
- codec 설치 비활성화
- driver 설치 비활성화

### SSH

- SSH 서버 설치
- password 인증 허용
- authorized key 미리 설정하지 않음

### Storage

- swap 비활성화
- layout: `lvm`
- sizing policy: `all`
- 대상 디스크: `/dev/nvme0n1`

### Network

- NIC `enp0s31f6` 매칭
- 인터페이스 이름을 `mgmt1`로 변경
- IP `192.168.0.205/24` 할당
- 기본 게이트웨이 `192.168.0.1`
- DNS `4.2.2.1`, `8.8.8.8`

## 오프라인 미디어 요구사항

설치에는 `THAKI_DATA` 라벨을 가진 읽기 가능한 USB 볼륨이 필요하다.

### USB 사용 방식

- 설치 중:
  - `/mnt/usbdata`에 마운트
  - Ubuntu 패키지 저장소 소스로 사용
  - autoinstall APT 접근을 위해 로컬 HTTP 서버로 노출
- 첫 부팅 후:
  - read-only로 다시 마운트
  - Harbor, Docker, PXE, 이미지 아카이브 소스로 사용

### USB에 있어야 하는 예상 콘텐츠

- `${THAKI_DATA}/init-pkg/harbor-offline-installer-thaki-v2.14.0.tgz`
- `${THAKI_DATA}/init-pkg/docker-compose-pack-28.5.1-1.tgz`
- `${THAKI_DATA}/init-pkg/create_harbor_projects.sh`
- `${THAKI_DATA}/init-pkg/artifact-scripts.tgz`
- `${THAKI_DATA}/pxe-netboot/pxe-netboot.tgz`
- `${THAKI_DATA}/cri-images/docker-io-images.tgz`
- `${THAKI_DATA}/cri-images/quay-io-images.tgz`
- `${THAKI_DATA}/cri-images/registry-kis-io-images.tgz`
- Ubuntu 저장소 콘텐츠:
  - `noble`
  - `noble-updates`
  - `noble-security`

## 첫 부팅 구성 요소

### `cp.env`

`/usr/local/bin/cp.env`는 주요 가변 설정 파일이다.

| 변수 | 현재 값 | 용도 |
| --- | --- | --- |
| `THAKI_PXENIC` | `mgmt1` | PXE/DHCP에 사용할 인터페이스 |
| `THAKI_DATA` | `/mnt/usbdata` | 소스 USB 마운트 경로 |
| `THAKI_ADMIN` | `tkadmin` | 생성된 보조 스크립트를 받을 관리자 계정 |
| `THAKI_SRC_DATA_LABEL` | `THAKI_DATA` | USB에서 기대하는 소스 디스크 라벨 |
| `THAKI_DST_DATA_LABEL` | `TKDATA` | 대상 라벨 선언값, 현재는 미사용 |
| `THAKI_DST_DISK` | 빈 값 | `/data`용 추가 디스크. 비어 있으므로 데이터 디스크 준비 단계는 건너뜀 |
| `HARBOR_URL` | `ar.thakicloud.net` | Harbor 호스트명 |
| `HARBOR_PORT` | `38088` | Harbor 포트 |
| `HARBOR_USER` | `admin` | Harbor 관리자 계정 |
| `HARBOR_PWD` | `Harbor12345` | Harbor 관리자 비밀번호 |
| `HARBOR_PROXIES` | `docker.io ghcr.io quay.io registry.k8s.io` | Harbor/containerd 프록시 대상 레지스트리 |
| `HARBOR_PKG` | `${THAKI_DATA}/init-pkg/harbor-offline-installer-thaki-v2.14.0.tgz` | Harbor 오프라인 설치 파일 |
| `THAKI_DOCKER_PKG` | `${THAKI_DATA}/init-pkg/docker-compose-pack-28.5.1-1.tgz` | Docker 패키지 번들 |
| `HARBOR_IP` | `192.168.0.205` | `/etc/hosts`에 강제로 기록할 Harbor IP |
| `HARBOR_PROJECT_SCRIPT` | `${THAKI_DATA}/init-pkg/create_harbor_projects.sh` | Harbor 프로젝트 생성 스크립트 경로. 현재 stage에서 직접 사용하지 않음 |
| `THAKI_PXE_PKG` | `${THAKI_DATA}/pxe-netboot/pxe-netboot.tgz` | PXE/TFTP 자산 번들 |

### `lib.sh`

`/usr/local/lib/cp/lib.sh`는 공통 함수들을 제공한다.

- `log`
- `warn`
- `die`
- `require_vars`
- `run_stage`

### `init-cp.sh`

`/usr/local/sbin/init-cp.sh`의 역할은 다음과 같다.

- `/var/log/init-cp.log`로 로그 기록
- `cp.env`의 모든 값을 로드하고 export
- 핵심 환경 변수 검증
- `/usr/local/lib/cp/stages/*.sh` 목록 수집
- 파일명 순서대로 stage 실행
- 완료 상태를 `/var/lib/cp`에 기록

## Stage 상세

### `10-usbdata-mount.sh`

목적:

- `/dev/disk/by-label/${THAKI_SRC_DATA_LABEL}`를 `${THAKI_DATA}`에 read-only로 마운트
- `/etc/fstab`에 영구 등록

동작:

- 라벨 장치가 없으면 실패
- `fstab` 변경 후 systemd mount 상태를 다시 읽음

### `11-prepare-data.sh`

목적:

- 선택적으로 `/data` 전용 디스크 준비
- 선택적으로 USB 콘텐츠를 `/data/thaki-pkg`로 background `rsync`

동작:

- `THAKI_DST_DISK`가 비어 있지 않을 때만 실행
- 루트 디스크로 보이는 장치는 포맷 거부
- 이미 다른 곳에 마운트된 디스크는 포맷 거부
- ext4로 포맷
- `/data`에 마운트
- UUID 기준으로 `fstab` 등록
- background `rsync` 시작
- 로그는 `/var/log/cp-rsync-usbdata-to-data.log`
- PID는 `/run/cp-rsync.pid`에 저장

현재 실제 동작:

- `THAKI_DST_DISK=""` 이므로 이 단계는 스킵됨

### `12-apt-usb-sources.sh`

목적:

- `${THAKI_DATA}`를 가리키는 `file:` 기반 APT source로 교체

동작:

- `/etc/apt/sources.list.d/thaki-usb.sources` 생성
- `/etc/apt/sources.list.d/ubuntu.sources` 제거
- `apt-get update || true` 실행

### `14-install-docker-pkgs.sh`

목적:

- 오프라인 tarball에서 Docker 관련 패키지 설치
- Harbor를 insecure registry로 사용하도록 Docker 설정

동작:

- `${THAKI_DOCKER_PKG}`를 임시 디렉터리에 해제
- 모든 `.deb`를 `dpkg -i`로 설치
- `${THAKI_ADMIN}`을 `docker` 그룹에 추가
- Harbor 엔드포인트를 포함하는 `/etc/docker/daemon.json` 생성
- Docker 재시작

전제조건:

- 패키지 번들이 정상 동작하는 Docker 서비스를 제공해야 함

### `16-config-pxetftp.sh`

목적:

- PXE/TFTP 자산 설치
- DHCP/TFTP용 `dnsmasq` 설정

동작:

- `${THAKI_PXE_PKG}`를 `/`에 해제
- `/etc/dnsmasq.d/pxe.conf` 생성
- `dnsmasq` enable 및 start

현재 PXE 설정값:

- 인터페이스: `${THAKI_PXENIC}`, `lo`
- DHCP 범위: `192.168.0.100` ~ `192.168.0.200`
- BIOS 부트 파일: `pxelinux.0`
- UEFI x86_64 부트 파일: `bootx64.efi`
- TFTP root: `/srv/tftp`
- subnet mask 옵션: `255.255.255.0`

### `20-prepare-harbor.sh`

목적:

- 관리자 홈 디렉터리에 Harbor 풀기
- Harbor hostname 매핑 강제
- 이후 Harbor 작업용 보조 스크립트 생성

동작:

- `${HARBOR_IP} ar.thakicloud.net`를 `/etc/hosts`에 기록
- `${HARBOR_PKG}`를 관리자 홈에 해제
- `harbor.yml.thaki`를 `harbor.yml`로 복사
- 다음 스크립트 생성:
  - `0-install-harbor.sh`
  - `1-craete-harbor-projects.sh`
  - `2-config-containerd-harbor.sh`
  - `3-setup-os-mirror.sh`
  - `4-unzip-container-images.sh`

보조 스크립트 요약:

- `0-install-harbor.sh`
  - Harbor `install.sh` 실행
- `1-craete-harbor-projects.sh`
  - Harbor 프로젝트 생성
- `2-config-containerd-harbor.sh`
  - `/etc/containerd/config.toml` 생성
  - `SystemdCgroup` 활성화
  - `config_path=/etc/containerd/certs.d` 설정
  - `HARBOR_PROXIES` 기준으로 레지스트리별 `hosts.toml` 생성
- `3-setup-os-mirror.sh`
  - Harbor compose/nginx 설정을 `.thaki` 버전으로 교체
  - `proxy` 서비스 재생성
- `4-unzip-container-images.sh`
  - 이미지 아카이브 해제
  - Harbor로 artifact push

중요:

- `20-prepare-harbor.sh`는 Harbor 실행 준비까지만 하며, 생성된 보조 스크립트는 자동 실행하지 않는다

## 운영 관점

### Idempotency

- 각 stage는 `/var/lib/cp/<stage>.done` 파일을 남긴다
- stamp 파일이 있으면 재실행 시 해당 stage를 건너뛴다
- 설치된 시스템에서 특정 stage를 다시 실행하려면 해당 stamp 파일을 삭제해야 한다

### 로그

- `/var/log/init-cp.log`
- `/var/log/usb-http.log`
- `/var/log/cp-rsync-usbdata-to-data.log`

### 보안 특성

현재 파일은 하드닝보다 부트스트랩 편의성에 더 초점이 맞춰져 있다.

- SSH password 로그인 허용
- SSH public key 미사전 설정
- USB APT source를 신뢰된 저장소로 처리
- Docker가 insecure registry 설정 사용
- Harbor 계정 정보가 `cp.env`에 평문으로 저장
- Harbor hostname 해석을 `/etc/hosts`로 강제

## 변경 가능성이 큰 지점

향후 수정 시 주로 바뀔 가능성이 높은 값은 다음과 같다.

- 호스트 식별 정보:
  - hostname
  - username
  - password hash
- 설치 대상:
  - 디스크 경로
  - storage layout
- 네트워크:
  - NIC 매칭 이름
  - 고정 IP
  - gateway
  - DNS
- 오프라인 소스:
  - USB 라벨
  - 패키지 경로
  - Ubuntu suite
- PXE:
  - 인터페이스
  - DHCP 범위
  - TFTP 자산
- Harbor:
  - hostname
  - IP
  - port
  - 계정 정보
  - proxy registry 목록
- 추가 데이터 디스크:
  - `THAKI_DST_DISK`

## 알려진 공백 사항

- `THAKI_DST_DATA_LABEL`은 정의되어 있지만 사용되지 않는다.
- `HARBOR_PROJECT_SCRIPT`는 정의되어 있지만 stage 로직에서 직접 사용되지 않는다.
- `1-craete-harbor-projects.sh` 파일명에는 오타가 있어 보인다.
- Stage 번호는 향후 삽입을 위해 일부 비워 둔 구조다.
- `11-prepare-data.sh`는 `rsync`를 필요로 하지만 `autoinstall.packages`에는 `rsync`가 없다.

## 수정 가이드

[autoinstall-user-data](/Users/joon/github/airgap-autoinstall/autoinstall-user-data)를 수정할 때는 다음을 지키는 것이 안전하다.

- YAML 들여쓰기를 엄격히 유지할 것
- 설치 시점 설정과 첫 부팅 `user-data`를 섞지 말 것
- 값만 바뀌는 경우에는 먼저 `cp.env`를 수정할 것
- 동작 변경이 필요한 경우에만 stage 스크립트를 수정할 것
- 의존성이 바뀌지 않는 한 stage 순서를 유지할 것
- storage, network 변경은 부팅 불가 또는 원격 접근 불가를 만들 수 있으므로 고위험 변경으로 다룰 것
