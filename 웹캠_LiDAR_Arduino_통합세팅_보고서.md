# 웹캠 LiDAR Arduino 통합 세팅 보고서

## 1. 목적

이 문서는 다른 PC에서 웹캠, LiDAR, Arduino, 모터 드라이버를 연동하여 차체 주행 제어 시스템을 테스트하기 위한 전체 설치 및 실행 절차를 정리한 문서이다.

최종 목표는 다음과 같다.

```text
웹캠
-> 목표 물체 인식
-> 목표 방향 판단

LiDAR
-> 차체 주변 안전박스 감지
-> 정지/감속 판단

Python / VS Code
-> 웹캠 판단 + LiDAR 판단 통합
-> 최종 주행 명령 생성
-> Arduino로 Serial 명령 전송

Arduino
-> Serial 명령 수신
-> 모터 드라이버 제어
-> 좌우 DC 모터 제어
```

## 2. 전체 시스템 구조

```text
PC / 노트북
├─ 웹캠 USB 연결
├─ LiDAR USB 연결
├─ Arduino USB 연결
│
├─ Python 코드 실행
│  ├─ 웹캠 영상 처리
│  ├─ YOLO 물체 인식
│  ├─ LiDAR 거리 처리
│  ├─ 안전박스 판단
│  └─ Arduino로 주행 명령 전송
│
└─ Arduino
   ├─ 모터 드라이버 연결
   ├─ 왼쪽 DC 모터 연결
   └─ 오른쪽 DC 모터 연결
```

## 3. 필요한 하드웨어

필수 하드웨어:

```text
1. Windows PC 또는 노트북
2. USB 웹캠 1개
3. YDLIDAR X2 또는 호환 LiDAR 1개
4. Arduino 보드 1개
5. 모터 드라이버 1개
6. DC 모터 2개
7. 모터용 배터리
8. USB 케이블
9. 점퍼 케이블
```

권장 하드웨어:

```text
1. 비상 정지 스위치
2. 모터 전원용 퓨즈
3. 공통 GND 연결용 단자
4. 안정적인 외부 전원
```

## 4. 필요한 프로그램

다른 PC에서 새로 세팅할 경우 아래 프로그램이 필요하다.

```text
1. Python 3.10 이상
2. VS Code
3. VS Code Python 확장
4. Arduino IDE
5. 필요 시 CH340 드라이버
```

Arduino 호환 보드가 CH340 USB 칩을 사용하는 경우, Windows 장치 관리자에서 COM 포트가 잡히지 않을 수 있다. 이 경우 CH340 드라이버를 설치해야 한다.

## 5. Python 라이브러리 설치

VS Code 터미널 또는 PowerShell에서 아래 명령을 실행한다.

```powershell
pip install pyserial opencv-python ultralytics numpy pillow
```

라이브러리 역할:

```text
pyserial
-> Arduino 및 LiDAR COM 포트 통신

opencv-python
-> 웹캠 영상 표시 및 처리

ultralytics
-> YOLO 물체 인식

numpy
-> 이미지 및 수치 처리

pillow
-> GUI 이미지 표시 보조
```

설치 확인:

```powershell
python -c "import serial, cv2, ultralytics, numpy, PIL; print('OK')"
```

`OK`가 출력되면 기본 Python 라이브러리 설치가 완료된 것이다.

## 6. 폴더 구성

권장 폴더 구성:

```text
project_folder
├─ Lidar_webcam_drive_decision.py
├─ Lidar_yolo_safetybox.py
├─ Lidar_safetybox_radar.py
├─ webcam_direction_test.py
├─ 모터2개_시리얼테스트.py
├─ best.pt
└─ README 또는 보고서 파일
```

중요:

```text
best.pt 파일은 Python 코드와 같은 폴더에 두는 것을 권장한다.
```

현재 테스트용 `best.pt`가 기본 YOLOv8n 모델이라면 `bottle`, `person`, `cup` 같은 일반 객체를 인식할 수 있다.

재활용 분류용 학습 모델을 사용할 경우, 기존 `best.pt`를 학습된 `best.pt`로 교체하면 된다.

## 7. 하드웨어 연결

### 7.1 PC 연결

```text
웹캠 -> PC USB
LiDAR -> PC USB
Arduino -> PC USB
```

각 장치는 서로 다른 포트로 인식되어야 한다.

예시:

```text
LiDAR   -> COM3
Arduino -> COM4
웹캠    -> CAMERA_INDEX = 2
```

### 7.2 Arduino와 모터 드라이버 연결

모터 드라이버 종류에 따라 핀 이름이 다를 수 있다.

일반적인 연결 개념:

```text
Arduino 디지털/PWM 핀
-> 모터 드라이버 입력 핀

모터 드라이버 출력 A
-> 왼쪽 모터

모터 드라이버 출력 B
-> 오른쪽 모터

모터 배터리
-> 모터 드라이버 전원

Arduino GND
-> 모터 드라이버 GND
-> 배터리 GND
```

