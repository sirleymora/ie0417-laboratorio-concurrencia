## Parte 1:

Se ejecutó el programa 5 veces. En todas las ejecuciones el orden de los mensajes fue distinto, y en varias algunas líneas salieron mezcladas (por ejemplo, "Hola desde el hilo Hola desde el hilo 1Hola desde el hilo 3").

1. **¿Los mensajes aparecen siempre en el mismo orden?** 
No. El orden cambió en cada ejecución, y el hilo 0 no siempre imprimió primero aunque fue el primero en crearse.

2. **¿Por qué podría cambiar el orden de impresión?** 
Porque el sistema operativo decide cuándo y en qué núcleo corre cada hilo, y esa decisión cambia en cada ejecución. Los hilos arrancan casi al mismo tiempo y compiten por usar `std::cout`. Como `std::cout` es un recurso compartido sin protección, incluso el texto de dos mensajes se puede mezclar.

3. **¿Qué función cumple `join()`?** 
Hace que el hilo que lo llama espere hasta que el hilo asociado termine. Así `main` no imprime "Todos los hilos finalizaron." ni termina el programa hasta que los 5 hilos acaben.

4. **¿Qué podría pasar si no se llama a `join()`?** 
Si un `std::thread` sigue "joinable" cuando se destruye, se llama a `std::terminate()` y el programa se aborta. Además, `main` podría terminar antes que los hilos, y estos quedarían sin ejecutarse completos.

## Parte 2:

Se ejecutó el programa 5 veces (4 hilos × 1 000 000 de incrementos):

| Ejecución | Valor esperado | Valor obtenido |
|-----------|----------------|----------------|
| 1 | 4000000 | 1040119 |
| 2 | 4000000 | 1315378 |
| 3 | 4000000 | 1713341 |
| 4 | 4000000 | 1390519 |
| 5 | 4000000 | 1208515 |

1. **¿El valor obtenido siempre coincide con el esperado?** 
No. En ninguna de las 5 ejecuciones coincidió, y el resultado fue distinto cada vez (entre aproximadamente 1.0 y 1.7 millones, en lugar de 4 millones).

2. **¿Por qué se pierden incrementos?** 
Porque dos o más hilos pueden leer el mismo valor de `contador`, sumarle 1 cada uno por separado y escribir el mismo resultado. Así dos incrementos terminan contando como uno y se pierde una actualización.

3. **¿La operación `contador++` es realmente una sola operación a nivel de CPU?** 
No. Equivale a tres pasos: leer `contador` de memoria a un registro, sumarle 1 y escribir el resultado de vuelta en memoria. El sistema puede cambiar de hilo entre esos pasos, así que no es atómica.

4. **¿Qué problema de concurrencia se está observando?** 
Una condición de carrera: varios hilos modifican una variable compartida sin sincronización y el resultado depende del orden en que se intercalan.

## Parte 3:

Se ejecutó el programa 5 veces:

| Ejecución | Valor esperado | Valor obtenido |
|-----------|----------------|----------------|
| 1 | 4000000 | 4000000 |
| 2 | 4000000 | 4000000 |
| 3 | 4000000 | 4000000 |
| 4 | 4000000 | 4000000 |
| 5 | 4000000 | 4000000 |

En todas las ejecuciones el valor obtenido coincidió con el esperado, a diferencia de la Parte 2, donde nunca coincidió.

1. **¿Qué cambió con respecto al programa anterior?** 
Se agregó `#include <mutex>`, un `std::mutex` global (`mtx`) y, dentro del ciclo, un `std::lock_guard<std::mutex>` que protege la línea `contador++`.

2. **¿Qué hace `std::mutex`?** 
Es un mecanismo de exclusión mutua: solo un hilo a la vez puede tenerlo bloqueado. Los demás hilos que intentan bloquearlo esperan hasta que se libere, así solo uno ejecuta la sección crítica.

3. **¿Qué hace `std::lock_guard`?** 
Bloquea el mutex al crearse y lo desbloquea automáticamente al destruirse, es decir, al salir del bloque donde se declaró. Así no se puede olvidar el `unlock()`.

4. **¿Por qué ahora el resultado sí debería ser correcto?** 
Porque la lectura, la suma y la escritura de `contador` se hacen completas dentro de la sección crítica, sin que otro hilo entre en medio. Cada incremento se aplica sobre el valor más reciente, así que no se pierden actualizaciones.

