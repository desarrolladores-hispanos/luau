---
permalink: /compatibility
title: Compatibilidad
toc: true
---

Luau está basado en Lua 5.1, y como tal lleva todas las funciones de 5.1, excepto por algunas que tuvimos que quitar debido a limitaciones de seguridad. Por restricciones de compatibilidad, no eliminamos funciones obsoletas en versiones más recientes (p. ej. aún soportamos `getfenv`/`setfenv`). Versiones de Lua más recientes presentan nuevas funciones al lenguaje y nuevas bibliotecas/funciones.

Nuestra meta es incluir funciones de versiones más recientes de Lua cuando para nosotros tiene sentido hacerlo - las motivaciones detrás de algunas nuevas funciones no son claras o no aplican al dominio en el que se utiliza Luau, y muchas funciones no valen la pena implementar. El resto de este documento describe el estado de todas las funciones de Lua 5.2 y más allá, con la siguiente clasificación:

- ✔️ - esta función está disponible en Luau
- ❌ -  esta función no está disponible en Lua porque no creemos que tiene sentido incluirlo
- 😞 -  esta función no está disponible en Luau por motivos de compatibilidad/seguridad
- 🔜 - esta función aún no está disponible en Luau pero nos gustaría incluirla y posiblemente estamos trabajando en ella
- 🤷‍♀️ - esta función aún no está disponible en Luau; no tenemos opiniones concretas sobre ella así que en algún punto la implementaremos

Por favor tomen en cuenta que todas estas decisiones no son finales, solo representan nuestra opinión actual. En algunos casos la evolución de nuestra MV (máquina virtual) puede hacer que una función que anteriormente no era práctica de soportar debido a complicaciones de rendimiento, factible. En algunos casos una función que no tenía uso fuerte ahora se gana uno, para nosotros implementarla.

## Límites de implementación

Luau tiene ciertas limitaciones sobre el número de variables locales, registros, upvalues, constantes e instrucciones. Estos límites suelen ser distintos a los límites impuestos por varias versiones de Lua, y están documentados aquí sin prometer que futuras versiones cumplirán con estos. Toma en cuenta que escribir código que está cerca de estos límites es peligroso porque este código puede resultar inválido a lo largo de la evolución de nuestra generación de código.

- Variables locales: 200 por función (igual que las demás versiones de Lua, esto incluye los argumentos de funciones)
- Upvalues: 200 por función (aumento desde 60 en Lua 5.1)
- Registros: 255 por función (igual que las demás versiones de Lua, esto incluye los argumentos de funciones)
- Constantes: 2^23 por función (aumento desde 2^18 en Lua 5.1)
- Instrucciones: 2^23 per function (aumento desde 2^17 en Lua 5.1, aunque en ambos casos el límite solo aplica al flujo de control)
- Funciones anidadas: 2^15 por función (disminución desde 2^18 en Lua 5.1)
- Profundidad de la pila: 20000 llamadas de Lua por hilo de Lua, 200 llamadas de C por hilo de C (p. e.j. el límite de anidamiento de coroutine.resume es 200)

Toma en cuenta que Lua 5.3 tiene un límite mayor de upvalues (255) y de constantes (2^26); los límites existentes de Luau probablemente son suficientes por usos razonables.

## Lua 5.1

Como varias funciones fueron eliminadas de Lua 5.1 por motivos de seguridad, esta tabla las enumera por completamiento.

| función | notas |
|---------|------|
| llamadas de cola | eliminadas para simplificar implementación y facilitar la depuración y el seguimiento de pilas |
| las bibliotecas `io`, `os`, `package` y `debug` | toma en cuenta que algunas funciones en `os`/`debug` aún siguen presentes |
| `loadfile`, `dofile` | eliminadas por seguridad, no acceso directo a los archivos |
| bytecode en `loadstring` y `string.dump` | es peligroso exponer bytecode por motivos de seguridad |

Los desafíos de seguridad [se cubren en la sección dedicada](sandbox).

## Lua 5.2