중요:

```text
Arduino와 모터 드라이버는 GND를 반드시 공유해야 한다.
모터 전원은 Arduino 5V에서 직접 공급하지 않는다.
모터는 별도 배터리 또는 전원 공급장치를 사용한다.
```

## 8. COM 포트 확인

Windows에서 확인:

```text
장치 관리자
-> 포트(COM & LPT)
```

Python에서 확인:

```powershell
python -m serial.tools.list_ports
```

또는 코드 실행 시 출력되는 COM 포트 목록을 확인한다.

설정해야 할 값:

```python
PORT = "COM3"        # LiDAR
MOTOR_PORT = "COM4"  # Arduino
CAMERA_INDEX = 2     # 웹캠
```

## 9. Arduino Serial 명령 규격

Python은 Arduino에 문자열 명령을 보낸다.

명령 형식:

```text
FORWARD\n
TURN_LEFT\n
TURN_RIGHT\n
STOP\n
```

Arduino는 줄바꿈 문자 `\n` 기준으로 명령을 읽는다.

권장 baudrate:

```text
115200
```

Python:

```python
MOTOR_BAUDRATE = 115200
```

Arduino:

```cpp
Serial.begin(115200);
```

## 10. 2모터 기본 동작 정의

초기 테스트는 좌우 모터 2개만 사용한다.

```text
FORWARD
-> 왼쪽 모터 전진
-> 오른쪽 모터 전진

TURN_LEFT
-> 왼쪽 모터 정지
-> 오른쪽 모터 전진

TURN_RIGHT
-> 왼쪽 모터 전진
-> 오른쪽 모터 정지

STOP
-> 왼쪽 모터 정지
-> 오른쪽 모터 정지
```

## 11. Arduino 코드 요구사항

Arduino 코드는 다음 기능을 가져야 한다.

```text
1. Serial.begin(115200)
2. Serial.readStringUntil('\n')으로 명령 수신
3. 명령 문자열 trim 처리
4. FORWARD, TURN_LEFT, TURN_RIGHT, STOP 처리
5. 알 수 없는 명령은 STOP 처리 권장
```

Arduino 로직 예시:

```cpp
String command = Serial.readStringUntil('\n');
command.trim();

if (command == "FORWARD") {
  // left motor forward
  // right motor forward
}
else if (command == "TURN_LEFT") {
  // left motor stop
  // right motor forward
}
else if (command == "TURN_RIGHT") {
  // left motor forward
  // right motor stop
}
else if (command == "STOP") {
  // both motors stop
}
else {
  // unknown command safety stop
}
```

## 12. 테스트 단계

### 12.1 1단계: 웹캠 단독 테스트

실행 파일:

```text
webcam_direction_test.py
```

실행:

```powershell
python webcam_direction_test.py
```

확인 내용:

```text
1. 웹캠 화면이 뜨는가
2. CAMERA_INDEX가 맞는가
3. YOLO 박스가 물체에 표시되는가
4. 목표 물체가 LEFT / CENTER / RIGHT로 구분되는가
```

웹캠이 안 뜨면:

```python
CAMERA_INDEX = 0
CAMERA_INDEX = 1
CAMERA_INDEX = 2
```

순서대로 바꿔 테스트한다.

### 12.2 2단계: LiDAR 단독 테스트

실행 파일:

```text
Lidar_safetybox_radar.py
```

실행:

```powershell
python Lidar_safetybox_radar.py
```

확인 내용:

```text
1. LiDAR COM 포트가 맞는가
2. 레이더 점이 표시되는가
3. 차체 박스가 표시되는가
4. 정지 박스/감속 박스 안 물체가 감지되는가
```

LiDAR가 안 잡히면:

```python
PORT = "COM3"
BAUDRATE = 128000
```

값을 확인한다.

### 12.3 3단계: Arduino 모터 단독 테스트

실행 파일:

```text
모터2개_시리얼테스트.py
```

실행:

```powershell
python 모터2개_시리얼테스트.py
```

키 입력:

```text
w -> FORWARD
a -> TURN_LEFT
d -> TURN_RIGHT
s -> STOP
exit -> 종료
```

확인 내용:

```text
1. Arduino COM 포트가 맞는가
2. w 입력 시 양쪽 모터가 도는가
3. a 입력 시 오른쪽 모터만 도는가
4. d 입력 시 왼쪽 모터만 도는가
5. s 입력 시 양쪽 모터가 멈추는가
```

### 12.4 4단계: 웹캠 + LiDAR 통합 판단 테스트

실행 파일:

```text
Lidar_webcam_drive_decision.py
```

실행:

```powershell
python Lidar_webcam_drive_decision.py
```

확인 내용:

