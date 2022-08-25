---
geometry: margin=1in
fontsize: 12pt
header-includes: |
    \usepackage[scaled]{beramono}
    \usepackage{setspace}\doublespacing
    \usepackage{url}
    \usepackage[numbers]{natbib}
    \usepackage{tikz}
    \usetikzlibrary{automata, arrows.meta, positioning}
    \usepackage{listings}
    \usepackage{color}
    \usepackage{xcolor}
    \pagenumbering{gobble}
    \usepackage{float}
    \floatplacement{figure}{H}
title: |
  | ![](logo.png){width=2in}
  | \vspace{1cm} \Huge Crowdsourcing EDR measurements using smartphones
  | \vspace{1cm} \large ISSN 2202-0303
author: |
  | Jaan Jaerving \vspace{0.2cm} 
  | Sæmundur Elías Þorsteinsson \vspace{0.2cm} 
  | Helgi Þorbergsson \vspace{4cm}
date: \small \today
...

\definecolor{pblue}{rgb}{0.13,0.13,1}
\definecolor{pgreen}{rgb}{0,0.5,0}
\definecolor{pred}{rgb}{0.9,0,0}
\definecolor{pgrey}{rgb}{0.46,0.45,0.48}

\lstset{language=Java,
  showspaces=false,
  showtabs=false,
  breaklines=true,
  showstringspaces=false,
  breakatwhitespace=true,
  commentstyle=\color{pgreen},
  keywordstyle=\color{pblue},
  stringstyle=\color{pred},
  basicstyle=\ttfamily,
  captionpos=b,                    
  moredelim=[il][\textcolor{pgrey}]{$$},
  moredelim=[is][\textcolor{pgrey}]{\%\%}{\%\%}
}

\pagebreak
\pagenumbering{arabic}

# Table of contents

\ttfamily

