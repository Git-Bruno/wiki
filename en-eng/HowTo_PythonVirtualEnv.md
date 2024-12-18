# Using Python Virtual Environment with Domoticz and Zigbee 4 Domoticz Plugin

## Why ?

It looks like recent linux distribution will prevent using `sudo python3 -m pip` installation. This is mainly driven to prevent any risk of breaking the system by overwriting libraries that where par of tools writen in Python whith the system.

To proper install the python libraries and applications required for the Z4D plugin, we need to create a virtual environment dedicated to __Domoticz__ and store all python modules/libraries required for the various python plugins.

This would prevent error messages like that one:

```log
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.

    If you wish to install a non-Debian-packaged Python package,
    create a virtual environment using python3 -m venv path/to/venv.
    Then use path/to/venv/bin/python and path/to/venv/bin/pip. Make
    sure you have python3-full installed.

    For more information visit http://rptl.io/venv

note: If you believe this is a mistake, please contact your Python installation or OS distribution provider. You can override this, at the risk of breaking your Python installation or OS, by passing --break-system-packages.
hint: See PEP 668 for the detailed specification.
```

## Create your python environment for Domoticz

### What is the python version running under Domoticz

Looks for the message in the Domoticz log.

`Status: PluginSystem: Started, Python version '3.10.11', 10 plugin definitions loaded.`

Conclusion: Domoticz is running python3.10 as thepython interpreter for all python plugins.

Check what is the default python3 interpreter version

`python3 --version`

If you get : `Python 3.10.11` in response to the command, then you have the default python3 interpreter matching the Domoticz one, if you get a different answer like `Python 3.12.7` it means that Domoticz is running a lower python interrpeter than the default on your system (which is not a problem), and you must load in the Python environment for Domoticz the librariries for the right interpreter version and in this case 3.10 and not 3.12. To do so , don't use python3 but python3.10 instead to specify the version you need to use.

### Create the python environment

We are suggesting to create the Domoticz Python Environment in the Domoticz home directory such as __/home/domoticz__

```bash
cd /home/domoticz
mkdir Domoticz_Python_Environment
```

```bash
cd /home/domoticz/plugins/Domoticz-Zigbee
````

```bash
python3 -m pip install -r requirements.txt --upgrade -t /home/domoticz/Domoticz_Python_Environment
```

or if you have to use a specific version.

```bash
python3.10 -m pip install -r requirements.txt --upgrade -t /home/domoticz/Domoticz_Python_Environment
```

### Makes Domoticz started with the Python Environment

Update the script which automaticaly start Domoticz and add the following environment variable

export PYTHONPATH=/home/domoticz/Domoticz_Python_Environment:$PYTHONPATH

* If you are using "domoticz.sh" you need to add `PYTHONPATH=/home/domoticz/Domoticz_Python_Environment:$PYTHONPATH` in the first part of the script to define where the python environment is located.

* if you are using a systemd unit definition you need to add `PYTHONPATH=/home/domoticz/Domoticz_Python_Environment:$PYTHONPATH` to the environment prior launching the domoticz daemon


As you are changing a startup script, you might also have to run `systemctl daemon-reload``
