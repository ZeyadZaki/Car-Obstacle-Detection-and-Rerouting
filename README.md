https://github.com/ZeyadZaki/Car-Obstacle-Detection-and-Rerouting

## Authors:
* Ahmed Abdelrahman 900161054 <br/>
* Nouran Flaifel 900160793 <br/>
* Zeyad Zaki 900160268 <br/>


## Project Description: 
In this project, we are designing and implementing a smart car that detects obstacles and reroutes. The car will be moving in a specific track to reach a final destination. It should be able to detect any obstacles that it faces and be able to reroute in order to avoid them while still aiming to reach the destination. The figure below shows the expected behaviour of the car in our system:   
![map](https://github.com/ZeyadZaki/Car-Obstacle-Detection-and-Rerouting/blob/main/images/Screen%20Shot%202020-12-16%20at%209.51.00%20PM.png)

## Software Requirements:
* Keil uvision <br/>
* Stm32 CubeMx <br/>
* Teraterm -_optional for debugging_- <br/>

## Hardware Requirements:
* Stm32 Microcontroller <br/>
* Dagu Wild Thumper 4WD Chassis <br/>
* Pololu TReX DC Motor Controller <br/>
* HC-SR04 ultrasonic ranging module <br/>

## Project Design:
* ### Block Diagram:
![block](https://github.com/ZeyadZaki/Car-Obstacle-Detection-and-Rerouting/blob/main/images/smallblock.png)

* ### CubeMx Configuration: 
  * The system clock frequency is set to 8MHz. 
  * UART1: to send the command packets from the microcontroller to the Pololu TReX DC Motor Controller. Should have baud rate 19200 <br/>
  * UART2: this is an optional connection to be able to observe the coordinates and the distances for debugging purposes. Should have baud rate 9600 <br/> 
  * Tim1: timer in input capture mode for the ultra sonic sensor ("Echo" pin). Prescaler is set to 7. <br/>
  * GPIO PIN: for the "Trig" pin in the ultrasonic sensor. <br/>    

![cubemx](https://github.com/ZeyadZaki/Car-Obstacle-Detection-and-Rerouting/blob/main/images/WhatsApp%20Image%202020-12-16%20at%208.15.51%20PM.jpeg)


## Implementation:
* ### Car Movement:
The Pololu TReX Motor Controller which is a versatile DC motor controller is designed to control two bidirectional motors through an asynchronous serial. The serial port has a default baud rate of 19,200 bps and can accept motor command packets to control the connected motors. Below are snippets of the command packets that are sent to the motor controller, to be communicated to the dagu, for motion from the microcontroller: <br/>
	uint8_t brake[4]={0xC0, 0x3F, 0xC8, 0x3F}; <br/>
	uint8_t turnRight[4]={0xC1, 0x00, 0xC9, 0x3F}; <br/>
	uint8_t turnLeft[4]={0xC1, 0x3F, 0xC9, 0x00}; <br/>
	uint8_t forward[4]={0xC1, 0x42, 0xC9, 0x40}; <br/>
The Dagu Wild Thumper 4WD Chassis each wheel is connected to a DC motor for 4-wheel-drive operation, the motors are connected to the TReX motor controller. The two left motors are connected to the controller “M1 outputs” port and the two right motors are connected to the “M2 outputs” port. 

* ### Obstacle Detection:
Ultrasonic sensor is used for the obstacle detection where it is placed in the front of the car. <br/>
The microcontroller controls the ultrasonic sensor using Timers input capture mode in conjunction with PWM. The ultrasonic sensor module generates the ultrasound by setting the Trig pin on a High State for 10 µs. The Module will then send out eight 40 kHz pulses which will travel at the speed sound and will be received by the Echo pin. The Echo pin will output the time the sound wave traveled in microseconds.<br/>
_Distance covered = (Duration of high level Echo pin *speed of sound)/2_ <br/>

Whenever the calculated distance using the front ultrasonic sensor is less than 50 cm the car automatically turns to the right. This is just a primary phase for implementation and we will later adjust it to reroute depending on the target destination. 50 cm was selected since we needed to account for 10 cm protruding from the front of the car while the other 40 cm are space for the car to detect, stop ,and reroute. <br/>

* ### Mapping and Localization:
Since the speed given to the Dagu is constant, we can use that directly to calculate the the location of the car in our map. <br/>
**First**, we needed to _**calculate the speed of the car**_ keeping in mind the time it takes to reach that speed. This is was done by having 3 trials, in each trial we calculate the average velocity:<br/>
1. Divided the area into 6 2m sections <br/>
2. Recorded the time taken to reach each section <br/>
3. Calculated the average velocity <br/>

<img src="https://github.com/ZeyadZaki/Car-Obstacle-Detection-and-Rerouting/blob/main/images/velocity.png" alt="alt text" width="550" height="320"> <br/>


**Then,** using the previous gathered information, we can calculate the exact location by just having a timer that calculates the elapsed time. We used the function _HAL_Gettick()_  that returns the time elapsed since the startup of the program in milliseconds. <br/>
**Finally,** we created a map with relative positioning, setting each direction to be one of the axes and the starting point the origin. We created a function that receives the final destination coordinates (x,y) in meters. This function moves the car in the x direction , then starts moving in the y direction. <br/>

* ### Rerouting:
The rerouting mechanism requires 4 steps; as if you are moving in a square around the obstacle and back to your original path. It firsts turn right, move 40 cm forward. Then turn left and move 40 cm forward; these 40 cm will actually be parallel to the path that our car will be moving in, so it must be accounted for in its current position calculations. After that, we currently passed the obstacle and wish to come back to our destination, so we turn left again, move for 40 cm and finally we turn right to have the car heading to its original path direction. <br/>

* ### Functions used:
   * **void change_dir(char dir):** <br/> This function is responsible for only changing the direction of the car, it takes a single character indicating whether you wish to turn left or right. It first brakes the movement of the thumper for 500ms, then it sends the turnleft/turnright instructions to the car for 900ms (this is the time for the car to make a 90 degrees shift). The total delay this function uses is 1400ms which should be accounted for when calculating the current location of the car in the map. <br/>
   * **void reroute(int dir):** <br/> This function is called when an obstacle is detected. It does the 4 steps described in the rerouting section. It receives an integer stating in which direction was the car moving when it detected that obstacle, to be able to either increment the car's position on the map in the x or in the y direction.
   * **void map(int x_final, int y_final):** <br/> This is the main function of our system; it is function that puts everything together. It receives two integer values; which are the final coordinates of the car in meters. It creates a loop which will not exit from until the x_pos and y_pos of the car are equal to the x_final and y_final parameters. It will first move in the x-direction, after reaching the destination in x, it will turn right and start moving in y-direction. While doing that it checks the distance coming from the ultrasonic sensor, if it is less than 50 cm, it will then call the reroute function and sends either 0 or 1 based on which direction it was moving in.


## Limitations and Furtherwork:
 * Initially, we were intending to get the distance covered by the car using the MPU6050 accelerometer by double integrating. However, after interfacing with the accelerometer and observing it's readings, we concluded that accelerometer readings are too sensitive since they fluctuate even when standing still; therefore, this will result in causing significant inaccuracies which will throw off our distance calculations.Thus, we abandoned using the accelerometer in our system and referred to the method stated previously. the figure below explain more.<br/>

<img src="https://github.com/ZeyadZaki/Car-Obstacle-Detection-and-Rerouting/blob/main/images/accelermoeter.png" alt="alt text" width="550" height="220"> <br/>

 * This project is implemented on this specific Dagu; therefore, for any other car ,the user has to experiment to calculate its own average velocity. <br/>

 * For any future work , a wheel encoder could be used to get the average velocity instead of the technique we used. 

  
