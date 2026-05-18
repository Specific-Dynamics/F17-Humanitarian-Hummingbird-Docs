*<div align="right"> Vikram Procter, Khai Huynh | Jan 13, 2026 </div>*

# BLDC Motor ESC Hardware
### *Specific Dynamics - F17 Humanitarian Hummingbird*

## Revision Notes:
-   Connect VDSLVL pin to GVDD through 100k, NOT TO GROUND (pg. 27)
-   Ensure MOSFETS are the right way
-   Remember using 3-pwm mode 

## Overview
Electronic Speed Controller (ESC) for a brushless 3 phase motor built around a STM32F405 to implement 6-step sensorless BEMF detection trapaziodal comutation. 

## Component Selction
- Drone Motors: COTS [YSIDO](https://www.aliexpress.com/item/1005005729701162.html)

- Microcontroller [STM32F405RGT6](https://www.digikey.ca/en/products/detail/stmicroelectronics/STM32F405RGT6/2754208) to run timing and phase control. (With 16MHz osc)

- 3V3 Buck [TPS563209DDCR](https://www.digikey.ca/en/products/detail/texas-instruments/TPS563209DDCR/5019042) 

- FET driver [DRV8329BREER](https://www.digikey.ca/en/products/detail/texas-instruments/DRV8329BREER/18178578) as gate driver for each phase mosfet. Each chip drives 6 fets or enough for 1 motor.

- NMOS selected [CSD17581Q5A](https://www.ti.com/product/CSD17581Q5A), [digikey](https://www.digikey.ca/en/products/detail/texas-instruments/CSD17581Q5A/6205539)

## Microcontroller
*Find CubeMX .ioc file in STM32_Brushless... folder for exact pin specifications*

**Back EMF (Bemf) Measurement**
All 3 ADCs and all channels will be used to sample the Bemf for positional and speed feedback correction. Two ADCs are responsible for 

## MOSFET Driver
**Sleep and Reset**  
*nSLEEP Pin*: When this pin is pulled low the device goes into sleep mode. By pulsing it low for 1-1.2us the device can be reset without entering sleep mode. This pin has an internal pulldown, so STM32 **must pull pin high!**

**Fault Detection:**  
*nFAULT pin*: Fault indicator output. This pin is pulled logic low during a fault condition and requires an external pull-up resistor to 3.3V to 5.0V. (Pg. 5)

**Current Sense Operation:**
- Monitors low side drain current by find voltage drop across R_sense. This voltage is then variably amplified.
- *SP, SN pins*: voltage drop across R_sense
- *CSAGAIN pin*: Resistor pull down determines gain (Pg. 23). Should be shorted to GND for 5x amplification. Could be replaced with 50kohm for 10x amplification. For 40x amplification connect CSAGAIN to AVDD or 3.3V 
    - $I=\frac{V_{so}-\frac{V_{CSAREF}}{8}}{CSAGAIN\times R_{SENSE}}$
    - For using a R_sense=0.5mΩ, and CSAGAIN=40x, and a $V_{CSAREF}$=3.3V, and a max V_so=3.3V:  
    - $I_{max}=\frac{3.3V-\frac{3.3V}{8}}{40\times 0.5m\Omega}=144A$
    - With this set up the resolution is 5A/0.1V

**Deadtime (Shoot Through Protection)**
- *DT pin*: Controlls deadtime (shoot through protection). By pulling the pin to ground via a resistor, the res val determines deadtime given by (pg.20):  
    - $R_{DT}(k\Omega)=\frac{Deadtime (ns)}{5}-10k\Omega$
    - If DT is grounded then default dead time of 55ns
    - 10k means 100ns, 20k means 150ns, 30k means 200ns

**Over Current Protection**
- *VDSLVL pin*: (Pg. 27) The voltage inputed to this pin determines threshold for an overcurrent event. The current is measured by finding the voltage drop across the mosfets as $I=V_{DS}*R_{DS(on)}$. When $V_{DS} > V_{DSLVL}$ for longer than the tDS_DG deglitch time, a VDS_OCP event is recognized. 
    - To set max current at 10A, with the R_DS(on)=3mOhm-4mOhm then V_DS=30mV-40mV
    - $V_{DSLVL}$ can range from 0.1-2.5V, meaning given this current limmmit and the R_DS(on) we cannot limmit internally.
    - Using the minimum $V_{DSLVL}=0.1V$ then limmit is set to 34.5A
    - So this feature will be disabled by connecting $V_{DSLVL}$ through **100kΩ** resistor to **GVDD** (pg. 27)
    - Same goes for VSENSE Overcurrent Protection (SEN_OCP) (Pg.27), which looks at the voltage drop across the R_sense

- *DRVOFF pin*: (pg. 24) When this pin is pulled high the Gate drivers are overriden and pulled low turning off all FETs. There is an strong internal pull down to prevent false positives. This pin will be connected to a comparitor monitoring the R_sense voltage to ensure when current max is reached the DRVOFF pin is pulled high

## Back EMF (BEMF) Detection
**Overview**
-   In the 6-step: The floating phase's voltage is measured 
-   We need to detect zero crossing vs virtual neutral (Vbus/2)
-   BEMF voltage is typically around battery voltage (8.4V-6.4V, nominal 7.4V)

**Amplitude Scaling**
-   A resistor divider to scale 8.4V to 3.3V max for STM32 ADC
-   At 40k electrical RPM = 667Hz/Step
-   For all six steps ~4kHz steps, which means ~250us 
-   OpAmps sourced have settling time of 0.5us should be fine but delay wont necessarily help computation if its not necessary
-   Resistor divder of R_1=22k and R_2=11k gives the ~0.33<=3.3V/8.4V
-   The STM ADC calls for <= 10kΩ source impedance, so R_1//R_2=7.3. Low enough for good readings (Pg. 136, equation 1)
-   100Ω series resistor to ADC pin should be put in to limit current for transient spikes

**Filtering**
-   Hardware filtering can be used just to remove PWM switching spikes so cutoff of 100kHz+
-   Heavy filtering can cause delay and therfore desync at high rpm
-   Maybe 10k res and 100pF – 1nF cap
-   Better might just be software filtering