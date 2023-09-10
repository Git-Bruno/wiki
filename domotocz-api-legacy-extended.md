==Overview==
This page is aimed at developers wishing to improve Domoticz functionality via Python Plugins. If you simply want to use a plugin that a developer has already created for you, please see [[Using Python plugins]]. To see the Plugins already developed (for a reference) got to page [[Plugins]].

The Domoticz Python Framework is not a general purpose Domoticz extension.  It is designed to allow developers to easily create an interface between a piece of Hardware (or Virtual Hardware) and Domoticz.  To that end it provides capabilities that seek to do the hard work for the developer leaving them to manage the messages that move backwards and forwards between the two.

The Framework provides the plugin with a full Python 3 instance that exists for as long as Domoticz is running.  The plugin is called by Domoticz when relevant events occur so that it can manage connectivity to the hardware and state synchronisation.

Multiple copies of the same plugin can run for users that have more than one instance of a particular hardware type.

===Plugin Framework types===
There are current two variations of the Plugin Framework available, plugin authors can use either but '''not both''' at the same time:

#Legacy Framework:  This has been stable for several years, invoked by importing 'Domoticz' into the plugin
#Extended Framework:
#*Built on Legacy Framework but has a new object model and event localisation
#*Invoked by importing 'DomoticzEx' into the plugin.
#*Creates a two layer mapping over the DeviceStatus table of Devices and Units
#*Plugins are not restricted to 256 Units (each Device can have 256 Units)

===Sleeping and multi-threading details:===
The following things should not be attempted using the Python Framework:

*Waiting or sleepingin Domoticz callbacks. Plugin callbacks are single threaded so the whole plugin system will wait.

The Python Framework has been uplifted to support multi-threaded plugins, key points:

*Use of asynchronous code or modules and callback functions are now supported. These should function as expected.
*All threads started within the plugin (either directly or by imported modules) must be terminated by the plugin prior to the plugin stopping ('onStop' is the recommended place for this). Failure to do this will result in Python aborting Domoticz during hardware 'Stops' &/or 'Updates'. The Plugin Framework cannot enumerate threads started by the plugin or stop them (Python limitation) and any active threads when the plugin interpreter is destroyed will cause Python to abort. '''YOU HAVE BEEN WARNED !'''

