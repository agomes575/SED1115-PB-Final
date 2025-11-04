# SED1115-PB-Final
from machine import Pin, UART, I2C         # Import classes for controlling pins, UART, and I2C
import utime                               # Import module for timing functions like sleep
from ads1x15 import ADS1015                # Import the ADS1015 class to use the external ADC
import adc1                                # Import your custom settings (I2C pins, ADC address, channel)

# Set up I2C communication using the correct pins from your settings file
i2c = I2C(1, sda=Pin(adc1.I2C_SDA_PIN), scl=Pin(adc1.I2C_SCL_PIN))

# Create an object to control the ADS1015 external ADC using I2C and its address
ads1015_adc = ADS1015(i2c, adc1.ADS1015_ADDR, gain=1)  # gain=1 sets voltage range to +-4.096V

# Set up UART1 communication on GPIO4 (TX) and GPIO5 (RX) at 9600 baud rate
uart = UART(1, baudrate=9600, tx=Pin(4), rx=Pin(5))

# Determine which ADS1015 channel to use for reading the potentiometer (usually channel 2)
adc_channel = adc1.ADS1015_PWM_CHANNEL

def read_analog_percent():
    """
    Get raw analog value from ADC and return it as a percent (0-100).
    """
    try:
        # Read analog value from the specified ADC channel at 1600 samples per second
        raw_adc_value = ads1015_adc.read(4, adc_channel)
        # If the value is negative (because ADC is signed), shift it to positive range
        if raw_adc_value < 0:
            raw_adc_value += 65536
        # Convert the value (0 to 65535) to a percentage (0 to 100)
        percent = int((raw_adc_value / 65535) * 100)
        # Make sure percent value isn't less than 0 or more than 100
        if percent < 0:
            percent = 0
        elif percent > 100:
            percent = 100
        return percent                        # Return the calculated percent
    except Exception as e:
        print("ADC read error:", e)           # Print an error if something goes wrong reading ADC
        return 0                              # Return 0 if read fails

def send_uart_message(message):
    """
    Send a message string to the other Pico using UART.
    """
    try:
        uart.write((message + '\n').encode()) # Send message as bytes with '\n' at the end
    except Exception as e:
        print("UART send error:", e)          # Print an error if UART write fails

def receive_uart_message(timeout_ms=1000):
    """
    Wait for a message from UART for up to given time (milliseconds),
    then return the message as a string, or None if not received.
    """
    start_time = utime.ticks_ms()             # Get the current time in ms
    while utime.ticks_diff(utime.ticks_ms(), start_time) < timeout_ms:
        if uart.any():                        # If there is data waiting in UART buffer
            line = uart.readline()            # Read a line (ends at '\n')
            if line:
                try:
                    return line.decode().strip() # Convert bytes to string and remove extra spaces
                except:
                    return None               # If decoding fails, return None
    return None                               # If timeout is reached, return None

def main_loop():
    while True:
        pot_percent = read_analog_percent()   # Get potentiometer value as a percentage from ADC
        print("Local potentiometer value:", pot_percent, "%")     # Show our value on the screen
        
        send_uart_message("POT:" + str(pot_percent))              # Send our value to partner Pico
        
        incoming = receive_uart_message()        # Try to receive potentiometer value from other Pico
        if incoming and incoming.startswith("POT:"):
            try:
                # Get just the number from the received message
                partner_percent = int(incoming.split(":")[1])
                print("Received partner potentiometer:", partner_percent, "%") # Show received value
            except ValueError:
                print("Received invalid data:", incoming)     # Handle messages that can't be parsed as numbers
        else:
            print("No valid potentiometer data from partner") # No good message was received
        
        utime.sleep(1)                        # Wait 1 second before repeating everything

# Start running the main loop forever
main_loop()

