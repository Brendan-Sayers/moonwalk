--- 
layout:  post
---

### Transistor Modulator

The first part of the buck project, to meet my "from scratch" requirement, is a ramp wave generator 
The idea here was to make a ramp wave generator, about 100 KHz,  that I can tune by swapping resistors or with a pot, on a board.


The basic concept is a constant current source, fed into a capacitor. This generates a linearly increasing voltage, with a dV/dt given by I/C.

The node is connected to a device that will trigger on when Vramp = Vthreshold, and off when Vramp is (about) 0V. When triggered on, it pulls the connected node to ground, turns off, and allows current source to begin recharging.

I had a couple ideas, but one I liked was ne was using two BJTs, a PNP and NPN, in a "Thyristor" pair. I went this for a couple reasons 
	1. It seemed easy to model and understand 
	2. I could peer into the circuit as I was modeling in spice, which fit my goals 
	3. It seemed very elegant 


I began buy modeling a circuit from this [video](https://www.youtube.com/watch?v=2a1I1X3RV0g), which worked easily enough, once "Use initial conditions" was turned off. 
![Basic Circuit](/assets/images/TransistorModulator/BasicCircuit.jpg)

It worked, the next step was running 100 KHz

The equation for slope is dV/dt = I/C.
For 100 Khz. 5vpk wave, that gives 0.5V / Us. 
The [datasheet](https://assets.nexperia.com/documents/data-sheet/PSSI2021SAY.pdf) showed the current source IC I found would output 50uA across the range of Vccs for the circuit.
50uA requires a 100pF capacitor, which seems to be the lower end of what seems comfortable. It's an amount were the traces could have single digit percent influence on the value, and a buffer amp maybe. A passive 'scope probe certainly does for this impedance. 
In LTSPICE
![at 100KHz...](/assets/images/TransistorModulator/PlotAt100khzAlmost.png)

What happened?
![Closer look at slope](/assets/images/TransistorModulator/NotSwithcingOffDetail.png)
The slope seems about right, which is a good start, and it triggers on, which means we're about half way there. It triggers on at the wrong voltage, but that's easier to tune. 
I think this is a good time to discuss criteria for switching on and off 

For it to switch on, the following sequence
Vramp goes about a diode drop above Vref 
Q1 starts conducting. The current it sinks begins turning Q2 on, which pulls down Vref to ~ 0V, which also increases Q1s drive, i.e. positive feedback  
This should continue until the pulse has removed (most of) the charge from C1, leaving it at about a diode drop, with Q1 and Q2 off. 

The issue here seems to be that current sources 50uA are able to leave both conducting, maybe better said enough for Q2 to pull down Vref, and keep Q1 on. 
The turn off action relies on basically reverse of the turn on feedback, which means it's three possible actions

	1) Current source outputs too much
	2) Vref output impedance is too high, small currents pull it below a diode drop
	3) Transistor has too much beta. 

Since the current source determines the slope, and we're already on the low end of that, the next step would be lowering the impedance of the biasing network. Some trial and error shows that a ~ 50 - 500 ohms for R1 will produce a switching waveform. But not exactly a ramp wave.
![Better, but not quite there](/assets/images/TransistorModulator/SwitchButLongTurnOff.png)



In a sense, it's about finding the right combination of the three. There's many paths that work, and no correct answer, as  any of the parameters can be adjusted so that it works or doesn't work. Consider for a second, that you could just select beta. You can't. or at least I can't, maybe a fab could. But consider say a beta of 1. That'd turn off beautifully; 50uA wouldn't pull down a 10k any. Though it wouldn't switch on either.

That being said, there is one approach that works very well. 
For turn off, you don't want lower gain, you want different gains at different points. When the transistor is off, you want a very high gain to turn it back on. When's already on, you don't want much gain. Enough that big output pulse keeps it on, but not so much that the Current source can.

You can get this by using a Schottky diode from the Q2s base to collector. The diode conducts away most or all of the base current when Vbe > Vramp. This has an added advantage, as it lowers the peak base current as well, a figure that generally goes unspecified on the datasheet
![Working model](/assets/images/TransistorModulator/WithSchottkyandZener.png)

Which gives me that, which looks pretty good. There's one issue, which is my threshold is wrong, but that's because I've switched to a 4.7V Zener as my reference. The threshold is then about 5.2V, though the slope is correct.

I did some work to establish a more formal criteria, though I think it's gonna be ugly. 