* 0 [Abstract ..................................................... 4](#abstract)
* 1 [Introduction ................................................. 5](#introduction)
    * 1.1 [What we learned from the previous project .............. 5](#what-we-learned-from-the-previous-project)
    * 1.2 [MQTT ................................................... 7](#mqtt)
    * 1.3 [Storage ............................................... 10](#storage)
* 2 [Android Application ......................................... 11](#android-application)
    * 2.1 [Performance and Stability ............................. 11](#performance-and-stability)
    * 2.2 [Problems with Power Saving ............................ 15](#problems-with-power-saving)
* 3 [Data & Processing ........................................... 17](#data-processing)
    * 3.1 [Latitude & Longitude .................................. 17](#latitude-longitude)
    * 3.2 [Altitude .............................................. 19](#altitude)
    * 3.3 [Speed ................................................. 21](#speed)
    * 3.4 [Acceleration .......................................... 23](#acceleration)
* 4 [Weather Data ................................................ 26](#weather-data)
    * 4.1 [Accessing Observations ................................. ?](#accessing-observations)
    * 4.2 [Interpolation .......................................... ?](#interpolation)
* 5 [Server Implementation ........................................ ?](#server-implementation)
    * 5.1 [Subscriber Step ........................................ ?](#subscriber-step)
    * 5.2 [Back-end & Storage ..................................... ?](#back-end-&-storage)
    * 5.3 [Access to Data ......................................... ?](#access-to-data)
* 6 [Conclusion ................................................... ?](#conclusion)
* 7 [References ................................................... ?](#references)
* 8 [Appendices ................................................... ?](#appendices)

\pagebreak

# Table of figures

* Figure 1: Comparison of FFT given by the Samsung and PX4
* Figure 2: Comparison of filtered acceleration
* Figure 3: Original Android application
* Figure 4: MQTT communication
* Figure 5: MongoDB database and collection format (three flights on two different days)
* Figure 6: Keflavík to Brussels flight path
* Figure 7: Brussels to Keflavík flight path
* Figure 8: Reykjavík to Egilsstaðir flight path
* Figure 9: Reykjavík to Ísafjörður flight path
* Figure 10: Keflavík to Brussels flight altitude
* Figure 11: Reykjavík to Ísafjörður flight altitude
* Figure 12: Keflavík to Brussels flight speed
* Figure 13: Reykjavík to Ísafjörður flight speed
* Figure 14: Reykjavík to Ísafjörður flight STD/RMS
* Figure 15: Reykjavík to Ísafjörður flight EDR
* Figure 16: Accelerometer measuring rate of different smartphones

\pagebreak

# Table of listings

* Listing 1: Sensor event listener
* Listing 2: Measurement object
* Listing 3: Measurement buffer
* Listing 4: Power management

# List of acronyms

* MQTT: Message Queueing Telemetry Transport
* PX4: Pixhawk4
* EDR: Eddy Dissipation Rate
* OASIS: Open Artwork System Interchange Standard
* ACK: Acknowledgment
* TCP: Transmission Control Protocol 
* RAM: Random Access Memory 
* FFT: Fast Fourier Transform
* GPS: Global Positioning System
* RMS: Root Mean Square

\pagebreak

\rmfamily

# Abstract

The project explores the viability of crowd sourcing measurements to map out areas of high turbulence in Iceland. Initial viability of measuring samples for vertical samples was established in the report preceding this one (ISSN 2772-1078). The objective of this report is to carry on where the previous report left off and establish a method to transport data off of the measuring smartphone and into a long term storage solution that can then be accessed later for further exploration and processing.

Data transportation is provided by a communication protocol called MQTT. Any smartphone that has measurements to share can publish them to the MQTT broker which then delegates these data-frames to subscriber processes that can do any required post processing such as de-compression, error correction or even adding weather data. Once the subscriber is done with post processing, measurements can be bulk inserted into a database that can be made accessible by providing a user facing API.

\pagebreak

# 1 Introduction

This project continues from the previous one called "Lýðvistun mælinga á ókyrrð í flugi". That previous project made great strides in showing how electronic filtering could be used to make use of smartphone sensors to measure EDR. The objective of this project is to continue onwards and set up a viable database system that could be used to store turbulence measurements from measuring smartphones as well as other wind phenomena. A starting point for this data gathering process was the idea to use the existing 4G connection that smartphones already have in tandem with the MQTT protocol for stable message passing to produce a data pipeline that could be readily received and stored.

These initial goals were implemented successfully, the Eclipse Paho MQTT client library \cite{paho} was used to implement MQTT communications for the Android application. Incoming data was received by a Mosquitto broker and sent onwards to a subscriber program written in NodeJS which would parse the data and store it in a MongoDB database. In addition to this base implementation the subscriber program also makes use of web scraping to retroactively add weather data from the Icelandic Met Office. In order to provide a full demo stack a public facing API as well as a simple front-end web application were also implemented to provide easy access to any collected data. This user accessible portion can be viewed at \url{http://31.209.145.132:3457} and the backend output can be inspected at \url{http://31.209.145.132:3456}.

## 1.1 What we learned from the previous project

### Vertical acceleration, filtering and comparison of measurements

The previous report in this series delves into the viability of measuring vertical acceleration on smartphones, filtering it using stable filter constants and producing EDR values that can be used to assess the turbulence of a given area at a given time. A Samsung Galaxy FE 20 was tested on a vibration motor as well as taken on a test flight to compare with the acceleration values given by a PX4 \cite{prev} the comparison can be seen in Fig. 1.

![Comparison of FFT given by the Samsung and PX4](original_accel.png){width=80%}

In the previous project a digital filter was designed using Matlab. This yielded filter coefficients that can be used in other programming languages to implement a filter. These filtering coefficients were also compared as can be seen in the following Fig. 2.

![Comparison of filtered acceleration](original_accel_filtered.png){width=80%}

From this we learned that vertical acceleration values given by a phone were very similar in form to the values produced by hardware that was already in use for this kind of purpose.

### Location inaccuracies

In the process of comparing the Samsung smartphone to the PX4 it was also noted that the altitude values provided by the smartphone were not the same as the ones shown by the PX4, in reality the smartphone produced altitude numbers which were 55m higher in altitude.

### The Android application

For the purpose of storing and filtering measurements an initial implementation of an Android application was created. For the current project this application provides a solid baseline to build from. The user interface for this initial implementation can be seen here in Fig. 3.

![Original Android application](original_app.png){height=50%}

## 1.2 MQTT

The MQTT protocol is an OASIS standard messaging protocol for the Internet of Things. It is designed as an extremely lightweight publish/subscribe messaging transport that is ideal for connecting remote devices with a small code footprint and minimal network bandwidth. This protocol is currently used in a wide variety of industries, such as automotive, manufacturing, telecommunications, oil and gas, etc \cite{mqtt_uses}.

### The publisher, subscriber and broker model

MQTT message passing works by using topics that publishing devices publish to and subscribing devices subscribe to. A broker oversees that information published to a topic is conveyed to the devices that are subscribed to that same topic \cite{mqtt_spec}. For our use case we would be publishing measurement data from our smartphone application to the broker which would forward it to any back-end applications that are subscribed to receive that data. There are many different ways to implement this interaction but the one we are interested in is the Paho MQTT client that provides an implementation standard that we can use on both the smartphone application for publishing purposes as well as the receiving computer for subscribing purposes. For our broker we can use an application called Mosquitto which will provide a lightweight, open source implementation of an MQTT broker server. The communication pattern can also be seen in Fig. 4.

\begin{figure}
\begin{center}
\resizebox{.6\textwidth}{!}{
\begin{tikzpicture} [node distance = 3cm, auto]
 
\node (q0) [state] {Paho 1};
\node (q1) [state, below = of q0] {Paho 2};
\node (q3) [state, right = of q1] {Mosquitto};
\node (q4) [state, right = of q3] {Back-end};

\path [-stealth, thick]
    (q0) edge [bend right] node [above right] {$Publish$} (q3)
    (q1) edge node [below] {$Publish$} (q3)
    (q3) edge [bend right] node [below] {$Publish$} (q4)
    (q4) edge [bend right] node [above] {$Subscribe$} (q3);
 
\end{tikzpicture}
}
\caption{MQTT communication}
\end{center}
\end{figure}

### Benefits of MQTT/Mosquitto

This kind of message passing implementation provides us several benefits that we do not have to implement ourselves. MQTT implements TCP for its transport layer for additional reliability, this means we can check ACKs on the Android application to see if the broker has received the data we published. The message passing between the three tiers described is data agnostic, the broker merely receives a topic and some data represented as bytes. The broker provides initial filtering and validation using our provided topics, it also provides a simple authentication for any connecting clients. Finally this format is easily scalable and components can be replaced or amended without necessarily having to change other components. Publishers, subscribers and topics can be added as needed; modifications can be made to the back-end or the Android application without necessarily having an impact on one another and without having to change how the broker is configured \cite{mqtt_spec}.

### Potential downsides

As always when using external code libraries there are some considerations to be made. During early implementation Android 12 began its roll-out which had an adverse affect on some of the features of the Paho library, this could be remedied by switching to the unstable version that was still in development but it is an example where third party libraries are not necessarily up to date as soon as a bleeding-edge operating system update rolls out.

Another potential downside of this middle-man format is that the publishing smartphones and the subscribing back-end application do not communicate directly, messages could be published to the broker successfully while the back-end is not running and the smartphones would not necessarily know anything was amiss. If back-end instability is a problem this could potentially be solved by letting the broker store messages that were not successfully subscribed to.

## 1.3 Storage

There are two main storage concerns for this messaging model. Temporary storage on smartphones for messages that are ready but have not yet been sent and permanent storage on the subscribing server for messages that have been received and processed.

### Smartphone storage

The initial variant of the Android application kept all measurements in RAM until they were ready to be exported. This is fine for relatively short test runs but can become a problem when dealing with longer sessions (an hour long session will produce nearly two million rows of measurements) or when we want to do multiple consecutive sessions such as connecting or back to back test flights. What is required then is to periodically write measurements to long term storage and free up RAM. Later, these data-frame files can be read from disk and sent whenever the possibility arises.

### Permanent storage (Database)

For this portion of our tech stack we chose to use NoSQL, MongoDB specifically. Due to the potential of large bulk inserts as well as large deletions following a standard SQL table format becomes problematic. While SQL primary and foreign keys can give us relatively quick select lookups these constraints within a single table also make insertions slower \cite{mongo}.

Since our main use cases do not include updating individual fields or complex joins between tables we decided to adopt a more flexible model. When measurements are stored a table is dynamically created for that specific date. For each date we can have an individual collection for both the values of interest as well as each measuring session that happened on that day. This format allows us to bulk insert with no indexing concerns but allows us to look and export entire sessions without having to explicitly search for values. In essence this is similar to a filing cabinet where each drawer is a date and each folder within a drawer is a specific session. The way this kind of segmenting works could be represented as can be seen in Fig. 5. 

\pagebreak

\begin{figure}
\begin{center}
\resizebox{0.6\textwidth}{!}{
\begin{tikzpicture} [node distance = 2cm, auto]
 
\node (q0) [state] {MongoDB};
\node (q1) [state, below right = of q0] {10-08-2022};
\node (q2) [state, below left = of q0] {09-08-2022};
\node (q3) [state, below left = of q1] {RKV-EGS};
\node (q4) [state, below = of q1] {AEY-RKV};
\node (q5) [state, below = of q2] {IFJ-AEY};

\path [-stealth, thick]
    (q0) edge node [above right] {$Database$} (q1)
    (q0) edge node [above left] {$Database$} (q2)
    (q1) edge node [above left] {$Collection$} (q3)
    (q1) edge node [right] {$Collection$} (q4)
    (q2) edge node [left] {$Collection$} (q5);


\end{tikzpicture}
}
\caption{MongoDB database and collection format (three flights on two different days)}
\end{center}
\end{figure}

# 2 Android Application 

## 2.1 Performance and Stability

Initially some performance adjustments had to be made to the original Android application that can be seen in Fig. 3. There was a suspicion that some select functionality could slow down how quickly the sensor data would be processed. Quickly checking an output file of a demo run done with the original application a simple time/rows calculation gives us that a measurement is only produced at a time interval of approximately `16ms`. We know that the accelerometer of that specific Samsung phone can measure at a rate of 500Hz which would ideally produce a measurement every `2ms`.

To ensure that measurements are produced at the rate the phone is capable of it would be ideal to any kind of longer processing functions, location functions and memory functions to separate threads from the thread that is providing our `onSensorChanged()` event listener. Instead of the main thread function containing all our logic and side functions we now have a relatively compact function that forks the main filter calculations to another handler thread. This implementation can be seen in Listing 1:

\singlespacing

\begin{lstlisting}[language=Java,caption=Sensor event listener]
public void onSensorChanged(SensorEvent event) {
    if (event.sensor.getType() == Sensor.TYPE_ACCELEROMETER) {
        DateFormat isoDate = new SimpleDateFormat(FileUtils.ISO_DATE);
        isoDate.setTimeZone(TimeZone.getTimeZone("UTC"));
        // Produce timestamp to indicate when measurement was received
        String time = isoDate.format(new Date());
        // Pass vertical acceleration and timestamp to the filter thread
        mMathThread.handleMeasurement(event.values[2], time);
    }
}
\end{lstlisting}

\doublespacing

Removing unnecessary actions such as address lookup and moving filtering and storage calls to their own threads was sufficient to get both our mean and median measuring speed to he be close to `2ms` for the Samsung Galaxy FE20 testing smartphone.

This not sufficient to produce a stable measuring rate however. Initial test runs after dividing our key functions into threads would produce standard deviations between measurements that could reach upwards of `60ms`. The initial implementation used a single array buffer which was protected by a semaphore to ensure that no race conditions would happen as we wrote our measurements to memory and then later to disk. Each measurement is defined as an object with its relevant fields. The format can be seen here in Listing 2:

\singlespacing

\pagebreak

\begin{lstlisting}[language=Java,caption=Measurement object]
public class Measurement implements Cloneable {
    private String time;    // UTC Timestamp
    private float lon;      // Longitude obtained from location
    private float lat;      // Latitude obtained from location
    private float alt;      // Altitude obtained from location
    private float ms;       // Estimated speed obtained from location
    private float ms0;      // Estimated speed obtained by calculating distance
    private float acc;      // Location accuracy (meters)
    private float z;        // z acceleration value read from the sensor
    private double fz;      // Filter z value result
    private double rms;     // Root mean square value of the measurement window
    private double edr_rms; // Eddy dissipation rate
    
    /* Constructors */

    /* Getters & Setters */
}
\end{lstlisting}

\doublespacing

This single memory safe single array format had performance issues, though. Every value produced by the sensor would allocate memory for a new measurement object as can be seen in Listing 2. Once a set of measurements would be written to disk the Java garbage collector would go to work to free up the RAM that was previously used by those measurements. Additionally in order to guarantee memory safety this array was protected by a Semaphore (effectively a lock to guarantee that the array can not be written to and read from at the same time). The end result of this frequent memory allocation and collection as well as delays in handing off the lock produced situations where we would end up with unreasonably long gaps in our measurements.

The solution to this problem was to use two preallocated arrays as circular buffers and switch between them instead of locking them. There is a theoretical risk of race conditions if measurement speed severely outpaces the rate at which we can write an array out to disk, but this does not seem to be a valid concern for the measurement rate of any phone we have seen so far. 

\pagebreak

The buffer implementation is shown in Listing 3:

\singlespacing

\begin{lstlisting}[language=Java,caption=Measurement buffer]
// Assign static memory for both buffers
public static final ArrayList<Measurement> DATA_1 =
  new ArrayList<>(MEASUREMENT_COUNT);
public static final ArrayList<Measurement> DATA_2 =
  new ArrayList<>(MEASUREMENT_COUNT);
// ...
// Swap between arrays according to known index
if (Measurements.sFirstArray) {
    if (Measurements.sCurrIdx >= Measurements.MEASUREMENT_COUNT) {
        flushMessages(true, Measurements.sCurrIdx);
        Measurements.sFirstArray = false;
        Measurements.sCurrIdx = 0;
    }
} else {
    if (Measurements.sCurrIdx >= Measurements.MEASUREMENT_COUNT) {
        flushMessages(false, Measurements.sCurrIdx);
        Measurements.sFirstArray = true;
        Measurements.sCurrIdx = 0;
    }
}
// Obtain memory for Measurement from existing buffer instead of allocating
Measurement m = Measurements.sFirstArray ?
  Measurements.DATA_1.get(Measurements.sCurrIdx) :
  Measurements.DATA_2.get(Measurements.sCurrIdx);
// ...
// Write current filter and measurement values to memory of m
// ...
// Increment index of buffer
Measurements.sCurrIdx++;
\end{lstlisting}

\doublespacing

After lowering our garbage collection footprint by replacing unneeded memory allocation with preallocated static memory we saw the measurement standard deviation drop into the `1ms` range which is the limit of accuracy provided by the standard Android system clock. If needed it would theoretically be possible to reduce measurement deviation further by running the application under system privileges or by providing a binary written in a language with manual garbage collection. Even with these changes, perfect measurement timing is likely not possible, the core interface for accessing the sensors of the device do not promise perfect accuracy \cite{sensors}.

## 2.2 Problems with Power Saving

There are a number of ways to ensure code execution within an Android application. A brute force approach would be to force the screen to remain on which effectively also forces the processor to remain awake. This carries with it a penalty of severe battery degradation just from keeping the screen on. For the usability of the application it was important to provide a way to measure and send data without having to keep the screen on. Android provides interfaces to write alarm intents, these are not ideal for measuring though as they are intended for longer interval actions such as fetching email.

Realistically what was required was to split measuring and sending into their individual services such that they were not bound to the user interface and would no longer be destroyed when the user interface left the foreground.

Android services come in two flavors, background and foreground. Any service that does not have foreground priority is subject to running its code within the code execution windows provided by the various battery saving features such as doze \cite{doze}.

For continuous measurements and faster sending of data it is imperative that these kinds of battery saving procedures do not interrupt the two main services we described. As such we are effectively left with the option of running our data services with foreground priority with a processor wake-lock as described in \cite{doze}.

Curiously even this is not a silver bullet solution though, certain versions of Android as well as certain vendor implementations of the same version of Android perform battery saving operations differently. 

\pagebreak

In order to cover all bases we effectively need to check a number of conditions as can be seen here in Listing 4:

\singlespacing

\begin{lstlisting}[language=Java,caption=Power management]
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
// For Android 6 or newer request permissions to ignore battery optimizations
    if (powerManager.isIgnoringBatteryOptimizations(this.getPackageName())) {
        sensorIntent // If permission to ignore optimizations exist then use it
          .setAction(Settings.ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS);
    } else { // If permission does not exist then request it before using
        sensorIntent
          .setAction(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS);
        sensorIntent
          .setData(Uri.parse("package:" + packageName));
    }
}

// For Android 8 or newer run the service with foreground permissions
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    startForegroundService(sensorIntent);
} else { // For older versions this permission does not exist
    // Resort to running as a background service with screen on
    getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
    startService(sensorIntent);
}
\end{lstlisting}

\doublespacing

For each individual smartphone, most of this code is unnecessary. Smartphones older than Android 6 do not have aggressive battery saving optimizations, smartphones older than Android 8 cannot run services with foreground processing priority at all.

Even ignoring Android versions the smartphones we tested implemented battery saving features differently, the main testing Galaxy FE20 would run both services just fine simply by requesting foreground, battery optimizations did not have to be ignored even though the Android version indicated that they would be. A OnePlus 10 Pro on the other hand would aggressively remove network permissions from the data sending service despite it being started as a foreground service that requested those permissions. Realistically the only way to be certain that the current solution works for a good subset of devices is to simply test a large number of smartphones.

# 3 Data & Processing

During development testing was actively being done to observe the quality of incoming data. Initial testing to obtain a more consistent measuring rate from the accelerometer sensor was already mentioned in Chapt. 1. Acceleration values are only one part of what needs to be tracked within the smartphone application, however.

## 3.1 Latitude & Longitude 

Raw location data is provided by the smartphone itself in the form of location updates. These are dependant upon available sources, setting the smartphone to airplane mode for example will remove 4G and WiFi triangulation \cite{location}. Even when set to airplane mode and fully reliant on GPS positioning it can be seen below in Fig. 6. That consistent location values can still be obtained and used to plot an accurate flight path, even for international flights.

\begin{figure}
  \centering
  \includegraphics[width=6cm]{brussels1_path.png}
  \includegraphics[height=6cm]{brussels1_coords.png}
  \caption{Keflavík to Brussels flight path}
\end{figure}

There are some caveats to obtaining location using a smartphone in this way. On a separate international flight we could see the path produced in the following Fig. 7.

\begin{figure}
  \centering
  \includegraphics[width=6cm]{brussels2_path.png}
  \includegraphics[height=6cm]{brussels2_coords.png}
  \caption{Brussels to Keflavík flight path}
\end{figure}

For this flight a window seat could not be obtained and the smartphone did not have good line of sight of the sky. Due to this handicap location values were not produced continuously, instead they appear as a sudden change when the smartphone briefly obtains a new location value. This indicates that in order to be able to use the application to mark areas of interest a good line of sight of the sky is required. We can see that this also holds for domestic flights by observing Fig. 8.

\begin{figure}
  \centering
  \includegraphics[width=6cm]{egs_path.png}
  \includegraphics[height=6cm]{egs_coords.png}
  \caption{Reykjavík to Egilsstaðir flight path}
\end{figure}

\pagebreak

Unlike the international flights that could be seen in Fig. 6. and Fig. 7. the flight in Fig. 8. was taken by the pilot, in that particular instance the smartphone was placed low in the cockpit. An additional test was made later where the smartphone was placed on top of the dashboard and the results of that flight can be seen here in Fig. 9.

\begin{figure}
  \centering
  \includegraphics[width=6cm]{isaf_path.png}
  \includegraphics[height=6cm]{isaf_coords.png}
  \caption{Reykjavík to Ísafjörður flight path}
\end{figure}

Once again with sufficient line of sight of the sky the smartphone is able to produce continuous location updates and it becomes possible to plot out a much more accurate flight path.

## 3.2 Altitude 

Altitude is also given by the location data of the device but this value comes with additional caveats. For every reported set of location values a GPS accuracy is given, this is an estimated horizontal accuracy in meters of the given location at the 68th percentile confidence level. It is explicitly noted that this GPS accuracy value makes no guarantees for vertical positioning \cite{location}. As a result even when longitude and latitude appear to be continuous we do not necessarily see this for altitude. If we look at the altitude plots in the following Fig. 10. and Fig. 11., representing the altitude for our international and domestic flights with good location data, we see that the altitude values we receive are not correct for the entire duration of the flight.

\pagebreak

![Keflavík to Brussels flight altitude](brussels1_alt.png){width=80%}

For the above Fig. 10. we see sharp changes in altitude where it is likely that the change should have been more gradual in reality.

![Reykjavík to Ísafjörður flight altitude](isaf_alt.png){width=80%}

For this domestic flight in Fig. 11. we see more gradual changes in altitude overall but obviously wrong values at around the 500th second of the flight where the altitude suddenly spikes back down to ground levels. It is also worth remembering that in \cite{prev} there appeared to be a difference in stable ground altitude between values produced by the smartphone and values produced by the PX4.

## 3.3 Speed

Travel speed values also had to be given special consideration. The formula for a given EDR value is as follows.

$$EDR = \frac{\sigma_{\ddot{z}}}{\sqrt{0.7 \cdot v^{\frac{2}{3}} \cdot I}}$$

A more in depth explanation can be seen in \cite{prev} but for this specific chapter we are mostly concerned with the variable for aerial velocity *v*. Since *v* is part of the divisor fluctuations similar to the ones seen in altitude can end up producing what could be considered false positives when it comes to EDR. And indeed a fluctuation similar to the one visible in Fig. 11. Is visible when observing speed values produced by the smartphone. We can see an example of this in Fig. 12. Here:

![Keflavík to Brussels flight speed](brussels1_speed.png){width=80%}

Any of these artificial valleys would produce extremely high EDR values due to a part of the divisor, *v* specifically, being 0 or near 0. After this was observed an effort was made to find an alternative way to obtain values for velocity and example output for the solution that was arrived at can be seen in the upcoming Fig. 13.

![Reykjavík to Ísafjörður flight speed](isaf_speed.png){width=80%}

Even though speed values may have severe errors as can be seen in Fig. 12. The same flight is very likely to have correct longitude and latitude data as can be seen in Fig. 6., the coordinate track for that same flight. In a private conversation concerning the viability of the Hvassahraun airport Þórgeir Pálsson suggested that it should be viable to use speed calculated from distance instead of the raw speed value produced by the smartphone. It stands to that it is possible to calculate the distance using the haversine formula for distance between two points on a sphere. These calculations are what appear in Fig. 13. As the red, calculated, track and are the values used for *v* in the current version of the smartphone application.

## 3.4 Acceleration

Some changes were also made to vertical acceleration. The initial version of the formula visible in Chapt. 3.3 uses standard deviation. In another conversation about the Hvassahraun airport it was noted by Gylfi Árnason that the final EDR value should likely use root mean square instead of standard deviation. To ensure correctness this was implemented and as a result the formula in use by the smartphone app currently is:

$$EDR = \frac{RMS_{\ddot{z}}}{\sqrt{0.7 \cdot v^{\frac{2}{3}} \cdot I}}$$

A comparison of the two methods can be seen in Fig. 14. Shown below:

![Reykjavík to Ísafjörður flight STD/RMS](isaf_variance.png){width=80%}

For standard test flights this change does not produce a significant difference in the EDR values produced. Some difference in the peaks can be seen at extremely low speeds but these values are unlikely to be significant due to the scaling effect low speed has on the divisor. An EDR track can also be drawn up for this flight using the speed values and RMS output we have seen previously, that track can be seen in the upcoming Fig. 15. In that figure it can be seen that values produced before takeoff and after landing will produce extremely large EDR values due to the drop in speed, it would therefore likely be useful to implement a minimum speed requirement for noteworthy measurements.

\pagebreak

![Reykjavík to Ísafjörður flight EDR](isaf_edr.png){width=80%}

It is also worth considering that the digital filter constants produced in the previous report were made with a 500Hz measuring rate in mind \cite{prev}. This is not really a rate that can be assumed for all phones, in reality we can observe a number of different measuring rates as can be seen in Fig. 16.

\begin{figure}
\begin{center}
\begin{tabular}{ |l|c| }
 \hline
 Smartphone model & Measuring rate (Hz) \\ 
 \hline\hline
 OnePlus 10 Pro & 506 \\
 OnePlus Nord & 400 \\
 Google Pixel 5 & 423 \\
 Google Pixel 6 Pro & 440 \\
 Samsung Galaxy A70 & 200 \\
 Samsung Galaxy FE20 & 500 \\ 
 Samsung Galaxy S22+ & 500 \\  
 \hline
\end{tabular}
\caption{Accelerometer measuring rate of different smartphones}
\end{center}
\end{figure}

The values in the above figure are obtained by measuring sampling rate using Phyphox \cite{phyphox} and also by checking frequency of measurements on volunteer data. For the sake of correctness it is likely that additional filter constants should be implemented and chosen between in order to accommodate a wider range of devices.

# 4 Weather Data

## 4.1 Accessing Observations

* Poor access to existing observations
  - Most APIs provide forecasts instead
* Scraping vedur.is and storing locally for better coverage

## 4.2 Interpolation

To interpolate for a given coordinate:

* For a set of stations
  - Order by ascending distance
  - Loop through three points at a time to create triangles
  - Check whether coordinate exists within triangle using addition of area
  - For a viable triangle calculate the barycentric weights to multiply observations with
  - Wind data for the initial coordinate is given by the measurements at the 3 stations multiplied with their given weights, added together

*NOTE: Show some formulas for this maybe*

# 5 Server Implementation

## 5.1 Subscriber Step

* Parsing Mosquitto data
* Unrolling the measurements and adding weather information
* Bulk insert into storage used by backend

## 5.2 Back-end & Storage

* Storage structure using MongoDB
  - Databases
    * Collections

## 5.3 Access to Data

* API endpoint logic to access stored data for further processing
* Website link for fun

# 6 Conclusion

* Problem sections e.g. phone positioning requirement
* Thoughts on how to continue from here
  - Obtain a list of I values
  - Correct for differences in measuring rate (additional constants)

# 7 References 

<!-- \bibitem{lamport94}
  Leslie Lamport,
  \textit{\LaTeX: a document preparation system},
  Addison Wesley, Massachusetts,
  2nd edition,
  1994. -->

<!-- For mongo: https://sci-hub.se/10.1016/j.matpr.2020.03.634 -->

\bibliographystyle{IEEEtran}
\renewcommand{\bibsection}{}
\begin{thebibliography}{9}

\bibitem{prev} R. E. Garðarsdóttir and S. E. Þorsteinsson, 2021, "Lýðvistun mælinga á ókyrrð í flugi," University of Iceland, ISSN 2772-1078, nr. 100

\bibitem{paho} Eclipse Foundation, "Eclipse Paho Android Service," \textit{eclipse.org}, 2022. [Online]. Available: \url{https://developer.android.com/guide/topics/sensors/sensors_motion}. [Accessed: Aug. 19, 2022].

\bibitem{mqtt_uses} ``MQTT Use Cases,`` \textit{mqtt.org}, 2022. [Online]. Available: \url{https://mqtt.org/use-cases}. [Accessed: Aug. 15, 2022].

\bibitem{mqtt_spec} IBM and Eurotech, "MQTT V3.1 Protocol Specification," 2022. [Online]. Available: \url{https://public.dhe.ibm.com/software/dw/webservices/ws-mqtt/mqtt-v3r1.html}. [Accessed: Aug. 15, 2022].

\bibitem{mongo} B. Jose and S. Abraham, "Performance analysis of NoSQL and relational databases with MongoDB and MySQL," \textit{Materials Today: Proceedings}, vol. 24, no. 3, pp. 2036-2043, March 2020. [Online]. Available: \url{https://doi.org/10.1016/j.matpr.2020.03.634}. [Accessed: Aug. 15, 2022].

\bibitem{sensors} Google Developers, "Motion sensors," \textit{developer.android.com}, 2022. [Online]. Available: \url{https://developer.android.com/guide/topics/sensors/sensors_motion}. [Accessed: Aug. 16, 2022].

\bibitem{doze} Google Developers, "Optimize for Doze and App Standby," \textit{developer.android.com}, 2022. [Online]. Available: \url{https://developer.android.com/training/monitoring-device-state/doze-standby}. [Accessed: Aug. 16, 2022].

\bibitem{location} Google Developers, "Location," \textit{developer.android.com}, 2022. [Online]. Available: \url{https://developer.android.com/reference/android/location/Location}. [Accessed: Aug. 22, 2022].

\bibitem{phyphox} S. Staacks, S. Hütz, H. Heinke and C. Stampfer, "Advanced tools for smartphone+based experiments: Phyphox," \textit{Phys. Educ. 53 045009}, pp. 1-6, 2018.

\end{thebibliography}

# 8 Appendices

* Some source code outtakes
* Direct links to the produced code
