# Laboratorio de programación concurrente y paralela en C++

## Estudiante

Nombre: Sirley Mora Chavarría

Carné: C15059

## Descripción

Este laboratorio explora conceptos básicos de programación concurrente y paralela usando C++: creación de hilos con `std::thread`, condiciones de carrera, sincronización con `std::mutex` y `std::lock_guard`, mediciones de rendimiento con `std::chrono`, la relación entre cantidad de hilos y rendimiento, y un ejemplo de deadlock con su corrección mediante `std::scoped_lock`.

Las tablas de resultados y las respuestas a las preguntas de análisis están en [`resultados/analisis.md`](resultados/analisis.md).

## Estructura

```text
laboratorio-concurrencia/
├── README.md
├── src/
│   ├── parte1_threads.cpp
│   ├── parte2_race_condition.cpp
│   ├── parte3_mutex.cpp
│   ├── parte4_benchmark.cpp
│   ├── parte5_hilos_vs_rendimiento.cpp
│   ├── parte6_deadlock.cpp
│   └── parte6_deadlock_corregido.cpp
└── resultados/
    └── analisis.md
```

## Requisitos

- g++
- C++17 o superior
- Sistema operativo Linux, macOS o Windows con soporte para compilación C++

Los comandos se ejecutan desde la carpeta `laboratorio-concurrencia`. En Windows (PowerShell), el ejecutable se corre con `.\parte1.exe` en lugar de `./parte1`.

## Compilación y ejecución

### Parte 1

```bash
g++ -std=c++17 -Wall -Wextra -pthread src/parte1_threads.cpp -o parte1
./parte1
```

### Parte 2

```bash
g++ -std=c++17 -Wall -Wextra -pthread src/parte2_race_condition.cpp -o parte2
./parte2
```

### Parte 3

```bash
g++ -std=c++17 -Wall -Wextra -pthread src/parte3_mutex.cpp -o parte3
./parte3
```

### Parte 4

```bash
g++ -std=c++17 -O2 -Wall -Wextra -pthread src/parte4_benchmark.cpp -o parte4
./parte4
```

### Parte 5

```bash
g++ -std=c++17 -O2 -Wall -Wextra -pthread src/parte5_hilos_vs_rendimiento.cpp -o parte5
./parte5
```

### Parte 6

Este programa está diseñado para bloquearse (deadlock); se detiene con `Ctrl + C`.

```bash
g++ -std=c++17 -Wall -Wextra -pthread src/parte6_deadlock.cpp -o parte6
./parte6
```

Versión corregida con `std::scoped_lock`, que termina normalmente:

```bash
g++ -std=c++17 -Wall -Wextra -pthread src/parte6_deadlock_corregido.cpp -o parte6_corregido
./parte6_corregido
```

## Conclusiones

Varios hilos permiten aprovechar mejor los núcleos del procesador, pero compartir datos sin sincronización produce resultados incorrectos o sea la condición de carrera. Un mutex lo corrige, aunque proteger cada operación pequeña con un candado tiene un costo, y tomar varios candados en distinto orden puede causar un deadlock, que se evita con `std::scoped_lock`. Además, más hilos no siempre significan más velocidad, en las pruebas, el tiempo mejoró hasta unos 4 hilos y después casi no cambió, porque la mejora depende de los núcleos disponibles, del tamaño del trabajo y de los costos de crear y administrar hilos.
