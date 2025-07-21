# Control de Acceso Inteligente para Habitación

## Integrantes

* Luis Fernando Castro
* William Camilo Obando

---

## Descripción del Proyecto

Este proyecto implementa un sistema embebido para el control de acceso y ventilación de una habitación utilizando un microcontrolador STM32. El sistema gestiona el acceso mediante un teclado y una pantalla OLED, validando una contraseña y controlando un ventilador con PWM basado en temperatura. Además, ofrece un modo de emergencia, control automático del ventilador y comunicación con un módulo ESP-01 para control remoto.

A continuación se presenta el diagrama de bloques del sistema:

```mermaid
graph TD
  A[Teclado Matricial 4x4] -->|Entradas de Clave| B[STM32 (MCU)]
  B -->|Control I2C| C[Pantalla OLED]
  B -->|PWM TIM3| D[Ventilador]
  B -->|Lectura ADC| E[Sensor LM35]
  B -->|UART3| F[Módulo ESP-01]
  B -->|GPIO| G[Circuito de Bloqueo de Puerta]
```

Este diagrama representa cómo cada componente del sistema se conecta con el microcontrolador central. El STM32 actúa como el núcleo del sistema, controlando entradas y salidas mediante periféricos como ADC, PWM, UART y GPIO.

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

La lógica del sistema está organizada como una máquina de estados, facilitando el flujo estructurado de control. Cada estado representa una fase del ciclo de acceso:

* `ROOM_STATE_LOCKED`: Puerta bloqueada, espera de clave.
* `ROOM_STATE_INPUT_PASSWORD`: Entrada de contraseña, con verificación automática y timeout.
* `ROOM_STATE_UNLOCKED`: Acceso concedido, puerta desbloqueada.
* `ROOM_STATE_ACCESS_DENIED`: Acceso denegado tras intento incorrecto, vuelve a LOCKED tras un tiempo.
* `ROOM_STATE_EMERGENCY`: Estado reservado para emergencias (a implementar).

El control se actualiza constantemente con base en temporizadores, eventos y entradas externas.

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

Este código centraliza toda la lógica de transición y actualización del sistema.

---

### Entrada de Teclado (Keypad)

Se emplea un teclado matricial que detecta dígitos ingresados y transiciones de estado. El buffer de clave se actualiza dígito a dígito hasta verificar contra la contraseña almacenada.

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

Durante la entrada de clave, se despliega “CLAVE:” junto a asteriscos que representan los dígitos ingresados. También se muestran mensajes de estado como “ACCESO CONCEDIDO” o “DENEGADO”.

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

El LM35 se conecta al pin ADC y su valor se convierte a grados Celsius. El sistema determina el nivel de ventilación adecuado y ajusta el PWM en consecuencia. Este control puede ser automático o forzado manualmente desde UART.

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

El módulo ESP-01 puede enviar comandos por UART al STM32. El parser de comandos detecta líneas completas (terminadas en `\n`) y las interpreta:

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

Comandos disponibles:

* `GET_TEMP`: devuelve temperatura actual.
* `SET_PASS xxxx`: cambia contraseña del sistema.
* `FORCE_FAN nivel`: fuerza nivel de ventilador (OFF, LOW, MED, HIGH).

---

## Inicialización en main.c

Inicialización del sistema y lectura periódica:

```c
room_control_init(&room_system);
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
```

Lectura de sensor LM35:

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

El proyecto demuestra una solución completa de control de acceso embebido, integrando múltiples periféricos del STM32 como ADC, UART, PWM y GPIO. El sistema es capaz de validar contraseñas, mostrar información visualmente, controlar temperatura ambiente y recibir comandos remotos. Su arquitectura modular permite futuras mejoras como integración con sensores de presencia, modo emergencia, o apertura remota mediante app móvil o web.