5. **¿Qué desventaja podría tener proteger cada incremento individual con un mutex?** 
Un costo de rendimiento: se bloquea y desbloquea 4 millones de veces, los hilos pasan mucho tiempo esperándose entre sí y el trabajo queda casi serializado, perdiendo el beneficio del paralelismo. Sería más eficiente que cada hilo acumule en una variable local y sume al contador compartido una sola vez al final, o usar `std::atomic<int>`.

## Parte 4:

Se ejecutó el programa 5 veces (100 000 000 de elementos, 4 hilos, compilado con `-O2`):

| Ejecución | Tiempo secuencial | Tiempo paralelo | ¿Cuál fue más rápido? |
|-----------|-------------------|-----------------|-----------------------|
| 1 | 72 ms | 37 ms | Paralelo |
| 2 | 72 ms | 37 ms | Paralelo |
| 3 | 73 ms | 29 ms | Paralelo |
| 4 | 66 ms | 26 ms | Paralelo |
| 5 | 67 ms | 43 ms | Paralelo |

Promedio: secuencial = 70 ms y paralelo = 34.4 ms, es decir, una aceleración de aproximadamente 2.0×.

1. **¿El resultado secuencial y el paralelo son iguales?** 
Sí. En todas las ejecuciones ambos dieron 100000000, que es lo esperado al sumar 100 millones de unos.

2. **¿La versión paralela siempre fue más rápida?** 
En estas 5 ejecuciones sí, pero no es una garantía. Depende del tamaño del problema, de los núcleos disponibles y de la carga del sistema en ese momento. Además, el tiempo paralelo varió bastante entre ejecuciones de 26 a 43 ms.

3. **¿Por qué dividir el vector en bloques permite paralelizar el trabajo?** 
Porque cada bloque es independiente, cada hilo suma un rango distinto del vector y escribe en su propia posición de `resultados`. No hay datos compartidos que se modifiquen, así que no se necesita sincronización y los hilos pueden trabajar a la vez en distintos núcleos. Al final solo se suman los resultados parciales.

4. **¿Qué costos adicionales tiene la versión paralela?** 
Crear y destruir hilos, hacer `join`, repartir el trabajo y combinar los resultados parciales. También puede haber competencia por el ancho de banda de memoria y por la caché, y el código es más complejo.

5. **¿Qué podría pasar si el vector fuera muy pequeño?** 
Que la versión paralela fuera más lenta que la secuencial, porque el costo de crear los hilos sería mayor que el tiempo de la suma misma, y no compensaría repartir el trabajo.

## Parte 5:

El programa se ejecutó 3 veces. El equipo tiene 8 núcleos físicos y 16 procesadores lógicos (`Get-CimInstance Win32_Processor`), y `std::thread::hardware_concurrency()` reportó 16.

| Cantidad de hilos | Corrida 1 | Corrida 2 | Corrida 3 | Promedio |
|-------------------|-----------|-----------|-----------|----------|
| 1 | 73 ms | 77 ms | 86 ms | 78.7 ms |
| 2 | 54 ms | 57 ms | 49 ms | 53.3 ms |
| 4 | 37 ms | 39 ms | 32 ms | 36.0 ms |
| 8 | 40 ms | 35 ms | 31 ms | 35.3 ms |
| 16 | 37 ms | 38 ms | 22 ms | 32.3 ms |

**Conclusión breve:** 
El tiempo mejoró claramente al pasar de 1 a 2 hilos (de 79 a 53 ms) y de 2 a 4 hilos (de 53 a 36 ms). A partir de 4 hilos la mejora casi desaparece: los promedios con 4, 8 y 16 hilos (36.0, 35.3 y 32.3 ms) están muy cerca y dentro de la variación que hay entre corridas de una misma configuración. La suma de un vector es una tarea muy corta y limitada por el acceso a memoria, por lo que agregar más hilos no sigue reduciendo el tiempo de forma proporcional.

1. **¿Cuál cantidad de hilos produjo el mejor tiempo?** 
El mejor tiempo individual fue de 22 ms con 16 hilos en la corrida 3, y 16 hilos también tuvo el mejor promedio de 32.3 ms. Sin embargo, esa ventaja sobre 4 y 8 hilos es pequeña, así que lo más claro es que la mejora real se logra hasta unos 4 a 8 hilos.

