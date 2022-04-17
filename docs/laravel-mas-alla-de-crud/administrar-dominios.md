# Administrar dominios

En los capítulos anteriores, analizamos los componentes básicos de nuestros dominios: DTO, modelos de acciones y varios conceptos relacionados. En este capítulo cambiaremos nuestro enfoque del lado técnico al lado filosófico: ¿cómo comienza a usar dominios, cómo identificarlos y cómo administrarlos a largo plazo?

## Trabajo en equipo

En el capítulo 1, afirmé que todos los paradigmas y principios sobre los que escribí tendrían un propósito: ayudar a los equipos de desarrolladores a mantener sus aplicaciones Laravel más grandes que el promedio a lo largo de los años.

Al escribir este libro y las publicaciones de blog anteriores, algunas personas expresaron su preocupación: ¿una nueva estructura de directorios y el uso de principios complejos no dificultarían que los nuevos desarrolladores se unan de inmediato?

Si eres un desarrollador familiarizado con los proyectos predeterminados de Laravel y con la forma en que se enseñan a los principiantes, es cierto que necesitarás dedicar un tiempo a aprender cómo se manejan estos proyectos. Sin embargo, esto no es un gran problema como algunas personas quieren que creas.

Imagina un proyecto con alrededor de 100 modelos, 300 acciones, casi 500 rutas. La principal dificultad en estos proyectos no es cómo está estructurado técnicamente el código; más bien se trata de la gran cantidad de conocimiento empresarial que hay que comprender. No puede esperar que los nuevos desarrolladores entiendan todos los problemas que este proyecto está resolviendo, en un instante. Se necesita tiempo para conocer el código, pero lo que es más importante: se necesita tiempo para conocer el negocio. Cuanta menos magia e indirectas haya, menos espacio habrá para cualquier confusión.

Es esencial entender el objetivo de la arquitectura que estoy desarrollando en este libro. No se trata de escribir el menor número de caracteres o de la elegancia del código. Se trata de hacer que las bases de código grandes sean más fáciles de navegar, para permitir el menor espacio posible para la confusión y para mantener el proyecto en buen estado durante mucho tiempo.

Tengo experiencia con este proceso en la práctica. Mi colega Rubén se unió como nuevo desarrollador backend en un equipo de tres desarrolladores en uno de nuestros proyectos.

La arquitectura era nueva para él, incluso si ya tenía experiencia con Laravel. Así que nos tomamos el tiempo para guiarlo. Después de solo unas pocas horas de información y programación en pareja, pudo trabajar de forma independiente en este proyecto. Definitivamente tomó varias semanas obtener una comprensión completa de toda la funcionalidad que brindaba el proyecto, pero afortunadamente, la arquitectura no se interpuso en su camino. Al contrario: ayudó a Rubén a centrarse en la lógica empresarial.

Si llegó hasta este punto de este libro, espero que comprenda que esta arquitectura no pretende ser la panacea para todos los proyectos. Hay muchos casos en los que un enfoque más directo podría funcionar mejor y, en algunos casos, en los que se requiere un enfoque más complejo.

## Identificación de dominios

Con el conocimiento que ahora tenemos sobre los componentes básicos del dominio, surge la pregunta de cómo empezamos a escribir código real exactamente. Hay muchas metodologías que puede usar para comprender mejor lo que está a punto de construir, aunque creo que hay dos puntos clave:

- Aunque sea un desarrollador, su objetivo principal es comprender el problema comercial y traducirlo en código. El código en sí mismo es simplemente un medio para un fin; siempre mantenga su enfoque en el problema que está resolviendo.
- Asegúrese de tener tiempo cara a cara con su cliente. Tomará tiempo extraer el conocimiento que necesita para escribir un programa que funcione.

De hecho, llegué a pensar en la descripción de mi trabajo más como "un traductor entre problemas del mundo real y soluciones técnicas", en lugar de "un programador que escribe código". Creo firmemente que esta mentalidad es clave si vas a trabajar en un proyecto de larga duración. No solo tiene que escribir el código, debe comprender los problemas del mundo real que está tratando de resolver.

Según el tamaño de su equipo, es posible que no necesite una interacción cara a cara entre todos los desarrolladores y el cliente, pero, no obstante, todos los desarrolladores deberán comprender los problemas que están resolviendo con el código.

Estas dinámicas de equipo son un tema tan complejo que merecen su propio libro. Por ahora lo mantendré así, porque de aquí en adelante podemos hablar sobre cómo traducimos estos problemas en dominios.

En el capítulo 1, escribí que uno de los objetivos de esta arquitectura es agrupar el código que pertenece, en función de su significado en el mundo real en lugar de sus propiedades técnicas. Si tiene una comunicación abierta con su cliente, notará que se necesita tiempo, mucho tiempo, para tener una buena idea sobre su negocio. A menudo, es posible que su cliente no lo sepa exactamente por sí mismo, y solo al sentarse puede comenzar a pensar en ello detenidamente.

Es por eso que no debe temer a los grupos de dominio que cambian con el tiempo. Puede comenzar con un dominio de Factura pero, medio año después, nota que ha crecido demasiado para que usted y su equipo lo comprendan por completo. Tal vez la generación de facturas y los pagos sean dos sistemas complejos en sí mismos y, por lo tanto, se puedan dividir en dos grupos de dominio más adelante.

Mi punto de vista es que es saludable seguir iterando sobre la estructura de su dominio para seguir refactorizándolo. Con las herramientas adecuadas, no es nada difícil cambiar, dividir y refactorizar dominios.

En resumen: no tenga miedo de comenzar a usar dominios porque siempre puede refactorizarlos más adelante.

Ese es el enfoque que tomaría si quisiera comenzar a usar esta arquitectura orientada al dominio: intente identificar los subsistemas dentro del proyecto, dándose cuenta de que pueden — y cambiarán — con el tiempo. Puede sentarse con su cliente y pedirle que escriba las cosas, incluso podría hacer sesiones de tormenta de eventos con él. Juntos forman una imagen de lo que debería ser el proyecto, y esa imagen bien podría ser refinada y también cambiada en el futuro.

Y debido a que nuestro código de dominio tiene dependencias mínimas, es muy flexible y no cuesta mucho mover cosas o refactorizarlas.
