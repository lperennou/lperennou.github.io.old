---
layout: post
title:  "Making a gas detector"
categories: [hardware]
---

### Introduction
Alex, my neighbour, sometimes has odorant gas coming out from the sewage pipes in his office. 
In order to show evidence to the office managers, he wants to deploy gas sensor in different rooms, collect data and finally generate graphs. 
Alex has already assembled the sensor boards, but he needs help with the programming and data collection. 
In this post, I introduce the hardware and software architectures that we have collectively come up with. 
This project was particularly interesting because it was my first opportunity to programm on an embedded platform. 



### Sensor Board
Alex has assembled boards such as the one below.
i
![png](/images/gas_detector/board.png)

A board comprises 4 components:
* An ESP32 microcontroller running micropython and with Wi-Fi connectectivity (black). 
* A gas sensor whose electric resistance varies with gas concentration in the air (grey).
* An analog-digital converter that measures the sensor's resistance and reports to the ESP32 using the I2C communication protocol (blue).
* A micro USB connector to power all above components. 

![png](/images/gas_detector/smell_detector_electrical.png)

### Sensor Electrical Wiring
The electrical schema above shows the sensor has three connectors. 
So how exactly does it work ? 
The sensor itself has two components : 
* An internal resistor (not shown) connected to ***VCC*** and ***GND*** that heats up. 
* A metal oxyde plate, connected to ***VCC*** and the green pin - but not ground.
When heated, the resistance of the metal oxyde plate varies with the gas concentration. 

Also note the presence of another resistor outside of the sensor. 
It is used to make a voltage divider to protect the ADC converter from being exposed to a high voltage. 

![png](/images/gas_detector/voltage_divider.png)

* From ohm's low, Va = Rs * I.
* Vcc = (Rr+Rs) x I ==> I = Vcc / (Rr+Rs). 
* Hence, Va = Rs / (Rr+Rs) * Vcc, and the bigger Rr, the smaller Va compared to Vcc.


### Software
Recall that we want to scatter several sensor boards across Alex's office, then collect and visualize data. 
In the architecture schema below, boards connect to Wi-Fi and send data to a server running on a rasperry pi. 
The server writes data to an InfluxDB database made for time series.
We use Grafana, an interactive visualization web application, to display information to users. 

![png](/images/gas_detector/software_architecture.png)

The sensor boards have to sample data, send the data, and listen to the server for updates in the sampling parameters such as the sampling frequency or batch size. 
However, with a single CPU core, the ESP32 is only able to perform one task at a time. 
In the next section, we explain how we used concurrent programming to solve this paradox. 



## Concurrent Programming with Coroutines
Sampling, sending or receiving data are IO-bound operations: the CPU will spend a large amount of time waiting for events such a sampling timer expiry or TCP acknowledgment. 
Instead of performing these tasks sequentially, we used concurrent programming to allow the CPU to switch to another task while one is waiting for an IO event. 
We implemented our sensor programme using micropython's uasyncio library, a port of python's asyncio library.  
The library allows the definition of *coroutines*, functions that will at some point wait for an IO event and temporarily release the CPU. 
Couroutines are prefixed with *async* in the function definition, and the function body contains an *await* statement. 
Corountines are secheduled for execution though asyncio's *create_task* functions. 
For what it's worth, one can think of the relationship between coroutines and tasks as the relationship between classes and objects. 

The main first creates a task that polls the ADC and sends data to the raspberry.
Then it creates a second task that continuously waits for parameter updates from the raspberry. Such updates trigger the execution of another polling task. 
```python
async def main():
    sensor=Sensor()

    #start sending data to rasp
    uasyncio.core.create_task(sensor.poll(nsamples=20, period_ms=1000, continuous=True)) 
    
    #wait for further instructions from raspberry
    server = uasyncio.stream.Server()
    task = uasyncio.core.create_task(server._serve(sensor.poll, '192.168.20.55', 1234, backlog=5))
    await task
```

The poll corountine first checks wether or not it has to stop due to polling parameters being obsolete. 
The CPU is released between successive samplings. 
After collecting a batch of samples, the task creates a *send_data* task. 
This way, the task that samples data and the one that sends it make use of each other's wait times. 
```python
class Sensor:
    async def poll(self, nsamples=1, period_ms=300, continuous=False):
        # Check if an identical coroutine is not already running.
        # In case client keeps polling with the same parameters
        # we don't want to send duplicate data
        if self.check_params(nsamples, period_ms, continuous):
            print("such co-routine is already running, exiting")
            return 
        self.set_params(nsamples, period_ms, continuous)
        pending = None
        while True :
            samples = []
            for _ in range(nsamples):
                samples.append(self.sample())
                await uasyncio.sleep_ms(period_ms if ((nsamples>1) or continuous) else 1)
            if pending : await pending #wait transmission of previous batch
            pending = uasyncio.core.create_task(self.send_data(samples)) #transmit current batch
            if not continuous : 
                print("batched finished, exiting")
                break
            if not self.check_params(nsamples, period_ms, continuous) :
                print("params have changed, exiting")
                break
        if pending: await pending
```
### Visualization

### Conclusion
This project has been a great opportunity to develop an IoT application, from hardware to software. 
I've learnt to program a microcontroller, developp some knowledge of electrical components and use concurrent programming to optimize IO efficiency. 
