## Depth Sensor BLE 운영 가이드

---

### 1. SSH 접속

```bash
ssh -p 8000 odroid@222.112.10.159
```

---

### 2. 기존 서비스 중지

```bash
sudo systemctl stop depth.service
```

> 먼저 하는 이유:
> 백그라운드에서 `depth.service → python3 start.py` 가 이미 실행 중 (보이지 않음)
> 중지하지 않으면 `python3 start.py` 중복 실행됨

---

### 3. depth 폴더 진입 후 수동 실행 (로그 확인용)

```bash
ls depth/
cd depth/
python3 start.py
```

---

### 실행 후 상태 정보 명령어

* `hciconfig`
  * Bluetooth 인터페이스(HCI) 상태 확인 (구식)
  * 현재 장치 상태 확인용

* `btmgmt`
  * 최신 방식의 BLE 관리 도구
  * Advertising, 페어링 등 설정 가능

* `systemctl status bluetooth`
  * BlueZ 데몬 상태 서비스 살아있는지 빠르게 확인할 때

* `sudo journalctl -u bluetooth -f`
  * BlueZ가 왜 오류 내는지 볼 때
  * 연결/끊김, 에러, BLE 동작 로그 확인 가능

* `sudo bluetoothctl show`
  * BLE 서버 실행 전/후 상태 빠르게 점검할 때

  | 항목 | 의미 |
  |------|------|
  | `Powered` | 블루투스 어댑터 켜짐/꺼짐 |
  | `Discoverable` | 앱에서 검색 가능 여부 — **no면 CONNECT 버튼 안 뜸** |
  | `Pairable` | 페어링 허용 여부 |
  | `Alias` | 앱에서 보이는 디바이스 이름 |
  | `ActiveInstances` | 현재 BLE 광고 중인 인스턴스 수 — **0x00이면 광고 안 되는 것** |
  | `UUID 목록` | 등록된 GATT 서비스 목록 |

  > 핵심 체크 항목: `Discoverable: yes` + `ActiveInstances: 0x01`
  > 이 두 개가 정상이면 앱에서 CONNECT 가능

* `sudo btmon`
  * 연결이 왜 끊기는지 패킷 레벨로 볼 때
  * 연결/끊김의 정확한 원인 코드 확인 가능
  * ex) `Remote User Terminated Connection (0x13)` — 앱 측에서 연결 해제

---

### Advertising 관리 방법

BLE 장치가 주변에 자신을 알리는 Advertising 신호는 BlueZ 스택에서 관리.
제어 방법은 크게 세 가지:

1. **최신 커맨드라인 방식 (btmgmt)**
```bash
   btmgmt advertise on
```
   * 광고 켜기/끄기, connectable 상태 등 세밀하게 제어 가능

2. **구식 커맨드라인 방식 (hciconfig)**
```bash
   hciconfig hci0 leadv
```
   * 단순하게 advertising 켜기만 가능
   * 구형 장치나 간단 테스트용

3. **Python 코드 (DBus API)**
   * Python에서 BlueZ의 DBus API 직접 호출
   * 자동화 가능, 연결 이벤트에 따라 광고 켜고 끄기 가능
   * `start.py` 에서 사용 중인 방식

---

### GATT 활성화란?

```
python3 start.py 실행 시,
BlueZ의 GATT 서버 API를 통해 서비스/캐릭터리스틱을 등록하는 것.
앱에서 연결 가능한 BLE 서버가 켜지는 것.
```

---

### 문제 발생 시 명령어

| 상황 | 명령어 |
|------|--------|
| BLE 연결이 계속 이상할 때 | `sudo systemctl restart bluetooth` |
| 서비스로 등록해서 운영할 때 중지 | `sudo systemctl stop depth.service` |
| 비정상 종료로 찌꺼기 남았을 때 | `sudo systemctl restart bluetooth` |

---

### 정상 동작 체크리스트

| 항목        | 확인 명령어              | 정상 상태                      |
| --------- | ------------------- | -------------------------- |
| BLE 서버 실행 | 터미널 로그              | `Bluetooth: Running (BLE)` |
| 광고 송출     | `bluetoothctl show` | `ActiveInstances: 0x01`    |
| 검색 가능     | `bluetoothctl show` | `Discoverable: yes`        |
| 앱 연결      | `btmon`             | `Device Connected`         |