# Bluetooth Interface Specification

DepthSensor 시스템과 Flutter 앱 간의 BLE GATT 통신 인터페이스 명세서입니다.

---

## 연결 정보

| 항목 | 값 |
|------|-----|
| **Device Name** | `DepthMain` |
| **Protocol** | Bluetooth Low Energy (BLE) GATT |
| **MTU** | 512 bytes (권장) |
| **연결 거리** | 3m 이내 권장 |

---

## Service & Characteristic UUID

### UUID 상수 (Flutter/Dart)

```dart
class DepthSensorBLE {
  // Device Name (스캔 시 필터용)
  static const String deviceName = 'DepthMain';

  // Service UUIDs
  static const String sensorServiceUuid    = '12345678-1234-5678-1234-56789abca001';
  static const String detectionServiceUuid = '12345678-1234-5678-1234-56789abca010';
  static const String statusServiceUuid    = '12345678-1234-5678-1234-56789abca020';
  static const String configServiceUuid    = '12345678-1234-5678-1234-56789abca030';

  // Characteristic UUIDs
  static const String gpsCharUuid       = '12345678-1234-5678-1234-56789abca002';
  static const String mpuCharUuid       = '12345678-1234-5678-1234-56789abca003';
  static const String detectionCharUuid = '12345678-1234-5678-1234-56789abca011';
  static const String statusCharUuid    = '12345678-1234-5678-1234-56789abca021';
  static const String classesCharUuid   = '12345678-1234-5678-1234-56789abca031';
}
```

> **중요**: Flutter BLE 라이브러리는 UUID를 **소문자**로 사용해야 합니다.

---

## Service 구조

### 1. Sensor Service (`...a001`)

GPS 및 IMU 센서 데이터 제공

| Characteristic | UUID | 속성 | 설명 |
|----------------|------|------|------|
| GPS | `...a002` | Read, Notify | GPS NMEA 데이터 |
| MPU | `...a003` | Read, Notify | 가속도/자이로 데이터 |

### 2. Detection Service (`...a010`)

객체 탐지 결과 제공

| Characteristic | UUID | 속성 | 설명 |
|----------------|------|------|------|
| Detection | `...a011` | Read, Notify | 탐지된 객체 목록 |

### 3. Status Service (`...a020`)

장치 상태 정보 제공

| Characteristic | UUID | 속성 | 설명 |
|----------------|------|------|------|
| Status | `...a021` | Read, Notify | 장치 상태 (카메라, GPS, MPU, FPS) |

### 4. Config Service (`...a030`)

설정 및 메타데이터 제공

| Characteristic | UUID | 속성 | 설명 |
|----------------|------|------|------|
| Classes | `...a031` | Read | 탐지 클래스 이름 목록 (29개) |

---

## 데이터 포맷

모든 데이터는 **UTF-8 인코딩 문자열**입니다.

### GPS 데이터 (`...a002`)

**포맷**: NMEA 원본 문자열

```
$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47
```

**파싱 예시 (Dart)**:

```dart
void parseGPS(List<int> data) {
  String nmea = String.fromCharCodes(data);

  if (nmea == 'NO_DATA') {
    // GPS 데이터 없음
    return;
  }

  // NMEA 파싱 로직
  if (nmea.startsWith('\$GPGGA')) {
    List<String> parts = nmea.split(',');
    // parts[2]: 위도, parts[4]: 경도 등
  }
}
```

| 필드 | 설명 |
|------|------|
| 최대 크기 | 512 bytes |
| 업데이트 주기 | 1Hz |
| 초기값 | `NO_DATA` |

---

### MPU 데이터 (`...a003`)

**포맷**: CSV (`ax,ay,az,gx,gy,gz`)

```
0.12,-0.98,9.81,0.01,0.02,-0.03
```

**파싱 예시 (Dart)**:

```dart
class MPUData {
  final double ax, ay, az;  // 가속도 (m/s²)
  final double gx, gy, gz;  // 자이로 (rad/s)

  factory MPUData.fromBLE(List<int> data) {
    String csv = String.fromCharCodes(data);
    List<String> parts = csv.split(',');

    return MPUData(
      ax: double.parse(parts[0]),
      ay: double.parse(parts[1]),
      az: double.parse(parts[2]),
      gx: double.parse(parts[3]),
      gy: double.parse(parts[4]),
      gz: double.parse(parts[5]),
    );
  }
}
```

| 필드 | 단위 | 범위 |
|------|------|------|
| ax, ay, az | m/s² | ±16g |
| gx, gy, gz | rad/s | ±2000°/s |
| 업데이트 주기 | - | 1Hz |

---

### Detection 데이터 (`...a011`)

**포맷**: 줄바꿈으로 구분된 CSV