2. **¿El tiempo mejoró siempre al aumentar los hilos?** 
No. Mejoró de 1 a 2 y de 2 a 4 hilos, pero de 4 en adelante prácticamente se estancó. En la corrida 1, por ejemplo, 8 hilos (40 ms) fue más lento que 4 hilos (37 ms), y en la corrida 2, 16 hilos (38 ms) fue más lento que 8 hilos (35 ms).

3. **¿Cuántos núcleos tiene la computadora donde se ejecutó el programa?**
8 núcleos físicos y 16 procesadores lógicos.

5. **¿Qué ocurre cuando se usan más hilos que núcleos disponibles?**
Los hilos no pueden correr todos a la vez: el sistema operativo los turna en los núcleos disponibles. En este equipo, con hasta 16 hilos todos caben en los 16 procesadores lógicos, pero si se usaran más de 16, habría hilos esperando su turno.

6. **¿Qué relación tiene esto con el context switching?** 
Cada vez que el sistema cambia el hilo que corre en un núcleo hace un cambio de contexto: guarda el estado del hilo actual y carga el del siguiente. Ese cambio no hace trabajo útil y además afecta la caché, así que cuantos más hilos compiten por los mismos núcleos, más cambios de contexto y más tiempo perdido en administración.

7. **¿Por qué la versión con 16 hilos podría no ser la mejor?** 
Porque cada hilo adicional agrega costos de creación y administración, y a partir de cierto punto los hilos compiten por el ancho de banda de memoria y por la caché sin que haya más núcleos físicos que aprovechar. Además, los 16 procesadores lógicos comparten recursos de 8 núcleos físicos, por lo que no rinden como 16 núcleos reales. En esta prueba 16 hilos resultó competitivo, pero su ventaja sobre 4 u 8 hilos fue pequeña e inconsistente, y con más hilos que procesadores lógicos o con tareas más cortas el exceso de hilos suele perjudicar.

## Parte 6:

Al ejecutar `parte6_deadlock` la salida fue:

```text
Hilo A intentando tomar recurso 1...
Hilo B intentando tomar recurso 2...
Hilo B intentando tomar recurso 1...
Hilo A intentando tomar recurso 2...
```

Después de esto el programa quedó detenido, nunca imprimió "obtuvo ambos recursos" y hubo que cortarlo con `Ctrl + C`.

1. **¿El programa termina normalmente?** 
No. Se queda bloqueado indefinidamente y hay que detenerlo manualmente con `Ctrl + C`.

2. **¿Qué recurso tomó primero el hilo A?** 
El recurso 1 (`recurso1`).

3. **¿Qué recurso tomó primero el hilo B?** 
El recurso 2 (`recurso2`).

4. **¿Por qué ninguno de los dos hilos puede continuar?** 
Porque cada uno retiene un recurso y espera el que tiene el otro: A tiene el recurso 1 y espera el 2, mientras B tiene el 2 y espera el 1. Ninguno libera lo que tiene hasta conseguir el segundo recurso, así que ninguno avanza. El `sleep_for(100 ms)` da tiempo a que ambos tomen su primer recurso antes de pedir el segundo, lo que hace que el bloqueo ocurra casi siempre.

5. **¿Qué significa espera circular?** 
Es una situación en la que cada hilo de un grupo espera un recurso que tiene el siguiente, formando un ciclo (A espera a B y B espera a A). Es una de las condiciones necesarias del deadlock, junto con exclusión mutua, retención y espera, y no apropiación.

6. **¿Cómo se podría evitar este problema?** 
Haciendo que todos los hilos tomen los recursos en el mismo orden, usando `std::scoped_lock` (o `std::lock`) para bloquear varios mutex a la vez, usando `try_lock` con reintentos o tiempos de espera, o reduciendo la cantidad de recursos que se mantienen bloqueados al mismo tiempo.

### Versión corregida

Se compiló y ejecutó `parte6_deadlock_corregido.cpp` 3 veces. En todas terminó normalmente y ambos hilos obtuvieron los recursos.

```text
--- ejecución 1
Hilo B obtuvo ambos recursos de forma segura.
Hilo A obtuvo ambos recursos de forma segura.
--- ejecución 2
Hilo A obtuvo ambos recursos de forma segura.
Hilo B obtuvo ambos recursos de forma segura.
--- ejecución 3
Hilo A obtuvo ambos recursos de forma segura.
Hilo B obtuvo ambos recursos de forma segura.
```

