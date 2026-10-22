# Android App for Mobile Intrusion Detection

Two Android phones form an intrusion alarm, built in Simulink. One phone guards a room and detects sudden light changes and movement while armed. It sends an alert with its GPS location and a timestamp over UDP to the second phone. The second phone plays a recorded alert message and can remotely control the first phone. Both phones run the same model and can act as either the Security Node or the Monitoring Console.

**Demo video (unlisted):** [Watch the demo on YouTube](https://www.youtube.com/watch?v=FAD9jujfgXU)

---

## What it does

| Requirement | How it is done |
|---|---|
| Armed security node | Phone A is set up as the Node and starts in the ARMED state |
| Light increase detection | Light Sensor compared with a moving average |
| Movement / tamper detection | Gyroscope rotation rate |
| Alert with GPS and timestamp | Latitude, longitude, altitude and time sent over UDP |
| Audible alert on the monitor | Recorded message (`Alert.wav`) plays on the Console |
| Remote siren | Console command makes the Node play `Siren.wav` |
| Remote arm / disarm | Console command toggles the Node's state |
| Interchangeable roles | One model on both phones, role set by the NODE button |

---

## Files

- `DSP_Simulink.slx`: the Simulink model (deployed to both phones)
- `Alert.wav`: recorded alert message (keep it in the same folder as the model)
- `Siren.wav`: siren sound (same folder)

---

## How the model works

The same model is deployed to both phones. Each phone only uses the blocks for its role, and the rest stay idle. All blocks run at a 0.1 s sample time.

### 1. Roles
Pressing **NODE** on a phone makes it the Security Node and sends a signal (UDP port 25004) that makes the other phone the Console. A Unit Delay keeps the chosen role, and this flag switches the blocks of the other role off.

### 2. Detection (Node)
- **Light:** the Light Sensor reading is compared with a 20-sample moving average (a Discrete FIR Filter with coefficients `ones(1,20)/20`, delayed by one sample). If the absolute difference is 20 or more, the light has jumped suddenly.
- **Movement:** |x| + |y| + |z| of the Gyroscope. If the sum is 1 or more, the phone is being moved.
- The two flags go into an **OR**.
- A 4-input **AND** lets the alert through only when all of these are true:
  breach (the OR), the phone is ARMED, the 5 s startup delay has passed (Step block), and this phone is the Node.

### 3. Arming (Stateflow chart)
A chart with two states, ARMED and DISARMED. A pulse on `toggle` flips the state, and `is_armed` is the output. A Detect Increase block turns a held button into a single pulse, so one press gives one flip.

### 4. Alert (Node to Console)
- When the AND is true, an Enabled Subsystem sends four values, [latitude, longitude, altitude, time], over UDP (port 25000).
- A live alert flag is sent separately (port 25005).

### 5. Console
- Receives the four values and holds them with an Enabled Subsystem (`Hold_Alert`). A Demux splits them into the Latitude, Longitude, Altitude and Timestamp displays.
- The alert flag passes through a 20-sample FIR filter that stretches it, so `Alert.wav` keeps playing while movement continues and stops about 2 seconds after it ends. It plays only on the Console.

### 6. Remote control (Console to Node)
The **State_change** button sends a command to the Node (UDP port 25002). Detect Increase turns it into a single pulse that flips the Chart between ARMED and DISARMED. The Node also plays `Siren.wav` and flashes a full-screen white Video Display (pulse generator, 0.25 s) while the Chart output is 0 (DISARMED).

### UDP ports

| Port | Direction | Data |
|---|---|---|
| 25000 | Node to Console | latitude, longitude, altitude, time |
| 25005 | Node to Console | live alert flag |
| 25004 | between the phones | NODE role signal |
| 25002 | Console to Node | State_change command |

---

## Requirements

- MATLAB R2023a with Simulink and the **Simulink Support Package for Android Devices**
- Android Studio Arctic Fox (2020.3.1) Patch 4, with SDK Platform 32, Build-Tools 32.0.0 and NDK 23.1.7779620
- Two Android phones with USB debugging enabled
- Both phones on the same Wi-Fi network or hotspot

## How to run

1. Open `DSP_Simulink.slx` from a folder outside OneDrive, with `Alert.wav` and `Siren.wav` in the same folder.
2. In the **UDP Send** blocks, set the **Remote address** to the IP address of the *other* phone, in single quotes. Phone 1's build points at Phone 2, and Phone 2's build points at Phone 1. Nothing else changes between the two builds.
3. Set **Hardware board** to **Android Device**, then click **Build, Deploy & Start** for each phone. Tap **Allow** on the permission requests (location, etc.).
4. On the phone that will stay in place, press **NODE**. It starts ARMED, and after about 5 seconds it watches for light changes and movement.
5. Move the Node or change the light on its sensor. The Console shows the location and time and plays the alert.
6. Use **State_change** on the Console to control the Node remotely.

---

## Known limitations

- **LED flash:** the Android support package has no flashlight block, so a flashing white screen on the Node stands in for the camera LED.
- **Timestamp:** the time value is seconds since the app started, not a clock time.
- **GPS:** latitude and longitude may read zero indoors until the phone gets a location fix.
- **Network:** UDP is unreliable by nature, so a packet can occasionally be lost. Some Wi-Fi networks (public or college) block phone-to-phone traffic, and a mobile hotspot works around this.
