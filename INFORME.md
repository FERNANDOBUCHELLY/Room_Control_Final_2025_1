# Control de Acceso Inteligente para Habitación

## Integrantes

* Luis Fernando Castro
* William Camilo Obando

---

## Descripción del Proyecto

Este proyecto implementa un sistema embebido para el control de acceso y ventilación de una habitación utilizando un microcontrolador STM32. El sistema gestiona el acceso mediante un teclado y una pantalla OLED, validando una contraseña y controlando un ventilador con PWM basado en temperatura. Además, ofrece un modo de emergencia, control automático del ventilador y comunicación con un módulo ESP-01 para control remoto.

---

## Funcionalidades Implementadas

* Ingreso de contraseña desde un teclado matricial (Keypad 4x4)
* Control de acceso (bloqueo/desbloqueo de puerta)
* Visualización en pantalla OLED (clave, mensajes de estado)
* Control automático del ventilador por temperatura (LM35)
* Control PWM del ventilador con distintos niveles (OFF, LOW, MED, HIGH)
* Cambio automático de estados según temporizadores o entradas
* Comunicación UART con un ESP-01 para comandos remotos
* Parser de comandos seriales (GET\_TEMP, SET\_PASS, FORCE\_FAN)
* Modo manual de control del ventilador
* Retroalimentación visual mediante asteriscos en pantalla al ingresar contraseña

---

## Instrucciones de Compilación y Uso

1. **Entorno Requerido:**

   * STM32CubeIDE o VSCode con entorno de compilación ARM
   * Bibliotecas de drivers HAL

2. **Conexiones:**

   * Teclado matricial a pines GPIO configurados como entradas/salidas
   * Pantalla OLED conectada vía I2C
   * Ventilador conectado a pin PWM (TIM3\_CH1)
   * Sensor de temperatura (LM35 en Pin ADC\_IN5)
   * UART2 (depuración), UART3 (ESP-01)

3. **Pasos para Ejecutar:**

   * Abre el proyecto en STM32CubeIDE
   * Compila y flashea el código a la placa Nucleo
   * Verifica conexiones con los pines
   * Interactúa con el sistema desde el teclado o comandos remotos

---

## Decisiones de Diseño y Explicaciones de Código

### Sistema de Control de Estados

La lógica principal del sistema se basa en una máquina de estados finitos:

* `ROOM_STATE_LOCKED`: Puerta bloqueada, espera de clave.
* `ROOM_STATE_INPUT_PASSWORD`: Entrada de contraseña, con timeout.
* `ROOM_STATE_UNLOCKED`: Acceso concedido, ventilador activo.
* `ROOM_STATE_ACCESS_DENIED`: Acceso denegado, espera y vuelve a LOCKED.
* `ROOM_STATE_EMERGENCY`: Estado reservado para emergencias.

```c
void room_control_update(room_control_t *room) {
    uint32_t current_time = HAL_GetTick();

    switch (room->current_state) {
        case ROOM_STATE_LOCKED:
            room->door_locked = true;
            break;

        case ROOM_STATE_INPUT_PASSWORD:
            if (current_time - room->last_input_time > INPUT_TIMEOUT_MS) {
                room_control_change_state(room, ROOM_STATE_LOCKED);
            }
            break;

        case ROOM_STATE_UNLOCKED:
            room->door_locked = false;
            break;

        case ROOM_STATE_ACCESS_DENIED:
            if (current_time - room->state_enter_time > ACCESS_DENIED_TIMEOUT_MS) {
                room_control_change_state(room, ROOM_STATE_LOCKED);
            }
            break;

        case ROOM_STATE_EMERGENCY:
            // Lógica futura
            break;
    }

    room_control_update_door(room);
    room_control_update_fan(room);

    if (room->display_update_needed) {
        room_control_update_display(room);
        room->display_update_needed = false;
    }
}
```

---

### Entrada de Teclado (Keypad)

Dependiendo del estado del sistema, la entrada del teclado permite iniciar ingreso de clave, ingresar dígitos, o bloquear el sistema:

