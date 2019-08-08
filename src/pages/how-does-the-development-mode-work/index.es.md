---
title: ¿Cómo funciona el modo de desarrollo?
date: '2019-08-04'
spoiler: Dead code elimination by convention.
---

Si tu base de código JavaScript es al menos moderadamente compleja, **probablemente tienes una forma de compilar en *bundles* código diferente en desarrollo y producción**.

Compilar en *bundles* y ejecutar código diferente en desarrollo y producción es poderoso. En modo de desarrollo, React incluye muchas advertencias que te ayudan a encontrar problemas antes de que lleven a bugs. Sin embargo, el código necesario para detectar tales problemas con frecuencia aumentan el tamaño del *bundle* y relentiza la aplicación.

La relentización es aceptable en desarrollo. De hecho, ejecutar el código más lentamente en desarrollo *puede hasta ser beneficioso* porque compensa parcialmente la discrepancia entre máquinas rápidas de desarrollo y los dispositivos promedio de los consumidores.

En producción no queremos pagar ese costo. Por lo tanto, omitimos estas verificaciones. ¿Cómo funciona eso? Veamos.

---

La forma exacta para ejecutar código en desarrollo depende de tu *build pipeline* de JavaScript (y de si tienes una). En Facebook es algo así:

```js
if (__DEV__) {
  hacerAlgoEnDesarrollo();
} else {
  hacerAlgoEnProd();
}
```

Aquí, `__DEV__` no es una variable real. Es una constante que se sustituye cuando los módulos son compilados para el navegador. El resultado se ve así:

```js
// En desarrollo:
if (true) {
  hacerAlgoEnDesarrollo(); // 👈
} else {
  hacerAlgoEnProd();
}

// En production:
if (false) {
  hacerAlgoEnDesarrollo();
} else {
  hacerAlgoEnProd(); // 👈
}
```

