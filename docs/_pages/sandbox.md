---
permalink: /sandbox
title: Seguridad
toc: true
---

Luau es seguro de integrar. Hablando en general, esto significa que en la cara de el código no confiable(y en el caso de Roblox, activamente malicioso), el lenguaje y la librería standard no permite ningún acceso inseguro a el sistema subyacente, y no contiene ningún bug que permite salir de la seguridad (ej. para obtener la ejecución de código nativo mediante artilugios ROP (Programación Orientada al Retorno) et al). Adicionalmente, la VM (Máquina Virtual) proporciona funciones extra para implementar aislamiento de código privilegiado desde código no privilegiado y proteger a uno del otro; esto es importante si el ambiente integrado (Roblox) decide exponer algunas APIs que pueden ser inseguras de llamar desde código no confiable, por ejemplo porque ellos proporcionan acceso controlado a el sistema subyacente o arriesgar exposición de PII (Información de Identificación Personal) mediante huellas dactilares, etc.

Esta seguridad es lograda mediante una combinación de retirar funciones de la librería estándar que son inseguras, agregando funciones a la MV que hacen posible implementar seguridad y aislamiento, y asegurándose que la implementación es segura de seguridad en la memoria usando fuzzing (pruebas de validación de datos).

Por supuesto, desde que la pila completa es implementada en C++, la seguridad no es formalmente demostrada - en teoría, el compilador o la librería estándar puede tener vulnerabilidades aprovechables. En pruebas estas son encontradas y arregladas rápidamente. Mientras implementar la pila en un lenguaje más seguro tal como Rust facilitaría proporcionar estas garantías, en nuestro conocimiento (basado en el código existente) esto haría imposible alcanzar el nivel de rendimiento requerido.

## Librería 

Partes de la librería estándar Lua 5.x son inseguras. Algunas de las funciones proporcionan acceso a el sistema operativo anfitrión, incluyendo procesos de ejecución y lectura de archivos. Algunas funciones carecen de suficientes verificaciones de seguridad de la memoria. Algunas funciones son seguras si todo el código no es confiable, pero puede romper la barrera de aislamiento entre código confiable y no confiable.

Las siguientes librerías y funciones globales han sido retiradas como resultado:

- la librería `io.` fue retirada completamente, debido a que le da acceso a archivos y permite ejecutar procesos.
- la librería `package.` fue retirada completamente, debido a que le da acceso a archivos y permite cargar módulos nativos.
- la librería `os.` fue eliminada del acceso a archivos y funciones ambientales(`execute`, `exit`, etc.). Las únicas funciones compatibles en la librería son `clock`, `date`, `difftime` y `time`.
- la librería `debug.` fue retirada a un largo alcance, debido a que este tiene funciones que no son seguras para la memoria y otras funciones rompen el aislamiento; las únicas funciones compatibles son `traceback` ~~y `getinfo` (con una funcionalidad reducida)~~.
- `dofile` y `loadfile` permitían el acceso a archivos del sistema y fueron retiradas.

Para lograr la seguridad de la memoria, el acceso a la función bytecode fue retirada. El Bytecode es complicado de validar y usando bytecode no confiable puede dirigirse a exploits. Por lo tanto, `loadstring` no funciona con entradas de bytecode, y `string.dump`/`load` fueron retiradas debido a que ya no son necesarias. Cuando se integra Luau, el bytecode debería ser encriptado para prevenir ataques mediante intermediarios al igual que, la MV asume que el bytecode fue generado por el compilador de Luau (el cual nunca produce bytecode invalido/inseguro).

Finalmente, para hacer el aislamiento posible dentro de la MV, las siguientes funciones globales fueron reducidas en su funcionalidad:

- `collectgarbage` solo funciona con el argumento `"count"`, modificando el estado del GC(colector de basura) puede interferir con las expectaciones del otro código que se ejecuta en el proceso. Tal como,`collectgarbage()` se convirtió en una versión inferior de `gcinfo()` y está obsoleta.
- `newproxy` solo funciona con los argumentos `true`/`false`/`nil`.
- `module` permitía anular paquetes globales y fue retirada como resultado.

> Nota: `getfenv`/`setfenv` resultan en desafíos de aislamiento adicionales, debido a que permiten inyectar globales dentro de scripts en la pila de llamadas. Idealmente, estas deberían ser deshabilitadas también, pero desafortunadamente la comunidad de Roblox confía en estas por varias razones. Esto puede ser migrado limitando la interacción entre código confiable y no confiable, y/o usando MV separadas.

## Ambiente

La modificación a la librería de funciones es suficiente para hacer la integración segura, pero no es suficiente para proveer aislamiento dentro de la misma MV. Debe ser señalado que para lograr una aislamiento garantizado, es aconsejable cargar código no confiable y confiable en MV separadas; sin embargo, incluso dentro de la misma MV Luau provee adicionalmente funciones seguras para hacer el aislamiento menos costoso.

Cuando se inicializa la tabla de globales por defecto, las tablas son protegidas de modificación:
  
- Todas las librerías (`string`, `math`, etc.) son marcadas como sólo para lectura
- La metatabla string es marcada como solo para lectura
- La tabla global por sí misma es marcada como solo para lectura

Esto es usando la función de la VM que no es accesible por los scripts, que evita todas las escrituras en la tabla, incluyendo asignaciones, `rawset` y `setmetatable`. Esto se asegura de que las globales no pueden ser monkey-patched en su lugar, y pueden ser sólo sustituidas mediante `setfenv`.

