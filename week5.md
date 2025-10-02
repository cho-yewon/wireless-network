# 라즈베리파이 + USB 카메라 연동 & 텔레그램 봇 타이머 촬영

라즈베리파이에 USB 카메라를 연결해 **실시간 영상 확인** 후, 텔레그램 봇으로 **N초 뒤 사진 촬영 & 전송**까지 구현합니다.

---

## 1) 라즈베리파이 + 카메라 연동

### 1-1. 카메라 연결 확인
```bash
lsusb
````

### 1-2. OpenCV 설치

```bash
pip install opencv-python
```

### 1-3. 테스트 스크립트 작성: `test_camera.py`

```python
import cv2

cam = cv2.VideoCapture(0)

if not cam.isOpened():  # isOpend -> isOpened
    print("Error: Could not open video device.")
    exit()

while True:
    ret, image = cam.read()
    if not ret:
        print("Error: Failed to grab frame")
        break

    cv2.imshow('USB Camera Feed', image)

    # 'q' 키로 종료
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

# 한 장 저장 (원하면 루프 안에서 조건 저장)
cv2.imwrite('/home/pi/captured_image.jpg', image)  # cautred -> captured

cam.release()
cv2.destroyAllWindows()
```

### 1-4. 실행

```bash
python test_camera.py
```

* 카메라 창이 뜨고, `q`를 누르면 종료되며 `/home/pi/captured_image.jpg`가 저장됩니다.

---

## 2) 텔레그램 봇 만들기

1. 텔레그램에서 **BotFather** 검색 → 채팅 시작
2. `/start` → `/newbot` → 봇 이름 지정
3. 발급된 **Bot Token** 복사

---

## 3) 라즈베리파이에서 텔레그램 라이브러리 설치

```bash
pip install "python-telegram-bot[job-queue]"
git clone https://github.com/python-telegram-bot/python-telegram-bot --recursive
```

(예제 파일 경로 참고: `/home/pi/python-telegram-bot/examples/timerbot.py`)

---

## 4) 타이머 촬영 봇 만들기

> 사용자가 `/set 5` 처럼 **초**를 입력하면, 그 시간 뒤에 USB 카메라로 **사진을 찍어 전송**합니다.

### 4-1. `timerbot.py` 작성

```python
import cv2
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# 사진 찍는 함수
def capture_image(filename="/tmp/captured_image.jpg"):
    cam = cv2.VideoCapture(0)  # 0번 카메라
    if not cam.isOpened():
        print("Error: Could not open video device.")
        return None

    ret, frame = cam.read()
    if not ret:
        print("Error: Failed to grab frame.")
        cam.release()
        return None

    cv2.imwrite(filename, frame)  # 이미지 저장
    cam.release()
    return filename

# 타이머 실행 시 호출되는 함수
async def alarm(context: ContextTypes.DEFAULT_TYPE) -> None:
    chat_id = context.job.chat_id
    delay = context.job.data

    filename = capture_image()
    if filename:
        with open(filename, "rb") as photo:
            await context.bot.send_photo(chat_id=chat_id, photo=photo,
                                         caption=f"{delay}초 뒤에 찍은 사진 📸")
    else:
        await context.bot.send_message(chat_id=chat_id, text="카메라 오류 발생!")

# 타이머 설정 함수 (/set)
async def set_timer(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    chat_id = update.effective_message.chat_id
    try:
        due = int(context.args[0])
        if due < 0:
            await update.message.reply_text("0보다 큰 수를 입력하세요.")
            return

        # N초 뒤 한 번 실행
        context.job_queue.run_once(alarm, due, chat_id=chat_id, name=str(chat_id), data=due)
        await update.message.reply_text(f"{due}초 뒤에 사진을 찍어드릴게요!")

    except (IndexError, ValueError):
        await update.message.reply_text("사용법: /set <초>")

# 메인 실행
def main():
    application = ApplicationBuilder().token("YOUR_BOT_TOKEN").build()
    application.add_handler(CommandHandler("set", set_timer))
    application.run_polling()

if __name__ == "__main__":
    main()
```

> ✅ `YOUR_BOT_TOKEN`을 BotFather에게 받은 **실제 토큰**으로 교체하세요.

### 4-2. 실행

```bash
python timerbot.py
```

* 텔레그램에서 나의 봇을 찾고 **/start** 입력
* **/set 5** → 5초 뒤 촬영된 사진이 채팅으로 전송됩니다.

---
