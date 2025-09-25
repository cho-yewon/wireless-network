# 🌫️ 미세먼지 센서를 이용한 아두이노 & 라즈베리파이 연동 실습

아두이노와 라즈베리파이를 연결해 **미세먼지 센서 데이터를 수집·시각화**하는 실습 가이드입니다.  
(Arduino IDE → Python Serial → InfluxDB → Grafana)

---

## 1️⃣ 준비물
- Raspberry Pi (예: Raspberry Pi 5)
- Arduino 보드
- 미세먼지 센서
- SD카드, HDMI 케이블, 랜선, 키보드, 마우스
- 인터넷 연결

---

## 2️⃣ 라즈베리파이 구동 & 아두이노 IDE 설치

```bash
sudo apt-get install arduino
````

## 3️⃣ 미세먼지 센서 연결 (아두이노 핀맵)

| 센서핀 | 아두이노 연결     |
| --- | ----------- |
| VCC | 5V          |
| LED | 2 (Digital) |
| OUT | A0 (Analog) |
| GND | GND         |

---

## 4️⃣ 아두이노 코드 작성

**Arduino IDE**에 아래 코드를 작성 & 업로드합니다.

```cpp
int Vo = A0;
int V_LED = 2;

float Vo_value = 0;

void setup() {
  Serial.begin(9600);
  pinMode(V_LED, OUTPUT);
  pinMode(Vo, INPUT); // 오타 주의: V0가 아닌 Vo
}

void loop() {
  digitalWrite(V_LED, LOW);
  delayMicroseconds(280);
  Vo_value = analogRead(Vo);
  delayMicroseconds(40);
  digitalWrite(V_LED, HIGH);
  delayMicroseconds(9680);

  Serial.println(Vo_value);

  delay(1000);
}
```

Arduino IDE의 **시리얼 모니터**에서 값이 출력되는지 확인합니다.

---

## 5️⃣ 라즈베리파이에서 Python으로 시리얼 읽기

```bash
sudo apt install vim
```

`serial_test.py` 작성:

```python
import time
import serial

seri = serial.Serial('/dev/ttyACM0', baudrate=9600, timeout=None)

while True:
    time.sleep(1)
    if seri.in_waiting != 0:
        content = seri.readline()
        a = float(content.decode())
        print(a)
```

## 6️⃣ InfluxDB 설치 및 설정

```bash
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c influxdata-archive_compat.key' | sha256sum -c && cat influxdata-archive_compat.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list

sudo apt-get update && sudo apt-get install influxdb -y
sudo service influxdb start
sudo service influxdb status
```

DB 생성:

```sql
-- influx 콘솔에서
create database dust;
show databases;
```

---

## 7️⃣ Python → InfluxDB 데이터 전송

`serial_dust.py` 작성:

```python
import time
from influxdb import InfluxDBClient as influxdb
import serial

seri = serial.Serial('/dev/ttyACM0', baudrate=9600, timeout=None)

while True:
    time.sleep(1)
    if seri.in_waiting != 0:
        content = seri.readline()
        a = float(content.decode())

        data = [{
            'measurement': 'dust',
            'tags': {
                'InhaUni': '2222',
            },
            'fields': {
                'dust': a,
            }
        }]

        client = None
        try:
            client = influxdb('localhost', 8086, 'root', 'root', 'dust')
        except Exception as e:
            print("Exception " + str(e))

        if client is not None:
            try:
                client.write_points(data)
            except Exception as e:
                print("Exception write " + str(e))
            finally:
                client.close()
                print("running influsdb OK")
```

### ⚠️ influxdb 라이브러리 오류 시 해결:

```bash
sudo rm /usr/lib/python3.11/EXTERNALLY-MANAGED
pip install influxdb
```

---

## 8️⃣ Grafana 설치 및 시각화

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update && sudo apt-get install grafana -y
sudo systemctl start grafana-server
```

초기 로그인:

* **ID**: `admin`
* **PW**: `admin` (변경하지 않음)

---

### Grafana 데이터 소스 연결

* `Connections → Add new connection → InfluxDB` 선택
* **Name**: 아무거나
* **URL**: `http://localhost:8086`
* **Database**: `dust`
* **User**: `root`

> `datasource is working. 1 measurements found` 메시지가 뜨면 성공입니다.

---

### 대시보드 구성

1. `Dashboard → Create dashboard → Add visualization`
2. 데이터 소스로 우리가 만든 `dust` 선택
3. 쿼리 설정(InfluxQL 기준):

   * **FROM**: `dust`
   * **WHERE**: `"InhaUni" = '2222'`
   * **SELECT**: `field("dust")`

그래프가 정상적으로 출력되는지 확인합니다.