Por sí mismo esto significa que el código que se ejecuta en Luau no puede usar globales en absoluto, 
ya que la asignar globales fallaría. Mientras esto es factible, en Roblox nosotros resolvimos esto creando una nueva tabla global para cada script, que usa `__index` para señalar a la tabla global incorporada. Esto asegura las globales integradas mientras aún permite escribir globales desde cada script. Esto también significa que, además de exponer globales compartidas especiales del host, todos los scripts están aislados entre sí.

## Identidad del hilo

La seguridad al nivel del ambiente es suficiente para implementar la separación entre el código confiable y no confiable, asumiendo que `getfenv`/`setfenv` no están disponibles (retiradas de las globales), o que el código confiable nunca interactúa con el código no confiable (el cual previene que el código no confiable tenga acceso a funciones confiables). Cuando se ejecuta código confiable, es posible inyectar globales extra desde el host dentro de la tabla global, proporcionando acceso a APIs especiales.

Sin embargo, en algunos casos es deseable restringir el acceso a funciones que son expuestas a código confiable y no confiable. Por ejemplo, ambos códigos podrían tener acceso a la global `game`, pero `game` puede exponer métodos que solo deberían funcionar desde código confiable.

Para lograr esto, cada hilo en Luau tiene una identidad de seguridad, las cuales solo pueden ser ajustadas por el host. Los nuevos hilos creados heredan identidades del hilo principal, y las funciones expuestas desde el host pueden validar la identidad del hilo llamado. Esto hace posible proveer APIs a códigos confiables mientras se limita el acceso a código no confiable.

> Nota: para lograr una mejor garantía de aislamiento entre código confiable y no confiable, es posible ejecutarlo en diferentes MVs de Luau, lo cual hace Roblox para una seguridad extra.

## `__gc`

Lua 5.1 expone un metamétodo de `__gc` para los datos de usuario, el cual puede ser usado en proxies (`newproxy`) para conectarse en el colector de basura. Las últimas versiones de Lua extienden este mecanismo para que funcione en tablas.

Este mecanismo es malo para el rendimiento, seguridad de la memoria y el aislamiento:

- En Lua 5.1, el soporte de `__gc` requiere atravesar listas de datos de usuario de forma redundante durante la recolección de basura para filtrar los objetos finalizables.
- En las últimas versiones de Lua, los datos de usuario que implementan `__gc` son separados en diferentes listas; sin embargo, la finalización prolonga el tiempo de vida de los objetos finalizados lo cual resulta en menos recuperación de memoria inmediata, y la destrucción en dos pasos resulta en 
pérdidas de caché extra para datos de usuario
- `__gc` se ejecuta durante la colección de basura en contexto de un hilo arbitrario el cual invalida el mecanismo de identidad del hilo descrito anteriormente
- Los objetos pueden ser retirados de tablas débiles *después* de ser finalizadas, lo cual significa que accediendo a estos objetos puede resultar en bugs en la seguridad de la memoria, a menos de que todos los métodos expuestos de los datos de usuario se protejan contra use-after-gc.
- Si el método `__gc` alguna vez se filtra a los scripts, estos pueden llamarlo directamente en un objeto y usar cualquier método expuesto por ese objeto después de esto. Eso significa que `__gc` y todos los otros métodos expuestos deben soportar la seguridad de la memoria cuando se solicita un objeto destruido.

Debido a estos problemas, Luau no soporta `__gc`. En cambio usa destructores basados en etiquetas que pueden llevar a cabo una limpieza de memoria adicional durante la destrucción de datos de usuario; crucialmente, estos son solo disponibles a el host (asi no pueden ser invocados manualmente), y estos se ejecutan justo después de liberar el bloque de memoria de datos de usuario el cual es óptimo para el rendimiento, y garantizado de ser seguro para la memoria.

Para monitorear el comportamiento del colector de basura, la recomendación es usar tablas débiles en su lugar.

## Interrupciones

En adición para prevenir el acceso a el API, puede ser importante para el aislamiento limitar el uso de memoria y CPU del código que se ejecuta dentro de la MV.

Por defecto, no hay límites de memoria impuestos en el código que se ejecuta, así que es posible agotar el espacio de direcciones del host; esto es fácil de configurar desde el host de las asignaciones de Luau, pero por supuesto con una superficie de API enriquecida expuesta por el host, es difícil eliminar esto como una posibilidad. El agotamiento de la memoria no resulta en problemas con la seguridad de la memoria o ningún riesgo en particular con el sistema que está ejecutando los procesos del host, que no sea el proceso del host que el sistema operativo termina.

Limitar el uso de la CPU puede ser igualmente desafiante con una rica API. Sin embargo, Luau provee una función a nivel de la MV para intentar contener runaway scripts lo cual permite que sea posible finalizar cualquier script externamente. Esto funciona mediante un mecanismo de interrupción global, donde el host puede configurar un controlador de interrupción en cualquier momento, y cualquier código de Luau es  “eventualmente” garantizado llamar este controlador (en práctica esto puede llegar a pasar en cualquier llamado de función o en cualquier iteración de loop). Esto aún deja la posibilidad de que un script que se esté ejecutando por mucho tiempo se abra y logre encontrar una manera de llamar una sola función C que lleva mucho tiempo, pero a falta de eso la interrupción es inmediata.

Roblox configura un controlador de interrupción usando un watchdog (Perro Guardián) que:

- Limita el tiempo de ejecución de cualquier script en Studio a 10 segundos (configurable mediante los ajustes del Studio) 
- Tras el cierre del cliente, se interrumpe la ejecución de cada script en ejecución 1 segundo después del cierre
