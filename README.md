
![Logo](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/RisitasLogo.png)

This document will contain the step by step with the necessary processes to carry out this project; which will include in detail what parts are required, What tools, versions, how to program computer systems, and review their states.



## Cadence Sensor

A cadence sensor is a device capable of measuring the revolutions per minute (RPM) of the connecting rod or pedals of the bicycle.

The idea with this sensor is to measure the speed of the rider, following the formula of the cutting speed that tells us that Velocity is equivalent to Phi(π) per Diameter per RPM, divided by 1000.


### Simplified operating steps

1. OVERVIEW OF OPERATION
* Cadence sensor
* UDP Client Architecture Segment
* Programs to use

2. BLUETOOTH LOW ENERGY PROTOCOL
* BLE Overview 
* Code initialization 
* State Machine 
* Calculating RPM

3. SIMPLE BICYCLE PHYSICS
* Connecting to the serial port 
* Code initialization 
* Thread removal process 
* RPM transformation to velocity

***Overview of operation***

**Cadence sensor**

[Bryton Rider magnet-less Dual](https://www.amazon.com/Bryton-Smart-Speed-Cadence-Sensor/dp/B07227TTX4/ref=sr_1_3?crid=JD6L4KAJQS94&keywords=bryton+cadence+sensor&qid=1663290246&sprefix=bryton+cadence+sensor%2Caps%2C134&sr=8-3)

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/CadenceSensor.png)

For this process we were implementing the Bryton cadence sensor; that allows us to place it easily  and operation through BLE protocol.

**UDP Client Architecture Segment**

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/Segmento1.png)

The cadence sensor will function as the first UDP connection; being also the only one that is done by Bluetooth.

**Programs to use**

*Visual studio*
![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/VisualStudio.png)

This will host the application that will act as UDP Server. For this it is necessary to add a reference to the project that allows us to assist language projections through readable metadata files; in this case to be able to use a library.

[Library System.Runtime](https://www.nuget.org/packages/System.Runtime.WindowsRuntime/)

*Unity*

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/Unitylogo.png)

The UDP client that will take the RPM and act with respect to the velocity that the user takes in real time. We will use editor version 2021.3.10f1


***Bluetooth Low Energy (BLE) protocol***

**BLE Overview**

