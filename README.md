

  Engineering analysis:

  1. My ISR works as my theme parks emergency stop button for an operator
    line by line by:

     int64_t now = esp_timer_get_time();
     This line records the isrs input so I can keep track of latency

     if (now - last_edge_us < DEBOUNCE_US) return;
      This line works as a debounce tool so I do not see many interrupts when
      the button is only pressed onces
  
    last_edge_us = now;
    This holds onto the buttons last stored input time

    gpio_set_level(ISR_PULSE_GPIO, 1);
    This line will allow for the logic analyzer to see when the interrupt starts

    isr_entry_time_us = now;
    This will save the timestamp for bottom half latency calcs

    presses_observed++;
    This will count the buttons presses

    BaseType_t higher_woken = pdFALSE;
    This will keep track if the isr wakes a high priority task

    
     
    xSemaphoreGiveFromISR(btn_sem, &higher_woken);
    This will wake the semaphore bottom half task

     
    vTaskNotifyGiveFromISR(task_notif_handle, &higher_woken);
    This will wake the notification task

    
    gpio_set_level(ISR_PULSE_GPIO, 0);
    This will lower teh gpio19 so the logic analyzer can see when its off
    
    portYIELD_FROM_ISR(higher_woken);
    this will request a context switch if a high task was woken

    whats not in the isr?
    There is no printf of any sort, there is no delay or any malloc.
    Thus the isr will simply wake the tasks if the emergency stop was pressed.

  2. 
    When running idle and completing 50 samples
    my worst time latency is:

      notif max = 30us
      sem max = 2410us

    When running it loaded at 1 and completing the samples 
    my worst time latency is:

      notif max = 2682 us
      sem  max = 2520 us

      The direct notification task was faster.

      Both logic analyzer pngs are in the zipfile!

  3. 
    When running ideal and completing 50 samples
    my worst time latency is:

      notif max = 30us
      sem max = 2410us

    When running it loaded at 1 and completing the samples 
    my worst time latency is:

      notif max = 2682 us
      sem  max = 2520 us

    
    the increase factor:
      notif max = 2682/30 = 89.4x 
      sem  max = 2520/2410 = 1.05x

      the large increase in latency comes form the priority of the tasks.
      Since there is a higher priority task (task A ). Due to task A being higher'
      it takes priority over the bottom half task which can cause an execution delay. The
      other three tasks all run at lower priorities then the isr meaning that
      they cannot preempt a higher task like the isr or task A. This means only
      task A effects the worst time latency and not the lower priority tasks. 



  
  
  4. The failure that i induced was removing portYIELD_FROM_ISR(higher_woken)
    The output that I expect to see is that i will see the notification but
    the task might not run until the next tick.

    What I ended up observing is that the system operated like normal
    and continued to fire both tasks. However, the latency became more
    inconsistent and at times larger than normal.


  

  AI Disclosure:

  No AI was used in this App

