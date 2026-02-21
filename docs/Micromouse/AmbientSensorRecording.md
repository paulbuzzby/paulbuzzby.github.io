# Capturing Sensor data while running

There was a request to capture ambient sensor reading during a run.

I have always struggled with being able to collect or dump a desired amount of telemetry data from my mouse. I think at least 80% of this is due to MicroPython. The rest very much user error. 
File writes are VERY slow, string manipulation is VERY slow due to how strings are handled. Streaming over Bluetooth also has limited speed due to the available buffer and software. I suspect that faster BLE throughput could be achieved with even more custom software but I am not at the point of writing an android app. I thought I had cracked it with a test around streaming binary data to a file offloaded to the second core on the RP2040 but I just cannot get the throughput. Very jealous of what Ian is able to achieve with the teensy and his data collection. It give me even more respect for how NASA managed to stream telemetry back in the 60's from space craft and how the first mice were even debugged at all.

Anyway, after a LOT of different experiments trying to work out how much data and how fast I could get data out of the system the end result is that I simply have to store data in some sort of ring buffer and the dump it once I have time. Which is code for "When the mouse is stopped".

I know the topic comes up a fair bit on the monthly calls. I have to say I have been having quite a bit of success in getting the AI to help carry out what I call "Donkey work". Some of its technical approaches are questionable at best and just down right wrong at worse but for the testing I have been doing it has been very quick to wrap up code in timing handlers so that call can be timed and output to the screen. A surprise use has been if you give the AI the method declaration on how data is going to be written and then give it the output it can process that data very well. Then it is able to compare one run to another as it has all the data in its memory and can provide analysis on what changes improve or worsen a system. On a similar note you can give it a whole log file and then the example bit that you want to process and it will write a function to pull out that data specifically from the log. If you go a bit more mad you can get that thrown into a python panda data frame for analysis. 

Back to the goal of getting sensor data. I have 4 sensors. So I want to capture the ambient reading and  what I call my RAW reading which is the value once I turn on the LED - Ambient. My value range is 0 - 32000 from my ADC (we ignore that I believe this is technically upscaled by the OS from 12 to 16 bits). Regardless this gives 8 data points per system tick to capture. 
Short version - impossible to capture all for any decent length of time.

The longer end result is that I stuff these 8 values into a single 13 byte array. With each sensor reading being stored as 13bits or 0-8191. Effectively resulting in the values from the ADC being divided by 4.  

I have ring buffer that can store 500 of these results before it will overwrite old data.
For my search run I set the record rate at 4hz (giving just over 2 minutes of record time) with the mouse dumping the buffer when the mouse stops.
For my home maze the speed run can be done in 25ish seconds so the sensor data is recorded at 20hz and dumped once the mouse gets to the goal.

The dumped data is multiplies back by 4, which is probably a waste but it gives the same number range I am used to seeing. It doesn't seem to make any difference to produced graphs. 

Attached are the 2 runs I have done on my home maze. The search run is a full out and back search with sensor data recorded every 250ms (4hz). The speed run is just from home to goal but recorded at 20hz. 
On my home setup my goal cell is (1,0)
The runs were done after sun set and the room is lit with LED bulbs.
The CSV files have the column headings
Front Left Ambient,Left Angle Ambient,Right Angle Ambient,Right Front Ambient,Front Left,Left Angle,Right Angle,Right Front

columns last 4 columns  have already been ambient corrected. 

My mouse uses the TLCR5800 LED's and the BPW96B Phototransistors. 

The maze config was 

```text
o---o---o---o---o---o---o---o---o---o . o . o . o . o . o . o . o
|                                   |   :   :   :   :   :   :   |
o   o---o---o---o---o---o---o---o   o . o . o . o . o . o . o . o
|                               |   |   :   :   :   :   :   :   |
o   o---o   o---o   o---o   o   o   o . o . o . o . o . o . o . o
|   |   |   |   |   |   |   |   |   |   :   :   :   :   :   :   |
o---o   o---o   o---o . o---o   o   o . o . o . o . o . o . o . o
|                       |       |   |   :   :   :   :   :   :   |
o   o---o---o---o   o---o   o---o   o . o . o . o . o . o . o . o
|   |           |           |       |   :   :   :   :   :   :   |
o   o---o   o   o---o---o---o---o   o . o . o . o . o . o . o . o
|   |       |                       |   :   :   :   :   :   :   |
o---o---o---o---o---o---o---o---o---o---o---o---o---o---o---o---o
```

The search route was
Route (cells only): (0,1) -> (0,2) -> (1,2) -> (2,2) -> (3,2) -> (4,2) -> (4,1) -> (5,1) -> (6,1) -> (6,2) -> (7,2) -> (7,3) -> (7,4) -> (6,4) -> (5,4) -> (4,4) -> (3,4) -> (2,4) -> (1,4) -> (0,4) -> (0,5) -> (1,5) -> (2,5) -> (3,5) -> (4,5) -> (5,5) -> (6,5) -> (7,5) -> (8,5) -> (8,4) -> (8,3) -> (8,2) -> (8,1) -> (8,0) -> (7,0) -> (6,0) -> (5,0) -> (4,0) -> (3,0) -> (3,1) -> (2,1) -> (1,1) -> (2,1) -> (2,0) -> (1,0) -> (2,0) -> (2,1) -> (3,1) -> (3,0) -> (4,0) -> (8,0) -> (8,1) -> (8,5) -> (7,5) -> (0,5) -> (0,4) -> (0,3) -> (0,4) -> (1,4) -> (2,4) -> (2,3) -> (2,4) -> (3,4) -> (4,4) -> (4,3) -> (4,4) -> (5,4) -> (6,4) -> (6,3) -> (6,4) -> (7,4) -> (7,3) -> (7,2) -> (6,2) -> (6,1) -> (5,1) -> (4,1) -> (4,2) -> (3,2) -> (0,2) -> (0,1) -> (0,0)

The speed run route was 
Route (cells with entry/exit headings): (2,0) E->N -> (2,1) N->E -> (3,1) E->S -> (3,0) S->E -> (4,0) E->E -> (8,0) E->N -> (8,1) N->N -> (8,5) N->W -> (7,5) W->W -> (0,5) W->S -> (0,4) S->E
(Some cells are missed as in a speed run the mouse will run straight cells in one move)


The ambient results of the search run where
![](images\AmbientSearchRun.png)
The red line in the top 4 graphs is an exponential moving average and the blue dots are the data points.
Bottom graph is all 4 moving averages on the same graph. They somewhat appear to match. I suspect this is showing when the mouse is and isn't facing towards the ceiling light. 

A missing capability in this method of data collection is the ability to accurately identify where the robot is in the maze as the points on the X axis. Just one other improvement to add to the TODO list.

