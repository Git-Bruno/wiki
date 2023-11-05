## Plugin Update

__ATTENTION:__ We assume here that you are already on Version 6.3. Otherwise please upgrade first to [version 6](Plugin_Version-6.md) and to [version 6.3](Release_6.3.md)

The plugin is regularly updated. For bug fixing or enhancement. In order to keep the plugin up to date


### From the WebUI

Since version 6.3, it is now possible to update the plugin form the plugin administration page : Admin > Plugin > Update


### Manually

* Go in your DomoticZ directory using a command line and open the plugins directory then the ZigBeeForDomoticZ-Plugin directory.

* Run the commands :

```bash
  git pull
  sudo python3 -m pip install -r requirements.txt --upgrade
 ```

__Note :__ Since version 6, the git pull is no longer enough.

* In case you have edited/updated some of the plugin controlled file, you might get an error message protecting any update. In such situation you can reset those files with the latest version with the following commands

  ```bash
  git reset --hard
  git pull --force
  ```

* Run: `sudo chmod +x plugin.py`
* Restart Domoticz.


## Plugin branches

The plugin is under constant development in order to bug fix and add new features as well as taking into account new devices.

The code is available under different branches

## About release channels

In order to provide stability and also provide more recent development, Zigate plugin has the following channels

### stable7

This is considered as a solid , reliable version.

### develop

This is the development channel

## How to switch from one channel to the other

`git pull`

`git checkout stable7 // will move you to the stable7 channel`

`git checkout develop    // will move you to the develop channel`