[Understand the basics](https://learn.adafruit.com/bluetooth-bicycle-speed-cadence-sensor-display-with-clue/understanding-ble)

To understand how we communicate between the cadence sensor and the UDP Server. it's first important to get an overview of how Bluetooth Low Energy (BLE) works in general. The cadence sensor uses Bluetooth Low Energy, or BLE which is a wireless communication protocol used by many devices, including mobile devices. 

We need to know that BLE devices have two modes:

* Broadcasting mode (also called **GAP** for **G**eneric **A**ccess **P**rofile)
* Connected device mode (also called **GATT** for **G**eneric **ATT**ribute Profile).

The GAP model consists of 2 users the *Broadcaster* who sends a series of data and advertisements; the *observer* that check advertisements being broadcast by the Broadcaster.
![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/mapa.jpg)

**Code initialization**

*Instances*

```c#
using System;
using System.Net;
using System.Net.Sockets;
using System.Threading;
using System.Threading.Tasks;
using Windows.Devices.Bluetooth;
using Windows.Devices.Bluetooth.GenericAttributeProfile;
using Windows.Devices.Enumeration;
using Windows.Storage.Streams;
```
To start with this process we will need to use several libraries including Sockets, threads, bluetooth devices...

*Initialization*

```c#
private static  string CYCLING_SPEED_AND_CADENCE_SERVICE_ID = "1816";
private static SensorState sensorState = SensorState.INIT;
private static DeviceWatcher deviceWatcher = null;
private static GattSession gattSession = null;
private static Socket s = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
private static IPAddress broadcast = IPAddress.Parse("127.0.0.1");
private static IPEndPoint ep = new IPEndPoint(broadcast, 3300);
private static UInt16 previousCumulativeCrankRevolutions = 0;
private static UInt16 previousLastCrankEventTime = 0;
private static Int32 RPM = 0;
```
In the initialization we can find the variables to implement; we find among them the ID of the cadence sensor, a variable that evaluates its status, the socket where the process will be saved, IP address, serial port and variables necessary to measure the RPM.

*Logic*

The process to obtain the RPM is done with a state machine taking into account the Broadcasting mode of the cadence sensor.

**State machine**

The state machine has **4** possible states in which the whole process is carried out and the user is constantly informed of the operation.

``` c#
public enum SensorState { INIT, QUERY_DEVICES, CONNECTED, DISCONNECTED };
```

*Init*

In the *INIT* state, the initialization process for data transfer is performed and the observer's watch is prepared, where it will remain until it finds a device that meets the requirements.

```c#
case SensorState.INIT:
    {   
        deviceWatcher.Stopped += DeviceWatcher_Stopped;
        // Start the watcher.
        deviceWatcher.Start();
        sensorState = SensorState.QUERY_DEVICES;
        Console.WriteLine("From init: waiting to detect cadence sensor Advertising");
        break;
    }
```
*Query devices*

In the QUERY_DEVICE state, the UDP server will start searching its surroundings for a device that is making announcements, until it confirms and can make a connection. 

``` c#
if (status == GattCommunicationStatus.Success)
    {
        characteristic.ValueChanged += Characteristic_ValueChanged;
        firstValueChange = false;
        sensorState = SensorState.CONNECTED;
        Console.WriteLine("Press Any Key to Exit application");
    }
```

*Connected*

In the connected state we can find the conditional process that gives rise to the RPM calculation, which will be executed until the sensor remains static for a few seconds.

```c#
 if(gattSession.SessionStatus != GattSessionStatus.Active)
    {
        sensorState = SensorState.QUERY_DEVICES;
        Console.WriteLine("From connected: waiting to detect cadence device Advertising");
    }
```

*Disconnected*

In the disconnected state we can find the process that returns everything to the search process.

**Calculating RPM**

```c#
 RPM = 0;
 if (deltaTime != 0) RPM =  (Int32) ((UInt32)(60 * deltaRevolutions * 1024) / (UInt32)deltaTime);
 else RPM = 0;
 Console.WriteLine($"RPM:{RPM}");
 s.SendTo(BitConverter.GetBytes(RPM), ep);
```

To calculate the RPM we must take into account two summations; the current sum of crank revolutions and the previous cumulative crank revolutions, and at the same time we calculate the time that the application has been active; in this way when the application is active and the sensor is connected every 60 seconds we will look at the difference between the 2 summations and multiply it by the diameter to give the current RPM.

*Please note that the BLE file presented in this document is not complete in the example codes; but you can find it in this repository inside the folder called bleUDP-main.*

***Simple bicycle physics***

Simple bicycle physics is a *AAA* E-sport Asset built to work in new generation games. Develop High Fidelity Bicycling Games at par with some of the Triple A titles. In this case we are going to use the controller features: Power, Torque, Steer Angle, Lean Dynamics, Pedaling Dynamics, Wheel Friction, Cycling Oscillation, Bunny Hop among many others, and the Animation Rigging is used for customizable IK to fit any style of riding. Procedurally / algorithmically generated IK.

[Assets used](https://assetstore.unity.com/packages/tools/physics/simple-bicycle-physics-206818)

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/bicycleRider.png)

**Connecting to the serial port**

To make the connection process with the UDP server we must have the notion that we are going to read the data that is being stored in the server coming from the cadence sensor. For this it is important that in the initialization process in the IPEndPoint variable we are using the same serial port; with this verified data we will be able to ensure a direct connection.

```c#
private IPEndPoint receiveEndPoint;
public string ip = "127.0.0.1";
public int receivePort = 3300;
private bool isInitialized;
private Queue receiveQueue;
private Thread receiveThread;
private UdpClient receiveClient;
```

**Code initialization**

During the initialization process we create the threads and assign the data reception, completely synchronizing the data transfer between the server and the experience.

```c#
private void Initialize()
    {
        receiveEndPoint = new IPEndPoint(IPAddress.Parse(ip), receivePort);
        receiveClient = new UdpClient(receivePort);
        receiveQueue = Queue.Synchronized(new Queue());
        receiveThread = new Thread(new ThreadStart(ReceiveDataListener));
        receiveThread.IsBackground = true;
        receiveThread.Start();
        isInitialized = true;
    }
```

**Thread removal process**

The process of killing the threads ensures that after the operation of a subprocess the resources chained to a task can be released; which is of vital importance because it gives space to the computer equipment, otherwise we will be putting at risk the integrity of this.

```c#
private void TryKillThread()
    {
    if (isInitialized)
        {
            receiveThread.Abort();
            receiveThread = null;
            receiveClient.Close();
            receiveClient = null;
            Debug.Log("Thread killed");
            isInitialized = false;
        }
    }
```

**RPM transformation to velocity**

As the experience within its narrative dictates a somewhat dystopian physics, taking this into account we will exchange the logic of the RPM; and depending on the value of tomen we will give them a fully inverted value in the respective speed in the game with a very simple conditional

``` c#
if (receiveQueue.Count != 0)
    {
        var value = (UInt32)receiveQueue.Dequeue();
        if (value > 0 && value <= 50)
        {
             currentRPM = 40;
        }
        else if (value > 51 && value <= 100)
        {
            currentRPM = 35;
        }
    }
```

## Directional sensor

### Simplified operating steps

1. OVERVIEW OF OPERATION
* M5Stack core2 sensor
* UDP Client Architecture Segment
* Programs to use

2. ARDUINO IDE PROTOCOL
* Code initialization 
* Draw map
* Calculating direction

3. SIMPLE BICYCLE PHYSICS
* Vector transformation to direction

***Overview of operation***

**M5Stack CORE2 sensor**

[M5Stack Kit de desarrollo IoT Core2 ](https://www.amazon.com/-/es/M5Stack-desarrollo-Core2-ESP32-EduKit/dp/B08VGRZYJR)

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/core2.jpg)

For the development of the movement, as already mentioned, we will use the M5Stack Core2 sensor making use of its gyroscope.

**UDP Client Architecture Segment**

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/Segmento2.png)

