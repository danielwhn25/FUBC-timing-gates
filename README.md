# UBC Formula Racing Design Team - Timing Gates System 
I have documented the full design cycle and my current progress. This includes:
  - Complete schematic and PCB layout files (Altium Designer)
  - Analog signal conditioning circuit design and dark current mitigation strategy
  - Firmware architecture (ESP-IDF, interrupt-driven timing, ESP-NOW 2.4 GHz Wi-Fi protocol)
  - Component and IC selection
  - Testing methodology and validation approach
  - Assembly and bring-up documentation
    
# Formula UBC Timing Gates System
## Project Lead: Daniel Ng
## September 2025 – Present
### Project Purpose
UBC's Formula Racing team needed a reliable system to measure lap times and vehicle speed during testing to prepare for our annual competition. The existing solution—manually operated stopwatches—was both inaccurate and labour-intensive, and existing commercial products are expensive and non-customizable. Therefore, I'm leading the design and development of a laser-based timing gate system that automatically records lap times and instantaneous speeds as the car passes through checkpoints on the track (as illustrated in the TinkerCAD below).

The system uses 650 nm red laser modules and receivers positioned on the track. When the car breaks the laser beam, receiver modules detect the interruption and timestamp the event. By placing two gates close together at a fixed distance, we can calculate instantaneous speed; by placing gates at the start/finish line, we can measure lap times. The data transmits wirelessly via 2.4 GHz Wi-Fi from a slave ESP32 MCU to a master ESP32 MCU, then to a laptop that the team can monitor during test days.

# Prototyping
<img width="353*2" height="148.6*2" alt="image" src="https://github.com/user-attachments/assets/7159bb5a-fd12-4fc4-b73b-40e9af208080" />

### My Role and Contributions
As project lead, I'm responsible for the complete design cycle—from initial component selection through PCB layout to firmware development. The Formula Racing team has mechanical subteams and an electrical subteam, but I'm the only one working on the hardware side of this project. The circuit design, schematic capture, and board layout decisions are mine, and two mechanical engineering team members designed the hardware enclosure.

**Hardware Design:** I selected the ESP32-S3 microcontroller as the core of each receiver module because it has built-in Wi-Fi (for wireless data transmission), dual-core processing, and enough GPIO pins to interface with the photodiode receiver and status LEDs. I designed a custom analog 3V3 front-end signal conditioning circuit to convert the photodiode output into clean digital signals that the MCU can read. This involved choosing appropriate op-amps for transimpedance amplification, adding filtering to reject ambient light noise, and designing comparator circuits with hysteresis to prevent false triggers from electrical noise (much of which will be described in depth in the challenges section).

### PCB Layout
I used Altium Designer for schematic capture and board layout. I routed the analog signal conditioning section separately from the digital MCU section, used a ground plane to minimize noise coupling, kept inductor magnetic fields isolated, and added decoupling capacitors close to every IC power and switching pin, along with other layout principles. I also had to be careful about trace impedance and impedance matching for the USB-C data lines (90-ohm differential pairs) to ensure reliable communication when programming the board or dumping data logs.

**Firmware Architecture (current stage):** I'm currently developing the embedded firmware in C using the ESP-IDF framework. I'm using hardware timer interrupts to capture timestamps when the laser beam is broken, and other pins and external interrupts to handle signals from our analog front end. Additionally, we're using ESP-NOW wireless protocol, Espressif's proprietary 2.4 GHz Wi-Fi low-latency communication protocol, to transmit timing data to the master MCU. I chose ESP-NOW over WiFi because it has lower latency and doesn't require connecting to an access point, which makes it more reliable in an outdoor testing environment.

### Testing and Validation
Beyond the timing gates themselves, I've been developing Python scripts for data analysis and visualization. I wrote a real-time stripchart plotter that displays speed and lap time data as it comes in, which helps the team spot performance trends during testing. I'm also using these scripts to validate the timing accuracy—I can compare our laser gate measurements against GPS data from the car's onboard sensors to verify we're getting consistent, reliable readings.

