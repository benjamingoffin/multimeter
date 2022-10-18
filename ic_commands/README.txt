The following script can be written on Arduino IDE and uploaded to the Arduino Uno board of the Low-Cost Electrical Resistivity Meter proposed by Benjamin D. Goffin and Lindsay Ivey-Burden in Computers and Geosciences (2022, under review):

//Include libraries
#include <Wire.h>
#include <Adafruit_INA219.h>
#include <LiquidCrystal_I2C.h>
#include <SD.h>

// INA219 Sensor and LCD screen attached to I2C (Inter-Integrated Circuit) bus as follows:
// SDA (Serial Data Line) - pin A4 (pink)
// SCL (Serial Clock Line) - pin A5 (white)

// Set the I2C address for each device
Adafruit_INA219 ina219;
LiquidCrystal_I2C lcd(0x27, 2, 1, 0, 4, 5, 6, 7, 3, POSITIVE);

// SD card attached to SPI (Serial Peripheral Interface) bus as follows:
// MOSI (Master Out Slave In) - pin 11 (yellow)
// MISO (Master In Slave Out) - pin 12 (green)
// SLK (Serial Clock) - pin 13 (orange)
// CS (Chip Select) - pin 10 (blue)
const int DefaultCS = 10; // Default output
// Set adequate length for string transfer to SD card
char buffer [10];

// Initialize reading sequence
int id = 1;

//-----------------------------------------------------------
// SETUP SETUP SETUP SETUP SETUP
//-----------------------------------------------------------

// void setup, to run once:
void setup() {

// Initialize communication with INA219 sensor
ina219.begin();

// Sensor defaults calibration in 32V, 2A range
// Set a lower 16V, 400mA calibration range to improve precision
ina219.setCalibration_16V_400mA();

// Initialize communication with lcd screen
lcd.begin(20, 4);

// Print title
lcd.setCursor(0, 0);
lcd.print("Resistivity Test");

// Print author
lcd.setCursor(0, 1);
lcd.print(" by BDG @ UVA");

// Wait
delay(500);

// Print Status 1
lcd.setCursor(0, 2);
lcd.print("Booting...");
lcd.setCursor(0, 3);

// Wait
delay(500);

// Set default CS pin as output
pinMode(DefaultCS, OUTPUT);

// See if SD card is present and can be initialized
if (!SD.begin(DefaultCS)) {
lcd.print("Card failed"); // Print Error 1
return; // Exit void setup
}

// See if file can be written/opened
File logFile = SD.open("LOG.csv", FILE_WRITE);

if (logFile) {
logFile.println(", ,"); // Write a leading blank line
String header = "I, V, R";
logFile.println(header); // Write header
logFile.close(); // and close file
lcd.print("File initialized..."); // Print Status 2
}
else {
lcd.print("File failed"); // Else, print Error 2
}

delay(1000);
}

//--------------------------------------------------------
// LOOP LOOP LOOP LOOP LOOP
//--------------------------------------------------------

// void loop, to run repeatedly:
void loop() {

// Clear screen
lcd.clear();

// Get current in mA from INA219 sensor
float current_mA = ina219.getCurrent_mA();

// Convert to SI unit
float current = (current_mA / 1000);

// Print current
lcd.setCursor(1, 0);
lcd.print("I= ");
lcd.print(current_mA,3);
lcd.print(" mAmps");

// Arduino senses voltage in 0-5V range
// A voltage divider creates an ouput voltage that is a fraction of the input
float Ra = 1000000; // input resistor value of voltage divider
float Rb = 100000; // output resistor value of voltage divider
float RR = (Rb/(Ra+Rb)); // voltage divider ratio

// Read analog value from sensor A0
int value0 = analogRead(A0);

// Calculate voltage sensed at output of voltage divider
float vraw = (value0*5.0/1024);

// Adjust voltage at input of voltage divider
float voltage = (vraw / RR);

// Print voltage
lcd.setCursor(1, 1);
lcd.print("V= ");
lcd.print(voltage,3);
lcd.print(" Volts");

// Calculate resistance based on Ohm's law
float resistance = (voltage / current);

// Print resistance
lcd.setCursor(1, 2);
lcd.print("Ra= ");
lcd.print(resistance,1);
lcd.print(" Ohms");
lcd.setCursor(1, 3);

// Compile current, voltage, and resistance data into a string
String dataString = String(dtostrf(current,8,4,buffer)) + "," + dtostrf(voltage,8,6,buffer) +
"," + dtostrf(resistance,8,4,buffer);

//See if file can be written/opened
File logFile = SD.open("LOG.csv", FILE_WRITE);
if (logFile) {
logFile.println(dataString); // Write string
logFile.close(); // and close file
lcd.print("[SD=On, #Rdgs="); // Print Status 3
lcd.print(id);
lcd.print("]");
}
else {
lcd.print("[SD=Off]"); // Else, print Error 3
}

id++;
delay(200); // Time interval between consecutive loops
}