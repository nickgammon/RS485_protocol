# RS485_protocol

A library for sending information between Arduinos connected via RS485 wiring as described [here](https://www.gammon.com.au/forum/?id=11428).

A non-blocking version is available at https://github.com/nickgammon/RS485_non_blocking


* Handles "packets" of between 1 and 255 bytes.

* Uses a "begin packet" character (Start of Text, STX, 0x02) to reliably indicate that a packet is starting.

* Uses an "end packet" character (End of Text, ETX, 0x03) to reliably indicate that a packet is ending.

* Each data byte (other than STX/ETX) is sent in a "doubled/inverted" form. That is, each nibble (4 bits) is sent twice, once normally, and once inverted. Thus the only valid values for each nibble are:

  ```
    0F, 1E, 2D, 3C, 4B, 5A, 69, 78, 87, 96, A5, B4, C3, D2, E1, F0
  ```
  
* The inverse (ones complement) of 0 is F, hence 0 becomes 0F. The inverse of 1 is E, hence 1 becomes 1E. And so on.

* This guards somewhat against "bursts" of noise. A burst of either 0s or 1s is unlikely to corrupt a byte preserving this normal/inverse relationship. Also there are only 16/256 valid combinations, so noise has only a 6% chance of becoming a valid byte.

* Because of this, also, the STX and ETX characters cannot appear in ordinary data (they are not one of the 16 valid values).

* Each packet is followed by a CRC (cyclic redundancy check). This is a further test that the packet was received completely. It guards against noise, or possibly some bytes just becoming missing.


## Callback functions


The library was written to allow for various hardware interfaces (eg. software serial, hardware serial, I2C). Thus when using it you supply three "callback" functions which have the job of doing the actual sending/receiving.

For hardware serial, they might look like this:

```c++
void fWrite (const byte what)
  {
  Serial.write (what);  
  }
  
int fAvailable ()
  {
  return Serial.available ();  
  }

int fRead ()
  {
  return Serial.read ();  
  } 
```


For software serial, you might use:

```c++
  #include <SoftwareSerial.h>
  
  SoftwareSerial rs485 (2, 3);  // receive pin, transmit pin
    
  void fWrite (const byte what)
    {
    rs485.write (what);  
    }
    
  int fAvailable ()
    {
    return rs485.available ();  
    }
  
  int fRead ()
    {
    return rs485.read ();  
    }
```

## Master


It is your responsibility to turn on the "write enable" pin before and after doing a "send". This configures the RS485 chip to allow writing to the network. An example master is:

```c++
#include "RS485_protocol.h"
#include <SoftwareSerial.h>

const byte ENABLE_PIN = 4;
const byte LED_PIN = 13;

SoftwareSerial rs485 (2, 3);  // receive pin, transmit pin

// callback routines
  
void fWrite (const byte what)
  {
  rs485.write (what);  
  }
  
int fAvailable ()
  {
  return rs485.available ();  
  }

int fRead ()
  {
  return rs485.read ();  
  }

void setup()
{
  rs485.begin (28800);
  pinMode (ENABLE_PIN, OUTPUT);  // driver output enable
  pinMode (LED_PIN, OUTPUT);  // built-in LED
}  // end of setup
  
byte old_level = 0;

void loop()
{

  // read potentiometer
  byte level = analogRead (0) / 4;
  
  // no change? forget it
  if (level == old_level)
    return;
      
  // assemble message
  byte msg [] = { 
     1,    // device 1
     2,    // turn light on
     level // to what level
  };

  // send to slave  
  digitalWrite (ENABLE_PIN, HIGH);  // enable sending
  sendMsg (fWrite, msg, sizeof msg);
  digitalWrite (ENABLE_PIN, LOW);  // disable sending

  // receive response  
  byte buf [10];
  byte received = recvMsg (fAvailable, fRead, buf, sizeof buf);
  
  digitalWrite (LED_PIN, received == 0);  // turn on LED if error    
  
  // only send once per successful change
  if (received)
    old_level = level;

}  // end of loop
```


This example demonstates how you might command a light in some other part of the house to dim up/down. It reads a potentiometer connected to pin A0 (with the other sides of the pot connected to +5V and Gnd). This gives an analog reading which is then sent to the slave.

We use a 3-byte message format:


* Address of slave (eg. 1 to 255)
* Command (eg. 2 = turn light on)
* Parameter (eg. 128 = half level)


Then we wait for a response from the slave to confirm it got the message. If not, we turn on an "error" LED.

## Slave



The code for the slave could be:

```c++
#include <SoftwareSerial.h>
#include "RS485_protocol.h"

SoftwareSerial rs485 (2, 3);  // receive pin, transmit pin
const byte ENABLE_PIN = 4;

void fWrite (const byte what)
  {
  rs485.write (what);  
  }
  
int fAvailable ()
  {
  return rs485.available ();  
  }

int fRead ()
  {
  return rs485.read ();  
  }
  
void setup()
{
  rs485.begin (28800);
  pinMode (ENABLE_PIN, OUTPUT);  // driver output enable
}

void loop()
{
  byte buf [10];
  
  byte received = recvMsg (fAvailable, fRead, buf, sizeof (buf));
  
  if (received)
    {
    if (buf [0] != 1)
      return;  // not my device
      
    if (buf [1] != 2)
      return;  // unknown command
    
    byte msg [] = {
       0,  // device 0 (master)
       3,  // turn light on command received
    };
    
    delay (1);  // give the master a moment to prepare to receive
    digitalWrite (ENABLE_PIN, HIGH);  // enable sending
    sendMsg (fWrite, msg, sizeof msg);
    digitalWrite (ENABLE_PIN, LOW);  // disable sending
    
    analogWrite (11, buf [2]);  // set light level
   }  // end if something received
   
}  // end of loop
```


The slave simply loops looking for incoming data. The library returns a non-zero "received count" if a valid message is received. Then the slave checks the address (first byte of the message) to see if it is addressed to it (rather than a different slave). If not, it ignores the message.

Then if checks for a valid command (eg. 2 = turn light on). If not, it ignores it.

Finally if it passes these checks, it sends back a response. A small delay (of 1 mS) is inserted to give the master time to prepare for a response. That way the master knows the slave is alive, and responding.

## Flushing the output


A small "gotcha" caught me when testing with hardware serial. The following code didn't work properly:

```c++
 digitalWrite (ENABLE_PIN, HIGH);  // enable sending
 sendMsg (fWrite, msg, sizeof msg);
 digitalWrite (ENABLE_PIN, LOW);  // disable sending
```


Whilst it worked fine with software serial, the code above "turns off" the RS485 chip too quickly, because the last byte is still being sent from the serial hardware port.

A couple of solutions worked:

```c++
 digitalWrite (ENABLE_PIN, HIGH);  // enable sending
 sendMsg (fWrite, msg, sizeof msg);
 delayMicroseconds (660);
 digitalWrite (ENABLE_PIN, LOW);  // disable sending  
```


I'm not very happy with hard-coded delays. Too low and it is still too quick, too high and the slave is responding before you turn the transmitter off. The exact value has to be carefully tuned, and depends very much on the baud rate in use. The value required appears to be appromately two character times. So for 28800 baud, one character time is 1/2880 which is 347 uS. Doubling that gives about 690 uS.

Another approach is for the Atmega328P is:

```c++
 digitalWrite (ENABLE_PIN, HIGH);  // enable sending
 sendMsg (fWrite, msg, sizeof msg);
 
 while (!(UCSR0A & (1 << UDRE0)))  // Wait for empty transmit buffer
     UCSR0A |= 1 << TXC0;  // mark transmission not complete
 while (!(UCSR0A & (1 << TXC0)));   // Wait for the transmission to complete

  digitalWrite (ENABLE_PIN, LOW);  // disable sending  
```


This requires fiddling with hardware registers (and the exact ones depend on whether you are using Serial, Serial1, Serial2 and so on). The first loop waits for the hardware chip's buffer to empty, at the same time setting the "transmission not complete" flag. The second loop waits for the final byte to be clocked out by the hardware.

### Termination resistors


If the transceivers are not at the ends of the cable termination resistors are probably necessary. Something like 120 ohms, connected between the A and B cables, at each end only, stop the signal reflecting back along the cable.