The address sensor (for simplicity) will work as a second UDP client, connected directly to a wifi app on the computer.

**Programs to use**

*Arduino IDE*

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/ArduinoLogo.png)

Arduino IDE will be used to program the microcontroller so that it knows that it must handle the gyroscope and transforms them to vectors; for this it is necessary to add in the card manager the M5Stack library, and the Core2 personal library. 

[Manufacturer's instructions](https://docs.m5stack.com/en/arduino/arduino_development)

[Support files](https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/arduino/package_m5stack_index.json)

[Boards manager](https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/arduino/package_m5stack_index.json)

*Unity*

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/Unitylogo.png)
The UDP client that will take the vector and change this with respect to the direction that the user takes in real time. We will use editor version 2021.3.10f1



***Arduino IDE protocol***

**Code initialization**

*Instances*

```c++
#define M5STACK_MPU6886
#include <M5Core2.h>
#include "password.h"
```

To start with the process we need to resort to the M5Core2 libraries; in addition to that we will save a file with the network name and password.

*Initialization*

```c++
WiFiUDP udpDevice;
uint16_t localUdpPort = 3302; // Sensor listening port
uint16_t UDPPort = 4000; // Port to which the sensor sends data
                         
const char *unityIP = "192.168.0.6"; //IP address of the equipment to which it must connect

float accX = 0.0F;  
float accY = 0.0F;  
float accZ = 0.0F;

float offsetArr[3] = {0.0F};
```
In the initialization we can see that what the gyroscope will do is to transform its data by a vector of 3 directions, which will result in the angle of movement of the handlebar every x amount of time.

**Draw map**

```c++
void drawGrid() {
  M5.Lcd.drawLine(41, 120, 279, 120, CYAN);
  M5.Lcd.drawLine(160, 1, 160, 239, CYAN);
  M5.Lcd.drawCircle(160, 120, 119, CYAN);
  M5.Lcd.drawCircle(160, 120, 60, CYAN);
}

void drawSpot(int ax, int ay) {
  static int prevx = 0;
  static int prevy = 0;
  int x, y;
  x = map(constrain(ax, -300, 300), -300, 300, 40, 280);
  y = map(constrain(ay, -300, 300), -300, 300, 240, 0);

  M5.Lcd.fillCircle(prevx, prevy, 7, BLACK);
  drawGrid();
  M5.Lcd.fillCircle(x, y, 7, WHITE);
  prevx = x;
  prevy = y;

}
```
The map is a short explanation that gives us feedback on the current process of reading the movement and the direction it is taking.

**Calculating direction**

```c++
if ( (currentTime - imuReadTime) > 100) {
    imuReadTime = currentTime;
    M5.IMU.getAccelData(&accX, &accY, &accZ);
    drawSpot((int)((accX - offsetArr[0]) * 1000), (int)((accY - offsetArr[1]) * 1000));
}

  currentTime = millis();
if ( (currentTime - printIMUTime ) > 100) {
    printIMUTime = currentTime;
    M5.IMU.getAccelData(&accX, &accY, &accZ);
    int x, y;
    x = (int)((accX - offsetArr[0]) * 1000);
    y = (int)((accY - offsetArr[1]) * 1000);
    printf("accX: %d, accY: %d\n", x, y);
    printf("Vbat:%f/Cbat:%f\n", M5.Axp.GetBatVoltage(), M5.Axp.GetBatCurrent());
    if (y >= 150) {
      auto m = (accX - offsetArr[0]) / (accY - offsetArr[1]);
      printf("m: %f\n", m);
      M5.Lcd.setCursor(0, 20);
      M5.Lcd.printf("m: %f", m);


      udpDevice.beginPacket(unityIP, UDPPort);
      udpDevice.write((uint8_t *)&m, 4);
      udpDevice .endPacket();
    }
}
 ```

 Each 100 ms the display is changing the direction with respect to the movement going through the gyroscope; and each second ends by evaluating the above positions to find a change and is transformed creating a vector of 3 variables. 

