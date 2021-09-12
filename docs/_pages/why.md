---
permalink: /why
title: ¿Por qué Luau?
---

Alrededor de 2006, [Roblox](https://www.roblox.com) comenzó a usar Lua 5.1 como lenguaje de programación para juegos. A través de los años el tiempo de ejecución debía ser alterado para proveer un entorno aislado seguro; gradualmente comenzamos a acumular pequeños cambios a las bibliotecas.

En el transcurso de los últimos años, en vez de usar pilas basadas en la Web para nuestras aplicaciones para jugador, IU dentro del juego basada en Lua o editor de IU basado en Qt, hemos comenzado a consolidar muchos de los esfuerzos y desarrollar todos estos usando el motor de Roblox, y Lua como el lenguaje de programación.

Habiendo desarrollado una base de código interno importante que necesitaba ser correcto y eficiente, y con su enfoque cambiando un poco, de desarrolladores de juegos principiantes a estudios profesionales construyendo juegos en Roblox y nuestros equipos de ingenieros construyendo aplicaciones, había necesidad de mejorar el rendimiento y la calidad del código que estábamos escribiendo.

A diferencia de la red principal de Lua, nosotros tampoco pudimos hacer cambios demasiado grandes al lenguaje (por eso la línea base del lenguaje 5.1 se mantuvo sin cambios por más de una década). Mientras que implementaciones más rápidas de Lua 5.1, como LuaJIT estuvieron disponibles, éstas no cumplían con nuestras necesidades en términos de portabilidad, facilidad de cambio y además no abordaban el problema de desarrollar código robusto a escala.

Todo esto nos motivó para comenzar a remodelar Lua 5.1 del que comenzamos, en un lenguaje nuevo, derivado que llamamos Luau. Nuestro enfoque está en hacer el lenguaje más eficiente y con varias funciones, y hacer más fácil el escribir código robusto por medio de una combinación de linting y comprobación de tipos usando un sistema de tipos gradual.

## ¿Reescritura completa?

Una gran parte del código base de Luau está escrito desde cero. Necesitábamos un set de herramientas para ser capaces de escribir herramientas de análisis de lenguaje; Lua tiene un analizador que está integrado con el compilador de bytecode, el cual lo hace inadecuado para análisis semánticos complejos. En la compilación de bytecode, mientras un compilador de un paso puede entregar un mejor rendimiento de compilación y es más simple que un full frontend/backend, este significativamente limita las optimizaciones que pueden ser realizadas a nivel del bytecode.

El compilador de Luau y las herramientas de análisis están escritas desde cero, siguiendo de cerca la sintaxis y semántica de Lua. Nuestro compilador no es de un paso, y en cambio depende de un set de pasos de análisis que corre en el AST para producir bytecode eficiente, seguido de algunas optimizaciones en procesos posteriores.

Con respecto al tiempo de ejecución, tuvimos que reescribir el intérprete desde cero para conseguir un rendimiento sustancialmente más rápido; usando una combinación de técnicas iniciadas por LuaJIT y optimizaciones personalizadas que son capaces de mejorar el rendimiento tomando control sobre la pila completa (lenguaje, compilador, intérprete, máquina virtual), somos capaces de conseguir un intérprete de rendimiento similar al de LuaJIT mientras usamos C como un lenguaje de implementación.

El recolector de basura y las librerías del núcleo representan más a un cambio incremental, donde usamos Lua 5.1 como línea base, pero también continuamos reescribiendo estos con el rendimiento en mente

Mientras Luau actualmente no implementa JIT/AOT, esto es probable a pasar en algún punto; más allá de los retos usuales de la implementación y las preocupaciones de seguridad, una limitación significativa es que no tenemos acceso a JIT en muchas plataformas, así que para nosotros mantener un rendimiento interpretado excelente para el gameplay y el código de la aplicación es más importante que alcanzar la cima de FLOPS(Operaciones de coma flotante por segundo) en código numérico.

## ¿Código abierto?

Por favor tenga en cuenta que en este momento no estamos preparados para hacer la implementación de Luau disponible al público en el formato fuente. En este momento la única interfaz a Luau es el motor de Roblox (expuesto en el cliente/servidor de Roblox y Roblox Studio).
