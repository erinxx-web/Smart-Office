# Smart-Office

This repository contains the implementation of a Smart Office system designed to monitor and control office conditions using embedded hardware and a web-based interface. The project integrates sensors, microcontrollers, and software logic to automate tasks such as environmental monitoring, device control, and system alerts, demonstrating practical IoT and system integration concepts.

```csharp
SPACE
import socketio
import time
import Adafruit_BBIO.GPIO as GPIO

sio = socketio.Client()
GPIO.setup("P9_15", GPIO.IN)

@sio.event
def connect():
    print('Connection established.')

@sio.event
def disconnect():
    print('Disconnected from server.')

while True:
    try:
        sio.connect('http://192.168.72.82:5000')
        break
    except:
        print("Try to connect to the server.")
        pass
    
MotionDetectionStatus = 0

while True:
    try:
        MotionDetectionStatus = GPIO.input("P9_15")
        if MotionDetectionStatus:
            sio.emit('BBBW1Event', {'data': MotionDetectionStatus})
            print('Data sent!')
            print("Motion is Detected")
        else:
            print("No Motion is Detected")
    except:
        print('Unable to transmit data.')
        pass
    time.sleep(0.5)

ROOM TEMPERATURE
import time
from Adafruit_BBIO.SPI import SPI
import Adafruit_BBIO.GPIO as GPIO
import Adafruit_BBIO.PWM as PWM
import board
import adafruit_bme680
import socketio

sio = socketio.Client()
while True:
    try:
        sio.connect("http://192.168.72.193:5000")
        break
    except Exception as e:
        print("trying to connect to Web Server...")
        time.sleep(2)

BUZZER_PIN = "P8_13"  # Slot 4 PWM-capable pin

def SevenSegInit():
    GPIO.setup("P8_19", GPIO.OUT)
    GPIO.setup("P8_14", GPIO.OUT)
    GPIO.output("P8_19", GPIO.HIGH)
    GPIO.output("P8_14", GPIO.HIGH)
    L_Spi0 = SPI(0,0)
    L_Spi0.mode = 0
    return L_Spi0

def SevenSegDisplay(L_Spi1, temperature):
    DigitList = [0x7E, 0x0A, 0xB6, 0x9E, 0xCA,
                 0xDC, 0xFC, 0x0E, 0xFE, 0xDE]
    temp_int = int(round(temperature))
    temp_int = max(0, min(temp_int, 99))  # Clamp to 0–99
    tens_digit = int(temp_int / 10)
    ones_digit = int(temp_int % 10)
    # Send digits (ones right, tens left)
    L_Spi1.writebytes([DigitList[ones_digit], DigitList[tens_digit]])

def EnvironmentInit():
    i2c = board.I2C()
    sensor = adafruit_bme680.Adafruit_BME680_I2C(i2c, 0x77)
    sensor.sea_level_pressure = 1008.5
    return sensor

def PlayBuzzer():
    print("Temp > 26°C: Playing buzzer...")
    PWM.start(BUZZER_PIN, 50)  # Start PWM 50% duty
    try:
        PWM.set_frequency(BUZZER_PIN, 523)  # C5
        time.sleep(1)
        PWM.set_frequency(BUZZER_PIN, 587)  # D5
        time.sleep(1)
        PWM.set_frequency(BUZZER_PIN, 659)  # E5
        time.sleep(1)
    finally:
        PWM.stop(BUZZER_PIN)
        print("Buzzer stopped.")

G_Spi1 = SevenSegInit()
BME680_Sensor = EnvironmentInit()
temperature_offset = -5  # Calibrate sensor
buzzer_triggered = False  # Prevent retrigger


try:
    while True:
        # Read and calibrate temperature
        temp_c = BME680_Sensor.temperature + temperature_offset
        print("\nTemperature: %0.1f °C" % temp_c)
        sio.emit('BBBW2Event', {'data':temp_c})
        print('data sent')
        # Update 7-Segment display
        SevenSegDisplay(G_Spi1, temp_c)

        # Play buzzer if temp > 26°C
        if temp_c > 26 and not buzzer_triggered:
            PlayBuzzer()
            buzzer_triggered = True
        elif temp_c <= 26:
            buzzer_triggered = False

        time.sleep(1)

except KeyboardInterrupt:
    print("\nExiting...")
    PWM.stop(BUZZER_PIN)
    G_Spi1.close()

PASSWORD DOOR
import socketio
from Adafruit_BBIO.SPI import SPI
import Adafruit_BBIO.ADC as ADC
import Adafruit_BBIO.GPIO as GPIO 
import Adafruit_BBIO.PWM as PWM 
import time
import math

sio = socketio.Client()

Buzzer = "P8_13"
ADC.setup()

password = ['T3', 'T2', 'T1']
input_sequence = []
unlocked = False


while True:
    try:
        sio.connect('http://192.168.72.193:5000')
        break
    except:
        print("Try to connect to the server.")
        pass   

def get_key():
    value = ADC.read("P9_38")
    if value >= 0.00 and value < 0.10:
        return None
    elif 0.16 < value < 0.18:
        return "T6"
    elif 0.33 < value < 0.35:
        return "T5"
    elif 0.50 < value < 0.52:
        return "T4"
    elif 0.67 < value < 0.69:
        return "T3"
    elif 0.84 < value < 0.86:
        return "T2"
    elif 0.90 < value < 1.10:
        return "T1"
    else:
        return None

print("Enter password using analog keys...")

while True:  # Infinite loop
    input_sequence = []
    unlocked = False

    while not unlocked:
        key = get_key()
        if key:
            if not input_sequence or key != input_sequence[-1]:  # debounce
                print(f"Key detected: {key}")
                input_sequence.append(key)
            time.sleep(0.5)
    
            if len(input_sequence) == len(password):
                if input_sequence == password:
                    print("password correct! Door unlocked.")
                    PWM.start(Buzzer, 50)
                    PWM.set_frequency(Buzzer, 2000)
                    time.sleep(0.1)
                    PWM.stop(Buzzer)
                    unlocked = True
                    print("Restart")
                else:
                    print("incorrect password.")
                    PWM.start(Buzzer, 50)
                    PWM.set_frequency(Buzzer, 1000)
                    time.sleep(0.2)
                    PWM.set_frequency(Buzzer, 2000)
                    time.sleep(0.2)
                    PWM.stop(Buzzer)
                    print("Retry")
                    print("Enter password using analog keys...")
                    input_sequence = []
                    sio.emit('BBBW3Event', {'data': key})

MAGNET STATUS
import time 
import Adafruit_BBIO.GPIO as GPIO 
import Adafruit_BBIO.PWM as PWM
import socketio

sio = socketio.Client()
GPIO.setup("P8_10", GPIO.IN) 


while True:
    try:
        sio.connect('http://192.168.72.193:5000')
        break
    except:
        print("Try to connect to the server.")
        time.sleep(1)

PWM.start("P8_19", 50)
PWM.stop("P8_19") 

door_open_time = None
buzzer_on = False

while True:
    MagneticStatus = GPIO.input("P8_10")  
    current_time = time.time()

    if MagneticStatus:  
        door_open_time = None  
        if buzzer_on:
            PWM.stop("P8_19")
            buzzer_on = False
        print("Closed")

    else:  
        if door_open_time is None:
            door_open_time = current_time  
        elif current_time - door_open_time >= 15:
            
            if not buzzer_on:
                PWM.start("P8_19", 50)
                buzzer_on = True
            PWM.set_frequency("P8_19", 1000)
            time.sleep(0.1)
            PWM.set_frequency("P8_19", 2000)
            time.sleep(0.1)
            print("Door opened too long")
        else:
            print(f"Door open... waiting: {int(current_time - door_open_time)}s")

    sio.emit('BBBW5Event', {'data': MagneticStatus})
    print("Data Sent!")

    time.sleep(0.3)

    LIGHT SYSTEM
    import time
import math
import socketio
import Adafruit_BBIO.GPIO as GPIO
import Adafruit_BBIO.ADC as ADC

GPIO.setup("P9_15", GPIO.IN)
ADC.setup()
motion = False
oldknobvalue = 0

sio = socketio.Client()

while True:
    try:
        sio.connect('http://192.168.72.193:5000')
        break
    except:
        print("Trying to connect to server")
        pass

while True:
    if GPIO.input("P9_15"):
        print("Motion is Dectected")
        motion = True
    else:
        print("Motion is NOT Dectected")
        motion = False
    sio.emit('BBBW4Event', {'data' : motion})
    print("Data Sent!")
    try:
        newknobvalue = ADC.read("P9_37")
        print("Digital Value: %f" % (newknobvalue))
        if(abs(newknobvalue - oldknobvalue) > 0.1):
            sio.emit('BBBW4Event', {'data': newknobvalue})
            print('Data sent!')
        oldknobvalue = newknobvalue
    except:
        print('Unable to transmit data.')
        pass
    time.sleep(1)