***Simple Bicycle physics***

**Vector transformation to direction**

To take the value of the direction, the vector 3 data is placed in a queue and is received in the unity application; so we assign the value to the customSteerAxis that redirects the character in the experience. 

```c#
if (receiveQueue2.Count != 0)
    {
        float value = (float)receiveQueue2.Dequeue();
        customSteerAxis = value;
        //customLeanAxis = value;
        Debug.Log("New target diretion" + value.ToString());
    }
```

*Remember that the implementation code is not fully written in this example code, this is only a readme; To fully view the functionality download a version from the repository*

## Additional applications 

Finally, application services, which run different orders of experience and are responsible for detonating different events in the main application. 

These applications run on android devices connected to the same local Wifi network as the other sensors to subsequently send the data and receive them by UDP. 
### Simplified operating steps

1. EXPLANATION OF THE FUNCTIONALITIES
* Master application
* Saboteur application

2. PROTOCOL OF ACTIVITIES
* Sending data
* Verify connections

***Explanation of the functionalities***

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/Segmento3.png)

External applications are part of the 3 and 4 UDP client; These affect their own actions directly.

**Master application**

The Master's device that will send the tasks to the saboteur and the advantages or disadvantages to the cyclist.Which has 3 types of data, The first is the conservation of names of both the rider and the saboteur:

```c#
public void ButtonContinuar()
    {
        sendStringDataCiclista(NombreCiclista.text);
        sendStringDataSabotaje(NombreSaboteador.text);
        Debug.Log(PlayerPrefs.GetInt("EstadoBotonBailar"));

        TryKillThread();
        SceneManager.LoadScene("ActividadesMaster");
    }
```

 The second is the type of activity that will be sent to the saboteur:

```c#
public void ButtonSentadilla()
    {
        sendStringDataSabotaje(0x06);
        PlayerPrefs.SetInt("EstadoBotonSentadilla", 1);
        PlayerPrefs.Save();
        int tmp = PlayerPrefs.GetInt("ValorDisponible") + 1;
        PlayerPrefs.SetInt("ValorDisponible", tmp);
        PlayerPrefs.Save();
        TryKillThread();
        SceneManager.LoadScene("RevisionMain");
    }
```

Finally the result of the activity carried out that directly affected the cyclist:

```c#
public void MasTiempo() 
    {
        sendStringDataCiclista(0x0A);
    }
public void MenosTiempo()
    {
        sendStringDataCiclista(0x0F);
    }
```
**Saboteur application**

A saboteur device that will give tasks to the user who fulfills this role. Saboteurs mostly receive data, which allows them to know what reaction or activity they should do, receive three types of data; The first is your name:

```c#
if (receiveQueue.Count != 0)
    {
            byte[] message = (byte[])receiveQueue.Dequeue();
        if (message == null)
            return;
        Debug.Log("Mensaje de llegada");
        _dataReceived = Encoding.Default.GetString(message); ;
        Debug.Log(_dataReceived);
        Nombre.text = _dataReceived;
    }
```

The second the activity to be carried out:

```c#
if (receiveQueue.Count != 0)
    {
        byte[] message = (byte[])receiveQueue.Dequeue();
        if (message == null)
         return;
        Debug.Log("Mensaje de llegada");
        dataReceived = message[0];
        Debug.Log(_dataReceived);
            
        if (_dataReceived==0X02){
            CartaBailar.SetActive(true);
        }
        else if (_dataReceived==0X03)
        {
            CartaCantar.SetActive(true);
        }
        else if (_dataReceived==0X04)
        {
            CartaGritar.SetActive(true);
        }
    }
```

The last the verification of the activity:

```c#
else if (_dataReceived==0X10)
    {
        CartaBailar.SetActive(false);
        CartaCantar.SetActive(false);
        CartaGritar.SetActive(false);
        CartaLagartija.SetActive(false);
        CartaSentadilla.SetActive(false);
        CartaTijera.SetActive(false);
        Paso.SetActive(true);
        Accion1.SetActive(true);
    }
    else if (_dataReceived==0X11)
    {
        CartaBailar.SetActive(false);
        CartaCantar.SetActive(false);
        CartaGritar.SetActive(false);
        CartaLagartija.SetActive(false);
        CartaSentadilla.SetActive(false);
        CartaTijera.SetActive(false);
        NoPaso.SetActive(true);
        Accion2.SetActive(true);
    }
```


