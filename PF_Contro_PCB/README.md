# PF_contro_PCB

**Water Analyzer · Proyecto final · PCB de control, revisión 1.0**

Este README documenta la composición, las conexiones y la puesta en servicio de la PCB del Water Analyzer. Las asignaciones corresponden a los esquemáticos y a la serigrafía de la revisión 1.0. Los límites de funcionamiento del conjunto se establecen mediante pruebas de banco.

## 1. Objetivo y usos

La PCB centraliza la adquisición de señales fisicoquímicas en puntos de uso de agua farmacéutica. Recibe las salidas de los módulos de pH, conductividad, ORP y turbidez, mide la temperatura con una sonda 1-Wire y recibe una señal digital para detectar el nivel de la cámara. El ESP32 procesa las mediciones y las entrega a una pantalla TFT táctil de 4 pulgadas. Cuatro canales de potencia permiten accionar bombas y electroválvulas externas de 12 V.

La placa se puede usar para probar sensores, observar variaciones en las mediciones y ensayar secuencias de llenado, medición y vaciado. La asignación física de bombas y válvulas a ACT1–ACT4 y la lógica de las secuencias se definen en el firmware y el cableado. El alcance de la revisión es el monitoreo operativo. La demostración de conformidad farmacopéica requiere los métodos y los instrumentos aplicables al grado de agua.

## 2. Componentes principales

| Referencia en el esquema | Componente | Función |
| --- | --- | --- |
| U8 | ESP32-S3-WROOM-1U-N16R8 | MCU a 3.3 V, con 16 MB de flash, 8 MB de PSRAM octal y conector para antena externa. |
| Q1 | DMP4015SPSQ-13 | MOSFET de canal P para protección contra inversión de polaridad en la entrada de 12 V. |
| D1 | SMBJ15A | Diodo TVS unidireccional para limitar transitorios en la línea de 12 V protegida. |
| C26, C30 | RZH1E471M1010 | Electrolíticos de 470 µF y 25 V en paralelo en la entrada protegida. Su capacitancia nominal combinada es 940 µF. |
| D2, D3 | B240A | Diodos Schottky que combinan la fuente de 12 V y el VBUS del USB hacia `V_SYS`. |
| U4 | TPS62162DSG | Convertidor buck de salida fija de 3.3 V. |
| U6 | TPS62163DSGR | Convertidor buck de salida fija de 5 V. |
| L2, L3 | MPIT3010-2R2M-LF | Inductores de 2.2 µH de los convertidores de 3.3 V y 5 V. |
| U2, U3 | ADS1115IDGSR | ADC I²C de 16 bits y cuatro entradas multiplexadas por dispositivo. |
| U1, U5 | UCC27524ADR | Drivers dobles de compuerta, alimentados a 5 V. |
| U_ACT1–U_ACT4 | TPM604NY3-2 | MOSFET de canal N por salida de potencia, en configuración de lado bajo. |
| D18–D21 | B340B-13-F | Diodos Schottky de rueda libre, uno por canal de actuador. |
| D15 | ESD441DPLR | Protección ESD en VBUS del micro-USB. |
| D4–D5, D7–D9, D11–D14, D16–D17 | ESD321DPYR | Protección ESD de pulsadores, UART, 1-Wire, entradas analógicas de sensores y líneas USB de datos según el bloque. |
| En2, boot2 | TS-1088-AR02016 | Pulsadores de reinicio y selección del modo de programación. |

La tabla recoge los componentes principales. Las resistencias SMD emplean encapsulados 0603 y 0805 y los capacitores cerámicos emplean 0603 y 0805, con tensiones nominales de 16 V o 25 V según la posición. Los electrolíticos de entrada se especifican aparte. Los indicadores son LED 0805. La lista de montaje indica rojo, aunque en el esquemático D10 figura como `LED_White Yellow`. Confirmar el color y la referencia de compra de D10 en la BOM final.

## 3. Subcircuitos

### 3.1 Entrada de 12 V y selección de alimentación

