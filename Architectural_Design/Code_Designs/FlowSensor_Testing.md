// Purpose: Test the functionality of the flowsensor that is going to be used to meassure the flowrate and total amount of liquid that has flown past it.

// Pin Definition
const int flowSensorPin = 15;
const int pumpPin = 13;
const int button_pin = 27;

// Variables for flow sensor
volatile long pulseCount = 0;
float flowRate = 0.0;
unsigned long oldTime = 0;
float total_pulse = 0;

// Calibration factor (determined through a series of tests)
const float calibrationFactor = 20.2;

// Interrupt function to count pulses from the flow sensor
void IRAM_ATTR pulseCounter() {
  pulseCount++;
}

void setup() {
  // Start the serial monitor
  Serial.begin(115200);

  // Set pin modes
  pinMode(flowSensorPin, INPUT_PULLUP);
  pinMode(pumpPin, OUTPUT);
  pinMode(button_pin, INPUT_PULLUP);

  // Ensure pump is off at start
  digitalWrite(pumpPin, LOW);

 // attaching an interupter to the flowsensor pin
  detachInterrupt(digitalPinToInterrupt(flowSensorPin)); // Reset just in case
  attachInterrupt(digitalPinToInterrupt(flowSensorPin), pulseCounter, RISING);

  oldTime = millis();
}

void loop() {
  // Update flow rate and check status every 1 second
  if ((millis() - oldTime) > 1000) {
    // Temporarily detach interrupt to read the pulse count safely
    detachInterrupt(digitalPinToInterrupt(flowSensorPin));
    
    // Calculate flow rate in Liters per minute (L/min)
    flowRate = ((1000.0 / (millis() - oldTime)) * pulseCount) / calibrationFactor;

    total_pulse = total_pulse + pulseCount;

    // Reset pulse count and timer
    pulseCount = 0;
    oldTime = millis();

    // Re-attach the interrupt
    attachInterrupt(digitalPinToInterrupt(flowSensorPin), pulseCounter, RISING);

    // Print flow rate to Serial Monitor
    Serial.print("Flow rate: ");
    Serial.print(flowRate);
    Serial.println(" L/min");
    Serial.print("Total pulse: ");
    Serial.println(total_pulse);
    Serial.println(digitalRead(button_pin));

    //pumpPin is a relay that's controlled by button definition 
    if (digitalRead(button_pin) == HIGH) {
      digitalWrite(pumpPin, LOW); // pumPin is the relay that controls the pump with "LOW" = "ON"
      Serial.println("Pump is ON");
    } else {
      digitalWrite(pumpPin, HIGH);  // pump is off when pumpPin is defined as HIGH
      Serial.println("Pump is OFF");
    }
  }
}