| función | estado | notas |
|---------|--------|------|
| pcall y metamétodos pausables | ✔️/❌ | pcall/xpcall soportan ser pausados pero los metamétodos no |
| tablas efímeras | ❌ | esto complica el colector de basura especialmente con grandes tablas débiles |
| colector de basura de emergencia | ❌ | Luau ejecuta en entornos donde manejar el agotamiento de memoria en situaciones de emergencia no es factible |
| instrucción goto | ❌ | esto complica el compilador debido a la manera de manejar las variables locales y no aborda una necesidad significante |
| finalizadores para las tablas | ❌ | no soportamos `__gc` debido a seguridad y rendimiento/complejidad |
| no más entorno de función para hilos o funciones | 😞 | nos encanta esto, pero rompe la compatibilidad |
| tablas respetan el metamétodo `__len` | ❌ | implicaciones de rendimiento, no hay usos fuertes |
| escapes hex y `\z` en cadenas de caracteres | ✔️ | |
| números flotantes hexadecimales | 🤷‍♀️ | no hay usos fuertes |
| metamétodos de orden funcionan con tipos distintos | ❌ | no hay usos fuertes y semánticas más complicadas + compatibilidad |
| instrucción vacía | 🤷‍♀️ | menos útil en Lua que en JS/C#/C/C++ |
| instrucción `break` puede aparecer en medio de un bloque | 🤷‍♀️ | nos gustaría hacerlo para return/continue también pero aquí hay dragones |
| argumentos para funciones llamadas por mediante de `xpcall` | ✔️ | |
| base opcional en `math.log` | ✔️ | |
| separador opcional en `string.rep` | 🤷‍♀️ | no hay usos reales |
| nuevos metamétodos `__pairs` e `__ipairs` | ❌ | nos gustaría reevaluar el diseño de iteración a largo plazo |
| patrones `%f` | ✔️ | |
| `%g` en patrones | ✔️ | |
| `\0` en patrones | ✔️ | |
| biblioteca `bit32` | ✔️ | |

Dos cosas que son importantes de resaltar son que hay varios nuevos metamétodos para tablas y el poder pausar en metamétodos. En ambos casos, hay implicaciones de rendimiento al soportar esto - nuestra implementación está *altamente* sintonizada para el rendimiento, así que cualquier cambio que afecta los fundamentales esenciales de como funciona Lua tiene un precio. Para soportar las pausas en metamétodos tendríamos que involucrar más al núcleo de la MV, ya que casi cada operación de código "interesante" tendría que aprender cómo reanudarse, lo cual complica futura historia de compilación en tiempo de ejecución/compilación anticipada. Los metamétodos en general son importantes para la extensibilidad, pero desafiantes de lidiar con en implementación, así que no soportamos nuevos metamétodos a menos de que aparezca una fuerte necesidad.

Para `__pairs`/`__ipairs`, no estamos seguros de que esta decisión de diseño es la correcta, tablas autoiteradoras por medio de `__iter` son muy agradables, y si podemos resolver algunos desafíos con el orden de iteración de listas, eso haría el lenguaje más accesible así que tal vez tomemos ese camino.

Es probable que implementemos tablas efímeras en algún ya que tienen usos válidos y hacen que las tablas débiles sean más limpias semánticamente, pero el mecanismo de limpieza es algo costoso y complicado, y por lo tanto esto solo se puede considerar después de completar la remodelación del colector de basura.

## Lua 5.3

| función | estado | notas |
|---------|--------|------|
| escapes `\u` en cadenas de caracteres | ✔️ | |
| números enteros (64 bits por defecto) | ❌ | compatibilidad e implicaciones de rendimiento |
| operadores bitwise | ❌ | la biblioteca `bit32` cubre esto |
| soporte básico para utf-8 | ✔️ | incluimos la biblioteca `utf8` y otras funciones de UTF8 |
| funciones para empacar y desempacar valores (string.pack/unpack/packsize) | ✔️ | |
| división entre enteros | ❌ | no hay usos fuertes, su sintaxis interfiere con los comentarios de C |
| `ipairs` y la biblioteca de `table` respetan los metamétodos | ❌ | no hay usos fuertes, implicaciones de rendimiento |
| nueva función `table.move` | ✔️ | |
| `collectgarbage("count")` ahora retorna un solo resultado | ✔️ | |
| `coroutine.isyieldable` | ✔️ | |

