# SED1115-PB-Final
from machine import Pin, PWM, UART, I2C    # Core hardware imports
from ads1x15 import ADS1015               # Import I2C ADC class
import utime

# ----- PIN AND UART SETTINGS -----
PWM_OUTPUT_PIN = 15        # PWM output pin
KNOB_INPUT_PIN = 28        # Knob analog input
UART_NUMBER = 0            # UART0 for serial comms
UART_BAUD_RATE = 9600

# ----- I2C + ADC SETTINGS -----
I2C_SDA_PIN = 14           # I2C SDA pin
I2C_SCL_PIN = 15           # I2C SCL pin
ADS1015_ADDR = 0x48        # I2C address
ADS1015_PWM_CHANNEL = 2    # PWM filtered analog in (ADS1015 channel 2)

# ----- HARDWARE SETUP -----
pwm_output = PWM(Pin(PWM_OUTPUT_PIN))
pwm_output.freq(1000)
knob_input = ADC(Pin(KNOB_INPUT_PIN))
uart = UART(UART_NUMBER, UART_BAUD_RATE)

i2c = I2C(1, sda=Pin(I2C_SDA_PIN), scl=Pin(I2C_SCL_PIN))
adc_chip = ADS1015(i2c, ADS1015_ADDR, gain=1)

def get_knob_percentage():
    knob_raw = knob_input.read_u16()
    knob_pct = int((knob_raw / 65535) * 100)
    if not (0 <= knob_pct <= 100):
        print("ERROR: Knob reading out of range ({}%).".format(knob_pct))
        return 50
    return knob_pct

def set_pwm_percentage(percent):
    duty_value = int(percent * 65535 // 100)
    pwm_output.duty_u16(duty_value)
    return duty_value

def get_ads1015_pwm():
    try:
        value = adc_chip.read(4, ADS1015_PWM_CHANNEL)  # 1600SPS, ch 2 (see datasheets)
        # Can convert to percent if you know range (gain range and calibration)
        return value
    except Exception as e:
        print("ERROR: I2C ADC read failed:", e)
        return None

def uart_send_message(message):
    try:
        uart.write((str(message) + '\n').encode())
    except Exception as e:
        print("ERROR: UART send failed:", e)

def uart_receive_message(timeout_ms=1000):
    start = utime.ticks_ms()
    while utime.ticks_diff(utime.ticks_ms(), start) < timeout_ms:
        if uart.any():
            try:
                return uart.readline().decode().strip()
            except Exception as e:
                print("ERROR: UART receive failed:", e)
                return None
    print("ERROR: UART receive timeout. Check TX/RX wires or partner Pico.")
    return None

def main():
    while True:
        local_pwm_percent = get_knob_percentage()
        set_pwm_percentage(local_pwm_percent)
        uart_send_message("PWM:{}".format(local_pwm_percent))

        filtered_ads_value = get_ads1015_pwm()
        if filtered_ads_value is None:
            print("ERROR: No filtered analog value, check I2C hardware/connections.")
        else:
            uart_send_message("ADC:{}".format(filtered_ads_value))

        partner_pwm_msg = uart_receive_message()
        if partner_pwm_msg and partner_pwm_msg.startswith("PWM:"):
            try:
                partner_pwm_percent = int(partner_pwm_msg.split(":")[1])
                print("Received partner PWM: {}%".format(partner_pwm_percent))
            except ValueError:
                print("ERROR: Invalid PWM message:", partner_pwm_msg)
        else:
            print("ERROR: No valid partner PWM received.")

        partner_adc_msg = uart_receive_message()
        if partner_adc_msg and partner_adc_msg.startswith("ADC:"):
            try:
                partner_adc_val = float(partner_adc_msg.split(":")[1])
                print("PWM sent: {}%, Partner ADC (raw): {}".format(local_pwm_percent, partner_adc_val))
            except ValueError:
                print("ERROR: Invalid ADC message:", partner_adc_msg)
        else:
            print("ERROR: No valid ADC from partner.")

        utime.sleep(1)

main()
