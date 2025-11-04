# SED1115-PB-Final
from machine import Pin, UART         # Import Pin and UART control classes
import time                          # Import time module for timing functions
import adc1                         # Import your ADC config and ready-to-use ADS1015 instance

# Configure UART communication parameters
UART_CHANNEL = 1                   # Use UART1 on Pico
UART_BAUD_RATE = 9600              # Communication speed: 9600 baud
UART_TX_PIN = 4                   # GPIO pin for UART transmit
UART_RX_PIN = 5                   # GPIO pin for UART receive

# Initialize UART with specified pins and baud rate
uart_device = UART(UART_CHANNEL, baudrate=UART_BAUD_RATE,
                   tx=Pin(UART_TX_PIN), rx=Pin(UART_RX_PIN))

# ADC channel for reading filtered PWM input from adc1 config (usually channel 2)
adc_channel_number = adc1.ADS1015PWM

# Time interval between sending successive messages (seconds)
send_interval_seconds = 0.5

# Variable to track last message sent time (milliseconds)
last_message_time = time.ticks_ms()


def send_message(message_text):
    """
    Send a string message over UART, adding newline for termination.
    """
    uart_device.write((message_text + "\n").encode())


def receive_message(timeout_ms=2000):
    """
    Listen for incoming UART string message ending with newline.
    Timeout after timeout_ms milliseconds if no message arrives.
    Returns the received string or None if timed out.
    """
    start_time = time.ticks_ms()  # Record start time for timeout
    received_bytes = b""           # Buffer for incoming bytes

    while time.ticks_diff(time.ticks_ms(), start_time) < timeout_ms:
        if uart_device.any():     # Check if there is data to read
            next_byte = uart_device.read(1)  # Read one byte
            if next_byte:
                received_bytes += next_byte  # Append byte to buffer
                if b"\n" in received_bytes:  # Check for newline terminator
                    try:
                        # Decode byte string to text and strip spaces/newlines
                        return received_bytes.decode().strip()
                    except:
                        return None  # Return None if decoding fails
        else:
            time.sleep_ms(5)  # Short delay to avoid CPU overuse
    return None               # Return None if timeout expires without data


def get_potentiometer_percentage():
    """
    Query the ADC to read the analog value and convert it to an integer percentage 0-100.
    Uses the ADS1015 ADC instance defined in adc1 module.
    """
    try:
        # Read raw signed value from ADC (1600 samples/s rate, configured adc channel)
        raw_adc_value = adc1.adc.read(4, adc_channel_number)

        # Convert from signed int to unsigned if negative
        if raw_adc_value < 0:
            raw_adc_value += 65536

        # Map 0-65535 to 0-100 percentage scale
        percentage = int((raw_adc_value / 65535) * 100)

        # Limit percentage between 0 and 100
        if percentage < 0:
            percentage = 0
        elif percentage > 100:
            percentage = 100

        return percentage

    except Exception as error:
        print("Error reading ADC:", error)
        return 0  # Return zero to indicate error safely


# Start of main operating loop
print(f"Starting UART communication on UART{UART_CHANNEL} (TX={UART_TX_PIN}, RX={UART_RX_PIN})")

while True:
    current_time = time.ticks_ms()  # Get current timestamp in milliseconds

    # Check if it is time to send a new message
    if time.ticks_diff(current_time, last_message_time) > send_interval_seconds * 1000:
        pot_percent = get_potentiometer_percentage()              # Read potentiometer %
        message_to_send = f"SET:{pot_percent}"                    # Format message string
        send_message(message_to_send)                              # Send over UART
        print("Sent message:", message_to_send)                   # Show message in console
        last_message_time = current_time                           # Reset timer

    # Listen for an incoming UART message for up to 200 ms
    incoming_message = receive_message(timeout_ms=200)

    if incoming_message:
        if incoming_message.startswith("SET:"):
            # Process SET command: send back measurement echo
            try:
                incoming_value = float(incoming_message[4:])
                echo_message = f"MEAS:{incoming_value:.1f}"
                send_message(echo_message)                         # Send back measurement
                print(f"Received: {incoming_message} | Replied: {echo_message}")
            except:
                send_message("ERR")                                # Error reply
        elif incoming_message.startswith("MEAS:"):
            # Display measurement values received from partner
            print("Measurement received:", incoming_message[5:])

    time.sleep(0.05)  # Small delay to reduce CPU load across loop iterations
