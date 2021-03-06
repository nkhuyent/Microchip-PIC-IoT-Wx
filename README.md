**Azure Ready \| Secured Core\***

**(\*branding is not yet defined, internal use only)**

**Repurpose PIC-IoT Wx Development Board to Connect to Azure through IoT
Hub Device Provisioning Service (DPS)**

Introduction
============

> This document describes how to connect the PIC-IoT Wx Development
> Board (featuring a 16-bit PIC24F MCU, ATECC608A secure element, and
> ATWINC1510 Wi-Fi module) to an Azure IoT Hub via Device Provisioning
> Service (DPS) while leveraging Microsoft’s Azure IoT Embedded C SDK.
> The PIC-IoT is provisioned for use with Azure using self-signed X.509
> certificate-based authentication.
>
> <img src=".//media/image1.png" />

Background Knowledge
====================

### **PIC-IoT Wx Development Board Overview & Features (SMART \| CONNECTED \| SECURE)**

> <img src=".//media/image2.png"/>
>
> Download the [PIC-IoT Wx HW User
> Guide](http://ww1.microchip.com/downloads/en/DeviceDoc/PIC-IoT-Wx-Hardware-User-Guide-DS50002964A.pdf)
> for more details

### **Microchip “Provisioning” vs. Microsoft “Provisioning”**

The
term “provisioning” will be use throughout this document (e.g. IoT
Provisioning Tool, provisioning key, provisioning device, device
provisioning service, etc.). On the Microchip side, the provisioning
process is to securely inject certificates into the hardware. From the
context of Microsoft, provisioning is defined in the relationship
between the hardware and the cloud, Azure. [Azure IoT Hub Device
Provisioning Service
(DPS)](https://docs.microsoft.com/en-us/azure/iot-dps/#:~:text=The%20IoT%20Hub%20Device%20Provisioning%20Service%20%28DPS%29%20is,of%20devices%20in%20a%20secure%20and%20scalable%20manner.)
allows the hardware to be provisioned securely to the right IoT Hub.

<img src=".//media/image3.png"/>

### **High Level Architecture between the Client (PIC-IoT) and the Cloud (Azure)**

This high-level architecture description summarizes the interactions
between the PIC-IoT board and Azure. These are the top six major puzzle
pieces that make up this enablement work of connecting PIC-IoT to Azure
through DPS using X.509-based authentication:

-   ATECC608A: a secure element from the Microchip CryptoAuthentication
    portfolio. It securely stores a private key that is used to
    authenticate the hardware with cloud providers to uniquely identify
    every board <https://www.microchip.com/wwwproducts/en/ATECC608A>

-   ATWINC1510: a low-power consumption Wi-Fi module that has access to
    the device certificate, signer CA certificate, and public key for
    mutual TLS handshaking between the board and the cloud
    <https://www.microchip.com/wwwproducts/en/ATWINC1510>

-   IoT Provisioning Tool: Microchip-provided tool for provisioning
    self-signed certificate utilizing the unique serial number and
    private key stored in the ATECC608A secure element.

-   Embedded C SDK for Azure IoT: Microsoft-provided API designed to
    allow small, low-cost embedded IoT devices to communicate with Azure
    services, serving as translation logic between the application code
    and transport client

-   Azure IoT Hub: IoT Hub is a managed service, hosted in the cloud,
    that acts as a central message hub for bi-directional communication
    between your IoT application and the devices it manages

-   Device Provisioning Service (DPS): a helper service for IoT Hub that
    enables zero-touch, just-in-time provisioning to the right IoT hub
    without requiring human intervention, allowing customers to
    provision millions of devices in a secure and scalable manner

> <img src=".//media/image4.png"/>

In a nutshell, we will use Microchip’s IoT Provisioning Tool to send a
Certificate Signing Request (CSR) to the ATECC608A to generate a
self-signed certificate chain which is then obtained by the ATWINC1510
Wi-Fi module to perform a TLS mutual handshake between the client
(PIC-IoT board) and the server (Azure), specifically using DPS.

Once successful, the PIC-IoT board will be provisioned to the correct
IoT Hub that is pre-linked to DPS during the setup process. We can then
leverage the Azure IoT Explorer which is a graphical tool for
interacting with and testing your IoT devices. Note that the ATECC608A
only contains the private key. The self-signed certificate chain
including root CA, signer CA (or intermediate CA), and device CA is
stored in the ATWINC1510 Wi-Fi module used for the TLS handshake.

### **Embedded C SDK for Azure IoT**

<img src=".//media/image7.png"/>

This is the high-level view of the Embedded C SDK which translates the application code into an Azure-friendly logic that can be easily understood by Azure IoT Hub. Note that Microsoft is only responsible for the logic in the green box; it is up to the IoT Developer to provide the remaining layers of application code, Transport Client, TLS, and Socket. In the provided demo project, Microchip provides the layers in blue.

### **TLS connection**

The TLS connection performs both authentication and encryption.
Authentication consists of two parts:

-   Server authentication; the board authenticates the server

-   Client authentication; the server authenticates the board

Server authentication happens transparently to the user since the
ATWINC1510 on the PIC-IoT board comes preloaded with the required CA
certificate. During client authentication the client private key must be
used, but since this is stored inside the ATECC608A chip and cannot be
extracted, all calculations must be done inside the ATECC608A. Normally,
these calculations would have been done by the ‘connectivity’ element
(the ATWINC1510 in this case). Since this is not possible because the
private key cannot be read out, the ATWINC1510 library offers an API to
delegate the TLS calculations to the main application. The main
application will in turn call the ATECC608A library API’s to perform the
calculations. Before the TLS connection is complete, a shared secret key
must be negotiated between the server and the client. This key is used
to encrypt all future communications during the connection.

### **MQTT Connection** 

After successfully connecting on the TLS level, the board starts
establishing the MQTT connection. Since the TLS handles authentication
and security, MQTT does not have to provide a username or password.

<img src=".//media/image8.ppm"/>

Checklist
=========

Here are major steps of this project. Track your progress using this
list as you complete each stage:

<img src=".//media/image5.png"/>

Setup Instructions 
==================

Prepare your development environment
------------------------------------

### Step 1: Set up Microchip’s MPLAB X IDE Tool Chain 

-   [MPLAB X IDE V5.30 or
    later](https://www.microchip.com/mplab/mplab-x-ide)

-   [XC16 Compiler v1.50 or
    later](https://www.microchip.com/mplab/compilers)

-   MPLAB Code Configurator 3.95 or later (once you finish the
    installation of the previous items, launch MPLAB X IDE &gt; click on
    Tools &gt; Plugins Download &gt; search for MPLAB Code Configurator
    and install it)

> <img src=".//media/image10.png"/>

### Step 2: Set up Azure cloud resources 

-   [Create an Azure free account for 30 day
    trial](https://azure.microsoft.com/en-us/free/)

-   [Set up Azure IoT Hub and Device Provisioning Service in Azure
    Portal](https://docs.microsoft.com/en-us/azure/iot-dps/quick-setup-auto-provision)

### Step 3: Set up Git

1.  Install latest version of [Git](https://git-scm.com/download/win).

Update the internal firmware of the ATWINC1510 Wi-Fi module 
-----------------------------------------------------------

Perform these steps to enable mutual TLS handshake between client’s ECC
and server’s RSA:

-   Download and extract the contents of the “PIC-IoT WINC OTA
    upgrade.zip” file.

-   Follow the steps in the file “PIC-IoT\_WINC\_OTA\_Procedure.pdf” to
    update the Wi-Fi module “Over the Air” (OTA) with the help of an
    HTTP File Server (HFS). This will enable the ATWINC1510 functions
    necessary for TLS handshaking between the PIC-IoT board’s ATECC608A
    and the Azure server’s RSA.

Provision the PIC-IoT (using Microchip’s IoT Provisioning Tool)
---------------------------------------------------------------

> Perform this section to create a self-signed certificate chain which
> acts as a device unique ID to enroll into DPS. The final result is the
> ATWINC1510 obtains the device certificate, CA certificate, and CA
> public key. The device certificate is based on a key pair that has
> previously been generated and is already pre-programmed in the
> ATECC608A. The device certificate is the client certificate used by
> the TLS layer for the client authentication (“device” and “client” are
> used interchangeably in this document). The end goals of the
> provisioning process include the following:

1.  The PC running the Python script requests and receives a Certificate
    Signing Request (CSR) based on the key pair stored in the ATECC608A

2.  The signer CA generates the certificate and returns it alongside
    with its own certificate and public key to the MCU; the two (signer
    and device) certificates are stored in the ATWINC1510

> <img src=".//media/image11.png"/>
>
> Here are the steps to perform the entire provisioning process:

-   Download and extract “iotprovision-azure.zip” into a
    conveniently-located directory (which is easily accessible from a
    command line window)

-   Launch PowerShell: click on Start &gt; type “PowerShell” in the
    Search field &gt; Open

-   Go to the directory where the “iotprovision-azure” executable file
    resides

-   Type the following command (this should generate and write all the
    certs to your disk):

> &gt; .\\iotprovision-azure.exe -c azure -m custom
>
> <img src=".//media/image12.png"/>

-   Ignore the following error message and warning at the end for
    setting the board configuration and disk link:

> Setting board configuration
>
> WARNING: Unable to find a later version to upgrade to
>
> Unsupported cloud provider: azure
>
> Configuring disk link failed

-   Take note of the generated cert location mentioned in the output
    message starting with “Saving to your …\[your
    path\]\\.microchip-iot\\MCHP3261021800001185.”. These certs will be
    used in IoT Hub DPS enrollment in the next step:

    -   root-ca.crt: self-signed root CA cert

    -   signer-ca.crt (aka. intermediate CA) is a uniquely generated by
        the root cert, which is then used to generate device cert in
        \[your-path\]\\.microchip-iot\\MCHP&lt;xxxxxxxxxxxxxxxx&gt;\\device.crt

> <img src=".//media/image13.png"/>
>
> **<u>\* Knowledge backfill:</u>** to understand the steps approaching
> in the upcoming session, please take some time review the document
> [Conceptual understanding of X.509 CA certificates in the IoT
> industry](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-x509ca-concept)
> which describes the value of using X.509 certificate authority (CA)
> certificates in IoT device manufacturing and authentication to IoT
> Hub.
>
> <img src=".//media/image14.gif" />

Enroll device into DPS 
----------------------

### Step 1: Preparing your environment for the certification verifying process:

-   Go to generated cert location in the previous step at “\[your
    path\]\\.microchip-iot”

-   Copy \*.crt files and rename them to \*.pem. (Note: DPS only accepts
    \*.pem or \*.cer file formats. If choosing \*.cer file, only base-64
    encoded certificate)

### Step 2: In Azure portal, upload the root CA cert (“root-ca.pem”) in DPS and do proof-of-possession for X.509 CA certificates with your Device Provisioning Service

#### Register the public part of an X.509 certificate and get a verification code

> Follow the “Register the public part of an X.509 certificate and get a
> verification code” section in [this
> document](https://docs.microsoft.com/en-us/azure/iot-dps/how-to-verify-certificates).
> Again, verification code is generated by encrypting the public key
> portion of your X.509 certification. It will be used to validate the
> uploaded certificate ownership. So make sure to copy the generated
> verification code to notepad for next step.

#### Digitally sign the verification code to create a verification certificate

> Now that you've registered your root CA with Azure IoT Hub, you'll
> need to prove that you actually own it by

1.  Generate the Certificate Signing Request (CSR) using the
    verification code.

2.  Then generate a verification certificate using CSR

> Once this done, you can upload your verification certificate to DPS to
> finish the [proof of
> possession](https://tools.ietf.org/html/rfc5280#section-3.1).

-   Open the Git Bash: Start menu &gt; type “Git Bash”

> A window like this will pop out:
>
> <img src=".//media/image15.png" style="width:3.21739in;height:0.94745in" alt="A picture containing ball, clock Description automatically generated" />

-   Change to your generated certification folder:

> cd \[your path\]\\.microchip-iot

-   **Generate a certification signing request (CSR)** by entering the
    > below command. CRS is generated by encrypting these two key
    > inputs:

    1.  The root-ca.key generated in previous step by Microchip IoT
        > Provisioning tool. This will be used in the openssl command
        > line below.

    2.  The “Verification Code” generated from DPS in previous step.

> You will be asked to provide this during the process of creating the
> CSR.

-   Note that once you enter to command below, you will then be asked to
    > enter information that will be used to will be incorporated into
    > your certificate request. Enter verification code (generated from
    > Azure portal) when prompt for CommonName. For the rest, you can
    > enter anything you want.

> openssl req -new -key root-ca.key -out azure\_signer\_verification.csr
>
> <img src=".//media/image16.png" style="width:5.12286in;height:2.47003in" alt="A screenshot of a cell phone Description automatically generated" />
>
> As the result, you should see this file
> ***azure\_signer\_verification.csr*** in your \\.microchip-iot folder.

-   **Generate a verification certificate** by entering the following.

> openssl x509 -req -in azure\_signer\_verification.csr -CA root-ca.crt
> -CAkey root-ca.key -CAcreateserial -out
> azure\_signer\_verification.cer -days 365 -sha256
>
> <img src=".//media/image17.png" style="width:5.14375in;height:0.784in" alt="A screenshot of a computer screen Description automatically generated" />
>
> As the result, you should see this file
> ***azure\_signer\_verification.cer*** in your \\.microchip-iot folder.

#### Upload the signed verification certificate

> Follow the “Upload the signed verification certificate” section in
> [this
> document](https://docs.microsoft.com/en-us/azure/iot-dps/how-to-verify-certificates).
> As a result, the status of your uploaded certification should be
> “Verified” as shown below (make sure to refresh the page to see the
> updated change in status)
>
> <img src=".//media/image18.png" style="width:5.256in;height:1.40665in" alt="A screenshot of a social media post Description automatically generated" />
>
> <u>\*Quick summary of this Step 2 (before heading to Step 3 below)</u>
> –
>
> By now, you have done verifying your X.509 CA certificate to DPS. To
> link all the above sub-steps and understand why this step is
> significant, I encouraged you to quickly re-visit the few introduction
> paragraphs of [this
> document](https://docs.microsoft.com/en-us/azure/iot-dps/how-to-verify-certificates).

### Step 3: Add a new enrollment group using the signer-ca.pem file

> In the Azure portal, navigate to your DPS &gt; Manage enrollments &gt;
> Select “Enrollment Groups” tab:
>
> <img src=".//media/image19.png" style="width:4.86246in;height:3.09722in" alt="A screenshot of a cell phone Description automatically generated" />
>
> Add enrollment group &gt; Enter “Group name” &gt; Choose Certificate
> as “Attestation Type” &gt; Choose “False” for IoT Edge Device &gt;
> Choose “Intermediate Certificate” as Certificate Type &gt; Upload
> “\[path\]\\.microchip-iot\\signer-ca.pem” to Primary Certificate &gt;
> select “Evenly weighted distribution” for how you want to assign
> devices to hub &gt; select your IoT Hub that this new enrollment group
> can assign to &gt; leave the rest as their existing defaults &gt; hit
> “Save”.
>
> Once this has been done, your enrollment group name should show up in
> the Enrollment Groups tab:
>
> <img src=".//media/image20.png" style="width:3.4806in;height:6.56317in" alt="A screenshot of a cell phone Description automatically generated" />
>
> At this point, as we have not yet programmed the PIC-IoT board with
> the demo firmware, the device should not show up anywhere in Azure.
> This can be verified in both DPS and in IoT Hub.
>
> In your DPS, Enrollment Group’s Registration Records should be empty.
> This can be verified by clicking on your newly enrolled group &gt;
> Registration Records &gt; observe that no device shows up.
>
> In your IoT Hub, your device ID should not show up in the IoT Devices.
> This can be verified by clicking on your IoT Hub that links to your
> DPS &gt; click “IoT devices” (on the left-hand side under “Explorers”
> &gt; observe that your PIC-IoT device ID “sn01237F696BEB9C89FE” does
> not show up.

Connect the PIC-IoT device to Azure
-----------------------------------

1.  Create a local folder to check out (clone) the MPLAB X demo project
    by issuing the following commands in a Command Prompt or PowerShell
    window:

> &gt; git clone
> <https://github.com/jasmineymlo/Microchip-PIC-MCU16-AzureIoT>
>
> &gt; cd Microchip-PIC-MCU16-AzureIoT
>
> &gt; git checkout –t origin/wip
>
> &gt; git submodule update --init

2.  Launch the MPLAB X IDE and then open the demo project (\*.X) located
    at:

> \[path\]\\Microchip-PIC-MCU16-AzureIoT\\myiot.X
>
> (Ignore the following warning messages that may show up in the Output
> window after the project has been loaded)
>
> <img src=".//media/image21.png" style="width:6.5in;height:1.00833in" alt="A screenshot of a cell phone Description automatically generated" />

3.  Modify myiot/Header Files/platform/config/conf\_winc.h with your
    wireless router’s SSID and password:

> \#define CFG\_MAIN\_WLAN\_**SSID** "&lt;*Your SSID*&gt;"
>
> \#define CFG\_MAIN\_WLAN\_**PSK** "&lt;*Your WiFi Password*&gt;"
>
> <img src=".//media/image22.png" style="width:1.96296in;height:2.09616in" alt="A screenshot of a cell phone Description automatically generated" />

4.  Modify myiot/Header Files/platform/config/
    IoT\_Sensor\_Node\_config.h with your DPS ID Scope and comment out
    the Hub Device ID:

<!-- -->

-  Comment out the definition for HUB\_DEVICE\_ID

> //\#define HUB\_DEVICE\_ID "01233EAD58E86797FE"

 - PROVISIONING\_ID\_SCOPE (copy the ID number directly from the Azure
    Portal)

> \#define PROVISIONING\_ID\_SCOPE "0ne0014B096"
>
> <img src=".//media/image23.png" style="width:6.5in;height:1.43056in" alt="A screenshot of a cell phone Description automatically generated" />

5.  Modify myiot\\Source Files\\platform\\application\_manager.c to set
    the highest severity debug level. Add the function call
    debug\_setSeverity(SEVERITY\_DEBUG) after the call to
    debug\_init(attDeviceID)

> <img src=".//media/image24.png" style="width:1.68518in;height:1.54649in" alt="A screenshot of a cell phone Description automatically generated" /><img src=".//media/image25.png" style="width:3.25894in;height:1.2956in" alt="A picture containing bird Description automatically generated" />

6.  Verify the project properties are set correctly before building the
    project

-   Connect the board to PC, make sure “CURIOSITY” device shows up as a
    disk drive in a File Explorer window.

-   Right-click the project myiot &gt; select “Properties” &gt; Verify
    that all Configuration settings are at least the minimum versions as
    shown in the below screenshot (and that your PIC-IoT board is
    selected as the Connected Hardware Tool). If any changes were made,
    make sure to hit the “Apply” button before hitting “OK”.

> <img src=".//media/image26.png" style="width:5.88652in;height:2.68981in" alt="A screenshot of a social media post Description automatically generated" />

7.  Build the project and program the device to connect to Azure with
    the following steps:

-   Open a serial terminal (e.g. PuTTY or TeraTerm) and connect to the
    > board at 9600 baud to view debug/status messages (open PuTTY
    > Configuration window &gt; choose “session” &gt; choose
    > “Serial”&gt; Enter the right COMx port (you can find the COM info
    > by opening your PC’s Device Manager &gt; expand “Ports(COM &
    > LPT) &gt; take note of Curiosity Virtual COM Port”)

> <img src=".//media/image27.png" style="width:3.17917in;height:2.98149in" alt="A screenshot of a cell phone Description automatically generated" />

-   Right-click the myiot project and select “Set as Main Project”

-   Right-click the myiot project and select “Make and Program Device”.
    > This will first automatically clean and build the project. After
    > the “BUILD SUCCESSFUL” message appears in the Output window, the
    > application HEX file will be programmed onto the PIC-IoT board.
    > Once programming has finished, the board will automatically reset
    > and start running its application code. After a few seconds, you
    > can check if the PIC-IoT board has successfully connected to your
    > Wi-Fi Access Point by observing 3 colored LED’s on the board:

    -   BLUE: Solid ON all the time (WIFI)

    -   GREEN: Solid ON all the time (COMM)

    -   ORANGE: Toggling every few seconds (DATA)

Verify the connection between PIC-IoT and Azure
-----------------------------------------------

> PIC-IoT and Azure connection can be verified by 1) viewing debug log
> messages in the serial terminal, 2) correct device ID shows up in DPS
> enrollment that was created earlier, 3) correct device ID shows up in
> the IoT Hub:

1.  Verify in the serial terminal window (e.g. PuTTY or TeraTerm). You
    may need to enable “Local Echo” in your Terminal settings to see
    your keystrokes displayed in the terminal window.

<!-- -->

- Once the PIC-IoT has established a successful connection, continuous
    MQTT messages involving “sendresult” and “Uptime SocketState” will
    be displayed on the terminal window

> sn01237F696BEB9C89FE" DEBUG NORMAL MQTT: sendresult (83)
>
> sn01237F696BEB9C89FE" INFO NORMAL CLOUD: Uptime 2315s SocketState (3)
> MQTT (3)

- Try disabling the status messages now by typing “debug 0” +
    \[ENTER\]

- Hit \[ENTER\] on the terminal window; a list of available commands
    will show up

- Type “device” and \[ENTER\] to verify that the correct device ID is
    displayed (e.g. sn01237F696BEB9C89FE)

- Type “debug 4” and \[ENTER\] and observe that the MQTT messages are
    being transmitted/received again

<!-- -->

2.  Verify in the Azure Device Provisioning Service

> In Azure Portal, go to your DPS &gt; click “Manage enrollments” &gt;
> under Enrollment Group, click “your group name” &gt; click
> “Registration Records” &gt; device should show up with the IoT Hub
> info that it got assigned to. Like so:
>
> <img src=".//media/image28.png" style="width:5.64758in;height:2.34954in" alt="A screenshot of a social media post Description automatically generated" />

3.  In the Azure Portal, go to your IoT Hub &gt; click “IoT
    Devices” &gt; click “Refresh” &gt; device should show up with the
    Status “Enabled” and Authentication Type of “SelfSigned”

> <img src=".//media/image29.png" style="width:4.91723in;height:2.58102in" alt="A screenshot of a cell phone Description automatically generated" />

View PIC-IoT board telemetry on Azure IoT Explorer
--------------------------------------------------

> Once the PIC-IoT connection to Azure has been verified in the previous
> step, the telemetry can be monitored by taking advantage of Azure IoT
> Explorer. The Azure IoT explorer is a graphical tool for interacting
> with and testing your IoT device on Azure. View [this
> document](https://docs.microsoft.com/en-us/azure/iot-pnp/howto-install-iot-explorer#install-azure-iot-explorer)
> for more details.
>
> Follow details in this [Install and use Azure IoT Explorer
> document](https://docs.microsoft.com/en-us/azure/iot-pnp/howto-install-iot-explorer#install-azure-iot-explorer)
> for instructions with these additional notes:

-   Install Azure IoT Explorer – install \*.msi file of the release
    0.12.1 or later.

-   Connect Azure IoT Explorer to IoT Hub by providing IoT Hub’s
    connection string. From the Azure Portal: click on your IoT Hub &gt;
    Shared access polices &gt; iothubowner &gt; connection
    string-primary key &gt; Copy to clipboard

> <img src=".//media/image30.png" style="width:3.00926in;height:5.85762in" alt="A screenshot of a cell phone Description automatically generated" />

-   Launch Azure IoT Explorer: Click on “Add connection” &gt; paste the
    Connection string &gt; Save

> <img src=".//media/image31.png" style="width:5.56482in;height:2.24436in" alt="A screenshot of a cell phone Description automatically generated" />
>
> <img src=".//media/image32.png" style="width:4.49781in;height:3.72222in" alt="A screenshot of a social media post Description automatically generated" />

-   Once the IoT Hub connection has been added, the list of devices
    connected to the hub appears. Verify that the correct PIC-IoT serial
    number is displayed, and then click on it:

> <img src=".//media/image33.png" style="width:5.76119in;height:1.45753in" alt="A screenshot of a cell phone Description automatically generated" />

-   Verify that the IoT Hub can send a command to the device. In the
    device window:

> Click Direct method tab &gt; enter “blink” in Method name &gt; enter
> {"duration":10} for payload &gt; Invoke method (a pop-up message
> displaying ERROR will appear, but that is expected since the IoT Hub
> is sending a simulated error condition to the PIC-IoT board).
>
> Observe that the RED LED (labeled as ERROR on the board) stays on for
> 10 seconds.
>
> <img src=".//media/image34.jpeg" style="width:1.65714in;height:2.32155in" alt="A circuit board Description automatically generated" />

-   Monitor telemetry sending from PIC-IoT to Azure IoT Hub. In the
    device window:

> Click Telemetry tab &gt; Start: after about 2 minutes, observe
> telemetry data is being updated in real-time approximately every 5s.
>
> <img src=".//media/image35.png" style="width:5.38735in;height:3.18982in" alt="A screenshot of a cell phone Description automatically generated" />

Further consideration
=====================

Instead of connection to IoT Hub and view the telemetry on Azure IoT
explorer, the PIC-IoT board can be provisioned to the IoT Central
instead, which has a built-in dashboard to monitor the telemetry. This
can be considered in your future project.

Conclusion 
==========

You are now able to connect PIC-IoT to Azure using self-signed cert base
authentication and have deeper knowledge of how all the pieces of puzzle
fit together from ATECC608 security element, WINC1510 Wi-fi, Azure
Embedded C SDK, Azure IoT Hub and DPS. Let’s start thinking out of the
box and see how you can apply this project to provision securely and
quick a massive number of Microchip devkits to Azure and safely manage
them through the whole device life cycle.

Support 
=======
