Issue #1: 
    During the testing of the solenoid valve in combination with the flow sensor and other componenets, whenever the valve was de-activated, the pulse count in the flow sensor increased by incriments of 1. Though this didn't affect flowrate, it heavily altered the calculated total passed water volume.
Possible Fixes:
    1, Includea pull down resistor for the flow sensor
    2, Sodder a diode onto the legs of the solenoid valve.
Verdict: The issue was caused by back EMF within the solenoid valve when it was deactivated. The collapsing magnetic field of the solenoid caused an induced voltage across its terminals. This induced voltage created a strong electromagnetic field that caused an induced current in surrounding wires. By adding a diode across the terminals of the valve, it suppresses the back EMF when it is formed, neutralizing the electromagnetic field and the induced current in the surrounding wires