En producción, también ejecutarías un *minifier* (por ejemplo, [terser](https://github.com/terser-js/terser)) en el código. La mayoría de los *minifiers* de JavaScript hacen una forma limitada de [eliminación de código muerto](https://es.wikipedia.org/wiki/Eliminaci%C3%B3n_de_c%C3%B3digo_muerto), como quitar las ramas `if (false)`. Entonces en producción sólo verías:

```js
// En producción (luego de minificar):
hacerAlgoEnProd();
```
*(Ten en cuenta que hay límites significativos en qué tan efectiva es la elimininación de código muerto con herramientas comunes de JavaScript, pero eso es un tema aparte.)*

Aunque puede que no estés usando la constante mágica `__DEV__`, si usas algún *bundler* de JavaScript popular como webpack, puede que haya otra convención que puedas seguir. Por ejemplo, es común expresar el mismo patrón de esta forma:

```js
if (process.env.NODE_ENV !== 'production') {
  hacerAlgoEnDesarrollo();
} else {
  hacerAlgoEnProd();
}
```

**Ese es exactamente el patrón usado por librerías como [React](https://es.reactjs.org/docs/optimizing-performance.html#use-the-production-build) y [Vue](https://vuejs.org/v2/guide/deployment.html#Turn-on-Production-Mode) cuando los importas de npm usando un *bundler*** (Las compilaciones de etiquetas `<script>` de un solo archivo ofrecen versiones de desarrollo y producción como archivos `.js` y `.min.js` separados.)

Esta convención en particular viene de Node.js. En Node.js, hay una variable global `process` que expone las variables de entorno de tu sistema como propiedades en el objeto [`process.env`](https://nodejs.org/dist/latest-v8.x/docs/api/process.html#process_process_env). Sin embargo, cuando ves este patrón en una base de código de *front-end*, no hay ninguna variable real `process` involucrada. 🤯

En su lugar, toda la expresión `process.env.NODE_ENV` se sustituye por un string literal en el momento de compilación, igual que nuestra variable mágica `__DEV__`:

```js
// En desarrollo
if ('development' !== 'production') { // true
  hacerAlgoEnDesarrollo(); // 👈
} else {
  hacerAlgoEnProd();
}

// En producción:
if ('production' !== 'production') { // false
  hacerAlgoEnDesarrollo();
} else {
  hacerAlgoEnProd(); // 👈
}
```
Dado que toda la expresión es constante (`'production' !== 'production'` siempre será `false`), un *minifier* también puede quitar la otra rama.

```js
// En producción (luego de minificar):
hacerAlgoEnProd();
```

Travesura lograda.

---

Ten en cuenta que esto **no funcionaría** con expresiones más complejas:

```js
let mode = 'production';
if (mode !== 'production') {
  // 🔴 no se garantiza que se elimine
}
```

Las herramientas de análisis estático de JavaScript no son muy inteligentes debido a la naturaleza dinámica del lenguaje. Cuando ven variables como `mode` en lugar de expresiones estáticas como `false` o `'production' !== 'production'`, generalmente se rinden.

De igual forma, la eliminación de código muerto en JavaScript generalmente no funciona bien fuera de los límites del módulo cuando usas la sentencia de nivel superior `import`:

```js
// 🔴 no se garantiza que se elimine
import {algunaFuncion} from 'some-module';

if (false) {
  algunaFuncion();
}
```

Por eso necesitas escribir código de forma muy mecánica tal que haga la condición *definitivamente estática* y asegure que *todo el código* que quieres eliminar se encuentre dentro.

---

Para que todo esto funcione, tu *bundler* necesita reemplazar `process.env.NODE_ENV` y necesita saber en qué modo *quieres* compilar el proyecto.

Hace unos años, solía ser común olvidarse configurar el entorno. Con frecuencia veías proyectos en modo de desarrollo desplegados en producción.

Eso es malo porque hace que el sitio web cargue y funcione más lentamente.

En los últimos dos años, la situación mejoró significativamente. Por ejemplo, webpack añadió una simple opción `mode` en lugar de tener que configurar manualmente el reemplazo de `process.env.NODE_ENV`. Ahora React DevTools también muestra un ícono rojo en sitios en modo de desarrollo, haciendo más fácil identificar e incluso [reportar](https://mobile.twitter.com/BestBuySupport/status/1027195363713736704).

![Advertencia de modo de desarrollo en React DevTools](devmode.png)

Instaladores dogmáticos como Create React App, Next/Nuxt, Vue CLI, Gatsby y otros hacen más difícil equivocarse con esto separando los *builds* de desarrollo y producción en dos comandos separados (por ejemplo, `npm start` y `npm run build`). Generalmente, solo se puede desplegar el build de producción por lo que el desarrollador ya no puede cometer este error.

Siempre existe un argumento que quizás el modo *producción* necesita ser el valor por defecto y el modo de desarrollo opcional. Personalmente no me convence este argumento. Las personas que se benefician más de las advertencias en el modo de desarrollo generalmente son nuevos en la librería. *No sabrían cómo encenderlo,* y evitarían encontra muchos bugs que con las advertencias se hacen evidentes.

Si, los problemas de performance son malos, pero también es enviar experiencias rotas y llenas de bugs a los usuarios finales. Por ejemplo, el [warning de *key* de React](https://es.reactjs.org/docs/lists-and-keys.html#keys) ayuda a prevenir bugs como enviar un mensaje a la persona equivocada o comprar el producto erróneo. Desarrollar con esta advertencia desactivada es un riesgo significativo para tí *y* tus usuarios. Si está desactivada pro defecto, entonces para el momento que encuentres la opción para encenderlo, tendrás demasiadas advertencias para limpiar. Así que muchas personas lo volverían a apagar. Este es el movito por el que necesita estar encendido desde el comienzo en lugar de activado luego.

Finalmente, incluso si las advertencias de desarrollo son opcionales y los desarrolladores *supieran* que deben encenderlas en una etapa temprana del desarrollo, simplemente volveríamos al mismo problema. ¡Alguien accidentalmente las dejaría encendidas en producción!

Y volvimos al primer casillero

Personalente creo en **herramientas que muestran y usan el modo correcto dependiendo si estás depurando o desplegando**. Casi cualquier entorno (sea mobile, escritorio o servidor) excepto el navegador ha tenido una forma de cargar y diferenciar builds de desarrollo y producción por décadas.

En lugar de dejar a las librerías inventar y nosotros depender de convenciones ad-hoc, quizás es momento que los entornos de JavaScript vean la distinción como una neesidad de primera clase.

---

¡Suficiente filosofía!

Miremos de nuevo a este código:

```js
if (process.env.NODE_ENV !== 'production') {
  hacerAlgoEnDesarrollo();
} else {
  hacerAlgoEnProd();
}
```

Quizás te preguntes: si no hay ningún objeto `process` en el código de front-end, ¿por qué las librerías como React y Vue dependen de ella en los builds de nmp?

*(To clarify this again: the `<script>` tags you can load in the browser, offered by both React and Vue, don’t rely on this. Instead you have to manually pick between the development `.js` and the production `.min.js` files. The section below is only about using React or Vue with a bundler by `import`ing them from npm.)*

Like many things in programming, this particular convention has mostly historical reasons. We are still using it because now it’s widely adopted by different tools. Switching to something else is costly and doesn’t buy much.

So what’s the history behind it?

Many years before the `import` and `export` syntax was standardized, there were several competing ways to express relationships between modules. Node.js popularized `require()` and `module.exports`, known as [CommonJS](https://en.wikipedia.org/wiki/CommonJS).

Code published on the npm registry early on was written for Node.js. [Express](https://expressjs.com) was (and probably still is?) the most popular server-side framework for Node.js, and it [used the `NODE_ENV` environment variable](https://expressjs.com/en/advanced/best-practice-performance.html#set-node_env-to-production) to enable production mode. Some other npm packages adopted the same convention.

Early JavaScript bundlers like browserify wanted to make it possible to use code from npm in front-end projects. (Yes, [back then](https://blog.npmjs.org/post/101775448305/npm-and-front-end-packaging) almost nobody used npm for front-end! Can you imagine?) So they extended the same convention already present in the Node.js ecosystem to the front-end code.

The original “envify” transform was [released in 2013](https://github.com/hughsk/envify/commit/ae8aa26b759cd2115eccbed96f70e7bbdceded97). React was open sourced around that time, and npm with browserify seemed like the best solution for bundling front-end CommonJS code during that era.

React started providing npm builds (in addition to `<script>` tag builds) from the very beginning. As React got popular, so did the practice of writing modular JavaScript with CommonJS modules and shipping front-end code via npm.

React needed to remove development-only code in the production mode. Browserify already offered a solution to this problem, so React also adopted the convention of using `process.env.NODE_ENV` for its npm builds. With time, many other tools and libraries, including webpack and Vue, did the same.

By 2019, browserify has lost quite a bit of mindshare. However, replacing `process.env.NODE_ENV` with `'development'` or `'production'` during a build step is a convention that is as popular as ever.

*(It would be interesting to see how adoption of ES Modules as a distribution format, rather than just the authoring format, changes the equation. Tell me on Twitter?)*

---

One thing that might still confuse you is that in React *source code* on GitHub, you’ll see `__DEV__` being used as a magic variable. But in the React code on npm, it uses `process.env.NODE_ENV`. How does that work?

Historically, we’ve used `__DEV__` in the source code to match the Facebook source code. For a long time, React was directly copied into the Facebook codebase, so it needed to follow the same rules. For npm, we had a build step that literally replaced the `__DEV__` checks with `process.env.NODE_ENV !== 'production'` right before publishing.

This was sometimes a problem. Sometimes, a code pattern relying on some Node.js convention worked well on npm, but broke Facebook, or vice versa.

Since React 16, we’ve changed the approach. Instead, we now [compile a bundle](https://reactjs.org/blog/2017/12/15/improving-the-repository-infrastructure.html#compiling-flat-bundles) for each environment (including `<script>` tags, npm, and the Facebook internal codebase).  So even CommonJS code for npm is compiled to separate development and production bundles ahead of time.

This means that while the React source code says `if (__DEV__)`, we actually produce *two* bundles for every package. One is already precompiled with `__DEV__ = true` and another is precompiled with `__DEV__ = false`. The entry point for each package on npm “decides” which one to export.

[For example:](https://unpkg.com/browse/react@16.8.6/index.js)

```js
if (process.env.NODE_ENV === 'production') {
  module.exports = require('./cjs/react.production.min.js');
} else {
  module.exports = require('./cjs/react.development.js');
}
```

And that’s the only place where your bundler will interpolate either `'development'` or `'production'` as a string, and where your minifier will get rid of the development-only `require`.

Both `react.production.min.js` and `react.development.js` don’t have any `process.env.NODE_ENV` checks anymore. This is great because *when actually running on Node.js*, accessing `process.env` is [somewhat slow](https://reactjs.org/blog/2017/09/26/react-v16.0.html#better-server-side-rendering). Compiling bundles in both modes ahead of time also lets us optimize the file size [much more consistently](https://reactjs.org/blog/2017/09/26/react-v16.0.html#reduced-file-size), regardless of which bundler or minifier you are using.

And that’s how it really works!

---

I wish there was a more first-class way to do it without relying on conventions, but here we are. It would be great if modes were a first-class concept in all JavaScript environments, and if there was some way for a browser to surface that some code is running in a development mode when it’s not supposed to.

On the other hand, it is fascinating how a convention in a single project can propagate through the ecosystem. `EXPRESS_ENV` [became `NODE_ENV`](https://github.com/expressjs/express/commit/03b56d8140dc5c2b574d410bfeb63517a0430451) in 2010 and [spread to front-end](https://github.com/hughsk/envify/commit/ae8aa26b759cd2115eccbed96f70e7bbdceded97) in 2013. Maybe the solution isn’t perfect, but for each project the cost of adopting it was lower than the cost of convincing everyone else to do something different. This teaches a valuable lesson about the top-down versus bottom-up adoption. Understanding how this dynamic plays out distinguishes successful standardization attempts from failures.

Separating development and production modes is a very useful technique. I recommend using it in your libraries and the application code for the kinds of checks that are too expensive to do in production, but are valuable (and often critical!) to do in development.

As with any powerful feature, there are some ways you can misuse it. This will be the topic of my next post!
