# Rfid_access.project-

#include <X113647Stepper.h>
#include   <EEPROM.h>     // We are going to read and write PICC's UIDs from/to EEPROM
#include   <SPI.h>        // RC522 Module uses SPI protocol
#include <MFRC522.h>  // Library   for Mifare RC522 Devices
#define ON HIGH
#define OFF LOW
static const int   STEPS_PER_REVOLUTION = 64 * 32;  // steps per revolution for stepper motor
int   i = 0; // Counter for alarm
int val = LOW, pre_val = LOW; // Alarming variables
const   int Buzz = A5;
const int greenLed = A4;
const int wipeB = 8;     // Button   pin for WipeMode
bool programMode = false;  // initialize programming mode to   false

uint8_t successRead;    // Variable integer to keep if we have Successful   Read from Reader

byte stobuzzerCard[4];   // Stores an ID read from EEPROM
byte   readCard[4];   // Stores scanned ID read from RFID Module
byte masterCard[4];    // Stores master card's ID read from EEPROM

// Create MFRC522 Pins.
constexpr   uint8_t RST_PIN = 9;
constexpr uint8_t SS_PIN = 10;

MFRC522 mfrc522(SS_PIN,   RST_PIN);
LiquidCrystal lcd(7, 6, 5, 4, 3, 2); // LCD Pins connection Rs,E,D4,D5,D6,D7
X113647Stepper   myStepper(STEPS_PER_REVOLUTION, A0, A1, A2, A3); //stepper on pins A0 through A3
//   Creat a set of new characters
byte smiley[8] = {0b00000, 0b00000, 0b01010, 0b00000,   0b00000, 0b10001, 0b01110, 0b00000};
byte armsUp[8] = {0b00100, 0b01010, 0b00100,   0b10101, 0b01110, 0b00100, 0b00100, 0b01010};
byte frownie[8] = {0b00000, 0b00000,   0b01010, 0b00000, 0b00000, 0b00000, 0b01110, 0b10001};

