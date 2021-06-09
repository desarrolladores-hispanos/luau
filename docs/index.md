---
title: Lua*u*
layout: splash
permalink: /

header:
  overlay_color: #000
  overlay_filter: 0.8
  overlay_image: /assets/images/luau-header.png
  actions:
    - label: Aprender más
      url: /why

excerpt: >
  Lua*u* (minúscula la *u*, /ˈlu.aʊ/) es un lenguaje de programación rápido, pequeño, seguro, y gradualmente tipado que se puede integrar en aplicaciones, derivado de Lua. Es usado por los desarrolladores de Roblox para escribir el código de sus juegos, al igual que los ingenieros de Roblox para implementar grandes partes de código para el usuario al igual que porciones del editor (Roblox Studio) como plugins.

feature_row1:
  - 
    title: Motivación 
    excerpt: >
      Alrededor de 2006, [Roblox](https://www.roblox.com) comenzó a usar Lua 5.1 como lenguaje de programación para juegos. A través de los años terminamos evolucionando sustancialmente la implementación y el lenguaje; para apoyar la creciente sofisticación de los juegos en la plataforma de Roblox, aumentando el tamaño de los equipos y grandes equipos internos que escriben grandes cantidades de código para la aplicación/el editor (con 1+MLOC en 2020), teníamos que invertir en el rendimiento, facilidad de uso y herramientas de lenguaje, e introducir un sistema de tipos gradual a el lenguaje.
    url: /why
    btn_label: Aprender más
    btn_class: "btn--primary"

  - 
    title: Seguridad
    excerpt: >
      Luau limita el set de librerías estándar expuestas a los usuarios e implementa funciones de seguridad extra para ser capaz de ejecutar código no privilegiado (escrito por nuestros desarrolladores) lado por lado con código privilegiado (escrito por nosotros). Esto da como resultado un entorno de ejecución diferente al habitual en Lua.
    url: /sandbox
    btn_label: Aprender más 
    btn_class: "btn--primary"

  - 
    title: Compatibilidad
    excerpt: >
      Cuando es posible, Luau apunta a ser compatible con versiones anteriores de Lua 5.1 y al mismo tiempo incorporar funciones de últimas revisiones de Lua. Sin embargo, Luau no es un super conjunto completo de versiones posteriores de Lua - nosotros no estamos de acuerdo con algunas decisiones de diseño tomadas por los autores de Lua, y tenemos diferentes casos de uso  y limitaciones. Todas las funciones posteriores a Lua 5.1, junto con su estado de soporte en Luau, [están documentadas aquí](compatibility).
    url: /compatibility
    btn_label: Aprender más
    btn_class: "btn--primary"

feature_row2:
  - 
    title: Sintaxis
    image_path: /assets/images/example.png
    excerpt: >
      Luau es sintácticamente retrocompatible con Lua 5.1 (el código que es válido en Lua 5.1 también es válido en Luau); sin embargo, hemos extendido el lenguaje con un conjunto de funciones sintácticas que hacen el lenguaje más familiar y ergonómico. El sintax [es descrito aquí](syntax).
    url: /syntax
    btn:label: Aprender más
    btn_class: "btn--primary"

feature_row3:
  - 
    title: Análisis
    excerpt: >
        Para facilitar la escritura de código correcto, Luau viene con un conjunto de herramientas de análisis que pueden revelar errores comunes. Estas consisten en un linter y un comprobador de tipos, coloquialmente conocido como un analizador de scripts, y puede ser usado desde [Roblox Studio](https://developer.roblox.com/en-us/articles/The-Script-Analysis-Tool). Las fases de linting [son descritas aquí](lint), y la guía de comprobación de tipos puede [ser encontrada aquí](typecheck).
    url: /typecheck
    btn_label: Aprender más
    btn_class: "btn--primary"

  - 
    title: Rendimiento
    excerpt: >
    En adición de un front end completamente personalizado que implementa el análisis, linting y comprobación de tipos, el tiempo de ejecución de Luau presenta un nuevo bytecode, interpretador y compilador que son densamente ajustados para un mejor rendimiento. Luau actualmente no implementa una compilación JIT (método justo a tiempo), pero su interpretador es frecuentemente competitivo con el interpretador de LuaJIT en un amplio conjunto de puntos de referencia. Nosotros continuamos optimizando el tiempo de ejecución y reescribiendo porciones de este para hacerlo aún más eficiente, incluyendo planes de tener un nuevo colector de basura y más optimizaciones de la biblioteca, al igual que una eventual opción de JIT/AOT. Mientras nuestro objetivo general es minimizar la cantidad de tiempo usada por los programadores para ajustar el rendimiento, algunos detalles acerca de las características del rendimiento son [proveídas para mentes inquisitivas](performance).
    url: /performance
    btn_label: Aprender más
    btn_class: "btn--primary"

  -
    title: Librerías
    excerpt: >
        Como un lenguaje, Luau es un super conjunto completo de Lua 5.1. En lo que respecta a la librería estándar, algunas funciones tuvieron que ser removidas de las librerías integradas, y algunas funciones tuvieron que ser agregadas. Adicionalmente, actualmente Luau es solamente ejecutable desde el contexto del motor de Roblox, el cual expone una gran superficie del API [documentada en el Roblox developer portal](https://developer.roblox.com/en-us/api-reference).

---

{% include feature_row id="feature_row1" %}

{% include feature_row id="feature_row2" type="left" %}

{% include feature_row id="feature_row3" %}