- `12V_inpu1` recibe la fuente externa prevista de **12 V CC / 5 A**. El pin 1 es GND y el pin 2 es el positivo. El fusible previsto se instala externamente en la línea positiva.
- Q1 protege contra polaridad invertida. R12 de 100 kΩ lleva su gate hacia GND. Después del MOSFET, D1 limita transitorios y C26/C30 aportan reserva de carga. C27 de 10 µF y C28 de 100 nF completan el filtrado de la línea.
- El positivo protegido alimenta directamente las cargas de 12 V y el buck de 5 V. D2 y D3 alimentan `V_SYS` desde los 12 V protegidos o desde el VBUS del micro-USB, con la caída directa del Schottky respectivo. Si ambas fuentes están conectadas y los 12 V son normales, conduce D2 y D3 queda polarizado en inversa.
- Con **solo USB**, se puede encender el dominio de 3.3 V para programación, sujeto a la corriente que entregue el puerto. El riel de 5 V de la placa y la potencia para actuadores dependen de la entrada externa de 12 V. VBUS y el riel regulado de 5 V son redes distintas.

### 3.2 Conversión a 3.3 V y 5 V

| Riel | Regulador y entrada | Cargas principales |
| --- | --- | --- |
| 3.3 V | U4 TPS62162DSG desde `V_SYS` | ESP32, ambos ADS1115, pull-ups I²C y 1-Wire, módulos de pH y EC, pantalla y lógica de expansión. |
| 5 V | U6 TPS62163DSGR desde `12V` protegido | Módulos ORP y turbidez, circuito de nivel y alimentación de los drivers de compuerta. |

