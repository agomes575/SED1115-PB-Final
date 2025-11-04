# SED1115-PB-Final
from machine import Pin, PWM, UART, I2C    # Import hardware modules for pin control, PWM, UART, and I2C communication
from ads1x15 import ADS1015               # Import ADS1015 ADC driver class for I2C analog readings
import utime                             # Import time library for delays and timing functions


# ----- PIN AND UART SETTINGS -----
PWM_OUTPUT_PIN = 15        # GPIO pin connected to PWM output signal
KNOB_INPUT_PIN = 28        # GPIO pin for analog knob input (ADC)
UART_NUMBER = 0            # Select UART0 interface for serial communication
UART_BAUD_RATE = 9600       # Baud rate for UART communication


# ----- I2C + ADC SETTINGS -----
I2C_SDA_PIN = 14           # GPIO pin for I2C data line (SDA)
I2C_SCL_PIN = 15           # GPIO pin for I2C clock line (SCL)
ADS1015_ADDR = 0x48        # I2C address of ADS1015 ADC chip
ADS1015_PWM_CHANNEL = 2    # ADS1015 channel connected to filtered PWM input (channel 2)


# ----- HARDWARE SETUP -----
pwm_output = PWM(Pin(PWM_OUTPUT_PIN))       # Initialize PWM on designated pin
pwm_output.freq(1000)                        # Set PWM frequency to 1 kHz
knob_input = ADC(Pin(KNOB_INPUT_PIN))       # Initialize ADC to read analog value from knob
uart = UART(UART_NUMBER, UART_BAUD_RATE)    # Initialize UART interface with defined parameters

# Initialize I2C bus and ADC chip with specified pins and address
i2c = I2C(1, sda=Pin(I2C_SDA_PIN), scl=Pin(I2C_SCL_PIN))
adc_chip = ADS1015(i2c, ADS1015_ADDR, gain=1)  # Create ADC instance with gain setting


def get_knob_percentage():
    """Read the raw knob value, convert to percentage (0-100). Return safe default if out of bounds."""
    knob_raw = knob_input.read_u16()                        # Read raw 16-bit ADC value from knob
    knob_pct = int((knob_raw / 65535) * 100)                # Scale raw value to percentage [0, 100]
    if not (0 <= knob_pct <= 100):                           # Check if knob reading is within valid range
        print("ERROR: Knob reading out of range ({}%).".format(knob_pct))  # Inform about error status
        return 50                                            # Return neutral default percentage if out of range
    return knob_pct                                          # Return actual knob percentage value


def set_pwm_percentage(percent):
    """Convert percentage to 16-bit duty cycle and set PWM output accordingly. Return raw duty value."""
    duty_value = int(percent * 65535 // 100)                 # Convert percentage [0,100] to 16-bit PWM duty range
    pwm_output.duty_u16(duty_value)                          # Apply duty cycle to hardware PWM output
    return duty_value                                        # Return duty cycle value for reference/logging


def get_ads1015_pwm():
    """Read filtered PWM value from ADS1015 over I2C. Returns raw ADC reading or None on error."""
    try:
        value = adc_chip.read(4, ADS1015_PWM_CHANNEL)       # Read ADC channel with 1600 samples per second setting
        # Note: You can convert raw value to percent based on calibration if desired
        return value                                         # Return raw ADC reading
    except Exception as e:
        print("ERROR: I2C ADC read failed:", e)             # Print error details
        return None                                          # Return None to indicate failure


def uart_send_message(message):
    """Send a string message (plus newline) over UART. Includes error catching."""
    try:
        uart.write((str(message) + '\n').encode())          # Encode string and write bytes over UART
    except Exception as e:
        print("ERROR: UART send failed:", e)                # Print send failure error


def uart_receive_message(timeout_ms=1000):
    """Wait for a UART message with optional timeout (milliseconds). Returns decoded string or None."""
    start = utime.ticks_ms()                                # Get current tick count (ms)
    while utime.ticks_diff(utime.ticks_ms(), start) < timeout_ms:  # While within timeout window
        if uart.any():                                      # If UART RX buffer has data
            try:
                msg = uart.readline().decode().strip()     # Read line, decode bytes to string, strip whitespace
                return msg                                  # Return received message string
            except Exception as e:
                print("ERROR: UART receive failed:", e)    # Print decoding error
                return None
    print("ERROR: UART receive timeout. Check TX/RX wires or partner Pico.")  # Timeout reached without data
    return None                                             # Indicate failure to receive


def main():
    """Main program loop performing PWM control, bidirectional UART communication, and error handling."""
    while True:
        local_pwm_percent = get_knob_percentage()          # Read local knob position as PWM percentage
        set_pwm_percentage(local_pwm_percent)               # Set PWM duty cycle according to knob

        uart_send_message("PWM:{}".format(local_pwm_percent))  # Transmit local PWM value to partner

        filtered_ads_value = get_ads1015_pwm()              # Read filtered PWM signal from ADC chip
        if filtered_ads_value is None:                       # Check for ADC read errors
            print("ERROR: No filtered analog value, check I2C hardware/connections.")  # Warn user
        else:
            uart_send_message("ADC:{}".format(filtered_ads_value))  # Send filtered ADC reading over UART

        partner_pwm_msg = uart_receive_message()             # Receive PWM value message from partner
        if partner_pwm_msg and partner_pwm_msg.startswith("PWM:"):  # Basic format check
            try:
                partner_pwm_percent = int(partner_pwm_msg.split(":")[1])  # Parse PWM percentage value
                print("Received partner PWM: {}%".format(partner_pwm_percent))  # Log received PWM
            except ValueError:
                print("ERROR: Invalid PWM message:", partner_pwm_msg)  # Parsing failure
        else:
            print("ERROR: No valid partner PWM received.")    # Missing or malformed message

        partner_adc_msg = uart_receive_message()              # Receive ADC reading message from partner
        if partner_adc_msg and partner_adc_msg.startswith("ADC:"):  # Basic format check
            try:
                partner_adc_val = float(partner_adc_msg.split(":")[1])  # Parse ADC float value
                print("PWM sent: {}%, Partner ADC (raw): {}".format(local_pwm_percent, partner_adc_val))  # Log data
            except ValueError:
                print("ERROR: Invalid ADC message:", partner_adc_msg)  # Parsing failure
        else:
            print("ERROR: No valid ADC from partner.")         # Missing or malformed message

        utime.sleep(1)                                         # Wait 1 second before next communication cycle


main()  # Start execution of main program loop