An example of a multi-threaded plugin including how to show running threads and thread shutdown can be found  [https://github.com/domoticz/domoticz/blob/development/plugins/examples/Mutli-Threaded.py here on Github]

==Overall Structure==
The plugin documentation is split into two distinct parts:

*Plugin Definition - Telling Domoticz about the plugin
*Runtime Structure - Interfaces and APIs to manage message flows between Domoticz and the hardware

===Getting started===
If you are writing a Python plugin from scratch, you may want to begin by using the '''Script Template''' in ''domoticz/plugins/examples/BaseTemplate.py'', which is also available from the github source repo at https://github.com/domoticz/domoticz/blob/master/plugins/examples/BaseTemplate.py

==Plugin Definition==
To allow plugins to be added without the need for code changes to Domoticz itself requires that the plugins be exposed to Domoticz in a generic fashion.  This is done by having the plugins live in a set location so they can be found and via an XML definition embedded in the Python script itself that describes the parameters that the plugin requires.

During Domoticz startup all directories directly under the 'plugins' directory are scanned for python files named <code>plugin.py</code> and those that contain definitions are indexed by Domoticz.  When the Hardware page is loaded the plugins defined in the index are merged into the list of available hardware and are indistinguishable from natively supported hardware.  For the example below the Kodi plugin would be in a file <code>domoticz/plugins/Kodi/plugin.py</code>.  Some example Python scripts can be found in the <code>domoticz/plugins/examples</code> directory.

Plugin definitions expose some basic details to Domoticz and a list of the parameters that users can configure. Each defined parameters will appear as an input on the Hardware page when the plugin is selected in the dropdown. 

Definitions look like this:
<syntaxhighlight lang="xml">
"""
<plugin key="Kodi" name="Kodi Players" author="dnpwwo" version="1.0.0" wikilink="http://www.domoticz.com/wiki/plugins/Kodi.html" externallink="https://kodi.tv/">
    <description>
        <h2>Kodi Media Player Plugin</h2><br/>
        <h3>Features</h3>
        <ul style="list-style-type:square">
            <li>Comes with three selectable icon sets: Default, Black and Round</li>
            <li>Display Domoticz notifications on Kodi screen if a Notifier name is specified and events configured for that notifier</li>
            <li>Multiple Shutdown action options</li>
            <li>When network connectivity is lost the Domoticz UI will optionally show the device(s) with a Red banner</li>
        </ul>
        <h3>Devices</h3>
        <ul style="list-style-type:square">
            <li>Status - Basic status indicator, On/Off. Also has icon for Kodi Remote popup</li>
            <li>Volume - Icon mutes/unmutes, slider shows/sets volume</li>
            <li>Source - Selector switch for content source: Video, Music, TV Shows, Live TV, Photos, Weather</li>
            <li>Playing - Icon Pauses/Resumes, slider shows/sets percentage through media</li>
        </ul>
    </description>
    <params>
        <param field="Address" label="IP Address" width="200px" required="true" default="127.0.0.1"/>
        <param field="Port" label="Port" width="30px" required="true" default="9090"/>
        <param field="Mode1" label="MAC Address" width="150px" required="false"/>
        <param field="Mode2" label="Shutdown Command" width="100px">
            <options>
                <option label="Hibernate" value="Hibernate"/>
                <option label="Suspend" value="Suspend"/>
                <option label="Shutdown" value="Shutdown"/>
                <option label="Ignore" value="Ignore" default="true" />
            </options>
        </param>
        <param field="Mode3" label="Notifications" width="75px">
            <options>
                <option label="True" value="True"/>
                <option label="False" value="False"  default="true" />
            </options>
        </param>
        <param field="Mode6" label="Debug" width="75px">
            <description><h2>Debugging</h2>Select the desired level of debug messaging</description>
            <options>
                <option label="True" value="Debug"/>
                <option label="False" value="Normal"  default="true" />
            </options>
        </param>
    </params>
</plugin>
"""
</syntaxhighlight>
Definition format details:
{| class="wikitable"
|-
!Tag
!Description/Attributes
|-
|<plugin>
|Required.
{| class="wikitable"
|-
!Attribute
!Purpose
|-
|key
|Required.
Unique identifier for the plugin. Never visible to users this should be short, meaningful and made up of characters A-Z, a-z, 0-9 and -_ only.
Stored in the ‘Extra’ field in the Hardware.
Used as the base name for the plugin Python file, e.g. ‘DenonAVR’ maps to /plugins/DenonAVR/plugin.py 
|-
|name
|Required.
Meaningful description for plugin.
Shown in the ‘Type’ drop down on the Hardware page.
|-
|version
|Required.
Plugin version.  Used to compare plugin version against repository version to determine if updates are available.
|-
|author
|Optional.
Plugin author.  Supplied to Hardware page but not currently shown.
|-
|wikilink
|Optional.
Link to the plugin usage documentation on the Domoticz wiki.
Shown on the Hardware page.
|-
|externallink
|Optional.
Link to an external URL that contains additional information about the device.
Shown on the Hardware page.
|-
|}
|-
|<description>
|HTML description, contained within <plugin> and <param> tags.
This is displayed in the Hardware page when configuring a plugin. Embedded HTML tags are allowed and respected. 
|-
|<params>
|Simple wrapper for param tags. Contained within <plugin>.
|-
|<param>
|Parameter definitions, Contained within <params>.
Parameters are used by the Hardware page during plugin addition and update operations.  These are stored in the Hardware table in the database and are made available to the plugin at runtime.
{| class="wikitable"
|-
!Attribute
!Purpose
|-
|field
|Required.
Column in the hardware table to store the parameter.<br>
Valid values:

*SerialPort - used by ‘serial’ transports
*Address - used by non-serial transports
*Port - used by non-serial transports
*Mode1 - General purpose
*Mode2 - General purpose
*Mode3 - General purpose
*Mode4 - General purpose
*Mode5 - General purpose
*Mode6 - General purpose
*Username - Username for authentication
*Password - Password for authentication
|-
|label
|Required.
Field label.  Where possible standard labels should be used so that Domoticz can translate them when languages other than English are enabled.
|-
|width
|Optional.
Width of the field
|-
|required
|Optional.
Marks the field as mandatory and the device will not be added unless a value is provided
|-
|default
|Optional.
Default value for the field. Ignored when <options> are specified.
|-
|password
|Optional.
Makes the text unreadable during entry and when hardware page is viewed subsequently. Ignored when <options> are specified.
|-
|}
|-
|<options>
|Simple wrapper for option tags. Contained within <param>. 
Parameter definitions that contain this tag will be shown as drop down menus in the Hardware page.  Available values are defined by the <option> tag.
|-
|<option>
|Instance of a drop down option. Contained within <options>. 
{| class="wikitable"
|-
!Attribute
!Purpose
|-
|label
|Required.
Text to show in dropdown menu.  Will be translated if a translation exists.
|-
|value
|Required.
Associated value to store in database
|-
|default
|Optional. Valid value:
true
|-
|}
|-
|}

==Runtime Structure==
Domoticz exposes settings and device details through four Python dictionaries.

===Settings===
Contents of the Domoticz Settings page as found in the Preferences database table. These are always available and will be updated if the user changes any settings.  The plugin is not restarted. They can be accessed by name for example: <code>Settings["Language"]</code>

===Parameters===
These are always available and remain static for the lifetime of the plugin.  They can be accessed by name for example: <code>Parameters["SerialPort"]</code>
{| class="wikitable"
|-
!
!Description
|-
|Key
|Unique short name for the plugin, matches python filename.
|-
|Name
|Name assigned by the user to the hardware.
|-
|HomeFolder
|Folder or directory where the plugin was run from.
|-
|Author
|Plugin Author.
|-
|Version
|Plugin version.
|-
|Address
|IP Address, used during connection.
|-
|Port
|IP Port, used during connection.
|-
|Username
|Username.
|-
|Password
|Password.
|-
|Mode1
|General Parameter 1
|-
|...
|
|-
|Mode6
|General Parameter 6
|-
|SerialPort
|SerialPort, used when connecting to Serial Ports.
|-
|DomoticzVersion
|Domoticz version, for instance 4.11774
|-
|DomoticzHash
|Domoticz hash, for instance a06400e05
|-
|DomoticzBuildTime
|Domoticz build time, for instance 2020-03-03 15:41:00
|}

===Extended Plugin Framework===

Available in Domoticz Stable 2022.1!

====Devices====
{| class="wikitable"
|-
!
!Description
|-
|Members
|DeviceID.
Devices are a container class designed to hold the Units that share a common DeviceID

{| class="wikitable"
|-
!Function
!Access
!Description
|-
|DeviceID
|Read
|External device identifier
|-
|TimedOut
|Read/Write
|Device TimedOut flag. 1=True, 0=False
|-
|Units
|Read
|Dictionary of Units that have the DeviceID
|-
|}
|-
|Methods
|Per device calls into Domoticz to manipulate specific devices
{| class="wikitable"
|-
!Function
!Description
|-
|__init__
|
{| class="wikitable"
|-
!Parameter
!Description
|-
|DeviceID
|Mandatory.<br>The DeviceID to be used with the device. <br>Device objects will be created automatically either during plugin load or when Unit objects are created with new DeviceIDs<br>Field type is Varchar(25)
|-
|}
|-
|Refresh
|Parameters: None.
Refreshes the Device and the Units with the matching DeviceID from the Domoticz database.

Not normally required because device values are updated when callbacks are invoked.
|-
|}
|-
|Callbacks
|If defined, these callbacks will be sent to the Device object rather than to the module level function. Note that the function definitions are different because the API only passes the required parameters so no Device details are passed.

To define these callback plugin authors need to register a local object that extends the Domoticz default.  See the Register function in the [[#C.2B.2B_Callable_API]] section for details
{| class="wikitable"
|+
!Callback
!Description
|-
|onDeviceAdded
|Parameters: Unit
Called just after a new device has been added to the hardware, for example by a Domoticz Web API call.

onDeviceAdded is not called when the plugin adds a device.
|-
|onDeviceModified
|Parameters: Unit
Called just after a device owned by the plugin has been modifed, for example by a Domoticz Web API call.

onDeviceModified is not called when the plugin initiated the change e.g. by calling Update.
|-
|onDeviceDeleted
|Parameters: Unit
Called just before a device owned by the plugin is been removed, for example by a Domoticz Web API call.

onDeviceRemoved is not called when the plugin removed the device by calling Delete.
|-
|onCommand
|Parameters: Unit, Command, Level, Hue
Called by Domoticz in response to a script or Domoticz Web API call sending a command to a Unit in the Device's Units dictionary
|-
|onSecurityEvent
|Parameters: Level, Description.
|}
<syntaxhighlight lang="python3">
import DomoticzEx as Domoticz

...

class hvacDevice(Domoticz.Device):

    def __init__(self, DeviceID):
        super().__init__(DeviceID)

    def onDeviceAdded(self, Unit):
        Domoticz.Log("Device onDeviceAdded for Unit: "+str(self.Units[Unit]))
        
    def onDeviceModified(self, Unit):
        Domoticz.Log("Device onDeviceModified for Unit: "+str(self.Units[Unit]))

    def onDeviceRemoved(self, Unit):
        Domoticz.Log("Device onDeviceRemoved for Unit: "+str(self.Units[Unit]))

    def onCommand(self, Unit, Command, Level, Hue):
        Domoticz.Log("onCommand called for Device '" + DeviceID + "', Unit " + str(Unit) + ": Parameter '" + str(Command) + "', Level: " + str(Level))

        Command = Command.strip()
        action, sep, params = Command.partition(' ')
        action = action.capitalize()

# Override the default Domoticz device object with custom one 
Domoticz.Register(Device=hvacDevice)

def onDeviceModified(self, DeviceID, Unit):
    # This will never be called because the Device specific version overrides it
    Domoticz.Log("Device onDeviceModified")
</syntaxhighlight>
|-
|}

====Units====
{| class="wikitable"
|-
!
!Description
|-
|Members
|Unit.
Unit number for the device as specified in the Manifest.
Note:  Units can be deleted in Domoticz so not all Units specified will necessarily still be present.
E.g: <code>Domoticz.Log(Devices["123456"].Units[2].Name)</code>

{| class="wikitable"
|-
!Function
!Access
!Description
|-
|ID
|Read
|The Domoticz Device ID
|-
|Unit
|Read
|The Unit number
|-
|Name
|Read/Write
|Current Name in Domoticz
|-
|nValue
|Read/Write
|Current numeric value
|-
|sValue
|Read/Write
|Current string value
|-
|SignalLevel
|Read/Write
|Numeric signal level
|-
|BatteryLevel
|Read/Write
|Numeric battery level
|-
|Image
|Read/Write
|Current image number
|-
|Type
|Read/Write
|Numeric device type
|-
|SubType
|Read/Write
|Numeric device subtype
|-
|Switchtype
|Read/Write
|Numeric device switchtype
|-
|Used
|Read/Write
|Device Used flag. 1=True, 0=False
|-
|Options
|Read/Write
|Current Device options dictionary
|-
|LastLevel
|Read/Write
|Last level as reported by Domoticz, used to control sliders on Dimmers and Blinds.
|-
|LastUpdate
|Read
|Timestamp of the last update, e.g: 2017-01-22 01:21:11
|-
|Description
|Read/Write
|Description of the device, visible in "Edit" dialog in Domoticz Web UI.
|-
|Color
|Read/Write
|Current color, see documentation of [[Developing a Python plugin#Callbacks|onCommand callback]] for details on the format.
|-
|Adjustment
|Read
|Float value representing the adjustment value set for the Unit (via the web UI)
Note: This is made available to the Plugin author so it can be applied if required, it is not applied by the framework automatically.
|-
|Multiplier
|Read
|Float value representing the multiplier value set for the Unit (via the web UI).  
Note: This is made available to the Plugin author so it can be applied if required, it is not applied by the framework automatically.
|-
|Parent
|Read
|The 'owning' Device object
|-
|}
|-
|Methods
|Per device calls into Domoticz to manipulate specific devices
{| class="wikitable"
|-
!Function
!Description
!Example
|-
|__init__
|
{| class="wikitable"
|-
!Parameter
!Description
|-
|Name
|Mandatory.<br>Is appended to the Hardware name to set the initial Domoticz Device name.<br>This should not be used in Python because it can be changed in the Web UI.
|-
|DeviceID
|Mandatory.<br>Set the DeviceID to be used with the device. Only required to override the default which is and eight digit number dervice from the HardwareID and the Unit number in the format "000H000U".<br>Field type is Varchar(25)
|-
|Unit
|Mandatory.<br>Plugin index for the Device. This can not change and should be used reference Domoticz devices associated with the plugin.  This is also the key for the Devices Dictionary that Domoticz prepopulates for the plugin.<br>Unit numbers must be less than 256.
|-
|TypeName
|Optional.<br>Common device types, this will set the values for Type, Subtype and Switchtype.<br>See [[#Available_Device_Types]]
|-
|Type
|Optional.<br>Directly set the numeric Type value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Subtype
|Optional.<br>Directly set the numeric Subtype value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Switchtype
|Optional.<br>Directly set the numeric Switchtype value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Image
|Optional.<br>Set the image number to be used with the device. Only required to override the default.<br>All images available by JSON API call "/json.htm?type=custom_light_icons"
|-
|Options
|Optional.<br>Set the Device Options field. A few devices, like Selector Switches, require additional details to be set in this field. It is a Python dictionary consisting of key values pairs, where the keys and values must be strings. See the example to the right.
|-
|Used
|Optional<br>
Values<br>
0 (default) Unused<br>
1 Used.<br>Set the Device Used field. Used devices appear in the appropriate tab(s), unused devices appear only in the Devices page and must be manually marked as Used.
|-
|Description
|Optional
The description of the Unit
|-
|}
|Both positional and named parameters are supported.<br>Creates a new Unit object in Python. E.g:<syntaxhighlight lang="python3">
myUnit1 = DomoticzEx.Unit("Total", "000A0001", 1, 113)
myUnit2 = DomoticzEx.Unit(Name="My Counter", DeviceID="000A0001", Unit=2, TypeName="Counter Incremental")
</syntaxhighlight>or<syntaxhighlight lang="python3">
import DomoticzEx

def onStart():
    if (len(Devices) == 0):
        DomoticzEx.Unit(Name="Status", \
            DeviceID="myDevice", Unit=1, \
            Type=17, Switchtype=17).Create()
</syntaxhighlight>||
|-
|Create
|Parameters: None, acts on current object.
Successfully created Units are immediately added to the Devices dictionary
|Creates the Unit in Domoticz from the object. E.g:<syntaxhighlight lang="python3">
myUnit = DomoticzEx.Unit(Name="Kilo Watts", DeviceID="Power", Unit=16, TypeName="kWh")
myUnit.Create()
Domoticz.Log("Created device: "+Devices["Power"].Units[16].Name+ \ 
             ", myUnit also still points to the Unit: "+myDev.Name)
</syntaxhighlight>or<syntaxhighlight lang="python3">
DomoticzEx.Unit(Name="Kilo Watts", DeviceID="Power", Unit=16, TypeName="kWh").Create()
Domoticz.Log("Created device: "+Devices["Power"].Units[16].Name)
</syntaxhighlight><br />
|-
|Update
|Applies the Unit's current values to the Domoticz database.
{| class="wikitable"
|-
!Parameter
!Description
|-
|Log
|Optional, Boolean, Default: False
During the update a row will be written to the log table.
|-
|TypeName
|Optional, String.<br>Common device types, this will set the values for Type, Subtype and Switchtype prior to the database update.<br>See [[#Available_Device_Types]]
|-
|}
If either the nValue or sValues have changed then the update function will signal the event and notification systems 
|<syntaxhighlight lang="python3">
Devices["myDevice"].Unit[1].nValue = 42
Devices["myDevice"].Unit[1].sValue = "Ultimate answer"
Devices["myDevice"].Unit[1].Update(Log=True)
</syntaxhighlight>

|-
|Delete
|Parameters: None, acts on current object.
Deleted Units are immediately removed from the Devices dictionary but local instances of the object are unchanged.
|Deletes the device in Domoticz.E.g:
  Devices["myDevice"].Unit[1].Delete()
or
  myDev = Devices["myDevice"].Unit[1]
  myDev.Delete()
|-
|Refresh
|Parameters: None.
Refreshes the values for the current Unit from the Domoticz database.

Not normally required because Unit values are updated when callbacks are invoked.
|<syntaxhighlight lang="python3">
Devices["myDevice"].Unit[1].Refresh()
</syntaxhighlight>
|-
|Touch
|Parameters: None.
Updates the Unit's 'last seen' time in the database and nothing else. No events or notifications are triggered as a result of touching a Unit.

After the call the Unit's LastUpdate field will reflect the new value.
|<syntaxhighlight lang="python3">
Devices["myDevice"].Unit[1].Touch()
</syntaxhighlight>
|-
|}
|-
|Callbacks
|
{|
|If defined, these callbacks will be sent to the Unit object rather than to the Device or module level function.  Note that the function definitions are different because the API only passes the required parameters so no Device or Unit details are passed.

To define these callback plugin authors need to register a local object that extends the Domoticz default.  See the Register function in the [[#C.2B.2B_Callable_API]] section for details
{| class="wikitable"
|+
!Callback
!Description
|-
|onDeviceAdded
|Parameters: None
Called just after a new device has been added to the hardware, for example by a Domoticz Web API call.

onDeviceAdded is not called when the plugin adds a device by calling Create.
|-
|onDeviceModified
|Parameters: None
Called just after a device owned by the plugin has been modifed, for example by a Domoticz Web API call.

onDeviceModified is not called when the plugin initiated the change e.g. by calling Update.
|-
|onDeviceRemoved
|Parameters: None
Called just before a device owned by the plugin is removed, for example by a Domoticz Web API call.

onDeviceRemoved is not called when the plugin removed the device by calling Delete.
|-
|onCommand
|Parameters: Command, Level, Hue
Called by Domoticz in response to a script or Domoticz Web API call sending a command to the Unit
|-
|onSecurityEvent
|Parameters: Level, Description.
|}
<syntaxhighlight lang="python3">
import DomoticzEx as Domoticz

...

class hvacDevice(Domoticz.Device):
    def __init__(self, DeviceID):
        super().__init__(DeviceID)

    def onDeviceModified(self, Unit):
        # This will never be called because the Unit specific version overrides it
        Domoticz.Log("Device onDeviceModified for Unit: "+str(self.Units[Unit]))

class hvacUnit(Domoticz.Unit):

    def __init__(self, Name, DeviceID, Unit, TypeName="", Type=0, Subtype=0, Switchtype=0, Image=0, Options="", Used=0, Description=""):
        super().__init__(Name, DeviceID, Unit, TypeName, Type, Subtype, Switchtype, Image, Options, Used, Description)

    def onDeviceAdded(self):
        Domoticz.Log("Unit onDeviceAdded for "+str(self.Name))

    def onDeviceModified(self):
        Domoticz.Log("Unit onDeviceModified for "+str(self.Name))

    def onDeviceRemoved(self):
        Domoticz.Log("Unit onDeviceRemoved for "+str(self.Name))

    def onCommand(self, Command, Level, Hue):
        Domoticz.Log("onCommand called for '" + str(self.Name)+ "': Parameters '" + str(Command) + "', Level: " + str(Level))

        Command = Command.strip()
        action, sep, params = Command.partition(' ')
        action = action.capitalize()

# Override the default Domoticz objects with custom ones 
Domoticz.Register(Device=hvacDevice, Unit=hvacUnit)

def onDeviceModified(self, DeviceID, Unit):
    # This will never be called because the Unit specific version overrides it
    Domoticz.Log("Device onDeviceModified")
</syntaxhighlight>
|}
|-
|}

===Legacy Plugin Framework===
====Devices====
{| class="wikitable"
|-
!
!Description
|-
|Key
|Unit.
Unit number for the device as specified in the Manifest.
Note:  Devices can be deleted in Domoticz so not all Units specified will necessarily still be present.
E.g: <code>Domoticz.Log(Devices[2].Name)</code>

{| class="wikitable"
|-
!Function
!Description
|-
|ID
|The Domoticz Device ID
|-
|Name
|Current Name in Domoticz
|-
|DeviceID
|External device identifier
|-
|nValue
|Current numeric value
|-
|sValue
|Current string value
|-
|SignalLevel
|Numeric signal level
|-
|BatteryLevel
|Numeric battery level
|-
|Image
|Current image number
|-
|Type
|Numeric device type
|-
|SubType
|Numeric device subtype
|-
|Switchtype
|Numeric device switchtype
|-
|Used
|Device Used flag. 1=True, 0=False
|-
|Options
|Current Device options dictionary
|-
|TimedOut
|Device TimedOut flag. 1=True, 0=False
|-
|LastLevel
|Last level as reported by Domoticz
|-
|LastUpdate
|Timestamp of the last update, e.g: 2017-01-22 01:21:11
|-
|Description
|Description of the device, visible in "Edit" dialog in Domoticz Web UI.
|-
|Color
|Current color, see documentation of [[Developing a Python plugin#Callbacks|onCommand callback]] for details on the format.
|-
|}
|-
|Methods
|Per device calls into Domoticz to manipulate specific devices
{| class="wikitable"
|-
!Function
!Description
!Example
|-
|__init__
|
{| class="wikitable"
|-
!Parameter
!Description
|-
|Name
|Mandatory.<br>Is appended to the Hardware name to set the initial Domoticz Device name.<br>This should not be used in Python because it can be changed in the Web UI.
|-
|Unit
|Mandatory.<br>Plugin index for the Device. This can not change and should be used reference Domoticz devices associated with the plugin.  This is also the key for the Devices Dictionary that Domoticz prepopulates for the plugin.<br>Unit numbers must be less than 256.
|-
|TypeName
|Optional.<br>Common device types, this will set the values for Type, Subtype and Switchtype.<br>See [[#Available_Device_Types]]
|-
|Type
|Optional.<br>Directly set the numeric Type value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Subtype
|Optional.<br>Directly set the numeric Subtype value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Switchtype
|Optional.<br>Directly set the numeric Switchtype value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Image
|Optional.<br>Set the image number to be used with the device. Only required to override the default.<br>All images available by JSON API call "/json.htm?type=custom_light_icons"
|-
|Options
|Optional.<br>Set the Device Options field. A few devices, like Selector Switches, require additional details to be set in this field. It is a Python dictionary consisting of key values pairs, where the keys and values must be strings. See the example to the right.
|-
|Used
|Optional<br>
Values<br>
0 (default) Unused<br>
1 Used.<br>Set the Device Used field. Used devices appear in the appropriate tab(s), unused devices appear only in the Devices page and must be manually marked as Used.
|-
|DeviceID
|Optional.<br>Set the DeviceID to be used with the device. Only required to override the default which is and eight digit number dervice from the HardwareID and the Unit number in the format "000H000U".<br>Field type is Varchar(25)
|-
|}
|Both positional and named parameters are supported.<br>Creates a new device object in Python. E.g:
  myDev1 = Domoticz.Device("Total", 1, 113)
  myDev2 = Domoticz.Device(Name="My Counter", Unit=2, TypeName="Counter Incremental")
or
  import Domoticz
  #
  def onStart():
    if (len(Devices) == 0):
        Domoticz.Device(Name="Status",  Unit=1, Type=17,  Switchtype=17).Create()
Options = {"LevelActions": "|| ||",
                   "LevelNames": "Off|Video|Music|TV Shows|Live TV",
                   "LevelOffHidden": "false",
                   "SelectorStyle": "1"}
        Domoticz.Device(Name="Source", Unit=2, \
                        TypeName="Selector Switch", Options=Options).Create()
        Domoticz.Device(Name="Volume", Unit=3, \
                        Type=244, Subtype=73, Switchtype=7, Image=8).Create()
        Domoticz.Device(Name="Playing", Unit=4, \
                        Type=244, Subtype=73, Switchtype=7, Image=12).Create()
        Domoticz.Log("Devices created.")

Device objects in Python are in memory only until they are added to the Domoticz database using the Create function documented below.
|-
|Create
|Parameters: None, acts on current object.
|Creates the device in Domoticz from the object. E.g:
  myDev = Domoticz.Device(Name="Kilo Watts", Unit=16, TypeName="kWh")
  myDev.Create()
  Domoticz.Log("Created device: "+Devices[16].Name+ \
               ", myDev also points to the Device: "+myDev.Name)
or
  Domoticz.Device(Name="Kilo Watts", Unit=16, TypeName="kWh").Create()
  Domoticz.Log("Created device: "+Devices[16].Name)
Successfully created devices are immediately added to the Devices dictionary.
|-
|Update
|Updates the current values in Domoticz.
{| class="wikitable"
|-
!Parameter
!Description
|-
|nValue
|Mandatory.
The Numerical device value
|-
|sValue
|Mandatory.
The string device value
|-
|Image
|Optional.
Numeric custom image number
|-
|SignalLevel
|Optional.
Device signal strength, default 12
|-
|BatteryLevel
|Optional.
Device battery strength, default 255
|-
|Options
|Optional.
Dictionary of device options, default is empty {}
|-
|TimedOut
|Optional.
Numeric field where 0 (false) is not timed out and other value marks the device as timed out, default is 0.<br />Timed out devices show with a red header in the Domoticz web UI.
|-
|Name
|Optional.<br>Is appended to the Hardware name to set the initial Domoticz Device name.<br>This should not be used in Python because it can be changed in the Web UI.
|-
|TypeName
|Optional.<br>Common device types, this will set the values for Type, Subtype and Switchtype.<br>See [[#Available_Device_Types]]
|-
|Type
|Optional.<br>Directly set the numeric Type value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Subtype
|Optional.<br>Directly set the numeric Subtype value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Switchtype
|Optional.<br>Directly set the numeric Switchtype value. Should only be used if the Device to be created is not supported by TypeName.
|-
|Used
|Optional<br>
Values<br>
0 (default) Unused<br>
1 Used.<br>Set the Device Used field. Used devices appear in the appropriate tab(s), unused devices appear only in the Devices page and must be manually marked as Used.
|-
|Description
|Optional<br>
|-
|Color
|Optional<br>
Current color, see documentation of [[Developing a Python plugin#Callbacks|onCommand callback]] for details on the format.
|-
|SuppressTriggers
|Optional<br>
Default: False
Boolean flag that allows device attributes to be updated without notifications, scene or MQTT, event triggers. nValue and sValue are not written to the database and will be overwritten with current database values.
|-
|}
|Both positional and named parameters are supported.
E.g
  Devices[1].Update(1,”10”)
  Devices[Unit].Update(nValue=nValue, sValue=str(sValue), SignalLevel=5, Image=8)
|-
|Delete
|Parameters: None, acts on current object.
|Deletes the device in Domoticz.E.g:
  Devices[1].Delete()
or
  myDev = Devices[2]
  myDev.Delete()
Deleted devices are immediately removed from the Devices dictionary but local instances of the object are unchanged.
|-
|Refresh
|Parameters: None.
|Refreshes the values for the device from the Domoticz database.
Not normally required because device values are updated when callbacks are invoked.
|-
|Touch
|Parameters: None.
|Updates the Device's 'last seen' time and nothing else. No events or notifications are triggered as a result of touching a Device.
After the call the Device's LastUpdate field will reflect the new value.
|-
|}
|-
|}

===Available Device Types===

Filling is in progress, table doesn't contain full available list yet. Look at the [[Domoticz API/JSON URL's|Domoticz API/JSON page section Update Devices]] for more background information.

{| class="wikitable"
! colspan="2" |Type
! colspan="2" |Subtype
! rowspan="2" |TypeName
! rowspan="2" |Description
|-
!ID
!Name
!ID
!Name
|-
|17||Lighting 2|| ||
| ||Behaves the same as Light/Switch,<br>''Preferable to use Type 244 instead''
|-
|80||Temp||5||LaCrosse TX3
|Temperature||Temperature sensor
|-
|81||Humidity||1||LaCrosse TX3
|Humidity||Humidity sensor
|-
|82||Temp+Hum||1||LaCrosse TX3
|Temp+Hum||Temperature + Humidity sensor
|-
|84
|Temp+Hum+Baro
|1
|THB1 - BTHR918, BTHGN129
|Temp+Hum+Baro
| rowspan="3" |Temperature + Humidity + Barometer sensor<br>
Device.Update(nValue, sValue)<br>
nValue is always 0,<br>
sValue is string with values separated by semicolon: Temperature;Humidity;Humidity Status;Barometer;Forecast<br>
Humidity status: 0 - Normal, 1 - Comfort, 2 - Dry, 3 - Wet<br>
Forecast: 0 = Heavy Snow, 1 = Snow, 2 = Heavy Rain, 3 = Rain, 4 = Cloudy, 5 = Some Clouds, 6 = Sunny, 7 = Unknown, 8 = Unstable, 9 = Stable
|-
| rowspan="2" |
| rowspan="2" |
|2
|THB2 - BTHR918N, BTHR968
|
|-
|16
|Weather Station
|
|-
|85||Rain||1||
|Rain||Rain sensor (sValue: "<RainLastHour_mm*100>;<Rain_mm>", Rain_mm is everincreasing counter)
|-
| rowspan="2" |86||Wind||1||
|Wind||Wind sensor (sValue: "<WindDirDegrees>;<WindDirText>;<WindAveMeterPerSecond*10>;<WindGustMeterPerSecond*10>;<Temp_c>;<WindChill_c>")
|-
|
|4
|
|Wind+Temp+Chill
|
|-
|87||UV||1||
|UV||UV sensor (sValue: "<UV>;<Temp>")
|-
|89||Current||1||
|Current/Ampere||Ampere (3 Phase)
|-
|93||Scale||1||Weight
| ||
|-
|113||Counter||0||
| ||RFXMeter


Additional attribute Switchtype is applicable
{| class="wikitable"
|-
!ID
!Name
|-
|0
|Energy
|-
|1
|Gas
|-
|2
|Water
|-
|3
|Counter
|-
|4
|Energy Generated
|-
|5
|Time
|}
|-
|241
|Color Switch
|1
|RGBW
|
|RGB + white, either RGB or white can be lit
|-
| rowspan="6" |
| rowspan="6" |
|2
|RGB
|
|
|-
|3
|White
|
|Monochrome white
|-
|4
|RGBWW
|
|RGB + cold white + warm white, either RGB or white can be lit
|-
|6
|RGBWZ
|
|Like RGBW, but allows combining RGB and white
|-
|7
|RGBWWZ
|
|Like RGBWW, but allows combining RGB and white
|-
|8
|Cold white + Warm white
|
|
|- 
|242||Thermostat||1||Setpoint
|Set Point||
|-
|243
|General
|1
|Visibility
|Visibility
|
|-
| rowspan="17" |
| rowspan="17" |
|2
|Solar Radiation
|Solar Radiation
|sValue: "float"
|-
|3||Soil Moisture
|Soil Moisture||
|-
|4||Leaf Wetness
|Leaf Wetness||
|-
|6||Percentage
|Percentage||
|-
|8||Voltage
|Voltage||
|-
|9||Pressure
|Pressure||
|-
|19||Text
|Text||
|-
|22||Alert
|Alert||
|-
|23||Ampere (1 Phase)
|Current (Single)||
|-
|24||Sound Level
|Sound Level||
|-
|26||Barometer
|Barometer||nValue: 0, sValue: "pressure;forecast"<br>Forecast:<br>0 - Stable<br>1 - Sunny<br>2 - Cloudy<br>3 - Unstable<br>4 - Thunderstorm<br>5 - Unknown<br>6 - Cloudy/Rain
|-
|27||Distance
|Distance||
|-
|28||Counter Incremental
|Counter Incremental||Incremental counter used to measure energy (used or produced), gas, water. Does not compute the power or flow rate.  nValue=0, sValue=INCREMENT_VALUE  to increase counter by INCREMENT_VALUE, or sValue=NEGATIVE_VALUE to reset the counter.


Additional attribute Switchtype is applicable
{| class="wikitable"
|-
!ID
!Name
|-
|0
|Energy
|-
|1
|Gas
|-
|2
|Water
|-
|3
|Counter
|-
|4
|Energy Generated
|-
|5
|Time
|}
|-
|29||kWh
|kWh||Electric (Instant+Counter)<br> nValue should be zero<br>
sValue are two numbers separated by semicolon like "123;123456" The first number is the actual power in Watt, the second number is actual energy in kWh. When the option "EnergyMeterMode" is set to "Calculated", the second value is ignored<br>
Optional argument Options can set the EnergyMeterMode.<br>
Use Options={'EnergyMeterMode': '1' } to set energyMeterMode to "Calculated". Default is "From Device"
|-
|30||Waterflow
|Waterflow||
|-
|31||Custom Sensor
|Custom||nValue: 0, sValue: "floatValue", Options: {'Custom': '1;<axisUnits>'}
|-
|33||Managed counter
| ||nValue is always 0<br>
sValue must be 2 semicolon separated values to update Dashboard, for instance "123456;78", 123456 being the absolute counter value, 78 being the usage (Wh). Set counter to -1 if you can't know the counter absolute value<br>
sValue must 3 semicolon separated values, the last value being a date ("%Y-%m-%d" format) to update last week/month/year history, for instance "123456;78;2019-09-24"<br>
sValue must 3 semicolon separated values, the last value being a date a space and a time ("%Y-%m-%d %H:%M:%S" format) to update last days history, for instance "123456;78;2019-10-03 14:55:00"<br>
<br>
Additional attribute Switchtype is applicable
{| class="wikitable"
|-
!ID
!Name
|-
|0
|Energy
|-
|1
|Gas
|-
|2
|Water
|-
|3
|Counter
|-
|4
|Energy Generated
|-
|5
|Time
|}
|-
| rowspan="2" |244
| rowspan="2" |Light/Switch
|62
|Selector Switch
|Selector Switch
| rowspan="2" |Additional attribute Switchtype is applicable
{| class="wikitable"
|-
!ID
!Name
!TypeName
!Description
|-
|0
|On/Off
|
|
|-
|1
|Doorbell
|
|
|-
|2
|Contact
|Contact
|Statuses:<br>Open: nValue = 1<br>Closed: nValue = 0
|-
|3
|Blinds
|
|
|-
|4
|X10 Siren
|
|
|-
|5
|Smoke Detector
|
|
|-
|6
|
|
|
|-
|7
|Dimmer
|Dimmer
|
|-
|8
|Motion Sensor
|Motion
|Statuses:<br>Motion: nValue = 1<br>Off: nValue = 0
|-
|9
|Push On Button
|Push On
|
|-
|10
|Push Off Button
|Push Off
|
|-
|11
|Door Contact
|
|Statuses:<br>Open: nValue = 1<br>Closed: nValue = 0
|-
|12
|Dusk Sensor
|
|
|-
|13
|Blinds Percentage
|
|
|-
|14
|Venetian Blinds US
|
|
|-
|15
|Venetian Blinds EU
|
|
|-
|16
|
|
|
|-
|17
|Media Player
|
|
|-
|18
|Selector
|
|
|-
|19
|Door Lock
|
|
|-
|20
|Door Lock Inverted
|
|
|-
|21
|Blinds Percentage With Stop
|
|
|-
|22
|
|
|
|}
|-
|73
|Switch
|Check:
Additional attribute Switchtype
|-
|246||Lux||1||Lux
|Illumination||Illumination (sValue: "float")
|-
|247||Temp+Baro||1||LaCrosse TX3
| ||Temperature + Barometer sensor
|-
|248||Usage||1||Electric
|Usage||
|-
|249||Air Quality||1||
|Air Quality||
|-
|250
|P1 Smart Meter
|1
|Energy
|
|
|-
|251
|P1 Smart Meter
|2
|Gas
|Gas
|
|}<br />
====Note on counters====
Usually, counters are updated daily to feed week/month/year history log views. Starting version 4.11774, if you want to disable that behavior to control everything from an external script or a plugin (already the case for managed counter), you can set the "DisableLogAutoUpdate" device option to "true", for instance in a Python plugin:
 Domoticz.Device(Name="MyCounter", Unit=1, Type=0xfa, Subtype=0x01, Options={"DisableLogAutoUpdate"<span> </span>: "true").Create()


Starting version 4.11774, you can too directly insert data in in history log. Set the "AddDBLogEntry" device option to "true", for instance in a Python plugin:
 Domoticz.Device(Name="MyCounter", Unit=1, Type=0xfa, Subtype=0x01, Options={"AddDBLogEntry"<span> </span>: "true").Create()
Then, depending on counters, you can insert values in history log. For most counters:
 Devices[Unit].Update(nValue=nValue, sValue="COUNTER;USAGE;DATE")

*COUNTER = absolute counter energy (Wh)
*USAGE = energy usage in Watt-hours (Wh)
*DATE = date with %Y-%m-%d format (for instance 2019-09-24) to put data in last week/month/year history log, or "%Y-%m-%d %H:%M:%S" format (for instance 2019-10-03 14:00:00) to put data in last days history log


For multi meters (P1 Smart Meter, CM113, Electrisave and CM180i):
 Devices[Unit].Update(nValue=nValue, sValue="USAGE1;USAGE2;RETURN1;RETURN2;CONS;PROD;DATE")

*USAGE1= energy usage meter tariff 1, This is an incrementing counter
*USAGE2= energy usage meter tariff 2, This is an incrementing counter
*RETURN1= energy return meter tariff 1, This is an incrementing counter
*RETURN2= energy return meter tariff 2, This is an incrementing counter
*CONS= actual usage power (Watt)
*PROD= actual return power (Watt)
*DATE = date with %Y-%m-%d format (for instance 2019-09-24) to put data in last week/month/year history log, or "%Y-%m-%d %H:%M:%S" format (for instance 2019-10-03 14:00:00) to put data in last days history log

or
 Devices[Unit].Update(nValue=nValue, sValue="USAGE1;USAGE2;RETURN1;RETURN2;CONS;PROD;COUNTER1;COUNTER2;COUNTER3;COUNTER4;DATE")
as previously, plus absolute counter values

for counters with custom units: Use RFXCom customer counter, e.g.
Domoticz.Unit(Name=name, Unit=unit, Type=113, Switchtype=3, Options={"ValueQuantity": "Custom", "ValueUnits": "customunit"}).Create()
 
<br />

===Connections===
Connection objects allow plugin developers to connect to multiple external sources using a variety of transports and protocols simultaneously.  By using the plugin framework to handle connectivity Domoticz will remain responsive no matter how many connections the plugin handles.<br>
Connections remain active only while they are in scope in Python after that Domoticz will actively disconnect them so plugins should store Connections that they want to keep in global or class variables.
{| class="wikitable"
|-
!Function
!Description/Attributes
|-
|__init__
|Defines the connection type that will be used by the object.  
{| class="wikitable"
|-
!Parameter
!Description
|-
|Name
|Required.<br>
Name of the Connection.  For incoming connections Domoticz will assign a unique name.
|-
|Transport
|Required.<br>
Valid values:

*TCP/IP: Connect over an IP network then send or receive messages<br />

See [https://github.com/domoticz/domoticz/blob/development/plugins/examples/HTTP.py HTTP/HTTPS Client example] that uses GET and POST over HTTP or HTTPS<br />
See [https://github.com/domoticz/domoticz/blob/development/plugins/examples/HTTP%20Listener.py HTTP Listener example] that acts as a lightweight webserver

*TLS/IP: Connect over an IP network using TLS security then send or receive messages<br />

See [https://github.com/domoticz/domoticz/blob/development/plugins/examples/HTTP.py HTTP/HTTPS Client example]

*UDP/IP: Send or recieve UDP messages, useful for discovering hardware on a network. <br />

See [https://github.com/domoticz/domoticz/blob/development/plugins/examples/UDP%20Discovery.py UDP Discovery example]<br />
See [https://github.com/domoticz/domoticz/blob/development/plugins/examples/Kodi.py UDP broadcast example] (onNotification function)

*ICMP/IP: Send or recieve ICMP messages, useful for discovering or pinging hardware on a network.<br />

See [https://github.com/domoticz/domoticz/blob/development/plugins/examples/Pinger.py Pinger example]

*Serial: Connect to serial ports, see [https://github.com/domoticz/domoticz/blob/development/plugins/examples/RAVEn.py RAVEn power monitoring example]
|-
|Protocol
|Required.
The protocol that will be used to talk to the external hardware. This is used to allow Domoticz to break incoming data into single messages to pass to the plugin.
Valid values:

*None (default)
*Line
*JSON
*XML
*HTTP
*HTTPS
*WS (Web Socket)
*WSS
*MQTT
*MQTTS
*ICMP
|-
|Address
|Optional.<br>TCP/IP or UDP/IP Address or SerialPort to connect to.
|-
|Port
|Optional.<br>TCP/IP & UDP/IP connections only, string containing the port number.
|-
|Baud
|Optional.<br>Serial connections only, the required baud rate.<br>
Default: 115200
|-
|}
This allows Domoticz to make connections on behalf of the plugin.
E.g: 
<syntaxhighlight lang="python">
 myConn = Domoticz.Connection(Name="JSON Connection", Transport="TCP/IP", Protocol="JSON", Address=Parameters["Address"], Port=Parameters["Port"])
 myConn.Connect()
 secureConn = Domoticz.Connection(Name="Secure Connection", Transport="TCP/IP", Protocol="HTTPS", Address=Parameters["Address"], Port="443")
 secureConn .Connect()
 mySerialConn = Domoticz.Connection(Name="Serial Connection", Transport="Serial", Protocol="XML", Address=Parameters["SerialPort"], Baud=115200)
 mySerialConn.Connect()
</syntaxhighlight>
Both positional and named parameters are supported. 
|-
|Name
|Returns the Name of the Connection.
|-
|Address
|Return/Set the Address associated with the Connection.
|-
|Port
|Return/Set the Port associated with the Connection.
|-
|Baud
|Returns the Baud Rate of the Connection.
|-
|Target
|Get or Set the event target for the Connection. See Target parameter description on Connect and Listen for more detail.
|-
|Parent
|Normally 'None' but for incoming connections this will hold the Connection object that is 'Listening' for the connection.
|-
|Connecting
|Parameters: None.
Returns True if a connection has been requested but has yet to complete (or fail), otherwise False.
|-
|Connected
|Parameters: None.
Returns True if the connection is connected or listening, otherwise False.
|-
|Connect
|Initiate a connection to a external hardware using transport details.
{| class="wikitable"
|+
!Parameter
!Description
|-
|Target
|Optional.
A Python object where the Connection's callbacks should be sent rather than sending them to the module level functions.

Applies to: onConnect, onMessage, onTimeout and onDisconnect.

This functionality is designed to allow plugin authors more flexibility and to segregate connection based code from other callbacks.
|-
|Timeout
|Optional.
Time in milliseconds to wait for either a connection to occur or data to arrive on a Connection before calling the onTimeout callback.

Applies to Serial, TCP and Secure TCP connections.

Valid values:

0 - No timeout

250 or greater - the milliseconds to wait before queuing an onTimeout event
|}
Connect returns immediately and the results of the actual connection operation will be returned via the onConnect callback.  If the address set via the Transport function translates to multiple endpoints they will all be tried in order until the connection succeeds or the list of endpoints is exhausted.<br>
The Connect operation works with connection based transports.<br>
Note that I/O operations immediately after a Connect (but before onConnect is called) may fail because the connection is still in progress so is technically not connected.
|-
|Listen
|Start listening on specifed Port using the specified TCP/IP, UDP/IP or ICMP/IP transport.
{| class="wikitable"
|+
!Parameter
!Description
|-
|Target
|Optional.
A Python object where the Connection's callbacks should be sent rather than sending them to the module level functions.

Applies to: onConnect, onMessage, onTimeout and onDisconnect.

This functionality is designed to allow plugin authors more flexibility and to segregate connection based code from other callbacks.
|}
Connection objects will be created for each client that connects and onConnect will be called.<br>
If a Listen request is unsuccessful the plugin's onConnect callback will be called with failure details. If it is successful then onConnect will be called when incoming Connections are made.
E.g:
<syntaxhighlight lang="python">
 self.httpServerConn = Domoticz.Connection(Name="WebServer", Transport="TCP/IP", Protocol="HTTP", Port=Parameters["Port"])
 self.httpServerConn.Listen()

 self.BeaconConn = Domoticz.Connection(Name="Beacon", Transport="UDP/IP", Address="239.255.255.250", Port="1900")
 self.BeaconConn.Listen()
</syntaxhighlight>Listening on UDP/IP traffic is normally unicast (point to point) but there are some special cases based on the specified Address:

*Addresses between "224.x.x.x" and "239.x.x.x" will be treated as multicast listeners
*Address "255.255.255.255" will be treated as a broadcast listener
|-
|Send
|Send the specified message to the external hardware.
{| class="wikitable"
|-
!Parameter
!Description
|-
|Message
|Mandatory.<br>Message text to send.<br>
For simple Protocols this can be of type String, ByteArray or Bytes.<br>
For structured Protocols (such as HTTP) it should be a Dictionary.
|-
|Delay
|Optional.<br>Number of seconds to delay message send.<br>Note that Domoticz will send the message sometime after this period. Other events will be processed in the intervening period so delayed sends will be processed out of order. This feature may be useful during delays when physical devices turn on.
|-
|}
Both positional and named parameters are supported.<br>
E.g: 
<syntaxhighlight lang="python">
 myConn.Send(Message=myMessage, Delay=4)

 Headers = {"Connection": "keep-alive", "Accept": "Content-Type: text/html; charset=UTF-8"}
 myConn.Send({"Verb":"GET", "URL":"/page.html", "Headers": Headers})

 postData = "param1=value&param2=other+value"
 myHttpConn.Send({'Verb':'POST', 'URL':'/MediaRenderer/AVTransport/Control', 'Data': postData})

 responseData = "<!doctype html><html><head></head><body><h1>Successful GET!!!</h1><body></html>"
 self.httpClientConn.Send({"Status":"200 OK", "Headers": {"Connection": "keep-alive", "Accept": "Content-Type: text/html; charset=UTF-8"}, "Data": responseData })

 udpBcastConn = Domoticz.Connection(Name="UDP Broadcast Connection", Transport="UDP/IP", Protocol="None", Address="224.0.0.1", Port="9777")
 udpBcastConn.Send("Hello World!")
</syntaxhighlight>
|-
|Disconnect
|Parameters: None.
Terminate the connection to the external hardware for the connection.<br />
Disconnect also terminates listening connections for all transports (including connectionless ones e.g UDP/IP).
|}

===Images===
Developers can ship custom images with plugins in the standard Domoticz format as described here: [http://www.domoticz.com/wiki/Custom_icons_for_webinterface#Creating_simple_home_made_icons].  Resultant zip file(s) should be placed in the folder with the plugin itself
{| class="wikitable"
|-
!
!Description
|-
|Key
|Base.
The base value as specified in icons.txt file in custom image zip file.
{| class="wikitable"
|-
!Function
!Description
|-
|ID
|Image ID in CustomImages table
|-
|Name
|Name as specified in upload file
|-
|Base
|This MUST start with (or be) the plugin key as defined in the XML definition.  If not the image will not be loaded into the Images dictionary.
|-
|Description
|Description as specified in upload file
|-
|}
|-
|Methods
|Per image calls into Domoticz to manipulate specific images
{| class="wikitable"
|-
!Function
!Description
|-
|__init__
|
{| class="wikitable"
|-
!Parameter
!Description
|-
|Filename
|Mandatory.<br>The zip file name containing images formatted for Domoticz.
|}
|Both positional and named parameters are supported.<br>Creates a new image object in Python.<br>
E.g To allow users to select from three device icons in the hardware page create three zip files with different bases:
  myPlugin Icons.zip -> icons.txt
  '''myPlugin''';Plugin;Plugin Description
and
  myPlugin1 Icons.zip -> icons.txt
  '''myPluginAlt1''';Plugin;Plugin Description
and
  myPlugin2 Icons.zip -> icons.txt
  '''myPluginAlt2''';Plugin;Plugin Description
and an XML parameter declaration like:<syntaxhighlight lang="XML">
  <plugin key="'''myPlugin'''" name="myPlugin cool hardware" version="1.5.0" >
    <params>
        <param field="Mode1" label="Icon" width="100px">
            <options>
                <option label="Blue"  value="<nowiki>'''myPlugin'''" default="true" />
                <option label="Black" value="'''myPluginAlt1'''"/>
                <option label="Round" value="'''myPluginAlt2'''</nowiki>"/>
            </options>
        </param>
        ...</syntaxhighlight>
then load the images into Domoticz at startup if they are not there and set the <syntaxhighlight lang="python3">
    def onStart(self):
        if ("'''myPlugin'''"     not in Images):
            Domoticz.Image('myPlugin Icons.zip').Create()
        if ("'''myPluginAlt1'''" not in Images):
            Domoticz.Image('myPlugin1 Icons.zip').Create()
        if ("'''myPluginAlt2'''" not in Images):
            Domoticz.Image('myPlugin2 Icons.zip').Create()
        
        if (1 in Devices) and (Parameters["Mode1"] in Images):
            if (Devices[1].Image != Images[Parameters["Mode1"]].ID):
                Devices[1].Update(nValue=Devices[1].nValue, \ 
                                  sValue=str(Devices[1].sValue), \
                                  Image=Images[Parameters["Mode1"]].ID)</syntaxhighlight>

Image objects in Python are in memory only until they are added to the Domoticz database using the Create function documented below.
|-
|Create
|Parameters: None, acts on current object.
|Creates the image in Domoticz from the object. E.g:<syntaxhighlight lang="python3">
  myImg = Domoticz.Image(Filename="Plugin Icons.zip")
  myImg.Create()</syntaxhighlight>
or<syntaxhighlight lang="python3">
  Domoticz.Image(Filename="Plugin Icons.zip").Create()</syntaxhighlight>
Successfully created images are immediately added to the Images dictionary.
|-
|Delete
|Parameters: None, acts on current object.
|Deletes the image in Domoticz.E.g:<syntaxhighlight lang="python3">
  Images['myPlugin'].Delete()</syntaxhighlight>
or<syntaxhighlight lang="python3">
  myImg = Images['myPlugin']
  myImg.Delete()</syntaxhighlight>
Deleted images are immediately removed from the Images dictionary but local instances of the object are unchanged.
|-
|}
|-
|}

==Callbacks==
Plugins are event driven.  If a callback is not used by a plugin then it should '''not''' be present in the plugin.py file, Domoticz will not attempt to call callbacks that are not defined.

Domoticz will notify the plugin when certain events occur through a number of callbacks, these are:
{| class="wikitable"
|-
!Callback
!Description
|-
|onStart
|Parameters: None.
Called when the hardware is started, either after Domoticz start, hardware creation or update.
|-
|onConnect
|Parameters: Connection, Status, Description
Called when connection to remote device either succeeds or fails, or when a connection is made to a listening Address:Port. Connection is the Domoticz Connection object associated with the event.  Zero Status indicates success.
If Status is not zero then the Description will describe the failure.<br>
This callback is not called for connectionless Transports such as UDP/IP.
|-
|onMessage
|Parameters: Connection, Data.
Called when a single, complete message is received from the external hardware (as defined by the Protocol setting).
This callback should be used to interpret messages from the device and set the related Domoticz devices as required.<br>
Connection is the Domoticz Connection object associated with the event.<br>
Data is normally a ByteArray except where the Protocol for the Connection has structure (such as HTTP or ICMP), in that case Data will be a Dictionary containing Protocol specific details such as Status and Headers.
|-
|onNotification
|Parameters: Name, Subject, Text, Status, Priority, Sound, ImageFile.
Called when any Domoticz device generates a notification.  Name parameter is the device that generated the notification, the other parameters contain the notification details.  Hardware that can handle notifications should be notified as required.
|-
|onCommand
|'''Legacy Plugin Framework ('import Domoticz')'''
*Parameters: Unit, Command, Level, Color

'''Extended Plugin Framework ('import DomoticzEx')'''

*Parameters: DeviceID, Unit, Command, Level, Color
*Can be overidden or Device or Unit objects, see docmentation elsewhere in this page


Called when a command is received from Domoticz.
The Unit parameters matches the Unit specified in the device definition and should be used to map commands to Domoticz devices. Level is normally an integer but may be a floating point number if the Unit is linked to a thermostat device.
This callback should be used to send Domoticz commands to the external hardware.
The Color parameter is valid if Command is "Set Color" and is a JSON serialized Domoticz color object.

Domoticz color format:<syntaxhighlight lang="python3">
                      ColorMode {
                      	ColorModeNone = 0,   // Illegal
                      	ColorModeWhite = 1,  // White. Valid fields: none
                      	ColorModeTemp = 2,   // White with color temperature. Valid fields: t
                      	ColorModeRGB = 3,    // Color. Valid fields: r, g, b.
                      	ColorModeCustom = 4, // Custom (color + white). Valid fields: r, g, b, cw, ww, depending on device capabilities
                      	ColorModeLast = ColorModeCustom,
                      };
                      
                      Color {
                      	ColorMode m;
                      	uint8_t t;     // Range:0..255, Color temperature (warm / cold ratio, 0 is coldest, 255 is warmest)
                      	uint8_t r;     // Range:0..255, Red level
                      	uint8_t g;     // Range:0..255, Green level
                      	uint8_t b;     // Range:0..255, Blue level
                      	uint8_t cw;    // Range:0..255, Cold white level
                      	uint8_t ww;    // Range:0..255, Warm white level (also used as level for monochrome white)
                      }</syntaxhighlight>
|-
|onHeartbeat
|Called every 'heartbeat' seconds (default 10) regardless of connection status.
Heartbeat interval can be modified by the Heartbeat command: <syntaxhighlight lang="python3">
Domoticz.Heartbeat(30) 
</syntaxhighlight> to set the Heartbeat interval to 30s.
Allows the Plugin to do periodic tasks including request reconnection if the connection has failed.
<br /><br />'''Warning:''' Setting this interval to greater than 30 seconds will cause a 'thread seems to have ended unexpectedly' message to be written to the log file every 30 seconds. The plugin will function correctly but this message can not be suppressed because it is a standard warning from Domoticz that a piece of hardware may have stopped responding.<br />If a plugin wants to heartbeat every 100 seconds it should be coded with the heartbeat interval set to 10, 20 or 25 seconds and only take action every 6th, 5th or 4th time the callback is invoked.
|-
|onTimeout
|Parameters: Connection
Called in response to a connection or read timeout.  


Timeout interval set configured for the connection by the 'Timeout' parameter optionally specified on the Connect method.
|-
|onDisconnect
|Parameters: Connection
Called after the remote device is disconnected, Connection is the Domoticz Connection object associated with the event<br>
This callback is not called for connectionless Transports such as UDP/IP.
|-
|onStop
|Called when the hardware is stopped or deleted from Domoticz.
|-
|onDeviceAdded
|'''Legacy Plugin Framework ('import Domoticz')'''
*Parameters: Unit

'''Extended Plugin Framework ('import DomoticzEx')'''

*Parameters: DeviceID, Unit
*Can be overidden or Device or Unit objects, see docmentation elsewhere in this page


Called just after a new device has been added to the hardware, for example by a Domoticz Web API call.<br />onDeviceAdded is not called when the plugin adds a device
|-
|onDeviceModified
|'''Legacy Plugin Framework ('import Domoticz')'''
*Parameters: Unit

'''Extended Plugin Framework ('import DomoticzEx')'''

*Parameters: DeviceID, Unit
*Can be overidden or Device or Unit objects, see docmentation elsewhere in this page


Called just after a device owned by the plugin has been modifed, for example by a Domoticz Web API call.<br />onDeviceModified is not called when the plugin initiated the change e.g. by calling Update.
|-
|onDeviceRemoved
|'''Legacy Plugin Framework ('import Domoticz')'''
*Parameters: Unit

'''Extended Plugin Framework ('import DomoticzEx')'''

*Parameters: DeviceID, Unit
*Can be overidden or Device or Unit objects, see docmentation elsewhere in this page


Called just before a device owned by the plugin is been removed, for example by a Domoticz Web API call.<br />onDeviceRemoved is not called when the plugin removed the device by calling Delete.
|-
|onSecurityEvent
|'''Legacy Plugin Framework ('import Domoticz')'''
*Parameters: Unit, Level, Description

'''Extended Plugin Framework ('import DomoticzEx')'''

*Parameters: DeviceID, Unit, Level, Description
*Can be overidden or Device or Unit objects, see docmentation elsewhere in this page
|}

==C++ Callable API==
Importing the ‘Domoticz’ module in the Python code exposes functions that plugins can call to perform specific functions. All functions are non-blocking and return immediately. 
{| class="wikitable"
|-
!Function
!Description/Attributes
|-
|Debug
|Parameters: String
Write a message to Domoticz log only if verbose logging is turned on.
|-
|Log
|Parameters: String.
Write a message to Domoticz log
|-
|Status
|Parameters: String.
Write a status message to Domoticz log
|-
|Error
|Parameters: String
Write an error message to Domoticz log
|-
|Debugging
|Parameters: Integer.
Set logging level and type for debugging.
{| class="wikitable"
|-
!Value
!Meaning
|-
|0
|None. All Python and framework debugging is disabled.
|-
|1
|All. Very verbose log from plugin framework and plugin debug messages.
|-
|2
|Mask value. Shows messages from Plugin Domoticz.Debug() calls only.
|-
|4
|Mask Value. Shows high level framework messages only about major the plugin.
|-
|8
|Mask Value. Shows plugin framework debug messages related to Devices objects.
|-
|16
|Mask Value. Shows plugin framework debug messages related to Connections objects.
|-
|32
|Mask Value. Shows plugin framework debug messages related to Images objects.
|-
|64
|Mask Value. Dumps contents of inbound and outbound data from Connection objects.
|-
|128
|Mask Value. Shows plugin framework debug messages related to the message queue.
|-
|}
Mask values can be added together, for example to see debugging details around the plugin and its objects use: <code>Domoticz.Debugging(62) # 2+4+8+16+32</code>
|-
|Heartbeat
|Parameters: Integer (Optional).
Set the heartbeat interval in seconds if parameter supplied (initial value 10 seconds). 

Values greater than 30 seconds will cause a message to be regularly logged about the plugin not responding.  The plugin will actually function correctly with values greater than 30 though.

Returns current Heartbeat value in seconds.
|-
|Notifier
|Parameters: Name, String.
Informs the plugin framework that the plugin's external hardware can comsume Domoticz Notifications.<br>
When the plugin is active the supplied Name will appear as an additional target for Notifications in the standard Domoticz device notification editing page.  The plugin framework will then call the onNotification callback when a notifiable event occurs.
|-
|Trace
|Parameters: N/A, Boolean. Default: False.
When True, Domoticz will log line numbers of the lines being executed by the plugin. Calling Trace again with False will suppress line level logging. Usage:<syntaxhighlight lang="python3">
  def onHeartBeat():
    Domoticz.Trace(True)
    Domoticz.Log("onHeartBeat called")
    ...
    Domoticz.Trace(False)</syntaxhighlight>
|-
|Configuration
|Parameters: Dictionary (Optional).
Returns a dictionary containing the plugin's configuration data that was previously stored. If a Dictionary paramater is supplied the database will be updated with the new configuration data.<br>
Values in the dictionary can be of types: String, Long, Float, Boolean, Bytes, ByteArray, List or Dictionary. Tuples can be specified but will be returned as a List.<br>
Configuration should not be confused with the Parameters dictionary.  Parameters are set via the Hardware page and are read-only to the plugin, Configuration allows the plugin store structured data in the database rather than writing files or creating Domoticz variables to hold it.<br>
Usage:<syntaxhighlight lang="python3">
    # Configuration Helpers
    def getConfigItem(Key=None, Default={}):
        Value = Default
        try:
            Config = Domoticz.Configuration()
            if (Key != None):
                Value = Config[Key] # only return requested key if there was one
            else:
                Value = Config      # return the whole configuration if no key
        except KeyError:
            Value = Default
        except Exception as inst:
            Domoticz.Error("Domoticz.Configuration read failed: '"+str(inst)+"'")
        return Value
        
    def setConfigItem(Key=None, Value=None):
        Config = {}
        try:
            Config = Domoticz.Configuration()
            if (Key != None):
                Config[Key] = Value
            else:
                Config = Value  # set whole configuration if no key specified
            Config = Domoticz.Configuration(Config)
        except Exception as inst:
            Domoticz.Error("Domoticz.Configuration operation failed: '"+str(inst)+"'")
        return Config</syntaxhighlight>
|-
|Register
|Parameters: Device (Class Object), Optional Unit (Class Object)
Must be positioned outside of any class code so that the Plugin Framework can process it during the module import.

Changes the object type that Domoticz uses to represent Devices (and Units where applicable) to a user defined type.  The specified class must inherit (directly or indirectly) from the default class type and the underlying object initialisation should be invoked (via super().__init__) to ensure there are no unexpected behaviours.


The purpose of this function is to support encapsulation and code segregation.



When using the legacy Plugin Framework the following code will cause Domoticz to populate the Devices dictionary with objects of type 'myDevice' rather than the default objects of type 'Domoticz.Device':<syntaxhighlight lang="python3">
import Domoticz
from Domoticz import Device

class myDevice(Domoticz.Device):
    def __init__(self, Name, Unit, TypeName="", Type=0, Subtype=0, Switchtype=0, Image=0, Options="", Used=0, DeviceID="", Description=""):
        super().__init__(Name=Name, Unit=Unit, TypeName=TypeName, Type=Type, Subtype=Subtype, Switchtype=Switchtype, Image=Image, Options=Options, Used=Used, DeviceID=DeviceID, Description=Description)
        # This code will run prior to onStart for existing data
        self.localVariable = "Hello"

    def localFunction(self):
        Domoticz.Log("Function called")

    # Callbacks cannot be overridden

Domoticz.Register(Device=myDevice)
</syntaxhighlight>

When using the extended Plugin Framework the following code will cause Domoticz to populate the Devices and Units dictionaries with objects of type 'myDevice' and 'myUnit' respectively rather than the default objects of type 'DomoticzEx.Device' and 'DomoticzEx.Unit:<syntaxhighlight lang="python3">
import DomoticzEx as Domoticz
from DomoticzEx import Device, Unit

class myUnit(Domoticz.Unit):
    def __init__(self, Name, DeviceID, Unit, TypeName="", Type=0, Subtype=0, Switchtype=0, Image=0, Options="", Used=0, Description=""):
        super().__init__(Name, DeviceID, Unit, TypeName, Type, Subtype, Switchtype, Image, Options, Used, Description)
        # This code will run prior to onStart for existing data
        self.myVar = 0

    # Optionally override onDeviceAdded, onDeviceModified, onDeviceRemoved or onCommand callbacks
    def onDeviceModified(self):
        Domoticz.Log("Unit onDeviceModified")

class myDevice(Domoticz.Device):
    def __init__(self, DeviceID):
        super().__init__(DeviceID)
        # This code will run prior to onStart for existing data
        self.localVariable = "Hello"

    def localFunction(self):
        Domoticz.Log("Function called")

    # Optionally override onDeviceAdded, onDeviceModified, onDeviceRemoved or onCommand callbacks
    def onDeviceModified(self, Unit):
        Domoticz.Log("Device onDeviceModified")

Domoticz.Register(Device=myDevice, Unit=myUnit)
</syntaxhighlight>
|-
|Dump
|Parameters: None.
Dumps current values to the Domoticz log, useful during debugging or exception handling.

Dumped values:

*Current context
*Local variables
*Global variables

Example:<syntaxhighlight lang="python3">
import DomoticzEx as Domoticz
from DomoticzEx import Device, Unit
import sys
import datetime
import json

modeCoolCmd = 'N000001{"SYST": {"OSS": {"MD": "C" } } }'
modeEvapCmd = 'N000001{"SYST": {"OSS": {"MD": "E" } } }'
modeHeatCmd = 'N000001{"SYST": {"OSS": {"MD": "H" } } }'

# Heater, Cooling and Evaporate common functionality
class hvacBase(Domoticz.Connection):
    def __init__(self, DeviceID, Name, Image, Address, Port):
        super().__init__(Name=Name, Transport="TCP/IP", Protocol="None", Address=Address, Port=Port)

        self.isActive = False
        self.DeviceID = DeviceID
        self.DeviceImage = Image
        self.Device = None
        self.reportedMode = DeviceID

    def onConnect(self, Connection, Status, Description):

        if (self != Connection):
            Domoticz.Error("Connection '"+Connection.Name+"' is not the same as '"+self.Name+"'")

        if (Status == 0):
            Domoticz.Log(self.Name+" connected successfully to: "+Connection.Address+":"+Connection.Port)
            for device in Devices:
                Devices[device].TimedOut = 0
        else:
            Domoticz.Log(self.Name+" failed to connect ("+str(Status)+") to: "+Connection.Address+":"+Connection.Port)
            for device in Devices:
                Devices[device].TimedOut = 1

class Heating(hvacBase):
    def __init__(self, DeviceID, Name, Image, Address, Port):
        super().__init__(DeviceID, Name, Image, Address, Port)

    def onMessage(self, Connection, Response):
        if (len(Response) > 10):    # filter out 'HELLO' message
            self.Disconnect()
            jsonPayload = json.loads(Response[7:])
            Domoticz.Dump()

</syntaxhighlight>would dump something like:<syntaxhighlight lang="text">
2021-07-06 15:18:34.296 Rinnai: (Rinnai) Context dump:
2021-07-06 15:18:34.296 Rinnai: (Rinnai) ----> 'Address' '192.168.999.999'
2021-07-06 15:18:34.297 Rinnai: (Rinnai) ----> 'Baud' '-1'
2021-07-06 15:18:34.297 Rinnai: (Rinnai) ----> 'Device' 'DeviceID: 'HGOM', Units: 1'
2021-07-06 15:18:34.298 Rinnai: (Rinnai) ----> 'DeviceID' 'HGOM'
2021-07-06 15:18:34.298 Rinnai: (Rinnai) ----> 'DeviceImage' '15'
2021-07-06 15:18:34.298 Rinnai: (Rinnai) ----> 'Name' 'Heating'
2021-07-06 15:18:34.299 Rinnai: (Rinnai) ----> 'Parent' 'None'
2021-07-06 15:18:34.299 Rinnai: (Rinnai) ----> 'Port' '27847'
2021-07-06 15:18:34.299 Rinnai: (Rinnai) ----> 'Target' 'Name: 'Heating', Transport: 'TCP/IP', Protocol: 'None', Address: '192.168.999.999', Port: '27847', Baud: -1, Timeout: 0, Bytes: 1071, Connected: True, Last Seen: 2021-07-06 15:18:34, Parent: 'None''
2021-07-06 15:18:34.300 Rinnai: (Rinnai) ----> 'isActive' 'True'
2021-07-06 15:18:34.300 Rinnai: (Rinnai) ----> 'reportedMode' 'HGOM'
2021-07-06 15:18:34.300 Rinnai: (Rinnai) Locals dump:
2021-07-06 15:18:34.300 Rinnai: (Rinnai) ----> 'self' 'Name: 'Heating', Transport: 'TCP/IP', Protocol: 'None', Address: '192.168.999.999', Port: '27847', Baud: -1, Timeout: 0, Bytes: 1071, Connected: True, Last Seen: 2021-07-06 15:18:34, Parent: 'None''
2021-07-06 15:18:34.301 Rinnai: (Rinnai) ----> 'Connection' 'Name: 'Heating', Transport: 'TCP/IP', Protocol: 'None', Address: '192.168.999.999', Port: '27847', Baud: -1, Timeout: 0, Bytes: 1071, Connected: True, Last Seen: 2021-07-06 15:18:34, Parent: 'None''
2021-07-06 15:18:34.301 Rinnai: (Rinnai) ----> 'Response' 'b'N000000[{"SYST": {"CFG": {"MTSP": ... "MT": "231" } } }]''
2021-07-06 15:18:34.302 Rinnai: (Rinnai) ----> 'jsonPayload' '[{'SYST': {'CFG': {'MTSP': ... 'MT': '231'}}}]'
2021-07-06 15:18:34.303 Rinnai: (Rinnai) Globals dump:
2021-07-06 15:18:34.304 Rinnai: (Rinnai) ----> 'Domoticz' '<module 'DomoticzEx' (built-in)>'
2021-07-06 15:18:34.304 Rinnai: (Rinnai) ----> 'sys' '<module 'sys' (built-in)>'
2021-07-06 15:18:34.304 Rinnai: (Rinnai) ----> 'datetime' '<module 'datetime' from 'C:\\Program Files (x86)\\Python39-32\\Lib\\datetime.py'>'
2021-07-06 15:18:34.304 Rinnai: (Rinnai) ----> 'json' '<module 'json' from 'C:\\Program Files (x86)\\Python39-32\\Lib\\json\\__init__.py'>'
2021-07-06 15:18:34.305 Rinnai: (Rinnai) ----> 'modeCoolCmd' 'N000001{"SYST": {"OSS": {"MD": "C" } } }'
2021-07-06 15:18:34.305 Rinnai: (Rinnai) ----> 'modeEvapCmd' 'N000001{"SYST": {"OSS": {"MD": "E" } } }'
2021-07-06 15:18:34.306 Rinnai: (Rinnai) ----> 'modeHeatCmd' 'N000001{"SYST": {"OSS": {"MD": "H" } } }'
2021-07-06 15:18:34.306 Rinnai: (Rinnai) ----> '_plugin' '<plugin.BasePlugin object at 0x008544E0>'
2021-07-06 15:18:34.306 Rinnai: (Rinnai) ----> 'Parameters' '{'HardwareID': 2, 'HomeFolder': 'plugins\\Domoticz-RinnaiTouch-Plugin\\', 'StartupFolder': '', 'UserDataFolder': '', 'WebRoot': '', 'Database': 'domoticz.db', 'Language': 'en', 'Version': '1.0.0', 'Author': 'dnpwwo', 'Name': 'Rinnai', 'Address': '', 'Port': '27847', 'SerialPort': '', 'Username': '', 'Password': '', 'Key': 'RinnaiTouch', 'Mode1': '', 'Mode2': '', 'Mode3': '', 'Mode4': '', 'Mode5': 'True', 'Mode6': '0', 'DomoticzVersion': '2021.1 (build 2107)', 'DomoticzHash': '2107', 'DomoticzBuildTime': '1970-01-01 11:35:07'}'
2021-07-06 15:18:34.307 Rinnai: (Rinnai) ----> 'Devices' '{'HGOM': <plugin.hvacDevice object at 0x008574E0>, 'SYST': <plugin.hvacDevice object at 0x00857570>}'
2021-07-06 15:18:34.307 Rinnai: (Rinnai) ----> 'Images' '{}'
2021-07-06 15:18:34.308 Rinnai: (Rinnai) ----> 'Settings' '{'DB_Version': '148', 'Title': 'Domoticz', 'LightHistoryDays': '30', ... 'MaxElectricPower': '6000'}'
</syntaxhighlight>
|-
|}

==Protocol Details==
{| class="wikitable"
|-
!Protocol
!Details
|-
|HTTP/HTTPS
|HTTP and HTTPS are handled the same from a protocol perspective. HTTPS signals the underlying transport to use SSL/TLS, see [https://www.domoticz.com/wiki/Developing_a_Python_plugin#Connections Connections]. 
The HTTP protocol handles incoming and outgoing messages for both client connections requesting content from remote websites and server side listeners waiting incoming requests.
Data is passed to the protocol (Connection.Send()) and passed back to the plugin (onMessage) using dictionaries. Key values in these directories are described below with an emphasis on the impact they have to what Domoticz transmits.
{| class="wikitable"
|-
!Key
!Details
|-
|Verb
|Any valid HTTP verb (GET, POST etc) although the passed value is not checked but is converted to upper case.
If this key is present then an HTTP Request will be created and sent
|-
|Status
|String value with the HTTP Response numeric code and text (e.g '200 OK' for success). The value is not checked but sent as is.
If this key is present then an HTTP Response will be created and sent. Will be ignored if 'Verb' was specified.
|-
|URL
|The URL HTTP requests.  Defaults to "/"
|-
|Headers
|A disctionary of headers to inject into the request/response.
Dictionary keys are header names and values are header values. No processing is done on the headers, they are passed as is but should ideally be capitalised by convention.<br />
Header values can be Unicode (string), Bytes, ByteArrays or Lists (for multiple values).<br />
Extra headers are conditionally injected when the 'Verb' or 'Status' key are present:
{| class="wikitable"
|-
!Header
!Details
|-
|Authorization:Basic
|If no 'Authorization:Basic' header is present for a Request but the Parameters dictionary has Username and Password values then one will be created.
|-
|Date
|If no 'Date' header is present for a Response then one will be injected with the value match the current system time.
|-
|User-Agent
|If no 'User-Agent' header is present for a Request then one will be injected with the value 'Domoticz/1.0'.
|-
|Server
|If no 'Server' header is present for a Response then one will be injected with the value 'Domoticz/1.0'.
|-
|Content-Length
|If no 'Content-Length' header is present then one will be injected with the value matching the length of the supplied Data.
|-
|Transfer-Encoding
|If no 'Transfer-Encoding' header is present and 'Status' and 'Chunk' keys are specified then one will be injected with the value 'Chunked'.
|}
|-
|Data
|The data to be sent.  This can be a unicode string, bytes or a byte array.
|-
|Chunk
|Used during Responses only, Value is irrelevant.
Presence of this key informs the framework that this that is a partial HTTP Response using [https://en.wikipedia.org/wiki/Chunked_transfer_encoding chunked transfer encoding]. This encoding is used for large responses to break them up or to return partial data to a client early.
Sending chunked messages should be be done using a sequence of Send directives like this:<syntaxhighlight lang="python3">
  myConnection.Send({'Status':'200 OK', 'Headers':{'Content-Type': 'text/html'}, 'Chunk':True, 'Data':'This is the first chunk'})
  myConnection.Send({'Chunk':True, 'Data':'This is the second chunk'})
  myConnection.Send({'Chunk':True, 'Data':'This is the last chunk'}, Delay=1)
  myConnection.Send({'Chunk':True}, Delay=1) # this will terminate the response</syntaxhighlight>
Domoticz will send the data as fast as it can so using the 'Delay' parameter to slow transmission of large responses can prevent overloading slow clients
|}
|-
|WS/WSS
|WebSockets (WS) and Secure WebSockets (WSS) are handled the same from a protocol perspective. WSS signals the underlying transport to use SSL/TLS.
WebSocket connections are initially made over HTTP and then 'upgraded' to be Web Sockets.  This means that the message parameters (and matching response) use the protocol details for HTTP until the Server switches to the WebSocket protocol.  Successful upgrades to the WebSocket protocol are signalled by an HTTP response with a "Status" of "101".


An example WebSockets client plugin can be found in the /plugins/examples folder.


During the initial post connection message, the following HTTP headers will be injected if they are not supplied but can be overridden:

*Accept (*/*)
*Accept-Language (en-US,en;q=0.9)
*Accept-Encoding (gzip, deflate)
*Connection (keep-alive, Upgrade)
*Sec-WebSocket-Version (13)
*Sec-WebSocket-Extensions (permessage-deflate)
*Upgrade (websocket)
*User-Agent (Domoticz/1.0)
*Pragma (no-cache)
*Cache-Control (no-cache)



WebSocket servers that the author tested with required values for 'Host', 'Origin' and 'Sec-WebSocket-Key' in addition to the above list for example:<syntaxhighlight lang="python3">
    def onConnect(self, Connection, Status, Description):
        if Status == 0:
            Domoticz.Log("Connected successfully to: " + Connection.Address + ":" + Connection.Port)
            # End point is now connected so request the upgrade to Web Sockets
            send_data = {'URL': Parameters["Mode1"],
                         'Headers': {'Host': Parameters["Address"],
                                     'Origin': 'https://' + Parameters["Address"],
                                     'Sec-WebSocket-Key': base64.b64encode(secrets.token_bytes(16)).decode("utf-8")}}
            Connection.Send(send_data)

</syntaxhighlight>Data is passed to the protocol (Connection.Send()) and passed back to the plugin (onMessage) using dictionaries. Key values in these directories are described below.
{| class="wikitable"
|+
!Key
!Details
|-
|Operation
|WebSocket control messages, can be sent or recieved.
Valid Values:  "Ping", "Pong" and "Close".

Not required (or supplied) for nornal data transfer mesages.

The plugin must respond to incoming "Ping" messages with a "Pong" otherwise the server may terminate the connection.<syntaxhighlight lang="python3">
    def onMessage(self, Connection, Data):
        if "Operation" in Data:  # WebSocket control message
            if Data["Operation"] == "Ping":
                Connection.Send({'Operation': 'Pong', 'Payload': 'Pong', 'Mask': secrets.randbits(32)})

</syntaxhighlight>
|-
|Payload
|Message data.
Can be of type String (Text Messages), Bytes or ByteArray (Binary messages).
|-
|Mask
|Data mask for security purposes.
Technically this is optional but many servers will require it to be specified.<syntaxhighlight lang="python3">
import secrets
...
        if self.websocketConn.Connected():
            self.websocketConn.Send({'Payload': 'Text message', 'Mask': secrets.randbits(32)})
...
</syntaxhighlight>
|-
|Finish
|Indicates that this is the final data for a message.
Domoticz will return each individual message to the plugin but the server may break a large message into several separate messages, when this flag is False the plugin should expect more data in a subsequent onMessage call. 
|}
|}

==UI==
There is no built-in capabilities to create UI for plugin but you can use workaround with Domoticz Custom Page

Example of complex UI implementation you can find in [https://github.com/stas-demydiuk/domoticz-zigbee2mqtt-plugin this plugin]

===Setup custom page within plugin===

#Create custom HTML file in your plugins' folder, let's assume it is index.html
#General idea is to install custom page when your plugin starts and remove it when plugin stops. To do so you need to modify your plugin's onStart and onStop methods. Base folder for Python will be Domoticz root (not your plugin root)<syntaxhighlight lang="python3">
...
from shutil import copy2
import os
...

class Plugin:
  def onStart(self):
    ...
    copy2('./plugins/myplugin/index.html', './www/templates/myplugin.html')

  def onStop(self):
    ...
    if (os.path.exists('./www/templates/myplugin.html')):
      os.remove('./www/templates/myplugin.html')
</syntaxhighlight>

===Connect your page to Domoticz UI core===
Domoticz uses AngularJS framework for its UI. Additionally it uses requirejs library for lazy loading. It is not required to follow the same patterns in your custom pages but following them will allow to access built-in classes and components. Every custom page would be loaded and rendered as template for AngularJS framework with access to $rootScope and other global variables.

Example of custom page with access to AngularJS and domoticzApi service:<syntaxhighlight lang="html">
<!-- Placeholder for page content -->
<div id="plugin-view"></div>

<!-- Template for custom component -->
<script type="text/ng-template" id="app/myplugin/sampleComponent.html">
    <div class="container">
        <a class="btnstylerev" back-button>{{ ::'Back' | translate }}</a>
        <h2 class="page-header">My Plugin UI</h2>
        <p>{{ $ctrl.message }}</p>
    </div>
</script>

<script>
    require(['app'], function(app) {
        // Custom component definition
        app.component('myPluginRoot', {
            templateUrl: 'app/myplugin/sampleComponent.html',
            controller: function($scope, domoticzApi) {
                var $ctrl = this;
                $ctrl.message = 'This is my plugin'
            }
        });

        // This piece triggers Angular to re-render page with custom components support
        angular.element(document).injector().invoke(function($compile) {
            var $div = angular.element('<my-plugin-root />');
            angular.element('#plugin-view').append($div);

            var scope = angular.element($div).scope();
            $compile($div)(scope);
        });
    });
</script>
</syntaxhighlight>

==Examples==
There are a number of examples that are available that show potential usages and patterns that may be useful that can be found [https://github.com/domoticz/domoticz/tree/development/plugins/examples on Github]:
{| class="wikitable"
|-
!Example
!Description
|-
|Base Template
|A good starting point for developing a plugin.
|-
|Denon/Marantz
|Support for Denon AVR amplifiers.

*TCP/IP connectivity over Telnet using Line protocol.
*Sending delays when the amplifier is powering on.
*Dynamic creation of Selector Switches
*Dynamic discovery using UDP/IP
|-
|DLink DSP-W215
|TCP/IP connectivity using HTTP protocol.

*Features use of SOAP to demonstrate use of HTTP headers
|-
|HTTP
|Shows how to do HTTP connections and handle redirects.
|-
|HTTP Listener
|Shows how to listen for incoming connections.

*Implements a simple webserver and connects to itself.
|-
|Kodi
|Alternate Kodi plugin to the built in one.

*TCP/IP connectivity and use of the JSON protocol.
*Features three additional devices in Domoticz
|-
|RAVEn
|Alternate RAVEn Energy Monitor plugin to the built in one.

*Serial Port use with the XML protocol
|-
|UDP Discovery
|Logs incoming UDP Broadcast messages for either Dynamic Device Discovery (DDD) or Simple Service Discovery Protocol (SSDP) protocols

*UDP/IP Listening
|-
|Mutli-Threaded
|Starts a Queue on a thread to write log messages and shuts down properly
|-
|Web Socket Client
|Establishes a Web Socket connection and shows how to send and recieve messages
|-
|}

==Troubleshooting==
===Importing Modules Fails===
Python 'import' that are unsuccessful will stop the entrie plugin from importing and running unless they are wrapped within a 'try'/'except' pair.  If a module is not found the Plugin Framework will provide as much information as it gets from Python.  For example, an attempt to <code>import fakemodule</code> in a plugin called 'Google Devices' will be reported like this in the Domoticz log:
  2019-03-23 17:17:44.483  Error: (GoogleDevs) failed to load 'plugin.py', Python Path used was '/home/domoticz/plugins/Domoticz-Google-Plugin/:/usr/lib/python36.zip:/usr/lib/python3.6:/usr/lib/python3.6:/usr/lib/python3.6/lib-dynload:/usr/local/lib/python3.6/dist-packages:/usr/lib/python3/dist-packages:/usr/lib/python3.6/dist-packages'.
  2019-03-23 17:17:44.483  Error: (Google Devices) Module Import failed, exception: 'ModuleNotFoundError'
  2019-03-23 17:17:44.483  Error: (Google Devices) Module Import failed: ' Name: fakemodule'
  2019-03-23 17:17:44.483  Error: (Google Devices) Error Line details not available.
the framework shows the directories it searches. Broadly these are: the plugin directory, the existing Python path as picked up when Domoticz started and the 'site' directories for '''system level modules''' installed with pip3. Local 'site' directories are not searched by defaul although plugin developers can add any directory the like to the path inside the plugin. Pip3 installs packages in system directories when sudo is used on linux platforms.

===Plugin Crashes Domoticz===
Yes, this is unfortunately possible. Python3 itself is vulnerable to segmentation faults and Domoticz just inherits this. If you don't believe me try typing <code>python3 -c "import ctypes; ctypes.string_at(0)"</code> at a linux command line with Python installed.
<br />
To help troubleshoot these issues the Plugin Framework attempts to load the standard 'faulthandler' module for each plugin just before the plugin itself is loaded.  If the plugin triggers any signals a thread by thread stack trace will be printed.  To see this you will need to run Domoticz from the command line or redirect sys.stderr to a file ([https://stackoverflow.com/questions/1956142/how-to-redirect-stderr-in-python how would I do that?])
<br /><br />
Output for a multi-threaded plugin will look something like this, a Python stack trace followed by a normal Domoticz one. The offending line in the plugin is shown in bold:
    Fatal Python error: Segmentation fault
    
    Thread 0xa8d0eb40 (most recent call first):
      File "/usr/local/lib/python3.6/dist-packages/pychromecast/socket_client.py", line 361 in run_once
      File "/usr/local/lib/python3.6/dist-packages/pychromecast/socket_client.py", line 341 in run
      File "/usr/lib/python3.6/threading.py", line 916 in _bootstrap_inner
      File "/usr/lib/python3.6/threading.py", line 884 in _bootstrap
    
    Current thread 0xab553b40 (most recent call first):
    ....
      File "<frozen importlib._bootstrap>", line 219 in _call_with_frames_removed
      File "<frozen importlib._bootstrap_external>", line 678 in exec_module
      File "<frozen importlib._bootstrap>", line 665 in _load_unlocked
      File "<frozen importlib._bootstrap>", line 955 in _find_and_load_unlocked
      File "<frozen importlib._bootstrap>", line 971 in _find_and_load
      '''File "/home/domoticz/plugins/Domoticz-Google-Plugin/plugin.py", line 322 in handleMessage'''
      File "/usr/lib/python3.6/threading.py", line 864 in run
      File "/usr/lib/python3.6/threading.py", line 916 in _bootstrap_inner
      File "/usr/lib/python3.6/threading.py", line 884 in _bootstrap
    
    Thread 0xad597b40 (most recent call first):
      File "/usr/lib/python3.6/threading.py", line 299 in wait
      File "/usr/local/lib/python3.6/dist-packages/zeroconf.py", line 1794 in wait
      File "/usr/local/lib/python3.6/dist-packages/zeroconf.py", line 1365 in run
      File "/usr/lib/python3.6/threading.py", line 916 in _bootstrap_inner
      File "/usr/lib/python3.6/threading.py", line 884 in _bootstrap
    
    Thread 0xb57cfb40 (most recent call first):
    2019-03-23 11:21:04.003  Error: Domoticz(pid:24259, tid:24500('PluginMgr')) received fatal signal 11 (Segmentation fault)
    2019-03-23 11:21:04.003  Error: siginfo address=0x5ec3, address=0xb7f51d09
    2019-03-23 11:21:04.741  Error: Thread 19 (Thread 0xab553b40 (LWP 24500)):
    2019-03-23 11:21:04.741  Error: #0  0xb7f51d09 in __kernel_vsyscall ()
    ...

===Debugging===
Debugging embedded Python can be done using the standard pdb functionality that comes with Python using the <code>rpdb</code> (or 'Remote PDB') Python library.<br>
<br>
The only restriction is that it can not be done from a debug instance of Domoticz running in Visual Studio on Windows.<br>
<br>
Download the <code>rpdb</code> library from https://pypi.python.org/pypi/rpdb/ and drop the 'rpdb' directory into the directory of the plugin to be debugged (or anywhere in the Python path). Alternatively, if pip3 is installed (<code>sudo apt install python3-pip</code> if on Raspian or a Debian distro) then install the debugger using <code>sudo pip3 install rpdb</code>

Import the library using something like:
    def onStart(self):
        if Parameters["Mode6"] == "Debug":
            Domoticz.Debugging(1)
            Domoticz.Log("Debugger started, use 'telnet 0.0.0.0 4444' to connect")
            import rpdb
            rpdb.set_trace()
        else:
          Domoticz.Log("onStart called")
The <code>rpdb.set_trace()</code> command causes the Python Framework to be halted and a debugger session to be started on port 4444. For the code above the Domoticz log will show something like:
  2017-04-17 15:39:25.448  (MyPlugin) Initialized version 1.0.0, author 'dnpwwo'
  2017-04-17 15:39:25.448  (MyPlugin) Debug log level set to: 'true'.
  2017-04-17 15:39:25.448  (MyPlugin) Debugger started, use 'telnet 0.0.0.0 4444' to connect
  pdb is running on 127.0.0.1:4444
Connect to the debugger using a command line tool such as Telnet. Attaching to the debugger looks like this if you start the session on the same machine as Domoticz, otherwise supply the Domoticz IP address instead of '0.0.0.0':
 pi@bob:~$ telnet 0.0.0.0 4444 
 Trying 0.0.0.0...
 Connected to 0.0.0.0.
 Escape character is '^]'.
 --Return--
 > /home/pi/domoticz/plugins/MyPlugin/plugin.py(30)onStart()->None
 -> rpdb.set_trace()
 (Pdb)
enter commands at the prompt such as:
 (Pdb) l  
  25  	    def onStart(self):
  26  	        if Parameters["Mode6"] == "Debug":
  27  	            Domoticz.Debugging(1)
  28  	            Domoticz.Log("Debugger started, use 'telnet 0.0.0.0 4444' to connect")
  29  	            import rpdb
  30  ->	            rpdb.set_trace()
  31  	        else:
  32  	          Domoticz.Log("onStart called")
  33  	
  34  	    def onStop(self):
  35  	        Domoticz.Log("onStop called")
 (Pdb) pp Parameters
  {'Address': '',''
  'Author': 'dnpwwo',
  'HardwareID': 7,
  'HomeFolder': '/home/pi/domoticz/plugins/MyPlugin/',
  'Key': 'Debug',
  'Mode1': '',''
  'Mode2': '',''
  'Mode3': '',''
  'Mode4': '',''
  'Mode5': '',''
  'Mode6': 'Debug',
  'Name': 'MyPlugin',
  'Password': '',''
  'Port': '4444',
  'SerialPort': '',''
  'Username': '',''
  'Version': '1.0.0'}
 (Pdb) b 53
 Breakpoint 1 at /home/pi/domoticz/plugins/MyPlugin/plugin.py:53
 (Pdb) continue
 > /home/pi/domoticz/plugins/MyPlugin/plugin.py(53)onHeartbeat()
 -> Domoticz.Log("onHeartbeat called")
 (Pdb) ll
  52  	    def onHeartbeat(self):
  53 B->	        Domoticz.Log("onHeartbeat called")
 (Pdb) 
Details of debugging commands can be found here: https://docs.python.org/3/library/pdb.html#debugger-commands<br>