```
0,0.95,1200,100,200,50,80
5,0.87,2500,300,150,60,90
```

각 줄: `class_id,score,distance,x,y,width,height`

**파싱 예시 (Dart)**:

```dart
class Detection {
  final int classId;      // 클래스 ID (0-28)
  final double score;     // 신뢰도 (0.0-1.0)
  final int distance;     // 거리 (mm)
  final int x, y;         // 바운딩 박스 좌상단
  final int width, height; // 바운딩 박스 크기

  factory Detection.fromCSV(String line) {
    List<String> parts = line.split(',');
    return Detection(
      classId: int.parse(parts[0]),
      score: double.parse(parts[1]),
      distance: int.parse(parts[2]),
      x: int.parse(parts[3]),
      y: int.parse(parts[4]),
      width: int.parse(parts[5]),
      height: int.parse(parts[6]),
    );
  }
}

List<Detection> parseDetections(List<int> data) {
  String raw = String.fromCharCodes(data);

  if (raw == 'NO_DETECTION') {
    return [];
  }

  return raw.split('\n')
    .where((line) => line.isNotEmpty)
    .map((line) => Detection.fromCSV(line))
    .toList();
}
```

| 필드 | 설명 |
|------|------|
| class_id | 객체 클래스 ID (Classes 참조) |
| score | 탐지 신뢰도 (0.0~1.0) |
| distance | 객체까지 거리 (mm) |
| x, y | 바운딩 박스 좌상단 좌표 (픽셀) |
| width, height | 바운딩 박스 크기 (픽셀) |
| 최대 객체 수 | 5개 |
| 초기값 | `NO_DETECTION` |

---

### Status 데이터 (`...a021`)

**포맷**: CSV (`camera,gps,mpu,fps`)

```
1,1,1,15
```

**파싱 예시 (Dart)**:

```dart
class DeviceStatus {
  final bool cameraOk;
  final bool gpsOk;
  final bool mpuOk;
  final int fps;

  factory DeviceStatus.fromBLE(List<int> data) {
    String csv = String.fromCharCodes(data);
    List<String> parts = csv.split(',');

    return DeviceStatus(
      cameraOk: parts[0] == '1',
      gpsOk: parts[1] == '1',
      mpuOk: parts[2] == '1',
      fps: int.parse(parts[3]),
    );
  }
}
```

| 필드 | 값 | 설명 |
|------|-----|------|
| camera | 0 또는 1 | 카메라 상태 |
| gps | 0 또는 1 | GPS 상태 |
| mpu | 0 또는 1 | IMU 상태 |
| fps | 0-30 | 현재 처리 FPS |

---

### Classes 데이터 (`...a031`)

**포맷**: 쉼표로 구분된 클래스 이름

```
bicycle,bus,car,truck,signage,person,wheelchair,stroller,kickboard,motorcycle,traffic_light,fire_hydrant,bench,chair,table,tree,plant,curb,pole,bollard,barricade,cone,manhole,pothole,crosswalk,speed_bump,tactile_paving,stairs,ramp
```

**파싱 예시 (Dart)**:

```dart
class ClassMapper {
  late List<String> _classes;

  void loadFromBLE(List<int> data) {
    String raw = String.fromCharCodes(data);
    _classes = raw.split(',');
  }

  String getClassName(int classId) {
    if (classId >= 0 && classId < _classes.length) {
      return _classes[classId];
    }
    return 'unknown';
  }

  int get classCount => _classes.length;
}
```

| 속성 | 설명 |
|------|------|
| Read-only | 앱 시작 시 1회 읽기 |
| 클래스 수 | 29개 |
| 인덱스 | 0부터 시작 |

**클래스 목록 (29개)**:

| ID | 이름 | 한글 |
|----|------|------|
| 0 | bicycle | 자전거 |
| 1 | bus | 버스 |
| 2 | car | 승용차 |
| 3 | truck | 트럭 |
| 4 | signage | 표지판 |
| 5 | person | 사람 |
| 6 | wheelchair | 휠체어 |
| 7 | stroller | 유모차 |
| 8 | kickboard | 킥보드 |
| 9 | motorcycle | 오토바이 |
| 10 | traffic_light | 신호등 |
| 11 | fire_hydrant | 소화전 |
| 12 | bench | 벤치 |
| 13 | chair | 의자 |
| 14 | table | 테이블 |
| 15 | tree | 나무 |
| 16 | plant | 화분 |
| 17 | curb | 연석 |
| 18 | pole | 전봇대 |
| 19 | bollard | 볼라드 |
| 20 | barricade | 바리케이드 |
| 21 | cone | 콘 |
| 22 | manhole | 맨홀 |
| 23 | pothole | 포트홀 |
| 24 | crosswalk | 횡단보도 |
| 25 | speed_bump | 과속방지턱 |
| 26 | tactile_paving | 점자블록 |
| 27 | stairs | 계단 |
| 28 | ramp | 경사로 |

