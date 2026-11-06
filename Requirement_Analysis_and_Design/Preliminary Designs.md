Water Transport:
    1. A connector tube/hose is used to facilitate the transport of fresh, filtered water between the ice maker and a water source
    2. An electric valve is to be positioned between the hose and the ice maker, opening the water source when more water is need, and closing it when there is suffiecent water in the ice maker.
    3. The valve is to be connected to a nearby arduino in order to know when to turn the water source open or closed.

Basic Internal Design:(ice maker)
    1. A food grade PP plastic float switch is to be attached to the front facing wall of the ice maker's "water area"(denoted in drawing), defining a level in which the ice maker is deemed "full".
    2. A leak/moisture sensor is to be placed on the floor of the cabinet in order to detect water in the case of spillage.

Basic Internal Design:(water source)
    1. An electric valve is the first thing attached to the exterior water source, controlling whether fluid can flow in or not.
    2. Pressure sensors are placed before and after the water filter in order to calculate its preassure difference. If there is a high pressure difference, the filter needs to be changed.
    3. water filter is situated between the preassure sensors and is responsible for cleaning inpurities out of the water.

Basic Software Design:
    1. The software used to moniter the ice maker is going to be seperated into 2 broad sections: overall quality and average sensor values.
    2, The overall quality is accounted by taking the information given by the sensors in order to  derive whether the ice maker system is functioning effectiently, or ineffeciently.
    3, Average sensor values takes an rough unwieghted average of the data the sensors give within a specific time frame(every 1 or 2 secounds), and displays them on the software.
    4, "Log Documents" are provided so that the user can moniter the change in average sensor values(1 given every 1-2 secounds) within a larger interval(like within 20 minutes or 40 minutes).