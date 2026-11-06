// Purpose: Create a set of code that can control a system of a pressure sensor, solenoid valve, pump, flow sensor, and float switch. This will closely mimic the hardware situation of the actual project but on a relatively minor scale.

// Define the pins
const int flowSensorPin = 5;
const int pumpPin = 13;          // Active-LOW Relay for Pump
const int floatSwitchPin = 27;   // Float Switch connected to GND and Pin 27
const int valvePin = 21;         // Active-LOW Relay for 12V Electric Valve
const int pressureSensorPin = 34; // Analog input pin (ADC1)

// Variables for flow sensor
volatile long pulseCount = 0;
volatile unsigned long lastPulseTime = 0; // Software noise filter
float flowRate = 0.0;
unsigned long oldTime = 0;
float totalPulse = 0.0;

// Variables for pressure sensor
float voltage = 0.0;
float pressurePSI = 0.0;
float pressureBar = 0.0;

// Calibration factor
const float calibrationFactor = 20.2;

// Pressure transducer specs
const float SENSOR_MIN_VOLTAGE = 0.8; 
const float SENSOR_MAX_VOLTAGE = 4.5; 
const float SENSOR_MAX_PSI = 80.0; 

// Interrupt function
void IRAM_ATTR pulseCounter() {
  unsigned long now = millis();
  if (now - lastPulseTime > 5) { // Ignore spikes faster than 5ms
    pulseCount++;
    lastPulseTime = now;
  }
}

void setup() {
  Serial.begin(115200);

  // Set ADC attenuation to 11dB for ESP32 full 0V-3.3V range
  analogSetAttenuation(ADC_11db);

  // Set pin modes
  pinMode(flowSensorPin, INPUT_PULLUP);
  pinMode(pumpPin, OUTPUT);
  pinMode(valvePin, OUTPUT);
  pinMode(floatSwitchPin, INPUT_PULLUP);

  // Ensure pump and valve are OFF at boot
  digitalWrite(pumpPin, HIGH);
  digitalWrite(valvePin, HIGH);

  // Attach interrupt
  attachInterrupt(digitalPinToInterrupt(flowSensorPin), pulseCounter, RISING);

  oldTime = millis();
}

void loop() {

  // Update sensors every 1 second
  if ((millis() - oldTime) > 1000) {

    // --- 1. READ FLOW SENSOR ---
    noInterrupts();
    long currentPulses = pulseCount;
    pulseCount = 0;
    interrupts();

    unsigned long elapsedTime = millis() - oldTime;
    oldTime = millis();

    totalPulse += currentPulses;
    flowRate = ((1000.0 / elapsedTime) * currentPulses) / calibrationFactor;
    float totalLiters = totalPulse / (calibrationFactor * 60.0);

    // --- 2. READ PRESSURE SENSOR ---
    int rawAnalog = analogRead(pressureSensorPin);
    
    // Convert raw ADC to voltage 
    voltage = (rawAnalog / 4095.0) * 3.3;

    // Convert voltage to PSI 
    if (voltage >= SENSOR_MIN_VOLTAGE) {
      pressurePSI = ((voltage - SENSOR_MIN_VOLTAGE) / (SENSOR_MAX_VOLTAGE - SENSOR_MIN_VOLTAGE)) * SENSOR_MAX_PSI;
    } else {
      pressurePSI = 0.0; // Voltage below offset considered 0 PSI
    }
    pressureBar = pressurePSI * 0.0689476; // Convert PSI to Bar

    // --- 3. PRINT RESULTS ---
    Serial.println("====================================");
    Serial.print("Raw ADC: ");
    Serial.print(rawAnalog);
    Serial.print(" | Voltage: ");
    Serial.print(voltage, 2);
    Serial.println(" V");

    Serial.print("Flow Rate: ");
    Serial.print(flowRate, 2);
    Serial.print(" L/min | Total Volume: ");
    Serial.print(totalLiters, 3);
    Serial.println(" L");

    Serial.print("Pressure: ");
    Serial.print(pressurePSI, 1);
    Serial.print(" PSI (");
    Serial.print(pressureBar, 2);
    Serial.println(" Bar)");
    Serial.println(totalPulse);

    // --- 4. FLOAT SWITCH, PUMP & VALVE CONTROL LOGIC ---
    // With INPUT_PULLUP:
    // Float Switch CLOSED (Triggered / Low Water) = LOW
    // Float Switch OPEN (Idle / Full Water) = HIGH
    
    if (digitalRead(floatSwitchPin) == LOW) {
      digitalWrite(pumpPin, LOW);   // Active-LOW relay ON (Pump runs)
      digitalWrite(valvePin, LOW);  // Active-LOW relay ON (Valve opens)
      Serial.println("System Status: ACTIVE (Float Switch Closed - Pump ON, Valve OPEN)");
    } else {
      digitalWrite(pumpPin, HIGH);  // Active-LOW relay OFF (Pump stops)
      digitalWrite(valvePin, HIGH); // Active-LOW relay OFF (Valve closes)
      Serial.println("System Status: IDLE (Float Switch Open - Pump OFF, Valve CLOSED)");
    }
  }
}