---

## Flutter 연동 가이드

### 권장 패키지

```yaml
dependencies:
  flutter_blue_plus: ^1.31.0  # BLE 통신
```

### 연결 흐름

```dart
import 'package:flutter_blue_plus/flutter_blue_plus.dart';

class DepthSensorConnection {
  BluetoothDevice? _device;
  Map<String, BluetoothCharacteristic> _chars = {};

  // 1. 스캔 및 연결
  Future<void> connect() async {
    // 스캔
    await FlutterBluePlus.startScan(
      withNames: ['DepthMain'],
      timeout: Duration(seconds: 10),
    );

    FlutterBluePlus.scanResults.listen((results) async {
      for (var r in results) {
        if (r.device.platformName == 'DepthMain') {
          await FlutterBluePlus.stopScan();
          _device = r.device;
          await _device!.connect();
          await _discoverServices();
          break;
        }
      }
    });
  }

  // 2. 서비스 탐색
  Future<void> _discoverServices() async {
    List<BluetoothService> services = await _device!.discoverServices();

    for (var service in services) {
      for (var char in service.characteristics) {
        _chars[char.uuid.toString()] = char;
      }
    }

    // 3. Classes 읽기 (1회)
    await _readClasses();

    // 4. Notification 활성화
    await _enableNotifications();
  }

  // Classes 읽기
  Future<void> _readClasses() async {
    var char = _chars[DepthSensorBLE.classesCharUuid];
    if (char != null) {
      List<int> data = await char.read();
      // 파싱...
    }
  }

  // Notification 활성화
  Future<void> _enableNotifications() async {
    // Detection (필수)
    var detChar = _chars[DepthSensorBLE.detectionCharUuid];
    if (detChar != null) {
      await detChar.setNotifyValue(true);
      detChar.onValueReceived.listen(_onDetection);
    }

    // Status
    var statusChar = _chars[DepthSensorBLE.statusCharUuid];
    if (statusChar != null) {
      await statusChar.setNotifyValue(true);
      statusChar.onValueReceived.listen(_onStatus);
    }

    // GPS (선택)
    var gpsChar = _chars[DepthSensorBLE.gpsCharUuid];
    if (gpsChar != null) {
      await gpsChar.setNotifyValue(true);
      gpsChar.onValueReceived.listen(_onGPS);
    }
  }

  void _onDetection(List<int> data) {
    // Detection 데이터 처리
  }

  void _onStatus(List<int> data) {
    // Status 데이터 처리
  }

  void _onGPS(List<int> data) {
    // GPS 데이터 처리
  }
}
```

### 재연결 처리

```dart
_device!.connectionState.listen((state) {
  if (state == BluetoothConnectionState.disconnected) {
    // 재연결 시도
    Future.delayed(Duration(seconds: 2), () {
      connect();
    });
  }
});
```

---

## 주의사항

### 1. Notification 재활성화

연결이 끊겼다가 재연결되면 **반드시 Notification을 다시 활성화**해야 합니다.

### 2. MTU 설정

최적의 성능을 위해 MTU를 512로 설정하세요:

```dart
await _device!.requestMtu(512);
```

### 3. 연결 거리

안정적인 연결을 위해 **3m 이내** 거리를 유지하세요.

### 4. 데이터 업데이트 주기

| Characteristic | 주기 |
|----------------|------|
| GPS | 1Hz |
| MPU | 1Hz |
| Detection | ~15Hz (FPS 기반) |
| Status | 1Hz |

### 5. 에러 처리

```dart
// 데이터가 없는 경우
if (raw == 'NO_DATA' || raw == 'NO_DETECTION') {
  // 빈 상태 처리
}

// 파싱 에러
try {
  // 파싱 로직
} catch (e) {
  print('Parse error: $e');
}
```

---

## 테스트

### nRF Connect 앱으로 테스트

1. [nRF Connect](https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp) 설치
2. `DepthMain` 스캔 및 연결
3. 4개 서비스, 5개 Characteristic 확인
4. Notification 활성화 테스트

### 체크리스트

- [ ] `DepthMain` 장치 스캔됨
- [ ] 4개 서비스 발견됨
- [ ] Classes Characteristic 읽기 성공 (29개 클래스)
- [ ] Detection Notification 수신됨
- [ ] Status Notification 수신됨
- [ ] 재연결 후 Notification 재활성화 동작

---

## 버전 정보

| 항목 | 값 |
|------|-----|
| 문서 버전 | 1.0 |
| 프로토콜 버전 | 1.0 |
| 최종 수정 | 2025.01 |
