/*************************************************** 
 * This is a sketch to measure air pressure with bmp180 and temperature/humidity with dht11
 * 
 *  Original code was written by Marco Schwartz for Open Home Automation and 
 *  Updated by Obaej Tareq. 
 *  
 *  Data is uploaded to "ThingSpeak".
 ****************************************************/

// Libraries
#include <Wire.h>
#include <Adafruit_CC3000.h>
//#include <Adafruit_Sensor.h> // don't need with earlier version of Adafruit_Sensor.h
#include <Adafruit_BMP085.h> //earlier version was used in Instructable link
#include <ccspi.h>
#include <SPI.h>
#include <string.h>
#include<stdlib.h>
#include "utility/debug.h"
#include "DHT.h"
#include <avr/wdt.h> //this is for the watchdog

// Define CC3000 chip pins
#define ADAFRUIT_CC3000_IRQ   3
#define ADAFRUIT_CC3000_VBAT  5
#define ADAFRUIT_CC3000_CS    10

// temperature and humidity sensor (dht11)
#define DHTPIN 2
#define DHTTYPE DHT11

DHT dht(DHTPIN, DHTTYPE);

//air pressure sensor (bmp180)
Adafruit_BMP085 bmp;

// Create CC3000 instances
Adafruit_CC3000 cc3000 = Adafruit_CC3000(ADAFRUIT_CC3000_CS, ADAFRUIT_CC3000_IRQ, ADAFRUIT_CC3000_VBAT,
SPI_CLOCK_DIV2); // you can change this clock speed, but do I need to?


// WLAN parameters at home
#define WLAN_SSID       "xxx"
#define WLAN_PASS       "xxx"

// Security can be WLAN_SEC_UNSEC, WLAN_SEC_WEP, WLAN_SEC_WPA or WLAN_SEC_WPA2
#define WLAN_SECURITY   WLAN_SEC_WPA2

// ThingSpeak Settings
char thingSpeakAddress[] = "api.thingspeak.com";
String writeAPIKey = "xxx";


uint32_t ip; //0 to 4,294,967,295

// Variable Setup
long lastConnectionTime = 0; 
boolean lastConnected = false;
int failedCounter = 0;


void setup(void)
{
  // Initialize
  Serial.begin(115200);
}

void loop(void)
{ 

  ConnectWifi();   //// Connect with Wifi
  
  dht.begin(); //play with adding/deleting this as it is not in any examples I've seen
  
  bmp.begin();

  /* Get barometric pressure from bmp180 */
  float p = (bmp.readPressure()/100.0); //pressure in hPa
  //char p_buffer[15];
  //String pres=dtostrf(p,0,5,p_buffer);
  Serial.print("Absolute Pressure = ");
  Serial.println(p); /*this is where I need to change the code to convert to QHN pressure */

  delay (5000); //This delay seemed to be necessary -- I think it is because of the dht11 sensor
  
  // Get temperature and humidity data from dht11
  float h = dht.readHumidity();
  float t = dht.readTemperature(true); //fahrenheit
  float hif = dht.computeHeatIndex(t, h);

  //print data to serial monitor
  Serial.print("Temperature = ");
  Serial.print(t);
  Serial.println(" F");
  Serial.print("Humidity = ");
  Serial.print(h);
  Serial.println(" %");
  Serial.print("Heat Index = ");
  Serial.print(hif);
  Serial.println(" F");
  Serial.print("QNH Pressure = ");
  Serial.print(((p)*((16000+(64*t))+140)/((16000+(64*t))-140))*0.0295300);
  Serial.println(" inHg");
  
  //get data ready to update ThingSpeak
  int temperature = (int) t;
  int humidity = (int) h;
  int heatindex = (int) hif;
  
  //convert absolute pressure in hPa to QNH pressure in inHg using 
  //int pressure = (int) (((p)*((16000+(64*temperature))+140)/((16000+(64*temperature))-140))*0.0295300);
  float pressure = (((p)*((16000+(64*temperature))+140)/((16000+(64*temperature))-140))*0.0295300);
  
  String tem=String(temperature, DEC); //Thingspeak only data stream is string type
  String hum=String(humidity, DEC);    // Converting integer data to string
  String heat=String(heatindex, DEC);
  //String pr =String(pressure, DEC);
  String inHgpres =String(pressure, DEC);
  
  Serial.print(F("api.thingspeak.com -> "));
  
  while  (ip  ==  0)  {
    if  (!  cc3000.getHostByName("api.thingspeak.com", &ip))  {
      Serial.println(F("Couldn't resolve!, Restarting....."));
      wdt_enable(WDTO_8S); // if host does not response wait 8 sec and restart
      while(1){}
    }
    
    delay(500); //I see this delay on the serial monitor
  }  
   cc3000.printIPdotsRev(ip);
  Serial.println(F(""));
  
   Adafruit_CC3000_Client client = cc3000.connectTCP(ip, 80);
   // Print Update Response to Serial Monitor
  if (client.available())
  {
    char c = client.read();
    Serial.print(c);
  }

 // Update ThingSpeak
  if(client.connected()) 
  {
    // Send temperature and humidity data to Thingspeak site
    updateThingSpeak("1="+tem+"&2="+hum+"&3="+inHgpres+"&4="+heat);  
    delay(25000);   // Thingspeak needs a 15s delay between 2 data stream. this seems to be very necessary!! 
    failedCounter=0;

  }
  else
  {
  Serial.println(F("Connection failed"));
  failedCounter++;
  if (failedCounter==10)
  {
    Serial.println(F("Connection failed for 10 times, Restarting......"));
    wdt_enable(WDTO_8S);   // if 10 continuous data upload failed, restart host after 8s.
    while(1){}
  }
  }

//close wifi connection -- I added this!!
client.close();

cc3000.disconnect();

Serial.println("closed");
Serial.println(F("Waiting 5 minutes before next sensor reading/transmission"));
  
  // Wait 5 minutes until next update
  delay(300000);
}

void updateThingSpeak(String tsData)
{
    Adafruit_CC3000_Client client = cc3000.connectTCP(ip, 80);
  
   
    Serial.println("Connecting to ThingSpeak...");
    Serial.println("Sending Data");
    client.print("POST /update HTTP/1.1\n");
    client.print("Host: api.thingspeak.com\n");
    client.print("Connection: close\n");
    client.print("X-THINGSPEAKAPIKEY: "+writeAPIKey+"\n");
    client.print("Content-Type: application/x-www-form-urlencoded\n");
    client.print("Content-Length: ");
    client.print(tsData.length());
    client.print("\n\n");

    client.print(tsData);
    Serial.println(F("Done"));
    Serial.println("_________");
}


void ConnectWifi(){
   Serial.println(F("\nInitializing..."));
  if (!cc3000.begin())
  {
    Serial.println(F("Couldn't begin()! Check your wiring?"));
    while(1);
  }
  // Connect to WiFi network
  cc3000.connectToAP(WLAN_SSID, WLAN_PASS, WLAN_SECURITY);
  Serial.println(F("Connected!"));

  /* Wait for DHCP to complete */
  Serial.println(F("Request DHCP"));
  while (!cc3000.checkDHCP())
  {
    delay(100);
  }  

}
