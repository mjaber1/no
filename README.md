# no
Voltage and Percentage of battery on RPI
#!/usr/bin/env python2.7
# explained here...
# http://raspi.tv/2013/controlled-shutdown-duration-test-of-pi-model-a-with-2-cell-lipo
# DO NOT use this script without a Voltage divider or other means of 
# reducing battery voltage to the ADC. This is explained on the above blog page
import time
import os
import subprocess
import smtplib
import string
import RPi.GPIO as GPIO
from time import gmtime, strftime

GPIO.setmode(GPIO.BCM)


########## Program variables you might want to tweak ###########################
# voltage divider connected to channel 0 of mcp3002
adcs = [0] # 0 battery voltage divider
reps = 10 # how many times to take each measurement for averaging
cutoff = 11.5# cutoff voltage for the battery
previous_voltage = cutoff + 1 # initial value
time_between_readings = 2 # seconds between clusters of readings

# Define Pins/Ports
SPICLK = 8             # FOUR SPI ports on the ADC 
SPIMISO = 23
SPIMOSI = 24
SPICS = 25

####### You shouldn't need to change anything below here #######################
################################################################################

# read SPI data from MCP3002 chip, 2 possible adc's (0 & 1)
# this uses a bitbang method rather than Pi hardware spi
# modified code based on an adafruit example for mcp3008
def readadc(adcnum, clockpin, mosipin, misopin, cspin):
    if ((adcnum > 1) or (adcnum < 0)):
        return -1
    if (adcnum == 0):
        commandout = 0x6
    else:
        commandout = 0x7

    GPIO.output(cspin, True)

    GPIO.output(clockpin, False)  # start clock low
    GPIO.output(cspin, False)     # bring CS low

    commandout <<= 5    # we only need to send 3 bits here
    for i in range(3):
        if (commandout & 0x80):
            GPIO.output(mosipin, True)
        else:   
            GPIO.output(mosipin, False)
        commandout <<= 1
        GPIO.output(clockpin, True)
        GPIO.output(clockpin, False)

    adcout = 0
    # read in one empty bit, one null bit and 10 ADC bits
    for i in range(12):
        GPIO.output(clockpin, True)
        GPIO.output(clockpin, False)
        adcout <<= 1
        if (GPIO.input(misopin)):
            adcout |= 0x1

    GPIO.output(cspin, True)

    adcout /= 2       # first bit is 'null' so drop it
    return adcout


#Set up ports
GPIO.setup(SPIMOSI, GPIO.OUT)       # set up the SPI interface pins
GPIO.setup(SPIMISO, GPIO.IN)
GPIO.setup(SPICLK, GPIO.OUT)
GPIO.setup(SPICS, GPIO.OUT)

try:
    while True:
        for adcnum in adcs:
            # read the analog pin
            adctot = 0
            for i in range(reps):
                read_adc = readadc(adcnum, SPICLK, SPIMOSI, SPIMISO, SPICS)
                adctot += read_adc
                time.sleep(0.05)
            read_adc = adctot / reps / 1.0
            print read_adc

            # convert analog reading to Volts = ADC * ( 3.33 / 1024 )
            # 3.33 tweak according to the 3v3 measurement on the Pi
            volts = read_adc * ( 3.33/ 1024.0) * 4.267
            voltstring = str(volts)[0:5]
            print "Battery Voltage: %.2f" % volts
print "Percentage:%.2f" volts*100/ 12.6 
GPIO.cleanup()
