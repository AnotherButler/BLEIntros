#Event-driven programming

mbed programming is event-driven, meaning it responds to interrupts coming from the hardware. Interrupts are generated by changes in electrical signals or system activity such as radio communication. In event-driven programming, interrupts often lead to the execution of special functions called *event handlers* by the OS. In the context of BLE, event handlers may be triggered quite regularly, for example if a sensor sends a measurement every x seconds, or they may be triggered intermittently, for example only by user interaction.

##waitForEvent() - sleep loops

Event handlers are able to pre-empt the main program, that is - interrupt its execution in order to do their work. The main program will resume when the interrupting event is fully handled. In the case of BLE, we expect the main program to be a sleep loop (``waitForEvent()``), which means that the device will sleep unless it receives an interrupt. Programming like this is necessary to take advantage of the low power nature of BLE.

<span style="text-align:center; display:block;">
![events](../Introduction/Images/EventHandle.png "An event interrupts the main loop and triggers an action. The interrupt is handled, and the event handler then returns control to main()")
</span>
<span style="background-color: #F0F0F5; display:block; height:100%; padding:10px;">
An event interrupts the main loop and triggers an action. The interrupt is handled, and the event handler then returns control to main()</span>

The relationship between ``main()`` and event handlers is all about timing, especially the decision about which code to move to an event handler and which to leave in ``main()``. Handler execution time is often not determined by the size of the code. It can instead be determined by how many times it must run - for example, how many iterations of a data-processing loop it performs. It can also be determined by communication with external components such as sensors (also called *polling*). Communication delays can range from a few microseconds to milliseconds, depending on the sensor involved. Reading an accelerometer can take around a millisecond, and a temperature sensor can take up to a few hundred microseconds. A barometer, on the other hand, can take up to 10 milliseconds to yield a new sensor value. 

An event, such as a sensor reading, wants to trigger an event handler that will wake the device and run immediately. This can happen if the event arrives when the program is in ``main()`` (when the device is sleeping, in our case). But if the event arrives when an event handler is being executed, it may have to wait for the first event to be handled in full. In this scenario, the first event is blocking the execution of the second event. Because event handlers can block each other, they are supposed to execute quickly and then return control to ``main()``. The quick return to ``main()`` allows the system to remain responsive. In the world of microcontrollers, anything longer than a few dozen microseconds is too long. A millisecond is an eternity. Therefore, activities longer than 100 microseconds, such as data processing and sensor communication, should be put in ``main()`` and not in an event handler. This allows event handlers to interrupt long-running processes, meaning the system remains responsive while running these processes. 

