## Theme Park Dispatch Controller — Real-Time Systems Final Capstone


 This project simulates a real-time theme park ride dispatch controller for a ride operator at a theme park.

## Demo
Video With my project demo
https://youtu.be/LqyWwRi0OeA

Live Wokwi: Miller-A3-RTS26Summer

https://wokwi.com/projects/468085698071913473


## Project Overview

This project demonstrates how a safety-critical ride control system responds to an emergency stop button. A GPIO interrupt immediately captures the event, records the interrupt latency, and signals a bottom-half task using both a binary semaphore and a direct task notification. The project measures ISR-to-task latency under idle and loaded CPU conditions to compare synchronization mechanisms while maintaining deterministic behavior.

## Architecture

                         Emergency Stop Button (GPIO18)
                                     │
                                     │ Falling Edge Interrupt
                                     ▼
                    ┌─────────────────────────────────────┐
                    │      GPIO Interrupt Service Routine │
                    │-------------------------------------│
                    │ • Timestamp ISR entry               │
                    │ • Debounce (200 μs)                 │
                    │ • GPIO19 latency pulse HIGH         │
                    │ • Signal Binary Semaphore           │
                    │ • Signal Task Notification          │
                    │ • GPIO19 latency pulse LOW          │
                    │ • portYIELD_FROM_ISR()              │
                    └──────────────┬───────────────┬──────┘
                                   │               │
                                   │               │
                         Binary Semaphore     Direct Task
                             btn_sem          Notification
                                   │               │
                                   ▼               ▼
                   ┌────────────────────┐  ┌────────────────────┐
                   │ Bottom-Half Task   │  │ Bottom-Half Task   │
                   │ (Semaphore)        │  │ (Notification)     │
                   │ Priority = 12      │  │ Priority = 12      │
                   │                    │  │                    │
                   │ Measure latency    │  │ Measure latency    │
                   │ Log E-Stop Event   │  │ Log Ride Alert     │
                   └─────────┬──────────┘  └─────────┬──────────┘
                             │                       │
                             └──────────┬────────────┘
                                        │
                                        ▼
                          Emergency Stop Event Logged


            Priority 15                   Priority 10               Priority 5             Priority 2
    
       ┌─────────────────┐          ┌─────────────────┐      ┌─────────────────┐    ┌─────────────────┐
       │task_dispatchLock│          │task_motorControl│      │task_operatorInput│    │   task_log     │
       ├─────────────────┤          ├─────────────────┤      ├─────────────────┤    ├─────────────────┤
       │ Period : 10 ms  │          │ Period : 20 ms  │      │ Period : 50 ms  │    │ Period :100 ms  │
       │ Xorshift RNG    │          │ FIR Filter      │      │ CRC32            │    │ Insertion Sort  │
       └────────┬────────┘          └────────┬────────┘      └────────┬────────┘    └────────┬────────┘
                │                            │                        │                      │
                └────────────────────────────┴────────────────────────┴──────────────────────┘
                                                 Running on Core 1
                          

1. The emergency-stop button generates a falling-edge interrupt on **GPIO18**.
2. The ISR timestamps the event and generates a pulse on **GPIO19** for latency measurement.
3. The ISR signals both the binary semaphore and direct task notification.
4. The bottom-half task wakes, measures ISR-to-task latency, and records the maximum observed latency.
5. Optional background tasks simulate processor contention to evaluate real-time performance under load.


## Task & Timing

| Task | Type | Period | Priority | Deadline | Purpose |
|------|:----:|:------:|:--------:|:--------:|---------|
| GPIO ISR | Interrupt | Event Driven | ISR | Immediate | Detect emergency stop button press |
| Bottom-Half (Notification) | Task | Event Driven | 12 | Immediate | Process ISR notification & measure latency |
| Bottom-Half (Semaphore) | Task | Event Driven | 12 | Immediate | Process semaphore signal & measure latency |
| task_dispatchLock | Periodic | 10 ms | 15 | 10 ms | High-priority processor workload |
| task_motorControl | Periodic | 20 ms | 10 | 20 ms | FIR filter computation |
| task_operatorInput | Periodic | 50 ms | 5 | 50 ms | CRC32 processing |
| task_log | Periodic | 100 ms | 2 | 100 ms | Worst-case insertion sort |