**Pregunta final: ¿por qué `std::scoped_lock` ayuda a evitar este deadlock?** 
Porque `std::scoped_lock` recibe ambos mutex y los bloquea juntos usando un algoritmo de evitación de deadlock: si no puede obtener los dos, libera el que haya tomado y reintenta, en lugar de quedarse reteniendo uno mientras espera el otro. Así se rompe la condición de "retención y espera" y no se forma la espera circular, sin importar en qué orden se listen los mutex. Además, al ser RAII, los libera automáticamente al salir del ámbito.

## Parte 7: Reflexión final

1. **¿Cuál fue la diferencia más clara que observó entre ejecución secuencial y ejecución con hilos?** 
En la Parte 4, la suma con 4 hilos tardó en promedio 34.4 ms contra 70 ms de la versión secuencial, es decir, cerca de 2 veces más rápida. También que los hilos no tienen un orden predecible de ejecución, como se vio en la Parte 1.

2. **¿Qué es una condición de carrera?** 
Es un error que ocurre cuando dos o más hilos acceden al mismo dato compartido al mismo tiempo, al menos uno lo modifica y no hay sincronización, de modo que el resultado depende del orden en que se intercalan los hilos.

3. **¿Por qué `contador++` puede fallar cuando muchos hilos lo ejecutan al mismo tiempo?** 
Porque no es una operación atómica, se compone de leer, sumar y escribir. Si dos hilos leen el mismo valor antes de que alguno escriba, ambos escriben el mismo resultado y se pierde un incremento. En la Parte 2 se obtuvieron valores entre 1.0 y 1.7 millones en lugar de 4 millones.

4. **¿Qué problema resuelve un mutex?** 
Garantiza exclusión mutua en una sección crítica, solo un hilo a la vez puede ejecutarla, lo que evita condiciones de carrera sobre datos compartidos.

5. **¿Qué ventaja tiene `std::lock_guard` sobre llamar manualmente a `lock()` y `unlock()`?** 
Libera el mutex automáticamente al salir del bloque (RAII), incluso si hay un `return` temprano o una excepción. Así no se olvida el `unlock()` y se evitan bloqueos permanentes por error.

6. **¿Por qué más hilos no siempre significan mejor rendimiento?** 
Porque la ganancia está limitada por los núcleos disponibles y por la parte del trabajo que se puede paralelizar, y cada hilo agrega costos de creación, cambios de contexto, sincronización y competencia por memoria. En la Parte 5 el tiempo mejoró de 1 a 4 hilos de 79 a 36 ms, pero de 4 a 16 hilos casi no hubo mejora.

7. **¿Qué es un deadlock?** 
Es un bloqueo mutuo en el que dos o más hilos esperan indefinidamente recursos que tiene otro hilo del grupo, de modo que ninguno puede continuar. En la Parte 6, A tenía el recurso 1 y esperaba el 2, y B tenía el 2 y esperaba el 1.

8. **¿Qué buenas prácticas aplicaría al programar con hilos?**
   - Compartir la menor cantidad posible de datos entre hilos; preferir datos locales por hilo y combinar resultados al final.
   - Proteger los datos compartidos con mutex (con `std::lock_guard` o `std::scoped_lock`) o usar `std::atomic` en casos simples.
   - Mantener las secciones críticas lo más cortas posible.
   - Tomar varios mutex siempre en el mismo orden o con `std::scoped_lock`.
   - Hacer `join()` de todos los hilos antes de terminar.
   - Ajustar la cantidad de hilos a los núcleos disponibles y medir antes de asumir que paralelizar mejora el rendimiento.

9. **¿En qué tipo de programas reales podría ser útil la programación concurrente?** 
En programas que deben atender varias tareas a la vez o no bloquearse mientras esperan: servidores web y de bases de datos que atienden muchos clientes, interfaces gráficas que siguen respondiendo mientras hacen otra tarea, aplicaciones con operaciones de entrada y salida (red, disco) y sistemas embebidos que manejan sensores y comunicaciones simultáneamente.

10. **¿En qué tipo de programas reales podría ser útil la programación paralela?** 
En programas con mucho cómputo que se puede dividir en partes independientes: procesamiento de imágenes y video, simulaciones científicas, análisis de grandes volúmenes de datos, entrenamiento de modelos de aprendizaje automático, renderizado de gráficos y cálculos con matrices.
