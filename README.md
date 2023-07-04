[main.md](https://github.com/amandusbengtson/IoT-project-2023/files/11897878/main.md)# IoT-project-2023
Description: IoT project code. This is the full code for the Smart Door Bell IoT LNU Project.

## Project Code

```{include} {python}
import machine
from machine import Pin, PWM, ADC #library import and define pin
from utime import sleep
import time #library import
import dht #temperatur sensor
import network
import urequests as requests
import random
from secrets import secrets
from time import sleep

DELAY = 2  # Delay in seconds

# Function for WiFi connection
def connect():
    wlan = network.WLAN(network.STA_IF)         # Put modem on Station mode
    if not wlan.isconnected():                  # Check if already connected
        print('connecting to network...')
        wlan.active(True)                       # Activate network interface
        # set power mode to get WiFi power-saving off (if needed)
        wlan.config(pm = 0xa11140)
        wlan.connect(secrets["WIFI_SSID"], secrets["WIFI_PASS"])  # Your WiFi Credential
        print('Waiting for connection...', end='')
        # Check if it is connected otherwise wait
        while not wlan.isconnected() and wlan.status() >= 0:
            print('.', end='')
            sleep(1)
    # Print the IP assigned by router
    ip = wlan.ifconfig()[0]
    print('\nConnected on {}'.format(ip))
    return ip

# Builds the json to send the request
def build_json(variable, value):
    try:
        data = {variable: {"value": value}}
        return data
    except:
        return None

# Random number generator
def random_integer(upper_bound):
    return random.getrandbits(32) % upper_bound

# Sending data to Ubidots Restful Webserice
def sendData(device, variable, value):
    try:
        url = "https://industrial.api.ubidots.com/"
        url = url + "api/v1.6/devices/" + device
        headers = {"X-Auth-Token": secrets["TOKEN"], "Content-Type": "application/json"}
        data = build_json(variable, value)

        if data is not None:
            print(data)
            req = requests.post(url=url, headers=headers, json=data)
            return req.json()
        else:
            pass
    except:
        pass

# Connect to the WiFi
connect()

buzzer = PWM(Pin(26)) #define buzzer 
tempSensor = dht.DHT11(machine.Pin(27))     # DHT11 Constructor #definining dht

tones = { #the song that plays to notify apartment owner. Activates when pushed button, plays as long as button is pushed down.
"B0": 31,
"C1": 33,
"CS1": 35,
"D1": 37,
"DS1": 39,
"E1": 41,
"F1": 44,
"FS1": 46,
"G1": 49,
"GS1": 52,
"A1": 55,
"AS1": 58,
"B1": 62,
"C2": 65,
"CS2": 69,
"D2": 73,
"DS2": 78,
"E2": 82,
"F2": 87,
"FS2": 93,
"G2": 98,
"GS2": 104,
"A2": 110,
"AS2": 117,
"B2": 123,
"C3": 131,
"CS3": 139,
"D3": 147,
"DS3": 156,
"E3": 165,
"F3": 175,
"FS3": 185,
"G3": 196,
"GS3": 208,
"A3": 220,
"AS3": 233,
"B3": 247,
"C4": 262,
"CS4": 277,
"D4": 294,
"DS4": 311,
"E4": 330,
"F4": 349,
"FS4": 370,
"G4": 392,
"GS4": 415,
"A4": 440,
"AS4": 466,
"B4": 494,
"C5": 523,
"CS5": 554,
"D5": 587,
"DS5": 622,
"E5": 659,
"F5": 698,
"FS5": 740,
"G5": 784,
"GS5": 831,
"A5": 880,
"AS5": 932,
"B5": 988,
"C6": 1047,
"CS6": 1109,
"D6": 1175,
"DS6": 1245,
"E6": 1319,
"F6": 1397,
"FS6": 1480,
"G6": 1568,
"GS6": 1661,
"A6": 1760,
"AS6": 1865,
"B6": 1976,
"C7": 2093,
"CS7": 2217,
"D7": 2349,
"DS7": 2489,
"E7": 2637,
"F7": 2794,
"FS7": 2960,
"G7": 3136,
"GS7": 3322,
"A7": 3520,
"AS7": 3729,
"B7": 3951,
"C8": 4186,
"CS8": 4435,
"D8": 4699,
"DS8": 4978
}

song = ["E5","G5","A5","P","E5","G5","B5","A5","P","E5","G5","A5","P","G5","E5"]

led = Pin(15, Pin.OUT)       # Set led pin as output #pinGD15
push_button = Pin(16, Pin.IN, Pin.PULL_UP)   # Set push button pin as input (It is connected to PULL_UP resistor #pinGD16
                                # and the default value is without pushing it is True)
vibratePin = Pin(27, Pin.IN)

# Pin setups for the lightsensor
ldr = ADC(Pin(28))
led = Pin(15, Pin.OUT)

while True: # While loop
    light = ldr.read_u16()
    #print("light before roundup ", light) #before roundup #debug strings
    darkness = round(light / 65535 * 100, 2)*100 #senses how dark it is to turn led on
    #print("darkness after roundup ", darkness) #after roundup debug string
    if darkness >= 70:
        print("Darkness is {}%, LED turned on".format(darkness))
        led.on()
        VARIABLE_LABEL = "led_status"
        returnValue = sendData(secrets["DEVICE_LABEL"], VARIABLE_LABEL, 1)
        sleep(DELAY)
    else:
        print("It is enough light, no need to turn the LED on")
        led.off() 
        VARIABLE_LABEL = "led_status"
        returnValue = sendData(secrets["DEVICE_LABEL"], VARIABLE_LABEL, 0)
        sleep(DELAY)
    time.sleep(1)

    time.sleep(0.1) #time before led identifying pushed button, a.k.a delay
    button_value = push_button.value()
    #print(button_value)
    if (button_value == 0):  # if the putton pushed
        def playtone(frequency):
            buzzer.duty_u16(1000)
            buzzer.freq(frequency)

        def bequiet():
            buzzer.duty_u16(0)

        def playsong(mysong):
            for i in range(len(mysong)):
                if (mysong[i] == "P"):
                    bequiet()
                else:
                    playtone(tones[mysong[i]])
                sleep(0.3)
            bequiet()
        playsong(song)
        try: #python keyword
                tempSensor.measure()
                temperature = tempSensor.temperature()
                VARIABLE_LABEL = "temperature"
                returnValue = sendData(secrets["DEVICE_LABEL"], VARIABLE_LABEL, temperature) #send data
                sleep(DELAY)
                humidity = tempSensor.humidity()
                VARIABLE_LABEL = "humidity"
                returnValue = sendData(secrets["DEVICE_LABEL"], VARIABLE_LABEL, humidity)
                sleep(DELAY)
                print("Temperature is {} degrees Celsius and Humidity is {}%".format(temperature, humidity))
        except Exception as error:
                print("Exception occurred", error)
        time.sleep(2)

```
