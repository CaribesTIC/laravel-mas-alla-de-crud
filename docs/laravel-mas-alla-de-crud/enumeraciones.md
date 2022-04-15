# Enumeraciones

## ¿Enumeraciones o estados?

Hablando de estados en el capítulo anterior, es posible que esté pensando en enumeraciones en este momento. ¿Cuál es la diferencia entre ellos y los estados? ¿Cómo y cuándo usar enumeraciones?

Es un tema complicado, especialmente porque tanto las enumeraciones como el patrón de estado se pueden usar para lograr el mismo resultado. ¿Recuerda uno de los ejemplos del capítulo anterior? Podríamos modelar un estado con una clase de enumeración y determinar el comportamiento específico del valor con condicionales, así:

```php
class InvoiceState extends Enum
{
    public function getColour(): string
    {
        return match($this->value) {
            self::PENDING => 'orange',
            self::PAID => 'green',
            default => 'grey',
        };
    }
}
```
Tenga en cuenta que estoy usando la expresión `match` de PHP 8 aquí, también puede reemplazarla con una instrucción `switch` o `if`.
De todos modos, la diferencia entre las implementaciones de enumeración y estado es que el patrón de estado elimina esas condiciones al proporcionar una clase dedicada para cada valor posible. En efecto, funciona al revés.

El objetivo del patrón de estado es deshacerse de todos esos condicionales y, en su lugar, confiar en el poder del polimorfismo para determinar el flujo del programa. Por lo tanto, podríamos argumentar que es una forma de elegir cuál implementar: usar el patrón de estado para deshacerse de los flujos condicionales en su código y las enumeraciones para todo lo demás.

Pero, ¿qué es _“todo lo demás”?_ Si usamos una enumeración, habrá lugares en nuestro código donde los verifiquemos, por ejemplo, `if InvoiceType is X`, `than do Y`. En esencia, una condición que podría modelarse con el patrón de estado. Entonces, ¿hay casos de uso válidos para las enumeraciones?

Creo que aquí hay un poco de pragmatismo. Claro que podríamos usar el patrón de estado para modelar todos los flujos condicionales en nuestra aplicación, pero debemos tener en cuenta que el patrón de estado conlleva una sobrecarga significativa: debe crear clases para cada estado, configurar transiciones entre ellos y debe mantenerlos.

En otras palabras: _hay_ espacio para enumeraciones en nuestra base de código.

Si necesita una colección de valores relacionados y hay pequeños lugares donde el flujo de la aplicación está realmente determinado por esos valores, entonces sí: siéntase libre de usar enumeraciones simples. Pero si te das cuenta de que les asignas más y más funcionalidades relacionadas con el valor, diría que es hora de comenzar a observar el patrón de estado.

No hay una regla clara que pueda darle para resolver esto, por lo que deberá decidir qué solución usar caso-por-caso. Tenga cuidado de no complicar demasiado su código aplicando _siempre_ el patrón de estado, pero también tenga cuidado de mantener ese código mantenible, usando el patrón de estado cuando sea relevante.

## Enums!
