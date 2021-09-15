---
permalink: /sandbox
title: Seguridad
toc: true
---

Luau es seguro de integrar. En términos generales, esto significa que a pesar del código no confiable (y en el caso de Roblox, activamente malicioso), el lenguaje y la librería estándar no permite ningún acceso inseguro al sistema subyacente, y no contiene ningún bug que permita salir del entorno seguro (ej. para obtener la ejecución de código nativo mediante artefactos con Programación Orientada al Retorno et al). Adicionalmente, la MV (Máquina Virtual) proporciona funciones extra para implementar aislamiento entre el código privilegiado y el código no privilegiado, y proteger a uno del otro; esto es importante si el entorno integrado (Roblox) decide exponer algunas APIs que pueden ser inseguras de llamar desde código no confiable, por ejemplo porque ellos proporcionan acceso controlado al sistema subyacente o arriesgan exposición de PII (Información de Identificación Personal) mediante huellas dactilares, etc.

Esta seguridad se logra mediante la combinación de retirar funciones de la librería estándar que son inseguras, agregar funciones a la MV que hacen posible implementar seguridad y aislamiento, y asegurarse que la implementación está a salvo de problemas de seguridad de la memoria usando fuzzing (pruebas de validación de datos).

Por supuesto, desde que la pila completa es implementada en C++, la seguridad no es formalmente demostrada - en teoría, el compilador o la librería estándar puede tener vulnerabilidades aprovechables. En pruebas estas son encontradas y arregladas rápidamente. Mientras implementar la pila en un lenguaje más seguro tal como Rust facilitaría proporcionar estas garantías, en nuestro conocimiento (basado en el código existente) esto haría imposible alcanzar el nivel de rendimiento requerido.

## Librería 

Partes de la librería estándar Lua 5.x son inseguras. Algunas de las funciones proporcionan acceso al sistema operativo anfitrión, incluyendo procesos de ejecución y lectura de archivos. Algunas funciones carecen de suficientes verificaciones de seguridad de la memoria. Algunas funciones son seguras si todo el código no es confiable, pero pueden romper la barrera de aislamiento entre código confiable y no confiable.

Las siguientes librerías y funciones globales han sido retiradas como resultado:

- la librería `io.` fue retirada completamente, debido a que le da acceso a archivos y permite ejecutar procesos.
- la librería `package.` fue retirada completamente, debido a que le da acceso a archivos y permite cargar módulos nativos.
- la librería `os.` fue eliminada del acceso a archivos y funciones del entorno (`execute`, `exit`, etc.). Las únicas funciones compatibles en la librería son `clock`, `date`, `difftime` y `time`.
- la librería `debug.` fue retirada en un largo alcance, debido a que este tiene funciones que no son seguras para la memoria y otras funciones rompen el aislamiento; las únicas funciones compatibles son `traceback` ~~y `getinfo` (con una funcionalidad reducida)~~.
- `dofile` y `loadfile` permitían el acceso a archivos del sistema y fueron retiradas.

Para lograr la seguridad de la memoria, el acceso a la función bytecode fue retirada. El Bytecode es complicado de validar y usar bytecode no confiable puede conducir a exploits. Por lo tanto, `loadstring` no funciona con entradas de bytecode, y `string.dump`/`load` fueron retiradas debido a que ya no son necesarias. Cuando se integra Luau, el bytecode debería ser encriptado para prevenir ataques mediante intermediarios también, mientras la MV asume que el bytecode fue generado por el compilador de Luau (el cual nunca produce bytecode invalido/inseguro).

Finalmente, para hacer el aislamiento posible dentro de la misma MV, las siguientes funciones globales fueron reducidas en su funcionalidad:

- `collectgarbage` solo funciona con el argumento `"count"`, ya que modificar el estado del GC (recolector de basura) puede interferir con lo esperado de otro código que se ejecuta en el proceso. Tal como,`collectgarbage()` se convirtió en una versión inferior de `gcinfo()` y está obsoleta.
- `newproxy` solo funciona con los argumentos `true`/`false`/`nil`.
- `module` permitía anular paquetes globales y como resultado fue retirada.

> Nota: `getfenv`/`setfenv` resultan en desafíos de aislamiento adicionales, debido a que permiten introducir globales dentro de scripts en la pila de llamadas. Idealmente, estas también deberían ser deshabilitadas, pero desafortunadamente la comunidad de Roblox depende de estas por varias razones. Esto puede ser mitigado limitando la interacción entre código confiable y no confiable, y/o usando MV separadas.

## Entorno

La modificación a las funciones de la librería es suficiente para hacer la integración segura, pero no es suficiente para proveer aislamiento dentro de la misma MV. Debe ser señalado que para lograr un aislamiento garantizado, es recomendable cargar código confiable y no confiable en MVs separadas; sin embargo, incluso dentro de la misma MV, Luau provee funciones seguras adicionales para hacer el aislamiento menos costoso.

Cuando se inicializa la tabla de globales predeterminadas, las tablas son protegidas de ser modificadas:
  
- Todas las librerías (`string`, `math`, etc.) son marcadas como solo lectura
- La metatabla string es marcada como solo lectura
- La tabla global en sí misma es marcada como solo lectura

Esto es usando la función de la MV que no es accesible desde los scripts, que evita todas las escrituras en la tabla, incluyendo asignaciones, `rawset` y `setmetatable`. Esto asegura que las globales no puedan ser monkey-patched (cambiar clases o módulos durante su ejecución) en su lugar, y solo pueden ser sustituidas mediante `setfenv`.

