

# MAX30102 heart rate monitor under STM32 control

Probably every reader of mine had to do with some sort of garment device, so-called wearables. Even the simplest fitness wristbands have a built-in heart rate sensor that measures most often on the wrist of a person using such a gadget. It turns out that this is not a mysterious technology that is only available to a closed group of consumer electronics manufacturers. Also on our desks we can make a simple measurement of our own heart rate. All
you need is the right layout.

### MAX30102 in the service of health

Company  *Maxim*  released several sensors designed pulse measurements. One of them is discussed today MAX30102 ( [MAX30102 Datasheet](https://msalamon.pl/download/720/) ). It is basically a device 2 for 1 in addition to pulse, you can also measure the oxygen saturation, the oxygenation of blood. This sensor is often used in smart phones, for example, in Samsung.

The sensor itself to power the logic requires a voltage of 1.8 V. Next to this is also required a second voltage to power the diodes. This may be 3.3 V, the diode but also quietly tolerate 5V power supply module has a built-in LED's at 5V, so no need to worry him unnecessarily.

This sensor has two LEDs illuminating the test tissue - red and infrared light. To collect the signal while a photo-detector is used.

The system has a very low power consumption mode ratio  *Shutdown* only 0.7uA. The overall current drawn by the sensor in the active state mainly depends on the current drawn by the LED, which is controlled by software. Not counting the diode sensor itself needs to work about 600 uA. Each of the current LED while the software is regulated in the range of 0 ÷ 51mA. Huh! 51mA per diode seems quite a lot. I did not dare let go of so much power ...

The sensor has an output INT. It can be used to signal several events. The most useful of them in my opinion is signaling a new sample and the upcoming FIFO overflow.

![img](https://msalamon.pl/wp-content/uploads/2019/04/MAX30102_baner-1024x341.jpg)

## Reflectance measurement of the heart rate and oxygen saturation

Used by our Max reflectance measurement is called because the LED and photo detector are located on the same side of the finger. The light emitted by the LEDs is reflected from the finger (specifically hemoglobin) and returns to the detector.

The measurement of oxygen saturation takes place through the phenomenon of light absorption by hemoglobin. Oxygenated hemoglobin to a low degree absorbs light having a wavelength of about 660 nm, while the heavily oxygenated hemoglobin will absorb more light having a wavelength of 940 nm. Diodes MAX30102 incorporated in the circuit have wavelengths 660 nm and 880 nm. Although we do not go perfectly in the infrared wavelength, it works correctly. The level of oxygen saturation is determined by the ratio of transmittance of light for both of these lengths.

Number of reflected light as the major component comprises a pulse wave from which the pulse is easily read. The following chart posted for the red LED, which I downloaded from my finger sensor discussed.

![img](https://msalamon.pl/wp-content/uploads/2019/03/max30102_samples.jpg)

## The schedule of CubeMX

Today, the system will use Nucleo STM32F103. Connection diagram of the sensor is simple thanks to the complete module which buy. Unfortunately pull-up on the module are connected to 1.8 V, which does not fully willing to cooperate with the STM32 speeding at 3.3 V or at least my copy. The outer pull-up therefore added to the supply voltage of the module and the I2C pins, and INT in the micro-controller I configured as  *No pull-up and pull-down no.*

![img](https://msalamon.pl/wp-content/uploads/2019/03/max30102_schematic.jpg)

I2C immediately set on 400 kHz.

![img](https://msalamon.pl/wp-content/uploads/2019/03/max30102_i2c_settings.jpg)

You need to act even interrupt input responsive to the falling edge. Make sure the cube is activated interrupt during initialization. They are sometimes the eggs. Anticipating comments - yes, HAL is the worst in the world for it

![img](https://msalamon.pl/wp-content/uploads/2019/03/max30102_int_settings.jpg)

## Library

We go to the most interesting or to how to use our sensor. Maxim your website provides sample code for the Arduino environment and the Mbed. While the same search algorithm peaks in the signal, converting them to heart rate and blood oxygenation counting algorithm are OK, as such sampling is hopeless

Unfortunately sample loop main index is dominated waiting for the emergence of a break from the sensor. Then I downloaded and converted are taken. After collecting the first 500 and then every 100 samples is activated algorithm. Samples unnecessarily in my opinion are moved to the left after calculations. It can not be that the micro controller specifically waiting for the break! In addition, all the time at full power LEDs, no matter if a finger is applied to the sensor or not. What a waste of electricity ...

Example also will not work with a different configuration of the sensor than 100 samples per second. I corrected it.

Action leaned my library on a simple state machine with four states, where in the meantime all the time interrupted collected samples. Comp goes to the circular buffer so that need not be moved each time releasing the algorithm. What happens in the wild states of the machine?

1. **MAX30102_STATE_BEGIN** . The finger is not applied to the sensor. The red LED is turned off, IR minimum. IR LED is designed to detect a finger placed on the sensor. Whenever the finger will be downloaded from the sensor, the program returns to the state of quenching diode.
2. **MAX30102_STATE_CALIBRATE** . After applying a finger to populate the buffer 'correct data'. By default, calculations are performed on samples from the last 5 seconds. Calibration, the buffer full enough so it will last.
3. **MAX30102_STATE_CALCULATE_HR** . After collecting an adequate number of samples are made by the calculation algorithm Maxim. In this step, the ring buffer is shifted by an amount of samples corresponding to one second.
4. **MAX30102_STATE_COLLECT_NEXT_PORTION** . In this state of operation of the sensor is the time when another portion is collected samples, that is, the next second. When it gets to the state number 3, and the cycle is repeated until the download of the finger sensor.

### Using the library

I wrote a library so that its use was as simple as possible. The whole is based on a few steps. First, you should initialize the library and the sensor specifying which I2C interface was used.

```c
MAX30102_STATUS Max30102_Init(I2C_HandleTypeDef *i2c);
```
Another important point is the service interruption. Redefine built-in function HAL *weak.*

```c
/* USER CODE BEGIN 4 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == INT_Pin) {
        Max30102_InterruptCallback();
    }
}
/* USER CODE END 4 */

```

The last condition is thrown into the main loop functions corresponding to the charge state machine

```c
void Max30102_Task(void);
```

Now you can read from a library of current heart rate and saturation using two functions.

```c
int32_t Max30102_GetHeartRate(void);
int32_t Max30102_GetSpO2Value(void);
```

### Additional configuration

Standard library and the sensor is configured for 5 seconds and measurement of 100 samples per second. You can freely modify. The library will automatically adjust the size of the sample buffers and will be able to customize the calculations

Additionally, it is set such as the number of samples to produce the interrupt *FIFO*  *Almost Full* . If for some reason the sample will not be received on time, they will be queued in the sensor. The queuing of the set number of samples will interrupt this event. Library detecting the interruption, read all the samples that are in the buffer, clearing it at the same time. The buffer can accommodate up to 32 samples. The default value of 17 is the minimum value that causes an interrupt. As for me, this is enough setting but did not check the sensor in a real battle. Perhaps the sensor will be extended application was doing better at a higher value.

```c
#define MAX30102_MEASUREMENT_SECONDS 5
#define MAX30102_SAMPLES_PER_SECOND 100 // 50, 100, 200, 400, 800, 100, 1600, 3200 sample rating
#define MAX30102_FIFO_ALMOST_FULL_SAMPLES 17
```

## Presentation of action

My sample code print calculated the terminal value.

![img](https://msalamon.pl/wp-content/uploads/2019/03/max30102_measurement_terminal.jpg)

Quite low pulse for the seat and patting code. And I thought it was a stressful job. View yet how the automatic switching on and off the LEDs depending on whether the sensor is applied to the finger.