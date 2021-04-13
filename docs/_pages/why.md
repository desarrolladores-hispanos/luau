---
permalink: /why
title: ¿Por qué Luau?
---

Alrededor de 2006, [Roblox](https://www.roblox.com) comenzó a usar Lua 5.1 como lenguaje de programación para juegos. A través de los años el tiempo de ejecución debía ser alterado para proveer un entorno aislado seguro; gradualmente comenzamos a acumular pequeños cambios a las bibliotecas.

A través del curso de los últimos años, en vez de usar pilas basadas en la Web para nuestras aplicaciones para el jugador, IU dentro del juego basada en Lua y editor de IU basado en Qt, comenzamos a consolidar grandes esfuerzos y desarrollar todos estos usando el motor de Roblox y Lua como el lenguaje de programación.

Teniendo sustancialmente una base de código interna aumentada que necesitaba estar correcta y eficiente, y con el foco cambiando un poco de desarrolladores de juegos principiantes a estudios profesionales construyendo juegos en Roblox y nuestros equipos de ingenieros construyendo aplicaciones, había una necesidad de mejorar el rendimiento y calidad del código que estábamos escribiendo.

A diferencia de la red principal de Lua, no pudimos permitir cambios mayores que rompían el código (por eso la base del lenguaje de 5.1 se mantuvo sin cambios por más de una década). Aunque implementaciones más rápidas de Lua 5.1, como LuaJIT fueron disponibles, estas no cumplían con nuestras necesidades en términos de portabilidad, facilidad de cambiar y estas no se dirigían al problema de desarrollar código robusto a escala.

Todas estas nos motivaron a comenzar a remodelar Lua 5.1 desde el que comenzamos en un nuevo, lenguaje derivado que nosotros llamamos Luau. Nuestro foco está en hacer el lenguaje más eficiente y con varias funciones, y hacer más fácil el escribir código robusto por medio de una combinación de linting y un comprobador de tipos usando un sistema de tipos gradual.

## ¿Reescritura completa?

Una gran parte de la base del código de Luau está escrita desde cero. Necesitábamos un conjunto de herramientas capaces de escribir herramientas de análisis del lenguaje; Lua tiene un analizador que está integrado con el compilador de bytecode, el cual no lo hace apto para análisis semánticos complejos. Para la compilación de bytecode, mientras un compilador de un paso puede entregar un mejor rendimiento de compilación y es más simple que un frontend/backend completo, esto significativamente limita las optimizaciones que pueden ser realizadas en el nivel del bytecode.

El compilador de Luau y las herramientas de análisis están escritas desde cero, siguiendo de cerca la sintaxis y las semánticas de Lua. Nuestro compilador no es de un paso, y en vez confía en un conjunto de análisis que corre en el árbol de sintaxis abstracta para producir bytecode eficiente, seguido por algunas optimizaciones en los procesos posteriores.

Para el tiempo de ejecución, tuvimos que reescribir el interpretador desde cero para conseguir sustancialmente un rendimiento más rápido; usando una combinación de técnicas propuesta por LuaJIT y optimizaciones personalizadas que son capaces de mejorar el rendimiento tomando control sobre la pila completa (lenguaje, compilador, interpretador, máquina virtual), somos capaces de conseguir un interpretador de rendimiento mientras usamos C como un lenguaje implementado.

El colector de basura y las bibliotecas importantes representan más a un cambio incremental, donde usamos Lua 5.1 como una base pero seguimos continuando a reescribir estos tal cual con el rendimiento en mente.

Mientras Luau actualmente no implementa compilación en tiempo de ejecución/compilación anticipada, esto es probable a que pase en algún momento; más allá de los retos usuales en la implementación y las preocupaciones de seguridad, una limitación significante es que no tenemos acceso a la compilación en tiempo de ejecución en muchas plataformas así que para mantener un rendimiento interpretado excelente para el gameplay y el código de la aplicación es más importante que alcanzar la cima de FLOPS (Operaciones de coma flotante por segundo) en código numérico.

## ¿Código abierto?

Por favor toma en cuenta que en este momento no estamos preparados en hacer Luau una implementación disponible públicamente en forma de fuente abierta. En este momento la única interfaz a Luau es el motor de Roblox (expuesta en el cliente/servidor de Roblox y Roblox Studio)
