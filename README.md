# NeuroPulse-Ambient: Product Design & BCI Case Study

This document walks through the proof-of-concept (POC) development for **NeuroPulse-Ambient**, a wearable single-channel Brain-Computer Interface (BCI). It also outlines the product roadmap to take this concept from a working desk prototype to a polished, custom-manufactured device designed to help parents and teachers track children's focus during math learning.

This project bridges R&D engineering with product strategy: proving the core signal processing works on low-cost hardware first, and then mapping out a clear path to a user-friendly product.

## 1. Project Overview & Executive Summary

NeuroPulse-Ambient was developed to measure mental focus and relaxation in real time without needing expensive medical equipment or complex multi-sensor caps.

Using three simple contact points on the forehead (two frontal electrodes and one reference point behind the ear), the device picks up subtle electrical activity from the brain. An onboard ESP32 microcontroller processes these signals locally, giving users instant visual feedback through a small, built-in ambient screen while wirelessly sending engagement data to a connected tablet or phone.

```
+-----------------------------------------------------------------------------------+
|                                 SYSTEM ARCHITECTURE                               |
+-----------------------------------------------------------------------------------+
  [ Frontal Electrodes ] ---> [ Analog Signal Filter ] ---> [ ESP32 Core 0: Signal Math ]
  [ Reference (Ear)    ]                                           |
                                                                   v
  [ Math Learning App ] <--- [ Wireless Data Stream ] <--- [ Core 1: Screen Graphics ]
                                                                   |
                                                                   v
                                                           [ Onboard Screen ]


```

## 2. What Was Sourced & Built (Proof of Concept)

To prove this idea was feasible without spending a fortune, the initial working prototype was built using affordable off-the-shelf components. The primary objective was capturing clean biopotential signals while keeping hardware costs as low as possible.

Here is the exact breakdown of what was used for the initial working prototype, alongside the planned cost targets for a 1,000 unit production run.

| Component | What Was Used in the POC | POC Cost (Actual) | Planned Production Component | Production Target Cost (@ 1k units) ( Tentative) | 
| ----- | ----- | ----- | ----- | ----- | 
| **Microcontroller** | ESP32-S3 Development Board | ₹380 | ESP32-S3 Micro Module (Surface Mount) | ₹150 | 
| **Analog Sensor Chip** | MAX30003 SPI Breakout Board   Alternative: AD8232 ( ₹400) | ₹3000 | Bare MAX30003 IC (Integrated onto main board) | ₹175 | 
| **Display Screen** | 1.54 inch Color LCD Screen | ₹500 | Custom 1.28 inch Round IPS Screen | ₹160 | 
| **Electrodes & Cables** | gel electrodes and snap connectors (included within MAX30003 board purchase) | 0 | Custom gold-plated spring pins | ₹70 | 
| **Battery & Charging** | Regular Powerbank | ₹300 | Integrated Battery Protection + Low-Noise Regulator | ₹80 | 
|  |  |  |  |  | 

### Key Achievements in the POC Phase:

* Successfully captured stable brainwave signals using low-cost forehead electrodes.

* Filtered out standard household electrical noise (50Hz powerline interference) directly on the microcontroller.

* Validated that the microchip could run frequency math (FFT transforms) in real time while simultaneously driving the screen smoothly.

## 3. Product Roadmap: Moving from Prototype to Final Device

With the working prototype complete, the next phase focuses on shrinking the hardware, integrating all components onto a single circuit board, and creating a comfortable enclosure.

```
+--------------------------+      +-----------------------+      +--------------------------+
| Phase 1: Completed POC   | ---> | Phase 2: Custom PCB   | ---> | Phase 3: Product Design  |
| Modular Breadboard Setup |      | Single Circuit Board  |      | Wearable Headband Module |
+--------------------------+      +-----------------------+      +--------------------------+


```

### Phase 2: Custom Electronics Integration (PCB Design)

* **Single Board Layout**: Combine the microcontroller, sensor chips, and power management onto one compact 4-layer printed circuit board.

* **Noise Protection**: Keep the sensitive analog sensing components physically separated from the wireless chip on the board to prevent Wi-Fi or Bluetooth transmission noise from distorting the brainwave readings.

* **Safety Features**: Add built-in surge protection components to keep the user completely isolated and safe from any power fluctuations.

### Phase 3: Industrial Design & Visual Appearance

* **Comfort & Fit**: Encase the electronics in a soft-touch matte plastic housing mounted on a lightweight adjustable headband. The electrode contacts use gentle spring tension so they sit comfortably on the forehead without leaving sharp marks.