## Utilization Calculations

Task A (Dispatch)

UA = 470 / 10000
   = 0.0470

Task B (Motor Control)

UB = 2931 / 20000
   = 0.1466

Task C (Operator Input)

UC = 4366 / 50000
   = 0.0873

Task D (Ride Log)

UD = 5191 / 100000
   = 0.0519

Total Utilization

U = UA + UB + UC + UD
= 0.0470+ 0.1466+ 0.0873+ 0.0519
= 0.3328

Rate Monotonic Bound (4 Tasks)

U = 0.3328

RM Bound = 0.7568

0.3328 < 0.7568  


EDF Bound

U = 0.3328

0.3328 < 1.0  Feasible

## Hazard Analysis
  | Hazard                                      | Possible Consequence                                                                    | Severity (1–10) | Mitigation                                                                                                                      |
| ------------------------------------------- | --------------------------------------------------------------------------------------- | :-------------: | ------------------------------------------------------------------------------------------------------------------------------- |
| Dispatch lock task misses its deadline      | Ride vehicles could be dispatched too closely together, increasing collision risk.      |      **10**     | Give Dispatch Lock the highest priority (15), monitor WCET, and verify utilization remains below the RMS bound.                 |
| Emergency stop ISR delayed                  | Emergency stop response is delayed, increasing stopping distance.                       |      **10**     | Keep the ISR short, avoid `printf`, delays, and dynamic memory, and use direct task notification to wake the responder quickly. |
| Motor control task overruns                 | Incorrect drive tire speed or braking commands could be issued.                         |      **9**      | Assign a high priority, measure WCET, and monitor execution time during testing.                                                |
| Operator input delayed                      | Dispatch button response feels sluggish or commands are processed late.                 |      **4**      | Run operator input at a lower priority than safety-critical tasks while ensuring it still meets its deadline.                   |
| Logging task delayed or skipped             | Diagnostic information may be lost, making troubleshooting difficult after an incident. |      **2**      | Keep logging at the lowest priority and buffer log messages so safety-critical tasks are unaffected.                            |
| Button bounce generates multiple interrupts | Multiple emergency-stop events may be recorded from one button press.                   |      **5**      | Use software debounce (`DEBOUNCE_US`) before accepting another interrupt.                                                       |
| Priority inversion/resource blocking        | High-priority safety task waits on a lower-priority task holding a resource.            |      **8**      | Protect shared resources with mutexes that support priority inheritance and minimize lock duration.                             |


During testing, CPU load was intentionally increased by enabling four periodic background tasks. The interrupt response latency increased slightly but remained deterministic because the emergency stop handler executes at a higher priority than most background tasks. The system continues servicing emergency events without failure even under processor contention.



As part of testing, portYIELD_FROM_ISR() was temporarily removed to demonstrate the impact on interrupt response time. This increased wake latency and verified the importance of requesting an immediate context switch after ISR completion.

 ## Build & Run
 ## Hardware
ESP32-S3 DevKitC-1
Pushbutton on GPIO18
GPIO19 latency pulse output
Wokwi Simulator
## Software
ESP-IDF
FreeRTOS
PlatformIO
Wokwi
## Running
1. Open project in Wokwi
2. Start simulation
3. Press GPIO18 button
4. Observe latency in Serial Monitor
5. Repeat under WITH_LOAD = 0 and WITH_LOAD = 1
   

## Tailored For

Controls Engineer/Hardware Engineer:

This project demonstrates core real-time embedded concepts commonly used in themed entertainment control systems. It emphasizes deterministic interrupt response, synchronization, timing analysis, and system behavior under processor load, all of which are essential for a real time ride control system.
