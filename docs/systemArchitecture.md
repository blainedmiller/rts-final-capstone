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
