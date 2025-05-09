# smart-door-lock-system-using-arduino
#include <Keypad.h>
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// LCD setup
LiquidCrystal_I2C lcd(0x27, 16, 2); // Try 0x3F if 0x27 doesn't work

// Keypad setup
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {9, 8, 7, 6};
byte colPins[COLS] = {5, 4, 3, 2};

Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Components
Servo myServo;
int servoPin = 10;
int buzzerPin = 11;

// Password system
String password = "1234";
String input = "";
int attempts = 0;
bool alarmTriggered = false;

void setup() {
  myServo.attach(servoPin);
  pinMode(buzzerPin, OUTPUT);
  myServo.write(0); // Locked

  lcd.begin();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Enter Password");

  Serial.begin(9600);
}

void loop() {
  char key = keypad.getKey();

  if (alarmTriggered) {
    // Buzzer stays on after 3 failed attempts
    digitalWrite(buzzerPin, HIGH); // Keep buzzer on
    lcd.setCursor(0, 0);
    lcd.print("!! ALARM TRIGGERED");
    lcd.setCursor(0, 1);
    lcd.print("Enter password   ");

    if (key) {
      input += key;
      if (key == '#') {
        if (input == password) {
          resetAlarm(); // Reset the alarm and turn off the buzzer
        }
        input = "";
      } else if (key == '*') {
        resetSystem(); // Reset system when '*' is pressed
      }
    }
    return; // Skip the rest of the loop if alarm is triggered
  }

  if (key) {
    Serial.println(key);
    
    if (key == '#') {
      if (input == password) {
        success();
        unlockDoor();
        attempts = 0;
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Access Granted");
        delay(2000);
        lcd.clear();
        lcd.print("Enter Password");
      } else {
        attempts++;
        failure();
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Wrong Password");
        lcd.setCursor(0, 1);
        lcd.print("Attempts: ");
        lcd.print(attempts);
        delay(2000);
        lcd.clear();
        lcd.print("Enter Password");

        // After 3 attempts, trigger the alarm
        if (attempts >= 3) {
          alarmTriggered = true;
          digitalWrite(buzzerPin, HIGH); // turn on buzzer
          Serial.println("ALARM TRIGGERED");
        }
      }
      input = "";
    } else if (key == '*') {
      resetSystem(); // Reset system when '*' is pressed
    } else {
      input += key;
      lcd.setCursor(0, 1);
      lcd.print("                "); // clear line first
      lcd.setCursor(0, 1);
      for (int i = 0; i < input.length(); i++) {
        lcd.print("*"); // Display '*' for each character entered
      }
    }
  }
}

void unlockDoor() {
  myServo.write(90); // Unlock the door
  delay(3000);       // Keep it unlocked for 3 seconds
  myServo.write(0);  // Lock the door again
}

void success() {
  tone(buzzerPin, 1000, 200); // Success beep
  delay(200);
  tone(buzzerPin, 1500, 200);
}

void failure() {
  tone(buzzerPin, 500, 500);  // Failure beep
  delay(500);
}

void resetAlarm() {
  alarmTriggered = false;      // Reset alarm state
  digitalWrite(buzzerPin, LOW); // Turn off buzzer
  attempts = 0;                // Reset attempt counter
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Alarm Reset");
  delay(2000);
  lcd.clear();
  lcd.print("Enter Password");
  success(); // Success tone
}

void resetSystem() {
  // Reset the system when '*' key is pressed
  alarmTriggered = false;     // Stop the alarm
  digitalWrite(buzzerPin, LOW); // Turn off buzzer
  attempts = 0;               // Reset attempts counter
  input = "";                 // Clear input buffer
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("System Reset");
  delay(2000);                // Show reset message for 2 seconds
  lcd.clear();
  lcd.print("Enter Password"); // Show password prompt again
}
