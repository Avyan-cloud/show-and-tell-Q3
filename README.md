# show-and-tell-Q3
from machine import Pin, I2C
from time import sleep
import ssd1306

rows = [Pin(2, Pin.OUT), Pin(3, Pin.OUT), Pin(4, Pin.OUT), Pin(5, Pin.OUT)]
cols = [Pin(6, Pin.IN, Pin.PULL_UP), Pin(7, Pin.IN, Pin.PULL_UP),
        Pin(8, Pin.IN, Pin.PULL_UP), Pin(9, Pin.IN, Pin.PULL_UP)]

i2c = I2C(0, scl=Pin(1), sda=Pin(0))
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

green_led = Pin(10, Pin.OUT)
red_led = Pin(11, Pin.OUT)
buzzer = Pin(12, Pin.OUT)

pir = Pin(13, Pin.IN)

password = "1234"
input_password = ""
digit_count = 0

keypad = [
    ['1','2','3','A'],
    ['4','5','6','B'],
    ['7','8','9','C'],
    ['*','0','#','D']
]

for row in rows:
    row.value(1)

attempts = 0

lockout_active = False
lockout_seconds = 60

idle_counter = 0
idle_limit = 20
oled_state = ""

while True:

    if lockout_active:

        while lockout_seconds > 0:
            oled.fill(0)
            oled.text("LOCKED OUT", 0, 0)
            oled.text(str(lockout_seconds) + " sec", 0, 20)
            oled.show()

            red_led.value(1)
            sleep(1)
            lockout_seconds -= 1

        lockout_active = False
        attempts = 0
        red_led.value(0)

    activity = False

    if pir.value() == 1:
        activity = True
        idle_counter = 0

        oled.fill(0)
        oled.text("Motion Detected!", 0, 0)
        oled.show()

        red_led.value(1)
        buzzer.value(1)
        sleep(0.5)
        buzzer.value(0)
    else:
        red_led.value(0)

    for row_num, row in enumerate(rows):
        row.value(0)

        for col_num, col in enumerate(cols):
            if col.value() == 0:
                activity = True
                idle_counter = 0

                key_pressed = keypad[row_num][col_num]
                input_password += key_pressed
                digit_count += 1

                while col.value() == 0:
                    sleep(0.01)

        row.value(1)

    if digit_count == 4:

        activity = True
        idle_counter = 0

        if input_password == password:
            oled.fill(0)
            oled.text("Access Granted", 0, 0)
            oled.show()

            green_led.value(1)
            sleep(3)
            green_led.value(0)

        else:
            attempts += 1
            oled.fill(0)
            oled.text("Access Denied", 0, 0)
            oled.show()

            red_led.value(1)
            buzzer.value(1)
            sleep(1)
            buzzer.value(0)

            if attempts >= 3:
                lockout_active = True
                lockout_seconds = 60

        input_password = ""
        digit_count = 0

    if not activity:
        idle_counter += 1
    else:
        idle_counter = 0

    if idle_counter >= idle_limit:
        if oled_state != "idle":
            oled.fill(0)
            oled.text("SYSTEM READY", 20, 20)
            oled.text("Waiting...", 30, 35)
            oled.show()
            oled_state = "idle"
    else:
        oled_state = "active"

    sleep(0.1)