The only data they send are the confirmation of sabotage:

```c#
public void ButtonAccion()
    {
        byte tmpByte = 0x01;
        sendStringData(tmpByte);
        Uso.SetActive(true);
        Accion1.SetActive(false);
                                         
    }
```

***Protocol of activities***

**Sending data**

To send and receive the data from the master, two IPEndPoint are generated whose objective is to be assigned to each of the other users to send the corresponding data.

```c#
    private Thread receiveThread;
    private UdpClient _dataReceiveCiclista;
    private UdpClient _dataReceiveSabotaje;
    private IPEndPoint _receiveEndPointDataCiclista;
    private IPEndPoint _receiveEndPointDataSabotaje;
    public string _ipDataCiclista = "192.168.100.13"; //IP PC
    private string _ipDataSabotaje = "127.0.0.1";//IP SAB
    public int Disponible = 3;

    private int _receivePortData = 44444;
    private int _sendPortData = 3100;
```

As seen in the code each one is assigned its IP address and its serial port to avoid failures.

**Verify connections**

To verify the connections between the applications we will use the scriptCommunicator program; facilitating the process of reviewing and filtering data.

![](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/script.png)

*Remember that application deployment code is not written in this repository; To see the full functionality download a version of the following link*

[Risitas Corp. APP Repository](https://github.com/IsabelaGAngel/RisitasCorp_App)


![Logo](https://github.com/IsabelaGAngel/RisitasCorp_Rider/blob/main/ImagenesReferencia/RisitasLogo.png)


In this part of the document you will find conceptualization, Experience Design (Ux) and Interface Design (UI) in addition to the evidence in photo format, and link to videos.


## Conceptualization

***Why was this project born?***

"The World Health Organization defines academic stress to the situations implicit in the educational process that are not controlled by the student and that can cause sleep disorders, behavior or even diseases."

This stress tends to be exacerbated in project season and final exams due to the lack of time organization. That is why we decided to create a project that is dedicated to de-stress and create a calmer environment for all those who go through difficult times.

Risitas Corp borned as a fictitious company that seeks help students with stress generated by “final exam’s time”. Aplying an absurdely narrative who distract the students and generate places to share beetwen them and erase the stress on those difficult times. 

### ***Narrative of the experience***

In a very happy world. The better-known company Risitas Corp. has committed a failure in one of the cheerful power plants;The world has been turned upside down and yet you must really be late for your work so as not to make it worse.

**What is the goal of Risitas Corp.?**

We’re a multinational conglomerate of companies working on responsible for energy, logistics, social, and scientific investigation. Our world is so different than yours, we have created a new virtual reality proyect who is seeking help you with the stress gerenerated by university exams. 

With a world governed by contradictory physics to ours. Risitas Corp. gave us the ability to change our interactions with its and generate funny situation who distract the users and viewers by the experience, therefore. In a pararell world you managed  to win your exam!


## How we achieve it?

The process of creating the project was based on 2 sections

***The technological***

Dedicated to all those circuits, and devices necessary for the functionality of the project.

***The design***

Comprising the creation process and user experience.

### ***Tech***

***Research***

The idea of the project was to create a race against the clock of a cyclist; which will be constantly sabotaged by his friends and by the altered physics of the game itself.

For the elaboration of the project we decided to work under a group of UDP clients who would join the project.

1. We would have a cadence sensor that would give us the RPM so we would measure the speed of the rider.

2. An M5Stack sensor that will be the one who directs the direction of the simulator

3. A saboteur device that will give tasks to the user who fulfills this role.

4. A Master's device that will send the tasks to the saboteur and the advantages or disadvantages to the cyclist.

**Evidence**

Many videos and photos

https://user-images.githubusercontent.com/44727218/202724318-024c4dca-e8d9-4170-8ce6-546afc3e0398.mp4

https://user-images.githubusercontent.com/44727218/202724792-61cc56b9-2a3d-4ba2-aa58-2aae03b457ee.mp4

New evidence, with virtual reality and matchmaking:

https://user-images.githubusercontent.com/44727218/202745653-cbffbd2b-42ce-4808-8387-863de9669a2d.mov

https://user-images.githubusercontent.com/44727218/202745735-9b77c1fa-7aa5-4abf-8db1-d656b7ad918e.mp4

https://user-images.githubusercontent.com/44727218/202746750-5189eaad-f6c7-4fcb-ad46-d88eb26b15e4.mp4