```c
void room_control_process_key(room_control_t *room, char key) {
    room->last_input_time = HAL_GetTick();

    switch (room->current_state) {
        case ROOM_STATE_LOCKED:
            room_control_clear_input(room);
            if (room->input_index < PASSWORD_LENGTH && key >= '0' && key <= '9') {
                room->input_buffer[room->input_index++] = key;
                room_control_change_state(room, ROOM_STATE_INPUT_PASSWORD);
            }
            break;

        case ROOM_STATE_INPUT_PASSWORD:
            if (key >= '0' && key <= '9' && room->input_index < PASSWORD_LENGTH) {
                room->input_buffer[room->input_index++] = key;
            }
            if (room->input_index == PASSWORD_LENGTH) {
                if (strncmp(room->input_buffer, room->password, PASSWORD_LENGTH) == 0) {
                    room_control_change_state(room, ROOM_STATE_UNLOCKED);
                } else {
                    room_control_change_state(room, ROOM_STATE_ACCESS_DENIED);
                }
            }
            break;

        case ROOM_STATE_UNLOCKED:
            if (key == '*') {
                room_control_change_state(room, ROOM_STATE_LOCKED);
            }
            break;
    }

    room->display_update_needed = true;
}
```

---

### Visualización en Pantalla OLED

Durante la entrada de clave, la pantalla muestra "CLAVE:" seguido de asteriscos representando los dígitos ingresados:

```c
ssd1306_SetCursor(10, 10);
ssd1306_WriteString("CLAVE:", Font_7x10, White);
ssd1306_SetCursor(10, 25);
for (uint8_t i = 0; i < room->input_index; i++) {
    ssd1306_WriteString("*", Font_7x10, White);
}
```

---

### Lectura de Temperatura y Control del Ventilador

El sistema lee un valor ADC del sensor LM35 y lo mapea a una temperatura entre 20 °C y 40 °C. El ventilador se ajusta automáticamente con PWM.

```c
void room_control_set_temperature(room_control_t *room, float temperature) {
    room->current_temperature = temperature;

    if (!room->manual_fan_override) {
        fan_level_t new_level = room_control_calculate_fan_level(temperature);
        if (new_level != room->current_fan_level) {
            room->current_fan_level = new_level;
            room->display_update_needed = true;
        }
    }
    room->display_update_needed = true;
}
```

PWM generado por TIM3:

```c
static void room_control_update_fan(room_control_t *room) {
    uint32_t pwm_value = 0;
    switch (room->current_fan_level) {
        case FAN_LEVEL_OFF: pwm_value = 0; break;
        case FAN_LEVEL_LOW: pwm_value = (30 * 99) / 100; break;
        case FAN_LEVEL_MED: pwm_value = (70 * 99) / 100; break;
        case FAN_LEVEL_HIGH: pwm_value = 99; break;
    }
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, pwm_value);
}
```

---

### Comunicación UART y Comandos Remotos

El sistema recibe comandos desde el ESP-01 vía UART3:

* `GET_TEMP`: Devuelve temperatura actual
* `SET_PASS xxxx`: Cambia contraseña
* `FORCE_FAN nivel`: Fuerza nivel de ventilador (OFF, LOW, MED, HIGH)

```c
void command_parser_process_char(command_parser_t *parser, char c) {
    if (c == '\n') {
        parser->command_buffer[parser->command_index] = '\0';
        command_parser_execute(parser);
        parser->command_index = 0;
    } else if (parser->command_index < COMMAND_BUFFER_SIZE - 1) {
        parser->command_buffer[parser->command_index++] = c;
    }
}
```

---

## Inicialización en main.c

```c
room_control_init(&room_system);
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
```

Lectura del sensor:

```c
HAL_ADC_Start(&hadc1);
if (HAL_ADC_PollForConversion(&hadc1, 10) == HAL_OK) {
    uint32_t adc_value = HAL_ADC_GetValue(&hadc1);
    float temperature = 20.0f + ((float)adc_value * 20.0f / 4095.0f);
    room_control_set_temperature(&room_system, temperature);
}
HAL_ADC_Stop(&hadc1);
```

Lectura del teclado:

```c
room_control_update(&room_system);
if (keypad_interrupt_pin != 0) {
    char key = keypad_scan(&keypad, keypad_interrupt_pin);
    if (key != '\0') {
        room_control_process_key(&room_system, key);
    }
    keypad_interrupt_pin = 0;
}
```

---

## Conclusión

El proyecto demuestra una solución de control de acceso embebido con retroalimentación visual, control de clima y entrada remota. Está diseñado con modularidad, usa periféricos del STM32 como GPIO, I2C, ADC, PWM y UART, y puede extenderse con nuevas funciones como apertura remota o detección de presencia.
