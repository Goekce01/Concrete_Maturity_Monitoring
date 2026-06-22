CONSTANTS:
  C_VALUE       ← 1.3          // empirical base for maturity model
  TEMP_INTERVAL ← 60 s         // temperature sampling period
  SD_INTERVAL   ← 600 s        // SD card write period
  MQTT_INTERVAL ← 600 s        // MQTT publish period

VARIABLES:
  tempA, tempB                  // instantaneous temperatures [°C], Sensors A and B
  tempAccA, tempAccB            // accumulated temperature sums for averaging
  tempSamples                   // number of samples in current accumulation window
  Mh_A, Mh_B                   // cumulative maturity index [°C·h], Sensors A and B
  strengthMPa_A, strengthMPa_B  // estimated compressive strength [MPa]
  startTime                     // system start timestamp
  lastMaturityUpdate            // timestamp of last maturity calculation

─────────────────────────────────────────────────────────
PROCEDURE SETUP:
  Initialise serial communication
  Initialise LCD display
  Initialise temperature sensor array (1-Wire bus)
  Initialise LED matrix display
  Attempt Wi-Fi connection
  Attempt SD card mount
  Begin NTP time client
  Set startTime ← current system time
  Set lastMaturityUpdate ← current system time

─────────────────────────────────────────────────────────
PROCEDURE MAIN LOOP (executes continuously):

  // --- Connectivity maintenance ---
  IF Wi-Fi check interval elapsed THEN
    IF Wi-Fi is disconnected THEN attempt reconnection

  IF SD check interval elapsed THEN
    IF SD card is not mounted THEN attempt remount

  // --- Sensor acquisition ---
  IF temperature sampling interval elapsed THEN
    CALL READ_TEMPERATURES()

  // --- Maturity and strength computation ---
  IF one-minute mark elapsed THEN
    CALL CALCULATE_MATURITY()
    strengthMPa_A ← STRENGTH_FROM_MATURITY(Mh_A)
    strengthMPa_B ← STRENGTH_FROM_MATURITY(Mh_B)
    Update LED matrix display

  // --- Output ---
  IF LCD update interval elapsed THEN
    CALL UPDATE_LCD()

  IF MQTT publish interval elapsed THEN
    Connect MQTT client if disconnected
    Publish {tempA, Mh_A, strengthMPa_A} to topic arduino/temp/A
    Publish {tempB, Mh_B, strengthMPa_B} to topic arduino/temp/B

  IF SD write interval elapsed THEN
    CALL WRITE_TO_SD()

  IF matrix alternation interval elapsed THEN
    Toggle display between Sensor A and Sensor B data
    Update LED matrix display

─────────────────────────────────────────────────────────
PROCEDURE READ_TEMPERATURES():
  Request temperature conversion from sensor array
  a ← temperature reading from Sensor A
  b ← temperature reading from Sensor B
  IF a is valid THEN
    tempA ← a
    tempAccA ← tempAccA + a
  IF b is valid THEN
    tempB ← b
    tempAccB ← tempAccB + b
  IF any reading is valid THEN tempSamples ← tempSamples + 1

─────────────────────────────────────────────────────────
PROCEDURE CALCULATE_MATURITY():
  // Uses Freiesleben Hansen & Pedersen equivalent-age model
  // adapted with empirical base C_VALUE
  IF tempSamples = 0 THEN RETURN

  avgA ← tempAccA / tempSamples
  avgB ← tempAccB / tempSamples

  base ← C_VALUE ^ (−2.245)

  M_A ← 10 · (C_VALUE ^ (0.1 · avgA − 1.245) − base) / ln(C_VALUE)
  M_B ← 10 · (C_VALUE ^ (0.1 · avgB − 1.245) − base) / ln(C_VALUE)

  dt ← elapsed time since lastMaturityUpdate [hours]
  Mh_A ← Mh_A + M_A · dt
  Mh_B ← Mh_B + M_B · dt
  lastMaturityUpdate ← current system time

  Reset tempAccA, tempAccB, tempSamples to 0

─────────────────────────────────────────────────────────
FUNCTION STRENGTH_FROM_MATURITY(Mh) → MPa:
  // Logarithmic strength–maturity relationship
  RETURN max(21.6 · ln(max(Mh, 0.01)) − 116.0,  0.0)

─────────────────────────────────────────────────────────
PROCEDURE WRITE_TO_SD():
  IF SD card is not mounted THEN RETURN
  Open DATA.csv in append mode
  IF header not yet written THEN
    Write column headers: Date, Time, RelTime, TempA, TempB, Mh_A, Mh_B, StrengthA, StrengthB
  Write one data row: [real date, real time, elapsed HH:MM, tempA, tempB, Mh_A, Mh_B, strengthMPa_A, strengthMPa_B]
  Close file

─────────────────────────────────────────────────────────
PROCEDURE UPDATE_LCD():
  Line 1: [elapsed HH:MM]  A[tempA °C]  B[tempB °C]
  Line 2: [Wi-Fi status] [SD status]
           IF strength > 0: display S A[strengthMPa_A] B[strengthMPa_B]
           ELSE:            display M A[Mh_A] B[Mh_B]