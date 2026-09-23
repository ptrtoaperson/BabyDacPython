# BabyDAC

###### Python API for the BabyDAC programmable voltage source

BabyDAC is a 24-channel programmable voltage source. This package gives you a
single Python class, `B_DAC`, to connect to a board over **Ethernet** or
**Serial (UART)** and control its channels: set DC voltages, run PWM outputs,
play soft/triggered voltage sequences, and chain channels together.

## Features

- Connect over **Ethernet (TCP)** or **Serial (USB/UART)** with the same API
- Set per-channel DC output voltages (-10 V to +10 V)
- Start/stop PWM outputs per channel
- Play software-timed and hardware-triggered voltage sequences
- Chain/fan-out sequences across multiple channels on shared IRQs

## Installation

```bash
pip install BabyDac
```

## Quick start

### Connect over Ethernet

```python
from babydac import B_DAC

k = B_DAC(ip_address="192.168.1.50", port="5000")

```

### Connect over Serial (UART)

```python
from babydac import B_DAC

k = B_DAC(serial_port="/dev/ttyUSB0") #for windows "COMx" where x is the respective com port

```

### Access channels by name

Every channel is available as a named attribute (`dac_01` ... `dac_24`):

```python
K.dac_01.set_voltage(5.0)
```

## Common operations
### PWM generation
PWM can be only generated at max of 2KHz 
```python
# PWM on channel 3, toggling between 0V and 5V at 1 kHz with 50% duty cycle
k.dac_03.start_pwm(tgp_id=0,low=0.0, high=5.0, freq=1000.0, duty_cycle=50)
k.dac_03.stop_pwm()

```
### Software-timed voltage sequence
```python
k.dac_01.soft_sequence(delay_ms=10, voltages=[0, 1, 2, 3, 0])
```

### External triggered voltage sequence 
```python
# Set the external trigger threshold level (-10 to + 10 volts), by default level is set to +3V
ext.set_level(3.3)
k.ext.edge = 0 #by default egde is 1 i.e. rising
k.ext.bind(k.dac_02)
k.dac_02.trigger_sequence(voltages=[-10,-5,0,5,10])
```


### Fan out external trigger to multiple channels 
```python
import numpy as np
vlt = np.linspace(-10, 10, 50)
# each (channel, edge) pair can use its own edge: 1 = rising, 0 = falling
k.ext.bind([(k.dac_03, 1), (k.dac_04, 0)])
k.dac_03.trigger_sequence(vlt)
k.dac_04.trigger_sequence(vlt)
```
### Internal IRQ fan-out (channel-to-channel triggering)

A source channel's soft sequence can fire an internal IRQ every N points that
starts follower sequences on other channels, so they run in lock-step with the
source without needing an external trigger line:

```python
import numpy as np

src = np.linspace(-10, 10, 50)
follower1 = np.linspace(-10, 10, 25)
follower2 = np.linspace(-10, 10, 10)

# dac_03 advances one point every 2 IRQs, dac_12 advances one point every 5 IRQs
k.dac_01.bind([(k.dac_03, 2), (k.dac_12, 5)])
k.dac_03.set_on_trigger(follower1)
k.dac_12.set_on_trigger(follower2)

# starts the source sequence, driving the bound followers via the internal IRQ
k.dac_01.soft_sequence(500, src)
```

### Reset all channels/sequences on the DAC
```python
k.clear_state()
```

### Batch multiple commands for simultaneous execution:

```python
with dac.batch():
    k.dac_01.set_voltage(1.0)
    k.dac_02.set_voltage(-1.0)

k.send()
```

### Changing the network configuration

The device must be power-cycled/reinitialized after this before the new settings
take effect.

```python
k.set_network_config(ip_address="192.168.1.60", port=5000)

# reconnect using the new address after reinitializing the device
k = B_DAC(ip_address="192.168.1.60", port=5000)
```