Es importante destacar el soporte para enteros y operadores bitwise. Para Luau, no es común que un tipo de dato de enteros de 64 bits sea necesario - tipos de doble precisión soportan los enteros de hasta 2^53 (en Lua que utiliza espacio integrado, los enteros pueden ser más agradables en entornos sin una unidad de coma flotante nativa de 64 bits). Pero, hay *mucho* valor en tener un solo tipo de número, desde una perspectiva de rendimiento y consistencia. Notablemente, Lua no maneja bien el desbordamiento de enteros, así que el uso de enteros también porta implicaciones de compatibilidad.

Si sacamos los enteros de la ecuación, los operadores bitwise tienen menos sentido; además, la biblioteca `bit32` es más completa en funciones (incluye operaciones comunes como rotaciones y desplazamiento aritmético; extracción/reemplazo de bits también es más legible). El agregar los operadores junto con sus metamétodos respectivos aumenta la complejidad, lo cual significa que esta función no vale la pena en la balanza.

La división entre enteros es menos dañina, pero se usa tan pocas veces que `math.floor(a/b)` se ve como una alternativa adecuada; además, `//` es usado para marcar comentarios en los lenguajes derivados de C, y puede ser que lo implementemos en adición a `--` en algún momento.

## Lua 5.4

| función | estado | notas |
|--|--|--|
| nuevo modo generacional para colección de basura | 🔜 | estamos trabjando en optimizaciones de colección de basura y el modo generacional está en nuestro radar
| variables a punto de ser eliminadas | ❌ | la sintaxis es horrenda e inconsistente con cómo nos gustaría hacer los atributos a largo plazo; no hay ningún uso fuerte en nuestro dominio |
| variables const (constantes) | ❌ | aunque hay demanda de variables constantes, nunca adoptaríamos esta sintaxis |
| implementación nueva de math.random | ✔️ | nuestro generador de números aleatorios está basado en PCG, no como Lua 5.4 el cual utiliza Xoroshiro |
| argumento opcional `init` de `string.gmatch` | 🤷‍♀️ | no hay usos fuertes |
| nuevas funciones `lua_resetthread` and `coroutine.close` | ❌ | no son útiles sin las variables a punto de ser eliminadas |
| coerciónes de cadenas de caracteres a números movidos a la biblioteca de string | 😞 | nos encanta esto, pero rompe la compatibilidad |
| nuevo formato `%p` en `string.format` | 🤷‍♀️ | no hay usos fuertes |
| biblioteca `utf8` acepta puntos de código de hasta 2^31 | 🤷‍♀️ | no hay usos fuertes |
| El uso del metamétodo `__lt` para emular el `__le` ha sido eliminado | 😞 | rompe la compatibilidad y no nos parece muy interesante |
| Al finalizar los objetos, Lua llamará los metamétodos `__gc` que no son funciones | ❌ | no hay soporte para `__gc` debido a la seguridad y el rendimiento/complejidad |
| La función print llama `__tostring` en lugar de tostring para formatear sus argumentos. | 🔜 | |
| Por defecto, las funciones de la biblioteca utf8 para decodificar no aceptan suplantes. | 😞 | romple la compatibilidad y no nos parece muy interesante |

Lua tiene una sintaxis muy bella y francamente estamos decepcionados de la sintaxis `<const>`/`<toclose>` lo cual disminuye esa belleza. Dejando la sintaxis alado, `<toclose>` no es muy útil en Luau - su uso dominante es para código que funciona con recursos externos como archivos o sockets, pero no proporcionamos tales interfaces de programación - y lleva un costo de complejidad muy grande, evidencias por muchas correciones de bugs desde la implementación inicial en versiones de trabajo de 5.4. `<const>` en Luau no importa para el rendimiento - nuestro compilador multipaso ya es capaz de analizar el uso de la variable para saber si está modificada o no y extraer las ganancias de rendimiento - así que el único uso aquí es para la legibilidad de código, donde la sintaxis `<const>` es...subóptima.

Si terminamos introduciendo las variables constantes, sería por medio de una sintaxis `const var = valor`, la cual es compatible por medio de una palabra clave sensible al contexto, similar a `type`.
