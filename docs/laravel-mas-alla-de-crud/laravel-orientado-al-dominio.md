# Laravel orientado al dominio

## Los humanos pensamos en categorías, nuestro código debería ser un reflejo de eso.

Lo primero es lo primero: no se me ocurrió el término "dominio", lo obtuve del popular paradigma de programación DDD, o "diseño impulsado por el dominio". Es un término bastante genérico y, según el Oxford Dictionary, un "dominio" puede describirse como "una esfera específica de actividad o conocimiento".

Si bien mi uso de la palabra "dominio" no tendrá exactamente el mismo significado que en la comunidad DDD, hay varias similitudes. Si está familiarizado con DDD, descubrirá estas similitudes a lo largo de este libro. Hice todo lo posible para mencionar cualquier superposición y diferencia cuando fuera relevante.

Entonces, dominios. También podría llamarlos "grupos", "módulos" o, como algunas personas los llaman, "servicios". Cualquiera que sea el nombre que prefiera, los dominios describen un conjunto de problemas comerciales que está tratando de resolver.

Espera... Me doy cuenta de que acabo de usar mi primer término "empresarial" en este libro: "el problema de los negocios". A lo largo de este libro, notará que hice lo mejor que pude para alejarme del lado teórico, de la alta dirección y de los negocios. Yo mismo soy un desarrollador y prefiero mantener las cosas prácticas. Entonces, otro nombre más simple sería "el proyecto".

Pongamos un ejemplo: una aplicación para gestionar reservas de hotel. Tiene que gestionar clientes, reservas, facturas, inventarios del hotel, etc.

Los marcos web modernos le enseñan a tomar un grupo de conceptos relacionados y dividirlos en varios lugares a lo largo de su base de código: controladores con controladores, modelos con modelos, etc. Obtienes el trato.

Así que detengámonos y pensemos en eso por un momento.

¿Alguna vez un cliente le ha dicho que "trabaje en los controladores ahora" o que "se concentre en el directorio de modelos"? No, le piden que trabaje en las funciones de facturación, gestión de clientes o reservas.

Estos grupos son lo que yo llamo dominios. Su objetivo es agrupar conceptos dentro de su proyecto que pertenecen juntos. Si bien esto puede parecer trivial al principio, es más complicado de lo que piensas. Es por eso que parte de este libro se centrará en un conjunto de reglas y prácticas para mantener su código bien ordenado.

Obviamente, no hay una fórmula matemática que pueda darte porque casi todo depende del proyecto específico en el que estés trabajando. Así que no pienses en este libro como dando un conjunto fijo de reglas. Más bien, piense en ello como si le entregara una colección de ideas que puede usar y desarrollar, como lo desee.

Es una oportunidad de aprendizaje, mucho más que una solución que puedes aportar a cualquier problema que encuentres.

## Dominios y aplicaciones
Si estamos agrupando ideas, evidentemente surge la pregunta: ¿Hasta dónde llegamos? Podría, por ejemplo, agrupar todo lo relacionado con la factura: modelos, controladores, recursos, reglas de validación, trabajos, ...

Sin embargo, este enfoque plantea un problema en las aplicaciones HTTP clásicas: a menudo no hay un mapeo uno a uno entre controladores y modelos. Por supuesto, en las API REST y para la mayoría de los controladores CRUD clásicos, puede haber una asignación estricta de uno a uno. Desafortunadamente, hay excepciones a las reglas, y esas nos harán pasar un mal rato.

Las facturas, por ejemplo, simplemente no se manejan de forma aislada; necesitan un cliente al que enviar, necesitan reservas para facturar, etc.

Es por eso que necesitamos hacer una distinción adicional entre lo que es un código de dominio y lo que no lo es.

Por un lado, está el dominio, que representa toda la lógica comercial y, por otro lado, tenemos el código que usa, es decir, consume, ese dominio para integrarlo con el marco y exponerlo al usuario final. Las aplicaciones proporcionan la infraestructura para que los usuarios finales accedan y manipulen la funcionalidad del dominio de una manera fácil de usar.

Ahora, dedicaremos un capítulo a profundizar en las diferencias entre el dominio y el código de la aplicación, pero en este momento ya es importante saber que haremos una distinción entre los dos. Te prometo que abordaremos varias preguntas que podrían estar surgiendo en tu cabeza en este momento, pronto.

## Domains in practice

Entonces, ¿cómo se ve en la práctica todo lo que describí anteriormente? El código de dominio constará de clases como modelos, generadores de consultas, eventos de dominio, reglas de validación y más; Veremos todos estos conceptos en profundidad.

La capa de aplicación será una o varias aplicaciones. Cada aplicación se puede ver como una aplicación aislada que puede usar todo ese código de dominio. En general, las aplicaciones no se comunican entre sí, al menos no directamente.

Un ejemplo podría ser un panel de administración HTTP estándar y otro podría ser una API REST. También me gusta pensar en la consola, el artesano de Laravel, como una aplicación propia.

`Una carpeta de dominio específica por concepto de negocio`
```
src/Domain/Invoices/
├── Actions
├── QueryBuilders
├── Collections
├── DataTransferObjects
├── Events
├── Exceptions
├── Listeners
├── Models
├── Rules
└── States
src/Domain/Customers/
// ...
```
