# home-assistant-config
Any useful Home Assistant Config I can share

## custom_components

### solax
This is actually the same as the offical integration, except it points at my solax lib instead of the current one, where I added a timeput to the http call. This is because aiohttp default timeout is 5 mins, which is too long for home assistant to loop through the inverter options

## Config

### Glow Display
I've created some automations for registering a glow display as a device via mqtt, assuming you have local mqtt set up
see: https://community.home-assistant.io/t/glow-hildebrand-display-local-mqtt-access-template-help/428638/241

Automations:
*	smart_meter_device_registry
	This sets up the device itsself, then disables itself (only needs runnng once). You need to add YOUR_ID
*	smart_meter_device_registry_activate
	This turns the automation on at home assistant start, meaning we run device registry each start just in case anythings changed
*	update_elec_peak_offpeak_rates
	This sets peak rate and off peak rate. Basically if rate changes it looks if its hiher or lower then previous and works out if that means its peak or offpeak, keeping teh right rate if yoru rates change
	It also sets utility meters to peak and offpeak at the same time, so you dont need times automations to do it, this takes care of it

NB delete bits you dont need (commented in the yaml


### Tado - Regulate temperature using the offset (No longer have Tado so not maintained!)
Like many people who enjoy home automation, Home Assistant and have a Tado system, I have been wanting to do a little more to manage my Tado thermostats, especially my TRVs to have them register the real temp in the room. Therefore, I set about adding the ability to get and set the offset for Tado devices in Home Assistant. This was implemented in the 2021.2 release of Home Assistant

Now that we have access to the offset we can use that to keep the device temp in sync with another thermostat in the room, placed in a better location than on the radiator (like the Tado TRVs are)

To take advantage of this I wanted one automation to handle the whole thing and allow me the flexibility to have different settings per device should I choose. I've created said automation and am sharing here for others to use should they wish. Its below for ease or the file can be downloaded from the Tado directory

NB there is a back off setting but that requires a further code update to Tado integration I have written. I'll get a PR in for that, for now it just wont do anything for others (doesnt break anything just wont have any effect)