De por sí esto significa que el código que se ejecuta en Luau no puede usar globales en lo absoluto, ya que asignar globales fallaría. Mientras esto es factible, en Roblox resolvimos esto creando una nueva tabla global para cada script, que usa `__index` para señalar a la tabla global incorporada. Esto asegura las globales integradas mientras aún permite escribir globales desde cada script. Esto también significa que además de exponer globales especiales compartidas del host, todos los scripts están aislados entre sí.

## Identidad del hilo

La seguridad al nivel del entorno es suficiente para implementar la distancia entre el código confiable y no confiable, asumiendo que `getfenv`/`setfenv` no están disponibles (eliminadas de las globales), o que el código confiable nunca interactúa con el código no confiable (lo cual previene que el código no confiable tenga acceso a funciones confiables). Cuando se ejecuta código confiable, es posible inyectar globales extra desde el host dentro de la tabla global, proporcionando acceso a APIs especiales.

Sin embargo, en algunos casos es conveniente restringir el acceso a funciones que son expuestas a ambos código confiable y no confiable. Por ejemplo, ambos códigos podrían tener acceso a la global `game`, pero `game` podría exponer métodos que solo deberían funcionar desde código confiable.

Para lograr esto, cada hilo en Luau tiene una identidad de seguridad, las cuales solo pueden ser establecidas por el host. Los nuevos hilos creados heredan identidades del hilo principal, y las funciones expuestas desde el host pueden validar la identidad del hilo llamado. Esto hace posible proveer APIs a códigos confiables mientras se limita el acceso a código no confiable.

> Nota: para lograr una mejor garantía de aislamiento entre código confiable y no confiable, es posible ejecutarlo en diferentes MVs de Luau, que es lo que hace Roblox para una seguridad extra.

## `__gc`

Lua 5.1 expone un metamétodo de `__gc` para los datos de usuario, el cual puede ser usado en proxies (`newproxy`) para conectarse al recolector de basura. Las últimas versiones de Lua extienden este mecanismo para que funcione en tablas.

Este mecanismo es malo para el rendimiento, seguridad de la memoria y aislamiento:

- En Lua 5.1, el soporte de `__gc` requiere recorrer listas de datos de usuario de forma redundante durante la recolección de basura para filtrar los objetos finalizables.
- En las últimas versiones de Lua, los datos de usuario que implementan `__gc` son separados en diferentes listas; sin embargo, la finalización prolonga el tiempo de vida de los objetos finalizados lo cual resulta en menos recuperación de memoria inmediata, y la destrucción en dos pasos resulta en 
pérdidas de caché extra para datos de usuario
- `__gc` se ejecuta durante la recolección de basura en el contexto de un hilo arbitrario, el cual invalida el mecanismo de identidad del hilo descrito anteriormente
- Los objetos pueden ser retirados de tablas débiles *después* de ser finalizadas, lo cual significa que acceder a estos objetos puede resultar en bugs en la seguridad de la memoria, a menos que todos los métodos de datos de usuario expuestos se protejan contra use-after-gc.
- Si el método `__gc` alguna vez se filtra a los scripts, estos pueden llamarlo directamente en un objeto y usar cualquier método expuesto por ese objeto después de esto. Eso significa que `__gc` y todos los otros métodos expuestos deben soportar la seguridad de la memoria cuando se solicitan en un objeto destruido.

Debido a estos problemas, Luau no soporta `__gc`. En cambio usa destructores basados en etiquetas que pueden llevar a cabo una limpieza adicional de memoria durante la destrucción de datos de usuario; crucialmente, estos solo están disponibles para el host (asi no pueden ser invocados manualmente), y estos se ejecutan justo antes de liberar el bloque de memoria de datos de usuario, lo cual es óptimo para el rendimiento y garantizado en ser seguro para la memoria.

Para monitorear el comportamiento del recolector de basura, la recomendación es usar tablas débiles en su lugar.

## Interrupciones

Además de prevenir el acceso al API, puede ser importante para el aislamiento limitar el uso de memoria y CPU del código que se ejecuta dentro de la MV.

Por defecto, no hay límites de memoria impuestos en el código que se ejecuta, así que es posible agotar el espacio de direcciones del host; esto es fácil de configurar desde el host de las asignaciones de Luau, pero por supuesto con una superficie de API enriquecida expuesta por el host, es difícil eliminar esto como posibilidad. El agotamiento de la memoria no resulta en problemas en la seguridad de la memoria o ningún riesgo en particular con el sistema que está ejecutando los procesos del host, que no sea el proceso del host que el sistema operativo termina.

Limitar el uso de la CPU puede ser igualmente desafiante con una API enriquecida. Sin embargo, Luau provee una función a nivel de la MV para intentar contener runaway scripts lo cual permite que sea posible finalizar cualquier script externamente. Esto funciona mediante un mecanismo de interrupción global, donde el host puede configurar un controlador de interrupción en cualquier momento, y cualquier código de Luau es garantizado de llamar “eventualmente” este controlador (en la práctica esto puede llegar a pasar en cualquier llamado de función o en cualquier iteración de loop). Esto aún deja la posibilidad de que un script que se esté ejecutando por mucho tiempo se abra y logre encontrar una manera de llamar una sola función C que toma mucho tiempo, pero a falta de eso la interrupción es muy pronta.

Roblox configura un controlador de interrupción usando un watchdog (Perro Guardián) que:

- Limita el tiempo de ejecución de cualquier script en Studio a 10 segundos (configurable mediante los ajustes del Studio) 
- Tras el cierre del cliente, se interrumpe la ejecución de cada script en ejecución 1 segundo después del cierre