* **Minimalist Ambient Mode**: The central round display uses smooth, calming color gradients (like soft blues and greens) that change as the user's focus shifts. This keeps the visual feedback intuitive and easy to understand at a quick glance without causing a distraction.

### Phase 4: Alternative Fun & Playful Visual Modes

To make cognitive tracking engaging and enjoyable for children, alternative screen modes can be selected through the settings menu:


* **Goofy Character & Dynamic Emojis**: An animated mascot evolves based on real-time brainwave activity. When the user enters a steady focus zone, the character puts on "brain-power glasses" or powers up with glowing aura effects. If the user drifts into a sleepy state, the avatar gently snoozes or shows sleepy emojis.

* **Interactive Activity Stopwatch**: An on-screen timer tracks active problem-solving sessions (e.g., 15-minute study sprints). It records exact focus streaks and displays a cute "Cognitive Battery" bar showing how much mental energy was sustained throughout the session.

* **Post-Activity Summary Badges**: At the end of a study timer, the screen displays fun achievements (e.g., *"Math Wizard Badge Unlocked"* or *"Deep Focus Streak: 8 Minutes"*), giving kids immediate, rewarding feedback for their effort.

## 4. How the Data Is Processed

The device reads raw electrical signals from the forehead at 256 checks per second. It runs these raw numbers through a basic step-by-step cleaning process:

1. **Noise Removal**: Filters out sudden body movements, muscle twitches, and ambient room noise.

2. **Frequency Division**: Separates the overall signal into standard brainwave frequencies:

   * **Alpha Waves (8–12 Hz)**: Higher when relaxed or sitting with eyes closed.

   * **Beta Waves (13–30 Hz)**: Higher when actively thinking, solving problems, or focusing.

   * **Theta Waves (4–7 Hz)**: Higher when sleepy or drifting off.

3. **Engagement Index Calculation**: Calculates a simple focus score by comparing active thinking waves against background relaxation waves:

$$
\text{Focus Score} = \frac{\text{Beta Power}}{\text{Alpha Power} + \text{Theta Power}}
$$

If the user blinks or clenches their jaw, the system recognizes the sudden spike in movement and temporarily pauses data updating so muscle movements do not mess up the focus score.

## 5. Applied Case Study: Supporting Children in Math Learning

One of the most practical applications for this device is helping kids learn math more effectively.

When children work through math problems, it can be hard for parents or teachers to tell whether a child is bored because the work is too easy, struggling because it is too hard, or engaged at just the right level.

```
          [ HIGH ] +------------------------------------------+
                   |                 FRUSTRATED /             |
                   |                 OVERWHELMED              |
                   |               (High Beta Spike)          |
  Task Difficulty  |                                  ENGAGED |
                   |                              (Balanced)  |
                   |  BORED /                                 |
                   |  DISENGAGED                              |
                   |  (High Alpha/Theta)                      |
          [ LOW  ] +------------------------------------------+
                   [ LOW ]      Mental Effort / Stress  [ HIGH ]


```

### How It Helps Parents & Teachers:

* **Identifying Difficulty Spikes**: When a child hits a problem that is too complex, their brainwave readings show a sudden spike in strain followed by a drop in focus. The device flags this moment so parents and teachers can see exactly which math concepts cause trouble.

* **Recognizing Boredom**: If the math problems are too repetitive, the Focus Score drops while relaxation waves rise. This signals that the child is ready for harder challenges.

* **Timely Help**: Instead of guessing when a child needs assistance, parents receive clear feedback showing when to step in with support and when to let the child work independently.

### Connected App Integration:

The device sends its real-time Focus Score over Bluetooth to an educational tablet app. The app can then automatically adjust the difficulty of the next question:

* If the child is bored $\rightarrow$ Introduce a slightly harder math problem.

* If the child is getting frustrated $\rightarrow$ Offer a hint or break the problem down into smaller steps.

* If the child is engaged $\rightarrow$ Keep the current pace.

## 6. Deliverables & Next Steps

* **Hardware Design**: Finalizing schematics and board files for the custom integrated circuit board.

* **3D Enclosure**: Printing initial headband prototypes to test real-world comfort on different head sizes.

* **Software Firmware**: Cleaning up the dual-core ESP32 code for production stability.

* **Mobile App Bridge**: Writing simple Bluetooth receiver code so iOS and Android educational apps can easily connect to the device.