```text
1. 웹캠 창이 뜨는가
2. LiDAR 레이더 창이 뜨는가
3. 목표 물체가 인식되는가
4. 목표 방향이 LEFT / CENTER / RIGHT로 바뀌는가
5. LiDAR 위험 감지 시 STOP이 우선되는가
6. 최종 명령이 화면에 표시되는가
```

### 12.5 5단계: 최종 Arduino 연동

최종 통합 시에는 아래 흐름이 필요하다.

```text
웹캠 판단
-> LiDAR 안전 판단
-> 최종 주행 명령 생성
-> Arduino로 Serial 전송
-> 모터 제어
```

권장 방식:

```text
이전 명령과 현재 명령이 다를 때만 Arduino로 전송한다.
```

예시:

```python
if command != last_command:
    send_command_to_arduino(command)
    last_command = command
```

## 13. 주행 판단 우선순위

최종 판단 우선순위:

```text
1. LiDAR 정지 박스 위험
   -> STOP

2. LiDAR 전방 위험
   -> STOP 또는 회피

3. 웹캠 목표 없음
   -> SEARCH 또는 STOP

4. 목표가 왼쪽
   -> TURN_LEFT

5. 목표가 오른쪽
   -> TURN_RIGHT

6. 목표가 중앙 + LiDAR 감속
   -> SLOW_FORWARD

7. 목표가 중앙 + 안전
   -> FORWARD
```

초기 2모터 테스트에서는 `SLOW_FORWARD`, `SEARCH`, `AVOID_LEFT`, `AVOID_RIGHT`는 나중에 확장해도 된다.

기본 4개 명령부터 안정화한다.

```text
FORWARD
TURN_LEFT
TURN_RIGHT
STOP
```

## 14. 자주 발생하는 문제

### 14.1 웹캠이 안 열림

원인:

```text
CAMERA_INDEX가 다름
다른 프로그램이 웹캠을 사용 중
드라이버 문제
```

해결:

```python
CAMERA_INDEX = 0
CAMERA_INDEX = 1
CAMERA_INDEX = 2
```

순서대로 테스트한다.

### 14.2 LiDAR 점이 안 보임

원인:

```text
COM 포트 오류
baudrate 오류
LiDAR 전원 문제
다른 프로그램이 포트 사용 중
```

확인:

```python
PORT = "COM3"
BAUDRATE = 128000
```

### 14.3 Arduino가 안 잡힘

원인:

```text
COM 포트 오류
CH340 드라이버 미설치
Arduino IDE Serial Monitor가 포트 사용 중
```

해결:

```text
장치 관리자에서 COM 포트 확인
Arduino IDE Serial Monitor 닫기
필요 시 CH340 드라이버 설치
```

### 14.4 모터가 안 돎

원인:

```text
모터 전원 없음
모터 드라이버 배선 오류
GND 미공유
PWM/방향 핀 불일치
Arduino 코드의 핀 번호 오류
```

확인:

```text
Arduino GND와 모터 드라이버 GND가 연결되어 있는가
모터 전원은 별도로 공급되는가
모터 드라이버 핀 번호가 Arduino 코드와 일치하는가
```

## 15. 다른 AI 모듈에 전달할 핵심 요구사항

Arduino 코드를 작성할 AI 모듈에 전달할 핵심 조건:

```text
1. Python은 Serial로 문자열 명령을 보낸다.
2. 명령은 줄바꿈 문자 '\n'으로 끝난다.
3. Arduino는 Serial.readStringUntil('\n')으로 명령을 읽는다.
4. baudrate는 115200이다.
5. 우선 지원 명령은 FORWARD, TURN_LEFT, TURN_RIGHT, STOP이다.
6. 좌우 DC 모터 2개를 사용한다.
7. FORWARD는 양쪽 모터 전진이다.
8. TURN_LEFT는 오른쪽 모터만 전진이다.
9. TURN_RIGHT는 왼쪽 모터만 전진이다.
10. STOP은 양쪽 모터 정지이다.
11. 알 수 없는 명령을 받으면 안전상 STOP 처리한다.
```

## 16. 최종 요약

이 시스템은 웹캠, LiDAR, Arduino를 결합하여 목표 물체 추적 주행을 구현하기 위한 구조이다.

```text
웹캠
-> 목표 물체 방향 판단

LiDAR
-> 안전박스 기반 위험 판단

Python
-> 최종 주행 명령 생성

Arduino
-> 명령 수신 후 모터 제어
```

초기 테스트는 반드시 아래 순서로 진행한다.

```text
1. 웹캠 단독 테스트
2. LiDAR 단독 테스트
3. Arduino 모터 단독 테스트
4. 웹캠 + LiDAR 통합 판단 테스트
5. Arduino 모터 제어 연동
```

처음부터 전체를 동시에 연결하면 원인 파악이 어렵다. 반드시 센서와 모터를 개별 테스트한 뒤 통합한다.