In these cases, the event handler is used not to perform functions but rather to enqueue work for the main loop. In the [heart rate demo](http://developer.mbed.org/teams/Bluetooth-Low-Energy/code/BLE_HeartRate/), the work of polling for heart rate data is communicated to the main loop through the variable ``triggerSensorPolling``, which gets set periodically from an event handler called ``periodicCallback()``. 

First, we see the ``triggerSensorPolling`` parameter and ``periodicCallback`` function declarations:

```c
	
	// the parameter triggerSensorPolling begins as FALSE. 
	// It will be set to TRUE in the function periodicCallback
	static volatile bool  triggerSensorPolling = false; 

	[...]
	
	void periodicCallback(void)
	{
		led1 = !led1; /* Do blinky on LED1 while we're waiting for BLE events */

		/* Note that the periodicCallback() executes in interrupt context, 
		* so it is safer to do
		* heavy-weight sensor polling from the main thread. */
		triggerSensorPolling = true; // sets TRUE and returns to main()
	}

```

Next, we tell the program (in ``main()``) to execute ``periodicCallback`` every second:


```c

	int main(void)
	{
		led1 = 1;
		Ticker ticker;
		ticker.attach(periodicCallback, 1); // calls the callback function every second

```


Finally, we use ``triggerSensorPolling`` in an infinite loop in ``main()``. Its value (TRUE or FALSE) determines which bit of the code is executed - the heart rate update or ``waitForEvent``:

```c

    // infinite loop
    while (1) {
        // check for trigger from periodicCallback()
        if (triggerSensorPolling && ble.getGapState().connected) { 

	/* if periodicCallback set the value of triggerSensorPolling to TRUE, 
	* we execute the code block that follow. 
	* The first thing it does is reset triggerSensorPolling to FALSE. 
	* Then it executes the interrupt action, which in our case is 
	* simply to change the heart rate */

            triggerSensorPolling = false; 

            // Do blocking calls or whatever is necessary for sensor polling.
            // In our case, we simply update the HRM measurement. 
		
			[...]

        } else { // if nothing came from the sensor, we stay with waitForEvent()
            ble.waitForEvent(); // low power wait for event
        }
    }
```

##Callback functions

To be energy efficient, a device should sleep as often as possible. Normal (sequential) programming doesn't support this ideal: code is executed one line at a time, meaning a line that doesn't finish running blocks the program's progress. The device is then forced to remain active without accomplishing anything. A good example is a line of code that requires user input that may come at any time at all (or even never). Ideally, a device would like to sleep while waiting for the input. But sequential programming will stall the device on the “wait for input” line instead of reaching the ``waitForEvent()`` function that allows the device to sleep. A clever “wait for input” line will itself put the device to sleep, which improves the device's energy use. But that may still be less than ideal, as the “wait for input” line forces the device to sleep without checking whether there is other code that could be running.

To get around this problem, the **mbed API** allows you to associate activity with a resource or a system event. The API will then enqueue and schedule activities for the resource, rather than force the main program to wait for the resource. This allows the main program to move on and eventually reach ``waitForEvent()``, where the device will rest. When an event arrives, or when a resource is either triggered (by an input) or released from previous activity, the device will wake. It will then perform the pending tasks enqueued by the API.

Associating an activity with a resources is done using **callback functions**. As we said, callback functions on mbed rely on the APIs. mbed_API and BLE_API support different sets of callback functions: mbed_API focuses on the platform and BEL_API focuses on the BLE controller and stack. Some examples of built-in BLE callback functions are:

1. ``onConnection`` and ``onDisconnection``, responding to connection and disconnection events.

2. ``onDataWritten``, responding to a client writing into a characteristic’s value.

3. ``onUpdatesEnabled``, responding to a client requesting to be notified of updates.

###Waiting for events

The first use of the callback function is to allow the program not to stall when it reaches an event that hasn’t happened yet. Instead, it will continue running and eventually allow the device to sleep. A good example is waiting for an input button to be pressed, as can be seen in our [input service templates](../Advanced/InputButton.md):

````c

	// Instantiating an object of the InterruptIn class for receiving interrupts
	InterruptIn button(BUTTON1); 
	…
	
	// the callback functions are not part of main(); 
	// they’re associated with their triggers  from main

	// reaction to falling edge - a button press changes 
	// the current received from the button

	void buttonPressedCallback(void) 	{
		buttonPressed = true;
		// gives the buttonState characteristic the value TRUE
		ble.updateCharacteristicValue(buttonState.getValueHandle(), 
			(uint8_t *)&buttonPressed, sizeof(bool));
	}

	// reaction to rising edge - releasing the button changes the current again
	void buttonReleasedCallback(void) 
	{
		buttonPressed = false;
		// gives the buttonState characteristic the value FALSE
		ble.updateCharacteristicValue(buttonState.getValueHandle(), 
			(uint8_t *)&buttonPressed, sizeof(bool));
	}
	...
	int main(void)
	{
	…
		// these lines tell the program to set up the functions and move on; 
		// mbed OS will ensure that the functions are called when the events occur. 
		// If these lines were the function code, rather than a call to the functions, 
		// we’d stay on the first line until the button was pressed and then
		// on the second line until the button was released
		
button.fall(buttonPressedCallback);// falling edge
		button.rise(buttonReleasedCallback);// rising edge


````

In mbed BLE we use callback functions to allow ``main()`` to reach ``waitForEvent()``. For example, in the heart rate monitor sample we call ``periodicCalback`` once a second. Between calls, the system sleeps within ``waitForEvent()``:

```c

int main(void)
{
led1 = 1;
Ticker ticker;
ticker.attach(periodicCallback, 1); //

while (1) {
        // check for trigger from periodicCallback()
 		       if (triggerSensorPolling && ble.getGapState().connected) {
    		        		triggerSensorPolling = false;
	
[... a bit more code here ...]

} else {

ble.waitForEvent():

```

Here’s another example from the heart-rate demo for setting up a callback to handle a disconnection event:

	    ble.onDisconnection(disconnectionCallback);

This is a call to the function ``onDisconnection`` that’s defined in ``BLEDevice.h`` (a part of ``BLE_API``). This call includes the callback function ``disconnectionCallback`` that was defined in ``main.cpp``:

```c

void disconnectionCallback(Gap::Handle_t handle, Gap::DisconnectionReason_t reason)
{
    		ble.startAdvertising(); // restart advertising
}

```


###Sequential calls

Another good use for a callback function is when we need two functions to work one after another, and the second function requires information produced by the first. It would be inefficient to  have ``main()`` call the first function, wait for results and only then call the second function. Instead, we let ``main()`` call the first function with the second function passed in as a parameter. Passing the second function as a parameter means it will be called at the end of the first function's execution. This stops the program from hanging while waiting for a function to conclude. 

So here’s a plain-language example:

```

	function 1 { 
		receive input from some blocking source like a sensor and create new output; }
	function 2 { 
		receive input from function 1 and process it; }

	main()
	{
	// this will trigger function 1 and then call function 2 at the tail end

	call function1(using function 2); 	
	}

```