void setup() {
   pinMode(Buzz, OUTPUT);// Buzzer pin as Output
  pinMode(greenLed, OUTPUT);//   buzzer Led pin as Output
  digitalWrite(Buzz, OFF);
  digitalWrite(greenLed,   OFF);
  pinMode(wipeB, INPUT_PULLUP);   // Enable pin's pull up resistor
   myStepper.setSpeed(6.5);// set the speed in rpm

  lcd.begin(16, 2);              //   initialize the lcd
  lcd.createChar (0, smiley);    // load character to the   LCD
  lcd.createChar (1, armsUp);    // load character to the LCD
  lcd.createChar   (2, frownie);   // load character to the LCD
  lcd.home ();                   //   go home
  lcd.print("  Inter. SUDAN ");
  lcd.setCursor ( 0, 1 );        //   go to the next line
  lcd.print (" University S&T");
  delay(3000);
   lcd.clear();
  lcd.home ();                   // go home
  lcd.print("*   RFID Access *");
  lcd.setCursor ( 0, 1 );        // go to the next line
   lcd.print ("Control SystemX");
  delay(2000);
  Serial.begin(9600);  //   Initialize serial communications with PC
  SPI.begin();           // MFRC522   Hardware uses SPI protocol
  mfrc522.PCD_Init();    // Initialize MFRC522 Hardware
   //If you set Antenna Gain to Max it will increase reading distance
  mfrc522.PCD_SetAntennaGain(mfrc522.RxGain_max);

   Serial.println(F("RFID Access Control System"));   // For debugging purposes
   //  ShowReaderDetails();  // Show details of PCD - MFRC522 Card Reader details

   //Wipe Code - If the Button (wipeB) Pressed while setup run (powebuzzer on) it   wipes EEPROM
  if (digitalRead(wipeB) == LOW) {  // when button pressed pin should   get low, button connected to ground
    lcd.clear();
    lcd.home ();                   //   go home
    lcd.print("Wiping 10 sec");
    lcd.setCursor ( 0, 1 );        //   go to the next line
    digitalWrite(Buzz, ON); // buzzer Led stays on to inform   user we are going to wipe
    Serial.println(F("Wipe Button Pressed"));
     Serial.println(F("You have 10 seconds to Cancel"));
    Serial.println(F("This   will be remove all records and cannot be undone"));
    bool buttonState = monitorWipeButton(10000);   // Give user enough time to cancel operation
    if (buttonState == true && digitalRead(wipeB)   == LOW) {    // If button still be pressed, wipe EEPROM
      Serial.println(F("Starting   Wiping EEPROM"));
      for (uint16_t x = 0; x < EEPROM.length(); x = x + 1)   {    //Loop end of EEPROM address
        if (EEPROM.read(x) == 0) {              //If   EEPROM address 0
          // do nothing, already clear, go to the next address   in order to save time and reduce writes to EEPROM
        }
        else {
           EEPROM.write(x, 0);       // if not write 0 to clear, it takes 3.3mS
         }
      }
      Serial.println(F("EEPROM Successfully Wiped"));
       lcd.print("EEPROM Wiped");
      lcd.print(char(1));
      digitalWrite(Buzz,   OFF);  // visualize a successful wipe
      delay(200);
      digitalWrite(Buzz,   ON);
      delay(200);
      digitalWrite(Buzz, OFF);
      delay(200);
       digitalWrite(Buzz, ON);
      delay(200);
      digitalWrite(Buzz, OFF);
     }
    else {
      Serial.println(F("Wiping Cancelled")); // Show some   feedback that the wipe button did not pressed for 10 seconds
      lcd.print("Wiping   Cancelled.");
      digitalWrite(Buzz, OFF);
      delay(1000);
    }
   }
  // Check if master card defined, if not let user choose a master card
   // This also useful to just redefine the Master Card
  // You can keep other   EEPROM records just write other than 143 to EEPROM address 1
  // EEPROM address   1 should hold magical number which is '143'
  if (EEPROM.read(1) != 143) {
     Serial.println(F("No Master Card Defined"));
    Serial.println(F("Scan   A PICC to Define as Master Card"));
    lcd.clear();
    lcd.home ();                   //   go home
    lcd.print("No Admin card");
    lcd.setCursor ( 0, 1 );        //   go to the next line
    do {
      successRead = getID();            // sets   successRead to 1 when we get read from reader otherwise 0
      digitalWrite(greenLed,   ON);    // Visualize Master Card need to be defined
      delay(200);
      digitalWrite(greenLed,   OFF);
      delay(200);
    }
    while (!successRead);                  //   Program will not go further while you not get a successful read
    for ( uint8_t   j = 0; j < 4; j++ ) {        // Loop 4 times
      EEPROM.write( 2 + j, readCard[j]   );  // Write scanned PICC's UID to EEPROM, start from address 3
    }
    EEPROM.write(1,   143);                  // Write to EEPROM we defined Master Card.
    Serial.println(F("Master   Card Defined"));
    delay(3000);
    lcd.clear();
    lcd.home ();                   //   go home
    lcd.print("Admin card OK");
    delay(1000);
  }

   Serial.println(F("-------------------"));
  Serial.println(F("Master Card's   UID"));
  lcd.clear();
  lcd.home ();                   // go home
  lcd.print("Admin   card UID:");
  lcd.setCursor ( 0, 1 );        // go to the next line
  for   ( uint8_t i = 0; i < 4; i++ ) {          // Read Master Card's UID from EEPROM
     masterCard[i] = EEPROM.read(2 + i);    // Write it to masterCard
    Serial.print(masterCard[i],   HEX);
    lcd.print(masterCard[i], HEX);
  }
  Serial.println("");
   Serial.println(F("-------------------"));
  Serial.println(F("Everything   is ready"));
  Serial.println(F("Waiting PICCs to be scanned"));
  cycling();
   delay(2000);
  lcd.clear();
  lcd.home ();                   // go home
   lcd.print("System is Ready");
  lcd.setCursor ( 0, 1 );        // go to the   next line
  testing123();

}
