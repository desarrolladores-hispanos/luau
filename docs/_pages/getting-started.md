---
permalink: /getting-started
title: Empezando
toc: true
---
Para empezar con Luau necesitarás instalar Roblox Studio, el cual puedes descargar [aquí](https://www.roblox.com/create).

## Creando un lugar

Si solo deseas experimentar con el lenguaje por sí mismo, puedes crear un simple juego baseplate.

<figure>
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/create-new-place.png">
</figure>

## Creando un script

Para crear tu propio script de prueba, dirigite a ServerScriptService en el explorador y agrega un objeto de Script.

<figure>
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/create-script.png">
</figure>

Haga doble click en el script y pegue esto:

```lua
function espositivo(x)
    return x > 0
end

print(espositivo(1))
print(espositivo("2"))

function isfoo(a)
    return a == "foo"
end

print(isfoo("bar"))
print(isfoo(1))
```

Observa que no hay advertencias llamando ``espositivo()`` con un string, o llamando ``isfoo()`` con un número. 

## Inferencia de tipos

Ahora modifica el script y agrega ``--!strict`` arriba del todo:

```lua
--!strict

function espositivo(x)
    return x > 0
end

print(espositivo(1))
print(espositivo("2"))

function isfoo(a)
    return a == "foo"
end

print(isfoo("bar"))
print(isfoo(1))
```

En el modo ``strict``, Luau inferirá tipos basándose en el análisis del flujo del código. También está el modo ``nonstrict``, en el cual el análisis es más conservativo y los tipos son interferidos más frecuentemente como ``any`` para reducir casos en los cuales código legítimo es marcado con advertencias.

En este caso, Luau usará la declaración ``return x > 0`` para inferir que ``espositivo()`` 
es una función que toma un número entero y devuelve un valor boolean. Similarly, usará la declaración ``return a == "foo"`` para inferir que ``isfoo()`` es una función que toma un string y devuelve un valor boolean. Observa que en ambos casos, no fue necesario agregar ningún tipo de anotaciones explícitas.

Basado en la inferencia de tipos de Luau, el editor ahora resalta las llamadas incorrectas a ``espositivo()`` y ``isfoo()``:

<figure>
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/error-ispositive.png">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/error-isfoo.png">
</figure>

## Anotaciones

Puedes agregar anotaciones a locales, argumentos, y tipos de retorno de funciones. Entre otras cosas, las anotaciones pueden ayudar a que no hagas algo estúpido accidentalmente. Aqui esta como agregaríamos anotaciones a  ``espositivo()``:

```lua
--!strict

function espositivo(x : number) : boolean
    return x > 0
end

local resultado : boolean
resultado = espositivo(1)

```

Ahora que le hemos dicho explícitamente a Luau que ``espositivo()`` acepta un número y regresa un valor booleano. Esto no era estrictamente necesario en este caso, porque la inferencia de Luau ya pudo deducir esto. Pero aun en este caso, hay ventajas para las anotaciones explícitas. Imaginate que luego decidimos cambiar ``espositivo()`` para que regrese un string.

```lua
--!strict

function espositivo(x : number) : boolean
    if x > 0 then
        return "si"
    else
        return "no"
    end
end

local resultado : boolean
resultado = espositivo(1)
```

Uy -- estamos regresando strings, pero nos olvidamos de actualizar el tipo de retorno de la función. Dado que le dijimos a Luau que ``espositivo()`` regresa un valor booleano (y así es como lo estamos usando), el lugar de la llamada no está marcado como error. Pero debido a que la anotación no coincide con nuestro código, recibimos una advertencia en el propio cuerpo de la función:

<figure>
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/error-ispositive-string.png">
</figure>

Repararlo es facil; solo cambia la anotación para declarar el tipo de regreso como un string:

```lua
--!strict

function espositivo(x : number) : string
    if x > 0 then
        return "si"
    else
        return "no"
    end
end

local resultado : boolean
resultado = espositivo(1)
```

Bueno, casi  - desde que declaramos ``resultado`` como un valor booleano, el lugar donde lo llamamos ahora está marcado:

<figure>
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/error-ispositive-boolean.png">
</figure>


Si actualizamos el tipo de la variable local, todo está bien. Observa que también podemos dejar que Luau infiera el tipo de ``resultado`` cambiandolo por la versión de solo una línea ``local resultado = espositivo(1)``.

```lua
--!strict

function espositivo(x : number) : string
    if x > 0 then
        return "si"
    else
        return "no"
    end
end

local resultado : string
resultado = espositivo(1)
```

## Conclusiones

Este fue un corto tour de la funcionalidad básica de Luau, pero hay mucho más por explorar. Si estás interesado en leer más, echa un vistazo a nuestras páginas de referencia principales [sintaxis](syntax) y [comprobación de tipos](typecheck).

