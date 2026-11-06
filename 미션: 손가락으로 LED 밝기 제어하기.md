<img width="1098" height="638" alt="image" src="https://github.com/user-attachments/assets/57a25809-5d9e-4276-84ae-a2557228b05d" />[아두이노코드,p5코드.md](https://github.com/user-attachments/files/34478070/p5.md)


https://github.com/user-attachments/assets/7a454241-b79a-4130-9c68-58cbebcf625f

## 1. Arduino IDE 코드
```cpp
const int LED_PIN = 5;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("READY");
}

void loop() {
  if (Serial.available() > 0) {
    String command = Serial.readStringUntil('\n');
    command.trim();
    if (command.length() > 0) {
      analogWrite(LED_PIN, constrain(command.toInt(), 0, 255));
    }
  }
}
```

## 2. HTML 코드 (index.html)
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <script src="https://cdn.jsdelivr.net/npm/p5@2.3.4/lib/p5.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/p5.sound@0.4.1/dist/p5.sound.min.js"></script>
    <script src="https://unpkg.com/ml5@1/dist/ml5.min.js"></script>
    <link rel="stylesheet" type="text/css" href="style.css">
    <meta charset="utf-8" />
  </head>
  <body>
    <main>
    </main>
    <script src="serial-helper.js"></script>
    <script src="sketch.js"></script>
  </body>
</html>
```

## 3. CSS 코드 (style.css)
```css
html, body {
  margin: 0;
  padding: 0;
}
canvas {
  display: block;
}
```

## 4. p5.js 코드 (sketch.js)
```javascript
let handPose, video, slider;
let serial, connectButton;
let hands = [];
let sliderValue = 50;
let connections;
let lastSent = -1;
let statusText = 'Arduino를 연결해 주세요.';

async function setup() {
  createCanvas(640, 480);

  slider = createSlider(0, 100, 50, 1);
  slider.style('writing-mode', 'vertical-rl');
  slider.style('height', '440px');
  slider.style('width', '20px');
  slider.style('transform: rotate(180deg)');
  slider.position(10, 20);
  slider.input(updateSliderValue);

  serial = new SerialHelper(9600, receiveLine, message => statusText = message);
  connectButton = createButton('Arduino 연결 / 해제');
  connectButton.mousePressed(() => serial.toggle());

  video = createCapture(VIDEO, {flipped:true});
  video.size(640, 480);
  video.hide();

  handPose = await ml5.handPose({flipped:true});
  handPose.detectStart(video, gotHands);
  connections = handPose.getConnections();
}

function draw() {
  image(video, 0, 0, width, height);

  drawSkeletons(hands);
  drawKeyPoints(hands);

  let adjustMode = hands.length > 0;
  if (adjustMode) {
    drawConnectedLine(hands);
    let v = getNormalizedDistance(hands);
    sliderValue += 0.2 * (v - sliderValue);
    slider.value(sliderValue);
  }

  sendBrightness();
  drawStatus();
}

function sendBrightness() {
  let brightness = round(map(Number(slider.value()), 0, 100, 0, 255));
  if (brightness === lastSent) return;
  lastSent = brightness;
  serial.send(String(brightness));
}

function receiveLine(line) {
  if (line.trim().toUpperCase() === 'READY') lastSent = -1;
}

function drawStatus() {
  noStroke();
  fill(0, 150);
  rect(50, 430, 580, 40, 8);
  fill(255);
  textSize(16);
  text('밝기 ' + max(lastSent, 0) + ' / 255  ·  ' + statusText, 65, 455);
}

function gotHands(results) {
  hands = results;
}

function drawKeyPoints(hs) {
  for (let i = 0; i < hs.length; i++) {
    let hand = hs[i];
    for (let j = 0; j < hand.keypoints.length; j++) {
      let keypoint = hand.keypoints[j];
      fill(255, 0, 0);
      noStroke();
      circle(keypoint.x, keypoint.y, 10);
    }
  }
}

function drawSkeletons(hs) {
  for (let i = 0; i < hs.length; i++) {
    let hand = hs[i];
    stroke(0, 255, 0);
    strokeWeight(2);
    for (const [a, b] of connections) {
      const start = hand.keypoints[a];
      const end = hand.keypoints[b];
      if (start && end) {
        line(start.x, start.y, end.x, end.y);
      }
    }
  }
}

function drawConnectedLine(hs) {
  if(hs.length > 0) {
    let hand = hs[0];
    const a = hand.keypoints[4];
    const b = hand.keypoints[8];
    stroke(255, 140, 0);
    strokeWeight(4);
    line(a.x,a.y,b.x,b.y);
    noFill();
    circle(a.x,a.y,22);
    circle(b.x,b.y,22);
  }
}

function getNormalizedDistance(hs) {
  if(hs.length > 0) {
    let ks = hs[0].keypoints;
    const a = ks[4];
    const b = ks[8];
    const c = ks[0];
    const d = ks[9];
    let v = dist(a.x,a.y,b.x,b.y);
    let p = dist(c.x,c.y,d.x,d.y);
    let s = round(100*(v/p))-15;
    return constrain(s,0,100);
  }
  return 0;
}

function updateSliderValue() {
  sliderValue = Number(slider.value());
}
```

## 5. 통신 코드 (serial-helper.js)
```javascript
class SerialHelper {
  constructor(baudRate, onLine, onStatus) {
    this.baudRate = baudRate;
    this.onLine = onLine;
    this.onStatus = onStatus;
    this.port = null;
    this.reader = null;
    this.writer = null;
    this.busy = false;
    this.closing = false;
  }

  async toggle() {
    if (this.busy) return;
    this.busy = true;
    try {
      if (this.port) { await this.disconnect(); return; }
      if (!('serial' in navigator)) {
        this.onStatus('Web Serial 지원 Chrome/Edge에서 실행해 주세요.');
        return;
      }
      this.port = await navigator.serial.requestPort();
      await this.port.open({ baudRate: this.baudRate });
      this.writer = this.port.writable.getWriter();
      this.closing = false;
      this.onStatus('연결됨 · 9600 baud');
      this.readTask = this.readLines();
    } catch (error) {
      this.onStatus(error.name === 'NotFoundError' ? '포트 선택을 취소했습니다.' : '연결 오류: ' + error.message);
      await this.disconnect(false);
    } finally { this.busy = false; }
  }

  async send(command) {
    if (!this.writer || this.closing) {
      this.onStatus('먼저 Arduino를 연결해 주세요.');
      return false;
    }
    try {
      await this.writer.write(new TextEncoder().encode(command + '\n'));
      this.onStatus('전송: ' + command + ' · 9600 baud');
      return true;
    } catch (error) {
      this.onStatus('전송 오류: ' + error.message);
      return false;
    }
  }

  readLines() {
    const decoder = new TextDecoder();
    let buffer = '';
    const reader = this.port.readable.getReader();
    this.reader = reader;

    return new Promise(resolve => {
      const finish = error => {
        reader.releaseLock();
        this.reader = null;
        const unexpected = !this.closing;
        if (unexpected) {
          this.onStatus(error ? '수신 오류: ' + error.message : '연결이 끊어졌습니다.');
        }
        resolve();
        if (unexpected) setTimeout(() => this.disconnect(false), 0);
      };

      const readNext = () => {
        if (this.closing) { finish(); return; }
        reader.read().then(({ value, done }) => {
          if (done || this.closing) { finish(); return; }
          buffer += decoder.decode(value, { stream: true });
          const lines = buffer.split('\n');
          buffer = lines.pop();
          lines.forEach(line => {
            const message = line.trim();
            if (message) this.onLine(message);
          });
          if (buffer.length > 4096) buffer = '';
          setTimeout(readNext, 0);
        }).catch(finish);
      };
      readNext();
    });
  }

  async disconnect(showStatus = true) {
    this.closing = true;
    if (this.reader) await this.reader.cancel().catch(() => {});
    if (this.readTask) await this.readTask.catch(() => {});
    this.readTask = null;
    if (this.writer) { this.writer.releaseLock(); this.writer = null; }
    if (this.port) await this.port.close().catch(() => {});
    this.port = null;
    this.closing = false;
    if (showStatus) this.onStatus('연결 해제됨');
  }
}
```
코드 작동방식
1. Arduino IDE
LED가 연결된 핀을 출력 모드로 설정한다.

Serial 통신으로 컴퓨터에서 밝기 값을 전달받는다.

전달받은 값을 0~255로 제한한 후 analogWrite()로 LED 밝기를 조절한다.

2. HTML
p5.js, ml5.js 등 필요한 라이브러리를 불러온다.

CSS, JavaScript 파일을 연결하여 전체 프로그램을 실행한다.

3. CSS
- 웹페이지의 기본 여백을 제거하고 Canvas 화면을 정리한다.

4. p5.js
웹캠을 실행하고 ml5.js의 handPose 모델로 손을 인식한다.

엄지와 검지 사이의 거리를 측정한다.

손의 크기를 기준으로 거리를 0~100 범위로 변환한다.

계산된 값을 슬라이더에 반영한다.

밝기 값을 0~255로 변환하여 Arduino에 전송한다.

손의 관절과 연결선을 화면에 표시한다.

5. Serial-helper.js
웹브라우저와 Arduino 사이의 Serial 통신을 담당한다.

Arduino 연결 및 해제를 관리한다.

밝기 값을 Arduino로 전송하고 통신 상태를 확인한다.

전체 작동 원리
웹캠으로 손을 인식한다.

엄지와 검지 사이의 거리를 측정한다.

측정값을 LED 밝기 값으로 변환한다.

Serial 통신으로 Arduino에 전달한다.

Arduino가 LED 밝기를 조절한다.

핵심: p5.js는 손 인식과 밝기 계산, Serial-helper.js는 데이터 전송, Arduino는 실제 LED 제어를 담당한다.



<img width="1098" height="638" alt="image" src="https://github.com/user-attachments/assets/c0b06afc-aff6-415b-9bb3-f1ca17d82dcb" />