Ambos convertidores tienen 10 µF y 100 nF en la entrada, un inductor de 2.2 µH y 22 µF más 100 nF en la salida. Cada salida tiene un LED con 510 Ω y un punto de prueba. Las salidas `PG` quedan sin conectar. TI especifica hasta **1 A** por regulador bajo sus condiciones de operación. La capacidad disponible para conectores externos depende del consumo de la placa, la alimentación y el comportamiento térmico. Los **5 A** corresponden a la fuente prevista y la corriente admisible de la placa requiere validación independiente. [Ficha de la familia TPS6216x](https://www.ti.com/lit/ds/symlink/tps62162.pdf)

### 3.3 Adquisición analógica

U2 y U3 se alimentan a 3.3 V. Comparten I²C con **SDA en GPIO8** y **SCL en GPIO9**. Las dos líneas llevan pull-up de 4.7 kΩ a 3.3 V. Cada ADC tiene desacoplo de 100 nF y 1 µF, y cada entrada analógica tiene 100 nF a GND. `ALERT/RDY` queda sin conexión, por lo que el firmware puede comprobar la disponibilidad de cada conversión vía I²C.

| ADC | ADDR | Dirección I²C de 7 bits | AIN0 | AIN1 | AIN2 | AIN3 |
| --- | --- | --- | --- | --- | --- | --- |
| U2 | GND | `0x48` | Turbidez | ORP | Conductividad | pH |
| U3 | 3.3 V | `0x49` | A0B | A1B | A2B | A3B |

Configurar los canales de los sensores como entradas simples referidas a GND. Para expansión, mantener las señales dentro de **0 a 3.3 V** en funcionamiento normal y compartir GND. El rango seleccionado en el PGA determina la escala de conversión. La alimentación del ADC sigue determinando el intervalo admisible en sus entradas. [Ficha del ADS1115](https://www.ti.com/lit/ds/symlink/ads1115.pdf)

### 3.4 Salidas para bombas y electroválvulas

U1 controla ACT1 y ACT2. U5 controla ACT3 y ACT4. Cada salida del driver acciona el gate a través de 33 Ω, con un pull-down de 10 kΩ entre gate y GND. Los dos drivers comparten `ACT_EN` (GPIO15), que tiene un pull-down de 10 kΩ. Cada driver tiene 100 nF y 1 µF de desacoplo. [Ficha del UCC27524A](https://www.ti.com/product/UCC27524A)

Cada conector de actuador tiene **pin 1 = +12 V protegido** y **pin 2 = retorno conmutado**. El MOSFET conecta el retorno a GND al activarse. El diodo de rueda libre queda conectado con el cátodo a +12 V y el ánodo al retorno conmutado. Conectar la carga entre ambos pines del conector.

En el firmware, inicializar `ACT_EN` y las cuatro señales ACT en bajo. Una vez configuradas las salidas y comprobadas las condiciones de la secuencia, poner `ACT_EN` en alto. Usar ON/OFF o PWM según la carga y su especificación. Como objetivo de diseño se había previsto **hasta 3 A combinados**, pendiente de confirmar con corriente de arranque, carga simultánea y temperatura de MOSFET, pistas y conectores.

### 3.5 Pantalla, reset y comunicaciones

- La pantalla TFT y el controlador táctil comparten MISO, MOSI y SCK. Cada dispositivo tiene su propia selección, `CS` para la TFT y `T_CS` para el táctil. La TFT también usa `DC` y `Rst`. El pin 1 de `Display1` queda sin conexión, por lo que cualquier IRQ que el módulo conecte a ese pin requeriría otra asignación. Confirmar el modelo de pantalla y su pinout antes de conectarla.
- `En2` lleva EN a GND al pulsarlo. R10 de 10 kΩ y C36 de 1 µF forman la red de arranque. `boot2` lleva GPIO0 a GND y esa línea tiene C37 de 100 nF. Ambos pulsadores tienen protección ESD.
- USB1 es micro-USB, con D− en GPIO19 y D+ en GPIO20 a través de resistencias serie, y protección ESD en datos y VBUS. Se comunica directamente con el ESP32-S3 por USB nativo. [USB Serial/JTAG de Espressif](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/usb-serial-jtag-console.html)
- El conector rotulado **`UART1` en la placa** lleva **UART0 del ESP32-S3**. Expone TXD0, RXD0, 3.3 V y GND para consola, depuración o recuperación del firmware.

## 4. Sensores e interacción con la PCB

| Parámetro | Sensor o fuente | Conector y destino | Acondicionamiento y uso |
| --- | --- | --- | --- |
| pH | DFRobot SEN0161-V2 | `pH_Sensor1` → U2 AIN3 | El módulo se alimenta a 3.3 V y su salida analógica entra directamente al ADC, con D14 para ESD. Calibrar con buffers de pH 7 y 4 conforme al procedimiento del sensor. [DFRobot SEN0161-V2](https://wiki.dfrobot.com/sen0161-v2/) |
| Conductividad (EC) | DFRobot DFR0300, K = 1 | `Conduc_Sensor1` → U2 AIN2 | Módulo a 3.3 V con salida directa al ADC y D13 para ESD. Verificar en banco la excursión real antes de aprobar todo el rango. El fabricante publica una salida de hasta **3.4 V**, por encima de los 3.3 V de alimentación del ADC. Convertir a conductividad con la calibración del módulo y usar la temperatura de la muestra cuando corresponda. [DFRobot DFR0300](https://wiki.dfrobot.com/dfr0300/) |
| ORP | DFRobot SEN0165 | `ORP_Sensor3` → U2 AIN1 | Módulo a 5 V. R15/R16, de 10 kΩ cada una, reducen la salida a aproximadamente la mitad. D12 protege el nodo del ADC. En el firmware, reconstruir el voltaje original multiplicando la lectura del ADC por 2 antes de aplicar la conversión y el ajuste de offset del módulo. [DFRobot SEN0165](https://wiki.dfrobot.com/sen0165/) |
| Turbidez | DFRobot SEN0189 | `Turb_Sensor1` → U2 AIN0 | Módulo a 5 V. Usar salida analógica del módulo. R13/R14, de 10 kΩ cada una, reducen la salida a aproximadamente la mitad y D11 protege el nodo. Multiplicar por 2 para recuperar el voltaje del módulo. Reportar cambios relativos de turbidez con este sensor. [DFRobot SEN0189](https://wiki.dfrobot.com/sen0189/) · [FAQ del fabricante](https://www.dfrobot.com/forum/topic/315216) |
| Temperatura | DS18B20 impermeable | `Temp_Sensor1` → GPIO10 | Conexión digital 1-Wire de tres hilos. R11 de 4.7 kΩ mantiene DATA a 3.3 V en reposo, D9 protege la línea y C47 de 1 µF desacopla la alimentación. El firmware puede asociar esta temperatura con las lecturas de EC de la misma muestra. [DS18B20 de Analog Devices](https://www.analog.com/en/products/ds18b20.html) |
| Nivel | Sonda fotoeléctrica prevista para SEN0205 | `Level_Sensor1` → GPIO42 | La PCB incorpora la interfaz. R40 de 330 Ω va en serie con la alimentación de la sonda. R39 de 4.7 kΩ eleva la señal a 5 V, C1 de 100 nF filtra el nodo y R38/R17 (10 kΩ/18 kΩ) adaptan el nivel a GPIO42. Confirmar orden de hilos, topología de salida, polaridad mojado/seco y márgenes de tensión con la sonda física. [DFRobot SEN0205](https://wiki.dfrobot.com/sen0205/) |
| TDS | Estimación a partir de EC | Cálculo en firmware | Mostrarlo como `TDS estimado` en mg/L si se implementa. El factor EC–TDS depende de la composición del agua y debe documentarse junto con la condición de temperatura empleada. |

Los conectores de pH, EC, ORP y turbidez reciben la salida de sus **módulos acondicionadores**, a los que se conectan las respectivas sondas. Cada uno tiene 1 µF local entre alimentación y GND. En el circuito de nivel, si la salida de la sonda queda abierta y R39 es el único pull-up, el GPIO recibe aproximadamente **2.75 V** con una fuente ideal de 5 V. Si un módulo fuerza el nodo de señal a 5 V, recibe aproximadamente **3.21 V**. Estos valores son nominales y sirven para orientar la comprobación de banco.

Para el uso farmacéutico, el DFR0300 tiene un rango recomendado de **1 a 15 mS/cm**, mientras que el criterio de la etapa 1 de USP 〈645〉 a 25 °C es **1.3 µS/cm** sin compensación de temperatura. La calificación de conductividad de Agua Purificada requiere instrumentación apropiada para ese rango. [USP 〈645〉](https://doi.usp.org/USPNF/USPNF_M99405_01_01.html) · [Tabla de etapa 1 publicada por USP](https://www.usp.org/sites/default/files/usp/document/health-quality-safety/usp-hand-sanitizer-ingredients.pdf)

## 5. Mapa de GPIO del ESP32-S3

| GPIO | Red | Función |
| --- | --- | --- |
| 0 | Boot | Pulsador `boot2`, selección de modo de arranque. |
| 2 | Rst | Reset de la TFT. |
| 4 | ACT1 | Mando del canal de potencia 1. |
| 5 | ACT2 | Mando del canal de potencia 2. |
| 6 | ACT3 | Mando del canal de potencia 3. |
| 7 | ACT4 | Mando del canal de potencia 4. |
| 8 | SDA | Datos I²C de ambos ADS1115. |
| 9 | SCL | Reloj I²C de ambos ADS1115. |
| 10 | Data | 1-Wire del DS18B20. |
| 11 | MOSI | SPI compartido entre TFT y táctil. |
| 12 | SCK | SPI compartido entre TFT y táctil. |
| 13 | MISO | SPI compartido entre TFT y táctil. |
| 14 | CS | Selección de la TFT. |
| 15 | ACT_EN | Habilitación común de los drivers. |
| 16, 17, 18 | IO16–IO18 | Expansión en `GPIO1`. |
| 19 | Du− | USB D− del ESP32, lado del MCU después de la resistencia serie. |
| 20 | Du+ | USB D+ del ESP32, lado del MCU después de la resistencia serie. |
| 21 | DC | Selección datos/comandos de la TFT. |
| 38, 39 | IO38, IO39 | Expansión en `GPIO1`. |
| 40 | T_CS | Selección del controlador táctil. |
| 41 | IO41 | Expansión en `GPIO1`. |
| 42 | IO42 | Entrada digital de nivel. |
| 43 | TXD0 | TX de la PCB en el conector `UART1`. |
| 44 | RXD0 | RX de la PCB en el conector `UART1`. |
| 45, 46 | IO45, IO46 | Pines de configuración de arranque con pull-downs de 10 kΩ, R19 y R18 respectivamente. |
| 47, 48 | IO47, IO48 | Expansión en `GPIO1`. |

`EN` es la habilitación/reinicio del módulo. GPIO1 y GPIO3 figuran sin conectar en U8. GPIO35–GPIO37 figuran sin conectar en el esquema y se reservan por el uso de PSRAM octal en esta variante del módulo. Los GPIO de expansión trabajan a 3.3 V. [Ficha del ESP32-S3-WROOM-1/1U](https://www.espressif.com/sites/default/files/documentation/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf)

## 6. Accesos, conectores y puntos de prueba

### 6.1 Conectores

El orden indicado es por **número de pin del esquemático**. Localizar el pin 1 en la serigrafía o por el pad cuadrado antes del cableado. En la vista superior, el micro-USB y los pulsadores están a la izquierda, la bornera de 12 V está abajo a la izquierda, la expansión está arriba, y las conexiones de sensores y actuadores se concentran en los bordes superior, derecho e inferior. La placa tiene cuatro orificios de montaje y un diseño de cuatro capas, con L2 destinada a GND y L3 a distribución de alimentación.

| Conector | Asignación por pin |
| --- | --- |
| `12V_inpu1` | 1 GND · 2 +12 V de entrada |
| `USB1` | 1 VBUS · 2 D− · 3 D+ · 4 ID sin conectar · 5 GND · blindaje a GND |
| `Turb_Sensor1`, `ORP_Sensor3` | 1 GND · 2 5 V · 3 salida analógica del módulo |
| `Conduc_Sensor1`, `pH_Sensor1` | 1 GND · 2 3.3 V · 3 salida analógica del módulo |
| `Temp_Sensor1` | 1 DATA · 2 GND · 3 3.3 V |
| `Level_Sensor1` | 1 GND · 2 GND · 3 alimentación desde 5 V a través de 330 Ω · 4 señal |
| `ACT1` a `ACT4` | 1 +12 V protegido · 2 retorno conmutado |
| `UART1` | 1 3.3 V · 2 TX de la PCB · 3 RX de la PCB · 4 GND |
| `Analog_Pin1` | 1 A0B · 2 A1B · 3 A2B · 4 A3B · 5 GND · 6 GND |
| `GPIO1` | 1–3 GND · 4 GPIO18 · 5 GPIO17 · 6 GPIO16 · 7 GPIO47 · 8 GPIO48 · 9 GPIO38 · 10 GPIO39 · 11 GPIO41 · 12 3.3 V · 13 5 V · 14 12 V protegido |

### 6.2 Conector de pantalla `Display1`

| Pin | Señal | Pin | Señal |
| --- | --- | --- | --- |
| 1 | Sin conectar | 8 | SCK |
| 2 | MISO táctil (`T_DO`) | 9 | MOSI |
| 3 | MOSI táctil (`T_DIN`) | 10 | DC |
| 4 | T_CS | 11 | Rst |
| 5 | SCK táctil (`T_CLK`) | 12 | CS de TFT |
| 6 | MISO | 13 | GND |
| 7 | 3.3 V | 14 | 3.3 V |

### 6.3 Puntos de prueba

| Punto | Red | Uso |
| --- | --- | --- |
| `P-12V1` | 12 V protegido | Verificar la salida de Q1 y observar caídas de tensión con cargas. |
| `TP-VBUS1` | VBUS | Verificar el voltaje del USB antes de D3. |
| `TP-PreBuck1` | `V_SYS` | Verificar la tensión seleccionada antes del buck de 3.3 V. |
| `TP-3.3V1` | 3.3 V | Verificar el riel de lógica y el arranque del ESP32. |
| `TP-5V1` | 5 V | Verificar el riel de sensores y drivers cuando se aplican 12 V. |
| `TP_SDA1`, `TP_SCL1` | SDA y SCL | Revisar niveles y actividad I²C con analizador lógico u osciloscopio. |

## 7. Programación y primera puesta en servicio

### 7.1 Carga por micro-USB nativo

1. Para la primera carga, desconectar los actuadores y periféricos externos. Conectar `USB1` al equipo mediante un cable **con datos**. Verificar el riel de 3.3 V si el puerto no aparece.
2. Configurar el proyecto para ESP32-S3, **16 MB de flash** y **PSRAM octal de 8 MB** según el entorno de compilación. Seleccionar el puerto USB Serial/JTAG. Con ESP-IDF se puede cargar desde el directorio del firmware mediante `idf.py -p <PUERTO> flash`.
3. Si la entrada automática al modo de carga falla, mantener presionado `boot2`, pulsar y soltar `En2` y después soltar `boot2`. Si el puerto cambia al entrar al cargador, seleccionar el nuevo puerto y cargar.
4. Al terminar, pulsar `En2` con `boot2` liberado para ejecutar el firmware. Si se requiere consola por el mismo puerto, habilitar USB Serial/JTAG como consola en la configuración del firmware. [Selección de modo de arranque de Espressif](https://docs.espressif.com/projects/esptool/en/latest/esp32s3/advanced-topics/boot-mode-selection.html)

### 7.2 Recuperación por UART0

Alimentar la placa por una de sus entradas previstas. Conectar un adaptador USB–UART de **lógica de 3.3 V** al conector `UART1`: GND con GND, TX del adaptador al pin 3 (RX de la PCB), y RX del adaptador al pin 2 (TX de la PCB). Dejar el pin 1 de 3.3 V sin conectar si la PCB ya está alimentada. Aplicar la misma secuencia `boot2`/`En2` y seleccionar el puerto del adaptador al cargar.

### 7.3 Comprobaciones de banco

1. Con solo USB, verificar `VBUS`, `V_SYS` y 3.3 V. Con la fuente externa, verificar 12 V protegido, `V_SYS`, 5 V y 3.3 V antes de conectar cargas. Tomar GND de la placa como referencia.
2. Escanear I²C y comprobar que aparecen `0x48` y `0x49`. Confirmar cada señal analógica en el canal correspondiente y revisar sus niveles antes de conectar los módulos de forma permanente.
3. Probar temperatura y comprobar el nivel en seco y mojado antes de usar GPIO42 para el control de llenado. Verificar la visualización TFT y las coordenadas del táctil por separado.
4. Con la cámara y el circuito hidráulico listos, probar un actuador a la vez y luego las combinaciones previstas. Medir corriente de arranque, caída en los rieles y temperatura de la etapa de potencia. Comprobar que los actuadores permanecen apagados durante el arranque y el reinicio.
5. Comparar las lecturas con patrones o instrumentos de referencia en el rango de trabajo. Repetir con los actuadores apagados y activos para identificar ruido de conmutación. Verificar también si la lectura cambia al energizar varias sondas en la misma muestra.

## 8. Oportunidades de mejora

| Mejora | Aplicación a una próxima revisión |
| --- | --- |
| **USB-C** | Sustituir el micro-USB por USB-C trabajando como dispositivo USB 2.0. Colocar una resistencia Rd de 5.1 kΩ a GND en cada línea CC1 y CC2, unir los contactos D+ duplicados entre sí y hacer lo mismo con D− cerca del conector. Mantener la protección ESD y revisar el ruteo del par y la puesta a tierra del blindaje. [Guía USB-C de Microchip](https://www.microchip.com/en-us/application-notes/an1953) |
| **Conductividad de bajo rango** | Si el uso previsto es Agua Purificada, incorporar una celda y electrónica adecuadas para el rango de µS/cm. Si se conserva el DFR0300 para ensayos, caracterizar su salida y prever atenuación o limitación compatible con la entrada de 3.3 V del ADC. |
| **Coordinación de protecciones** | Verificar la tensión residual de D1 frente al margen de los bucks. El SMBJ15A tiene 15 V de tensión de trabajo y puede limitar a aproximadamente 24.4 V en su pulso de especificación. Los TPS6216x tienen 17 V como máximo de operación recomendado. Comprobar también el arranque con la capacitancia de entrada y la conmutación simultánea de cargas. [Ficha SMBJ15A](https://www.littelfuse.com/products/tvs-diodes/surface-mount/smbj/smbj15a) · [Ficha TPS6216x](https://www.ti.com/lit/ds/symlink/tps62162.pdf) |
| **Medición de turbidez y nivel** | Si el informe requiere turbidez cuantitativa, evaluar un turbidímetro calibrable en el rango de interés. Caracterizar la interfaz de nivel con la sonda real y considerar una entrada con histéresis si aparecen transiciones inestables. |
| **Diagnóstico y mantenimiento** | Añadir accesos de prueba a las entradas analógicas acondicionadas, `ACT_EN` y las señales de mando. Documentar en serigrafía la polaridad y reasignar el rótulo `UART1` a `UART0` para reflejar el periférico real. |
| **Interferencia entre sondas** | Si las pruebas con varias sondas en la misma cámara muestran interacción, evaluar medición secuencial, alimentación conmutada o aislamiento donde el ensayo identifique el acoplamiento. |

## 9. Referencias visuales de la revisión 1.0

- [Esquemático del ESP32-S3 y la UART](https://chatgpt.com/library?file_id=libfile_7d32f168a6ac81919c8d5fec6f96290c&account_id=personal)
- [Ruteo de la PCB](https://chatgpt.com/library?file_id=libfile_7149e8ada4688191b3fc5cf668dfa2bb&account_id=personal)
- [Vista 3D superior de la PCB](https://chatgpt.com/library?file_id=libfile_6da0e10a4c088191a52f7a4e6080ad2a&account_id=personal)
