# Concrete_Maturity_Monitoring
Arduino-based embedded system for real-time concrete maturity monitoring and compressive strength estimation

# Concrete Maturity Monitor

Embedded firmware for an Arduino UNO R4 WiFi that monitors
in-situ concrete curing via dual DS18B20 temperature sensors.
Computes the maturity index (°C·h) and estimates compressive
strength (MPa) in real time using a logarithmic strength–maturity
relationship.

## Hardware
- Arduino UNO R4 WiFi
- 2× DS18B20 temperature sensors (1-Wire)
- 16×2 I²C LCD display
- MicroSD card module
- Arduino LED Matrix

## Libraries Required
- hd44780
- DallasTemperature / OneWire
- WiFiS3
- NTPClient
- ArduinoMqttClient
- ThingSpeak

## Output
- Real-time LCD display
- SD card logging (DATA.csv)
- MQTT publish every 10 minutes

## Author
Gökce Yesiloglu — M.Sc Construction and Robotics, RWTH Aachen University, 2026
