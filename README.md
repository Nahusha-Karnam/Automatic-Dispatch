# Automatic-Dispatch

![image](https://github.com/user-attachments/assets/2483106e-560c-41c0-900b-435eb8d79b5f)

![a49899fc-1aa1-4baa-bdef-72e322cf8e1d](https://github.com/user-attachments/assets/fdecbbc3-8889-4b59-b4e3-4ff2debf8b3f)

![6ef3206c-c90b-422a-8663-30a5306c6bc1](https://github.com/user-attachments/assets/88f90c34-5ed1-4804-a5de-9cc68e051316)

Program

```

#include <SPI.h>
#include <Wire.h>
#include <MFRC522.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// RFID 1 Pins
#define SS_1 15
#define RST_1 14

// RFID 2 Pins
#define SS_2 13
#define RST_2 11

SPIClass SPI(1);

// Create instances for each reader
MFRC522 mfrc522_1(SS_1, RST_1);
MFRC522 mfrc522_2(SS_2, RST_2);

TwoWire Wire(0);  // I2C-1
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define OLED_RESET     -1
#define SCREEN_ADDRESS 0x3C
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Function to convert UID to string
String uidToString(MFRC522 &reader) {
  String uid = "";
  for (byte i = 0; i < reader.uid.size; i++) {
    uid += String(reader.uid.uidByte[i], HEX);
  }
  uid.toUpperCase();
  return uid;
}

// Match UID and return Box name
String getBoxName(String uid) {
  if (uid == "DCB2FA0") return "Box 1";
  if (uid == "E3152224") return "Box 2";
  return "Unknown";
}

void setup() {
  Serial.begin(115200);
  pinMode(9, INPUT);

  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println("SSD1306 allocation failed");
    while (true);
  }

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.display();

  while (!Serial);

  SPI.begin();  // Initialize SPI bus
  mfrc522_1.PCD_Init();
  mfrc522_2.PCD_Init();

  Serial.println("Scan RFID cards on either reader...");
}

void loop() {
  // Reader 1 -> Room 1
  if(digitalRead(10)==0){
    display.clearDisplay();
    display.setCursor(0, 0);
    display.print("Item Detected in ");
    display.setCursor(0, 10);
    display.print("Room 1");
    display.display();
  }else if(digitalRead(9)==0){
    display.clearDisplay();
    display.setCursor(0, 0);
    display.print("Item Detected in ");
    display.setCursor(0, 10);
    display.print("Room 2");
    display.display();
  }
  if (mfrc522_1.PICC_IsNewCardPresent() && mfrc522_1.PICC_ReadCardSerial()) {
    display.clearDisplay();
    
    
    String uid = uidToString(mfrc522_1);
    String box = getBoxName(uid);
    
    
    display.setCursor(0, 0);
    display.print("Item Detected in ");
    display.setCursor(0, 10);
    display.print("Room 1");
    display.setCursor(0, 25);
    display.print(box);
    display.print(",Room 1");
    display.display();
    
    Serial.print(box);
    Serial.println(",Room 1");

    mfrc522_1.PICC_HaltA();
    mfrc522_1.PCD_StopCrypto1();
    delay(1000);
  }

  // Reader 2 -> Room 2
  if (mfrc522_2.PICC_IsNewCardPresent() && mfrc522_2.PICC_ReadCardSerial()) {
    display.clearDisplay();
  
    
    String uid = uidToString(mfrc522_2);
    String box = getBoxName(uid);
    
    display.setCursor(0, 0);
    display.print("Item Detected in ");
    display.setCursor(0, 10);
    display.print("Room 2");
    display.setCursor(0, 25);
    display.print(box);
    display.print(",Room 2");
    display.display();
    Serial.print(box);
    Serial.println(",Room 2");

    mfrc522_2.PICC_HaltA();
    mfrc522_2.PCD_StopCrypto1();
    delay(1000);
  }
}

```

Python code

```
import serial
import threading
from flask import Flask, render_template_string


SERIAL_PORT = "COM11"
BAUD_RATE = 115200


room_boxes = {
    "Room 1": set(),
    "Room 2": set()
}
all_logs = []  

app = Flask(__name__)


HTML_TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
    <title>Room Box Tracker</title>
    <meta http-equiv="refresh" content="3">
    <style>
        body { font-family: Arial; background: #f0f0f0; padding: 30px; }
        .room { background: white; padding: 20px; margin-bottom: 30px; border-radius: 10px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        h2 { margin-top: 0; }
        ul { list-style-type: none; padding-left: 0; }
        li { padding: 5px 0; }
    </style>
</head>
<body>
    <h1>Room Box Tracker</h1>
    
    {% for room, boxes in room_boxes.items() %}
        <div class="room">
            <h2>{{ room }}</h2>
            <p><strong>Boxes currently in:</strong></p>
            <ul>
                {% for box in boxes %}
                    <li>{{ box }}</li>
                {% endfor %}
            </ul>
        </div>
    {% endfor %}

    <div class="room">
        <h2>Recent Activity</h2>
        <ul>
            {% for log in all_logs[-10:] %}
                <li>{{ log }}</li>
            {% endfor %}
        </ul>
    </div>
</body>
</html>
"""

@app.route('/')
def index():
    return render_template_string(HTML_TEMPLATE, room_boxes=room_boxes, all_logs=all_logs)

def read_serial():
    try:
        ser = serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1)
        print(f"Connected to {SERIAL_PORT}")
    except serial.SerialException:
        print(f"Failed to connect to {SERIAL_PORT}")
        return

    while True:
        try:
            line = ser.readline().decode('utf-8').strip()
            if line:
                print("Received:", line)
                parts = [x.strip() for x in line.split(",")]
                if len(parts) == 2:
                    box, room = parts
                    if room not in room_boxes:
                        print("Unknown room:", room)
                        continue

                    if box in room_boxes[room]:
                        room_boxes[room].remove(box)
                        all_logs.append(f"{box} went OUT of {room}")
                    else:
                        room_boxes[room].add(box)
                        all_logs.append(f"{box} came IN to {room}")
        except Exception as e:
            print("Error reading serial:", e)


threading.Thread(target=read_serial, daemon=True).start()


if __name__ == "__main__":
    app.run(debug=False, port=5000)

```

Working Video
https://drive.google.com/file/d/1QdDMYfZM6c_gcKFSPs8uqjuAJ3kWCu1z/view?usp=sharing
