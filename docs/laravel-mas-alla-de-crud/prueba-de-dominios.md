# Prueba de dominios

Hasta este punto, he explicado los componentes básicos del código de dominio y cómo administrarlo a lo largo del tiempo. Todavía falta un aspecto crucial: cómo vamos a probar todo esto. Ya mostré algunas pruebas aquí y allá y le dije cómo nuestra arquitectura modular permite realizar pruebas unitarias sencillas, pero en este capítulo, le mostraré los entresijos de las pruebas de código de dominio.

Pero antes de hacerlo, hay un patrón más que quiero explicarles porque hará que nuestras pruebas sean diez veces más fáciles. Veamos el patrón de fábrica

## Test factories
