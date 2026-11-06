// Purpose: Create usable code for a system composed of a pressure sensor, flow sensor, and pump with ability to display the pressure within the system as well as calculating the flowrate and flow amount of the passing water.

// Define the pins
const int flowSensorPin = 15;
const int pumpPin = 13;
const int buttonPin = 27;
const int pressureSensorPin = 34; // Analog input pin (ADC1)

// Variables for flow sensor
volatile long pulseCount = 0;
volatile unsigned long lastPulseTime = 0; 
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
// Adjust these max values if your transducer has a different rating (e.g. 150 PSI or 0.8 MPa)
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

  // Set pin modes
  pinMode(flowSensorPin, INPUT_PULLUP);
  pinMode(pumpPin, OUTPUT);
  pinMode(buttonPin, INPUT_PULLUP); 
  // Note: no pinMode needed for a GIPO 34
  

  // Ensure pump is OFF at boot (Active-LOW relay)
  digitalWrite(pumpPin, HIGH);

  // Attach interrupt cleanly
  attachInterrupt(digitalPinToInterrupt(flowSensorPin), pulseCounter, RISING);

  oldTime = millis();
}

void loop() {

  // Update sensors every 1 second
  if ((millis() - oldTime) > 1000) {

    // --- 1. SAFELY READ FLOW SENSOR ---
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
    // Read raw 12-bit ADC value (0 to 4095)
    int rawAnalog = analogRead(pressureSensorPin);
    
    // Convert raw ADC to voltage (ESP32 ADC ranges ~0.0V to 3.3V)
    voltage = (rawAnalog / 4095.0) * 3.3;

    // Convert voltage to PSI 
    if (voltage >= SENSOR_MIN_VOLTAGE) {
      pressurePSI = ((voltage - SENSOR_MIN_VOLTAGE) / (SENSOR_MAX_VOLTAGE - SENSOR_MIN_VOLTAGE)) * SENSOR_MAX_PSI;
    } else {
      pressurePSI = 0.0; // Voltage below offset considered 0 PSI
    }
    pressureBar = pressurePSI * 0.0689476; // Convert PSI to Bar

    // --- 3. PRINT RESULTS  ---
    Serial.println("====================================");
    Serial.print("Flow Rate: ");
    Serial.print(flowRate, 2);
    Serial.print(" L/min | Total Volume: ");
    Serial.print(totalLiters, 3);
    Serial.println(" L");

    Serial.print("Pressure: ");
    Serial.print(pressurePSI, 1);
    Serial.print(" PSI (");
    Serial.print(pressureBar, 2);
    Serial.print(" Bar) | Voltage: ");
    Serial.print(voltage, 2);
    Serial.println(" V");
    Serial.println(totalPulse);

    // --- 4. BUTTON & PUMP CONTROL LOGIC ---
    // INPUT_PULLUP logic: Pressed = LOW, Released = HIGH
    if (digitalRead(buttonPin) == HIGH) {
      digitalWrite(pumpPin, LOW); // Active-LOW relay ON
      Serial.println("Pump Status: ON (Button Pressed)");
    } else {
      digitalWrite(pumpPin, HIGH); // Active-LOW relay OFF
      Serial.println("Pump Status: OFF");
    }
  }
}