# 공공데이터 API 미세먼지 수집 + TinyOS 환경 구축 & LED 제어

1) **공공데이터포털(에어코리아) API**로 미세먼지(PM10/PM2.5) 수집  
2) **TinyOS** 환경 구축  
3) **TinyOS 예제(nesC)**로 LED 1개/3개 점멸 제어

---

## 1. 공공데이터포털 에어코리아 API 신청

1) 접속: https://www.data.go.kr  
2) 회원가입 → 로그인  
3) 검색창에 **“미세먼지”** 검색  
4) **오픈 API - 한국환경공단_에어코리아_대기오염정보** 선택  
5) **활용신청** 진행  
6) **측정소별 실시간 측정정보 조회**(getMsrstnAcctoRltmMesureDnsty) **활용신청**

---

## 2. Python 코드로 미세먼지 수집 (BeautifulSoup + requests)

### 2.1 라이브러리 설치 (Windows)

```bat
cd C:\Users\PC\AppData\Local\Programs\Python\Python313\Scripts
pip3.13.exe install bs4
pip3.13.exe install requests
````

### 2.2 `dustAPI.py` 작성
```python
from bs4 import BeautifulSoup
from urllib.parse import urlencode, quote_plus
import requests

# 본인의 서비스 키 입력
service_key = '여기에_서비스키_입력'

url = 'http://apis.data.go.kr/B552584/ArpltnInquireSvc/getMsrstnAcctoRltmMesureDnsty'

queryParams = '?' + urlencode({
    quote_plus('serviceKey'): service_key,
    quote_plus('returnType'): 'xml',
    quote_plus('numOfRows'): '10',
    quote_plus('pageNo'): '1',
    quote_plus('stationName'): '주안',
    quote_plus('ver'): '1.0'
})

res = requests.get(url + queryParams)
soup = BeautifulSoup(res.content, 'xml')
data = soup.find_all('item')
print(data)

for item in data:
    dataterm = item.find('dataTime')
    pm25value = item.find('pm25Value')
    if pm25value and dataterm:
        print(pm25value.get_text())
        print(dataterm.get_text())
```

### 2.3 실행

```bat
python dustAPI.py
```
---

## 3. TinyOS 환경 구축

### 3.1 Cygwin 설치

1. [https://www.cygwin.com/install.html](https://www.cygwin.com/install.html) 접속 → **setup-x86\_64.exe** 다운로드/실행
2. 기본 Next 진행 → **Select Your Internet Connection**에서 **Direct Connection** 선택
3. **Choose A Download Site**에서 `https://ftp.kaist.ac.kr` 선택 → 설치 완료

### 3.2 FTDI VCP 드라이버 설치

* 구글에서 **“FTDI VCP Drivers”** 검색 → 제조사 페이지 진입
* **VCP Drivers** 다운로드/설치

### 3.3 MSVCR71.dll 오류 해결 (필요 시)

* 구글에서 **“MSVCR71.dll”** 검색
* **웅카콜라의 티스토리** 에서 다운로드
* 파일명이 이상할 경우 **반드시 `m_71_auto.exe`로 변경** 후 실행
---

## 4. TinyOS 예제 — LED 점멸 (1개)

아래 3개 파일을 같은 폴더에 준비합니다.

### 4.1 `BlinkAppC.nc` (configuration)

```c
configuration BlinkAppC { }
implementation {
  components MainC, BlinkC, LedsC;
  components new TimerMilliC() as Timer;

  BlinkC -> MainC.Boot;
  BlinkC.Timer -> Timer;
  BlinkC.Leds -> LedsC;
}
```

### 4.2 `BlinkC.nc` (module)

```c
module BlinkC @safe() {
  uses interface Timer<TMilli> as Timer;
  uses interface Leds;
  uses interface Boot;
}
implementation {
  event void Boot.booted() {
    call Timer.startPeriodic(250);
  }

  event void Timer.fired() {
    call Leds.led0Toggle();
  }
}
```

### 4.3 `Makefile`

```make
COMPONENT=BlinkAppC
include $(MAKERULES)
```

### 4.4 빌드/설치

```bash
make telosb install.10
```

---

## 5. TinyOS 예제 — LED 3개 제어 (0/1/2번 각각 주기 다르게)

아래 3개 파일을 새 폴더에 준비합니다.

### 5.1 `BlinkAppC.nc`

```c
configuration BlinkAppC { }
implementation {
  components MainC, BlinkC, LedsC;
  components new TimerMilliC() as Timer0;
  components new TimerMilliC() as Timer1;
  components new TimerMilliC() as Timer2;

  BlinkC -> MainC.Boot;

  BlinkC.Timer0 -> Timer0;
  BlinkC.Timer1 -> Timer1;
  BlinkC.Timer2 -> Timer2;

  BlinkC.Leds -> LedsC;
}
```

### 5.2 `BlinkC.nc`

```c
module BlinkC @safe() {
  uses interface Timer<TMilli> as Timer0;
  uses interface Timer<TMilli> as Timer1;
  uses interface Timer<TMilli> as Timer2;
  uses interface Leds;
  uses interface Boot;
}
implementation {
  event void Boot.booted() {
    call Timer0.startPeriodic(250);
    call Timer1.startPeriodic(500);
    call Timer2.startPeriodic(750);
  }

  event void Timer0.fired() {
    call Leds.led0Toggle();
  }
  event void Timer1.fired() {
    call Leds.led1Toggle();
  }
  event void Timer2.fired() {
    call Leds.led2Toggle();
  }
}
```

### 5.3 `Makefile`

```make
COMPONENT=BlinkAppC
include $(MAKERULES)
```

### 5.4 빌드/설치

```bash
make telosb install.10
```

설치 후 **LED 3개가 서로 다른 주기로 깜빡**이면 성공입니다.

---