### Technical Challenges and Solutions
### Challenge 1: Signal Conditioning for Photodiode Detection. How can we ensure that our MCU's reading is reliable?

Photodiodes generate current when light is shone onto them. Since these output currents are in the range of microamps, we must amplify this signal using an op amp—specifically, the TI OPA380 transimpedance amplifier (TIA), because it is excellent at amplifying weak signals. However, photodiodes have a phenomenon known as dark current. Since photodiodes are triggered via photons, they are highly sensitive to light. Even in an isolated, light-free environment, a photodiode connected to 5 V will still output a current. Since our output signal to the MCU relies on the photodiode's frontend current, this dark current must be mitigated. If not carefully attuned, the MCU may read the dark current as a logic one. To mitigate this, we must measure the baseline dark current (which varies with temperature) and determine the TIA's output voltage. 

**Step 1:** Measure baseline dark current and record the TIA's output voltage in the worst-case scenario—the highest reasonable outdoor temperature. 

### Challenge 2: Assuming we have found the worst-case output voltage, how can we filter out this worst-case noise? 
Assuming the worst-case dark current generates a $V_{out} = x$ Volts, we must ensure that all input signals are less than $x$ Volts. The challenge was finding an IC that would do this, and the first thought was a comparator (LM393). By "comparing" our TIA's output voltage, which is our comparator's inverting input, to the non-inverting input, we can essentially filter out all values less than $x$.

Mathematically speaking:

Let $V_{out, TIA} = V_{in, comparator} = V_{inverting\ input, comparator} = V_n$

$V_p = V_{non-inverting\ input, comparator}$

**If** $V_n < V_p$, then $V_{out, comparator} = V_{CC_{comparator}}$

**Else**, $V_{out, comparator} = 0\ V$

During the testing phase, we can adjust the ambient temperature and record the output voltage at each temperature. Greater temperatures ⇒ greater dark current ⇒ greater output voltage noise. Depending on our photodiode's dark current sensitivity, our noise levels will vary. Therefore, our reference comparison must also be adjustable. If we know what temperature our laser receiver system will be operating in, we should be able to adjust the "comparison voltage" (which is $V_p$) by using a potentiometer at the non-inverting input. Essentially, we have a "voltage profile" for each temperature, based on the temperature data plotting we gathered in Step 1. 

This comparator setup is known as the Schmitt trigger, designed to output clean, digital signals from analog inputs. Essentially, we have just created our custom analog front end—taking in a noisy and weak input signal, amplifying it, and filtering out false signals using a comparator and sending a clean $V_{out, comparator}$ to our ESP32's GPIO. 

In addition to electrical noise, we have to take into account ambient outdoor noise and false triggers. Since we are operating this receiver system in an outdoor environment, the sun and other environmental elements may input light onto our photodiode. To mitigate this, we can use non-reflective material to shroud our PCB and add a 650 nm red-light filter to block out the noise.

### Next Steps
The schematic design and PCB layout are complete, and I'm currently working on board bring-up and firmware development. I plan to get my PCB manufactured by JLCPCB and solder the components in-house. 

Once assembled, I will measure the base current and TIA output voltage, as outlined in Challenge #1, and adjust the potentiometer to match my findings. Following this would be outdoor testing with the non-reflective shroud to test our theoretical circuit against real noise. 

For future development, in extreme or varying temperatures, we can add a small servo/arm to adjust the comparator's non-inverting input voltage by rotating the potentiometer knob automatically. 

# Finite State Machine Logic
<img width="353*2" height="148.6*2" alt="image" src="https://github.com/user-attachments/assets/ead815e7-2066-435c-8249-6020ee32722e" />

# Laser Receiver (Photodiode) Schematic
<img width="353*2" height="148.6*2" alt="image" src="https://github.com/user-attachments/assets/4a2e9f8b-d89f-4256-a426-806c3c7f4640" />

<img width="353*2" height="148.6*2" alt="image" src="https://github.com/user-attachments/assets/59e6bc49-c12a-4646-bde4-1609f6b82256" />
