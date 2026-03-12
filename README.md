[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/KzqfxGd5)
[![Open in Visual Studio Code](https://classroom.github.com/assets/open-in-vscode-2e0aaae1b6195c2367325f4f02e2d04e9abb55f0b24a779b69b11b9e10269abc.svg)](https://classroom.github.com/online_ide?assignment_repo_id=22939935&assignment_repo_type=AssignmentRepo)
# Lab02 - Caracterización de osciladores (externo vs. interno)


## 1. Integrantes

* [Santiago Leonardo Molina Bogotá](https://github.com/SaintGao-cmd)

* [Hellen Julieth Rincón Orjuela](https://github.com/hellenjurinconor-crypto)

* [Miguel Angel Tarazona Peinado](https://github.com/miguetarazona) 

## 2. Documentación

### 2.1 Descripción del laboratorio

### 2.2 Explicación del código implementado

## 2.2 Explicación del código implementado

El siguiente código configura el oscilador según el modo seleccionado (interno, cristal HS o RC externo) y hace parpadear un LED en RC0 con un período de 1 segundo (frecuencia de 1 Hz). También permite activar el PLL para multiplicar la frecuencia por 4 en los modos que lo soportan.

```c
#include <xc.h>
#include <stdint.h>

// ================ CONFIGURACIÓN GENERAL ==================
#pragma config WDTEN  = OFF
#pragma config LVP    = OFF
#pragma config PBADEN = OFF
#pragma config CP0 = OFF, CP1 = OFF, CP2 = OFF, CP3 = OFF
#pragma config BOREN  = OFF
#pragma config FCMEN  = OFF
#pragma config IESO   = OFF

// =================== MODO DE OSCILADOR ===================
// 1 = INTOSC interno
// 2 = Cristal externo HS
// 3 = RC externo
#define MODE    3
#define USE_PLL 0

#if MODE == 1
    #pragma config FOSC = INTIO67
#elif MODE == 2
    #pragma config FOSC = HSHP   // RA6=OSC2, RA7=OSC1 reservados para cristal
#elif MODE == 3
    #pragma config FOSC = RC
#else
    #error "Modo de oscilador inválido. Usa 1, 2 o 3."
#endif

// =============== FRECUENCIA DEL OSCILADOR ================
// Ajusta este valor al cristal físico que tengas conectado
#if MODE == 1
    #if USE_PLL
        #define _XTAL_FREQ 64000000UL   // 16 MHz * 4 (PLL)
    #else
        #define _XTAL_FREQ 16000000UL   // 16 MHz interno
    #endif
#elif MODE == 2
    #if USE_PLL
        #define _XTAL_FREQ 64000000UL   // 16 MHz cristal * 4 (PLL)
    #else
        #define _XTAL_FREQ 16000000UL   // Cambia según tu cristal (4/8/16/20 MHz)
    #endif
#elif MODE == 3
        #define _XTAL_FREQ 16000000UL       // Ajustar según R y C del circuito RC
#endif

// ====================== FUNCIONES ========================
void delay_ms(uint16_t ms) {
    while(ms--) {
        __delay_ms(1);
    }
}

void init_oscillator(void) {
#if MODE == 1
    // Oscilador interno: configurar OSCCON por software
    OSCCONbits.IRCF = 0b111;        // 16 MHz
    while(!OSCCONbits.HFIOFS);      // Esperar estabilización

    #if USE_PLL
        OSCCONbits.SPLLEN = 1;      // Activar PLL x4
    #endif

#elif MODE == 2
    // Cristal externo: el hardware espera solo la estabilización.
    // No se necesita configurar OSCCON.
    // El PIC no ejecuta código hasta que el cristal esté estable.
    #if USE_PLL
        OSCCONbits.SPLLEN = 1;      // PLL solo si se solicita
    #endif

#elif MODE == 3
    // RC externo: sin configuración de oscilador por software.
    // La frecuencia depende de los valores R y C del circuito.
    // Solo ajustar _XTAL_FREQ para que los delays sean correctos.
#endif
}

void init_pins(void) {
    // Poner puertos a digital
    ANSELA = 0x00;
    ANSELB = 0x00;
    ANSELC = 0x00;

    // RC0: salida para LED
    TRISCbits.TRISC0 = 0;
    LATCbits.LATC0   = 0;

    // RA6: solo disponible en MODE 1 (sin cristal en RA6/RA7)
    // En MODE 2, RA6=OSC2 y RA7=OSC1 están reservados para el cristal.
    // Configurarlos como I/O dañaría la señal del oscilador.
#if MODE == 1
    TRISAbits.TRISA6 = 0;
    LATAbits.LATA6   = 0;
#endif
}

// ================== PROGRAMA PRINCIPAL ===================
void main(void) {
    init_oscillator();  // Primero el clock
    init_pins();        // Luego los pines

    while(1) {
        LATCbits.LATC0 = 1;
        delay_ms(500);      // 500 ms encendido
        LATCbits.LATC0 = 0;
        delay_ms(500);      // 500 ms apagado
                            // Período total = 1 segundo → 1 Hz
    }
}

### 2.3 Análisis y comparación

#### Tabla 1: Medición en frío

| Modo de oscilador | Freq. teórica Fosc | RA6 medible (CLKO)? | Freq. medida RA6 (Hz) | Freq. teórica RC0 (Hz)| Freq. medida RC0 (Hz) | Error RC0 (%) |  
|------------------|------------------|---------------------|---------------|---------------------|---------------|---------------|
| INTOSC (interno) | 16,000,000       | Sí                 |                     |                500                 |               |               | |
| HS (cristal externo 16 MHz) | 16,000,000 | No |     NA      |               500                 |               |               |
| RC externo       | ~16,000,000*     | No                                    |       N/A        | 500                 |               |               | |

#### Tabla 2: Medición con calor

| Modo de oscilador | Freq. teórica Fosc | RA6 medible (CLKO)? | Freq. medida RA6 (Hz) | Freq. teórica RC0 (Hz)| Freq. medida RC0 (Hz) | Error RC0 (%) |  
|------------------|------------------|---------------------|---------------|---------------------|---------------|---------------|
| INTOSC (interno) | 16,000,000       | Sí                 |                     |                500                 |               |               | |
| HS (cristal externo 16 MHz) | 16,000,000 | No |     NA      |               500                 |               |               |
| RC externo       | ~16,000,000*     | No                                    |       N/A        | 500                 |               |               | |

#### Tabla 3: Deriva

| Modo de oscilador |RC0 deriva (Hz) |
|------------------|--------------------|
| INTOSC (interno) |                    |                
| HS (cristal externo 16 MHz) |                |                |
| RC externo       |                 |                


<!-- Agregar tablas para valores usando PLL -->

<!-- Complemente con análisis de lo registrado en tablas -->

## 2.4 Diagramas

## 2.5 Formas de onda

### INTOSC (interno) 


### HS

## RC

## 3. Evidencias de implementación

## 4. Preguntas

* ¿En qué modo se obtuvo la medición más cercana a la frecuencia teórica?

* ¿Fue posible evidenciar el fenómeno de deriva? ¿Qué factores podrían explicar la variación de frecuencia al calentar el PIC?

* ¿Cuál es más preciso en cuanto a frecuencia teórica vs. medida?


* Explique cómo usar RC0 para estimar la frecuencia del oscilador cuando RA6 no está disponible.

* Si se quisiera duplicar la frecuencia del PIC usando PLL, ¿en qué modos se podría aplicar?

* Enliste ventajas y desventajas de cada modo.

## 5. Referencias