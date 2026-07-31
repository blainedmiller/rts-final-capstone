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
