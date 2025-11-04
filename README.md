# SED1115-PB-Final
from machine import Pin, UART, I2C
import utime
from ads1x15 import ADS1015
import adc1  # Your config with pins and address

# === Initialize I2C bus with pins defined in adc1.py ===
i2c = I2C(1, sda=Pin(adc1.I2C_SDA_PIN), scl=Pin(adc1.I2C_SCL_PIN))

# === Initialize ADS1015 ADC on the I2C bus with configured address ===
ads1015_adc = ADS1015(i2c, adc1.ADS1015_ADDR, gain=1)  # gain=1 for +-4.096V range

# === UART1 setup for TX on GP4, RX on GP5 at 9600 baud ===
uart = UART(1, baudrate=9600, tx=Pin(4), rx=Pin(5))

# ADS1015 channel to read analog input (e.g. channel 0)
adc_channel = adc1.ADS1015_PWM_CHANNEL  # Usually 2 as per your adc1.py

def read_analog_percent():
    """
    Read raw ADC value from ADS1015 on configured channel.
    Convert raw reading to 0-100% scale.
    """
    try:
        raw_adc_value = ads1015_adc.read(4, adc_channel)  # 4 = 1600 samples/sec rate per ads1x15.py default
        # ADS1015 is 16-bit signed: range approx -32768 to +32767, make unsigned 0 to 65535 for percentage
        if raw_adc_value < 0:
            raw_adc_value += 65536  # Convert signed to unsigned by adding 2^16
        percent = int((raw_adc_value / 65535) * 100)
        # Clamp percentage to 0-100 just in case
        if percent < 0:
            percent = 0
        elif percent > 100:
            percent = 100
        return percent
    except Exception as e:
        print("ADC read error:", e)
        return 0  # Return safe default

def send_uart_message(message):
    """
    Send a text message over UART, ending with newline.
    """
    try:
        uart.write((message + '\n').encode())
    except Exception as e:
        print("UART send error:", e)

def receive_uart_message(timeout_ms=1000):
    """
    Receive a line message from UART with a timeout.
    Returns string or None.
    """
    start_time = utime.ticks_ms()
    while utime.ticks_diff(utime.ticks_ms(), start_time) < timeout_ms:
        if uart.any():
            line = uart.readline()
            if line:
                try:
                    return line.decode().strip()
                except:
                    return None
    return None

def main_loop():
    while True:
        # Read analog percentage from ADS1015
        pot_percent = read_analog_percent()
        print("Local potentiometer value:", pot_percent, "%")
        
        # Send this value using UART with a "POT:" label
        send_uart_message("POT:" + str(pot_percent))
        
        # Attempt to receive partner's potentiometer value over UART
        incoming = receive_uart_message()
        if incoming and incoming.startswith("POT:"):
            try:
                partner_percent = int(incoming.split(":")[1])
                print("Received partner potentiometer:", partner_percent, "%")
            except ValueError:
                print("Received invalid data:", incoming)
        else:
            print("No valid potentiometer data from partner")
        
        utime.sleep(1)  # 1-second delay for next cycle

# Start the program
main_loop()
