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

## Enumeraciones!

Entonces, con esa pregunta aclarada, me gustaría discutir algunas formas de agregar enumeraciones en PHP, porque no hay una implementación nativa para ellos.

Como breve resumen: un tipo de enumeración, _"enum"_ para abreviar, es un tipo de datos para categorizar valores con nombre. Las enumeraciones se pueden usar en lugar de cadenas codificadas para representar, por ejemplo, el estado de una publicación de blog de una manera estructurada y escrita.

PHP ofrece una implementación SPL muy básica, pero esto realmente no es suficiente. Por otro lado, hay un paquete popular escrito por Matthieu Napoli llamado `myclabs/php-enum` que es un paquete que muchas personas, incluido yo mismo, hemos estado usando en innumerables proyectos. Es realmente asombroso.

Ahora, quiero explorar algunas de las dificultades que puede encontrar al intentar resolver enumeraciones en el espacio del usuario y decirle mi solución preferida.

Entonces, volvamos a nuestro ejemplo de factura. Imagina que escribiríamos algo como esto:

```php
class Invoice
{
    public function setType(InvoiceType $type): void
    {
        $this->type = $type;
    }
}
```
Y asegúrese de que el valor de `Invoice::$type` sea siempre una de dos posibilidades: `credit` o `debit`.

Digamos que guardaríamos esta `Invoice` en una base de datos, su estado se representaría automáticamente como una cadena.

El paquete `myclabs/php-enum` nos permite escribir esto:

```php
use MyCLabs\Enum\Enum;

class InvoiceType extends Enum
{
    const CREDIT = 'credit';
    const DEBIT = 'debit';
}
```
Ahora, el ejemplo anterior de sugerencia de tipo `InvoiceType` obviamente no funciona: no podemos asegurar que sea uno de estos dos valores. Entonces podríamos usar los valores constantes directamente:

```php
class Invoice
{
    public function setType(string $type): void
    {
        $this->type = $type;
    }
}

// ...

$invoice->setType(InvoiceType::DEBIT);
```
Pero esto nos impide hacer una verificación de tipo adecuada, ya que cada cadena podría pasarse a `Invoice::setType`

```php
$invoice->setType('Whatever you want');
```
Un mejor enfoque es usar un poco de magia introducida por la biblioteca. Hacemos que los valores constantes sean privados, para que no sean accesibles directamente, y podemos usar métodos mágicos para resolver su valor:
```php
use MyCLabs\Enum\Enum;

class InvoiceType extends Enum
{
    private const CREDIT = 'credit';
    private const DEBIT = 'debit';
}

$invoice->setType(InvoiceType::CREDIT());
```
Usando el método mágico `__callStatic()` debajo, se construye un objeto de la clase `InvoiceType`, con el valor `'credit'` en él.

Ahora podemos escribir verificar para `InvoiceType` y asegurarnos de que la entrada sea uno de los dos valores definidos por la enumeración.

Here's the problem with the `myclabs/php-enum` package though: by relying on `__callStatic()`, we lose static analysis benefits like auto-completion and refactoring. Your IDE or static analysis tool won't know there's a `InvoiceType::CREDIT` or `InvoiceType::DEBIT` method available:

Sin embargo, aquí está el problema con el paquete `myclabs/php-enum`: al confiar en `__callStatic()`, perdemos los beneficios del análisis estático como el auto-completado y la refactorización. Su IDE o herramienta de análisis estático no sabrá que hay un método `InvoiceType::CREDIT` o `InvoiceType::DEBIT` disponible:

```php
$invoice->setType(InvoiceType::CREDIT());
```
Afortunadamente, este problema se puede resolver con DocBlocks:
```php
use MyCLabs\Enum\Enum;

/**
 * @method static self CREDIT()
 * @method static self DEBIT()
 */
class InvoiceType extends Enum
{
    private const CREDIT = 'credit';
    private const DEBIT = 'debit';
}

$invoice->setType(InvoiceType::CREDIT());
```
Pero ahora mantenemos el código duplicado: están los valores constantes y los bloques de documentos.

En este punto es hora de parar y pensar. En un mundo ideal, tendríamos enumeraciones integradas en PHP:
```php
enum InvoiceType {
    DEBIT, CREDIT;
}
```
Dado que ese no es el caso en este momento, estamos atascados con las implementaciones del espacio de usuario. Extender el sistema de tipos de PHP en el espacio del usuario probablemente signifique dos cosas: magia y reflexión.

Si ya confiamos en estos dos elementos, ¿por qué no hacer todo lo posible y hacer nuestra vida lo más simple posible?

Así es como escribo enumeraciones hoy:

```php
use Spatie\Enum\Enum;

/**
 * @method static self CREDIT()
 * @method static self DEBIT()
 */
class InvoiceType extends Enum
{
}
```

Este enfoque simplemente omite la necesidad de duplicar el código y determinará los valores de enumeración en función de los DocBlocks - DocBlocks que necesitamos escribir de todos modos - para garantizar un IDE adecuado y soporte de análisis estático.

De opinión, ¿verdad? Es menos código para mantener, con más beneficios. Una vez más, escribí un paquete para esto, se llama `spatie/enum`, y también hay una implementación de Laravel llamada `spatie/laravel-enum` que agrega conversión automática para las propiedades del modelo y cosas por el estilo.

Ahora, no tienes que usar ninguno de esos paquetes si no quieres. Simplemente quería explicar cómo puede tener un comportamiento similar a una enumeración, manteniendo los beneficios de un sistema fuertemente tipado y analizado estáticamente, sin soporte PHP nativo.

_Sé_ que esto está lejos de ser una situación ideal. Sería increíble ver un día soporte integrado para enumeraciones en PHP. Pero hasta entonces, esto es lo que hay.

