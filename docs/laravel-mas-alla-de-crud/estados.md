# Estados

El patrón de estado es una de las mejores formas de agregar un comportamiento específico del estado a los modelos, sin dejar de mantenerlos limpios.

Este capítulo hablará sobre el patrón de estado y, específicamente, cómo aplicarlo a los modelos. Puede pensar en este capítulo como una extensión del capítulo 4, donde escribí sobre cómo pretendemos mantener nuestras clases modelo manejables evitando que manejen la lógica empresarial.

Alejar la lógica empresarial de los modelos plantea un problema con un caso de uso muy común: ¿qué hacer con los estados del modelo?

Una factura puede estar pendiente o pagada, un pago puede fallar o tener éxito. Dependiendo del estado, un modelo debe comportarse de manera diferente; ¿Cómo acortamos esta brecha entre los modelos y la lógica empresarial?

Los estados y las transiciones entre ellos, son un caso de uso frecuente en grandes proyectos y de hecho son tan frecuentes que merecen un capítulo aparte.

## The state pattern
