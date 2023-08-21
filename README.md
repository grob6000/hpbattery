# HP Battery

A Li-Po battery replacement design for HP35/HP45/HP55/HP65/HP67 calculators. The original batteries had ~ 500mAh capacity, while modern Ni-MH replacements are typically ~1500mAh.

While a higher capacity is not necessarily feasible, Li-Po has the advantage of far reduced self-discharge in comparison to Ni-MH, which is an advantage for an intermittently-used device.

Ni-MH / Ni-CD batteries are notorious for leaking, which is somewhat traded-off against the notoriety of Li-Pos to self-immolate.

## Features / design goals:
- Fit into existing compartment with no modifications
- Similar or ideally exceed capacity of standard Ni-Cd or Ni-MH 3xAA types (1200~2200mAh)
- Dual charge - USB (externally) or in-place using original HP charger
- Battery charge control and protection, OK-ish to accidentally leave on
- Maximise battery life:
    - Minimise Iq
    - Minimise voltage drop / series resistance of circuits
    - Optimise Vout to minimise calculator power dissipation

## References 
Thanks to the below references, I intend to utilise elements of these designs but with some functional improvements.
 - https://www.edn.com/classic-hp-35-calculator-comes-back-to-life/
   - GOOD: Regulated (see reasoning in the article, and discussion below)
   - BAD: Has no charging circuit / must remove to charge
   - BAD: Cannot connect HP charger (would cause dangerous overcharge condition; best case trip battery OC protection)
 - https://hackaday.io/project/175815-classic-hp-calculator-lipo-battery-pack
   - GOOD: Has on-board charge controller
   - BAD: Has no regulation
   - GOOD: Can connect HP charger, but
   - BAD: Unable to charge from HP charger / must remove to charge

## Key Components
 - Battery:
     - Found a variety of low cost 50x34mm cells which may be made to fit:
       - 6mm thick: 1000mAh
       - 8mm thick: 1500mAh
       - 11mm thick: 2000mAh
 - Charge controller: [TP4056](https://dlnmh9ip6v2uc.cloudfront.net/datasheets/Prototyping/TP4056.pdf)
     - Very common and cheap charge controller for LiPo batteries
 - Regulator: [TPS73101](https://www.ti.com/product/TPS73101-EP)
     - Ultra low dropout (~30mV)
     - 150mA current (HP35 calculator uses ~120mA worst case)
 - Output protection diode: [MAX40200](https://www.analog.com/en/products/max40200.html)
     - Low voltage drop (~50mV) ideal diode
 - Battery protection: [S-8261ABJMD](https://www.ablic.com/en/doc/datasheet/battery_protection/S8261_E.pdf)
     - Pin-compatible with [DW01A](https://datasheet.lcsc.com/szlcsc/1901091236_PUOLOP-DW01A_C351410.pdf)
     - Type J: Discharge protection @ 3.0V (DW01 is too low at 2.5V). Overcharge protection at 4.28V.
 - Battery disconnector: [FS8205](https://datasheet.lcsc.com/szlcsc/Fortune-Semicon-FS8205_C32254.pdf)
     - Typically paired with DW01A, circuit has been kepts equivalent to typical online modules.

## Function Description
![Block diagram of battery](https://github.com/grob6000/hpbattery/blob/master/blockdiagram.png?raw=true)
### Discharging, calculator on:
 - TP4056 is powered down (VCC < VBAT)
 - B+ supplies TPS73101 regulator, which regulates to 3.60V
 - Output delivered to calculator via MAX40200
 - Up to 16h run time estimated
### Charging, USB:
 - VBUS at Q1 G pulls TP4056 prog down to 5k (250mA)
 - VBUS supplies VCC via D1 (5V down to 4.7V)
 - TP4056 charges battery at 250mA (<0.2C, 6 hour charge time)
 - Physically not possible to do inside the calculator, assume HP input disconnected
### Charging, HP charger:
 - Q1 off; TP4056 prog at 25k (40mA setting)
 - TP4056 charges battery at 40mA, uses 1mA for LED, remainder available to bias D2 and control VCC
 - U2 (MAX40200) reverse biased; V at PAD+ will be ~5.4V while VREG is 3.6V max; HP input does not interfer with battery charging.
### Battery Exhausted:
 - HP35 shows low battery (decimal points) indication at V+ of 3.50V
   - This represents about 6-7% charge state of LiPo
   - User may switch off and recharge now...
 - HP35 will cease to function at ~3.25V, still draws ~75mA
   - User probably should switch off and recharge now...
 - U4 (S-82612ABJMD) protects battery at VB+ = 3.0V via U5 (FS8205)
   - E.g. if left on accidentally.
   - Reset by applying VCC to TP4056 (will charge battery and raise voltage above recovery threshold for U4)

## Notes on optimisation:

Refer to spreadsheet

Addition of a series regulator is not an immediately obvious means to maximise efficiency.
  - As input voltage rises, input current to the HP35 rises also, likely due to increased dissipation by the logic circuits as voltage rises, some level of shunt regulation in the power supply input, and particularly the increased current through the LEDs as the voltage rises (fixed resistor current limiting). As such, minimising the Vreg while ensuring operation of the calculator is desirable to minimise the consumption. Measurements of a HP35 were taken with (0.) displayed and (8888888888.-88) displayed, and are included in the excel.
  - Series regulator dropout voltage will result in Vout reaching the low battery threshold (3.50V) at a higher remaining battery capacity than without. As such, selection of a series regulator with extremely low Vdo is important to minimise the unusable capacity. TPS73101 was found which has a Vdo of ~30mV at the current levels anticipated. It is capable of 150mA, which is just enough to supply the calculator.

The discharge is modelled using measured load data from a HP35, an approximate Li-Ion discharge curve from a forgotten online source. I found that for regulation voltages below about 3.8V, the usable operating time of the HP35 would in fact be longer than without a regulator. Minimising the output voltage, assuming the function of the calculator is assured, maximises the running time. I've selected VREG=3.6V, which the model estimates (with a 1500mAh battery) at 16.1h vs. 15.4h for no regulation; i.e. 42mins extra or 4.5% improvement.

![Graph of discharge performance model](https://github.com/grob6000/hpbattery/blob/master/dischargeperformance_3v6.png?raw=true)

### Why not a switching regulator?

In theory, replacing the series linear regulator with a switching regulator may allow use of the battery capacity right down to 3.0V, and more efficiently use the currently wasted energy of the full battery (VB+ > 3.63V) currently being dissipated by the linear reg.

Unfortunately, as the VB+ is close to Vout required, a topology able to deal with VB+ both above and below Vout is needed. The target efficiency to outperform the linear regulator would be 93.8%, which is difficult for low power simple topologies. As such, the linear regulator may remain the optimal solution.
