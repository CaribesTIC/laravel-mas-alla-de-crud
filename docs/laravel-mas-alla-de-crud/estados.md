# Estados

El patrón de estado es una de las mejores formas de agregar un comportamiento específico del estado a los modelos, sin dejar de mantenerlos limpios.

Este capítulo hablará sobre el patrón de estado y, específicamente, cómo aplicarlo a los modelos. Puede pensar en este capítulo como una extensión del capítulo 4, donde escribí sobre cómo pretendemos mantener nuestras clases modelo manejables evitando que manejen la lógica empresarial.

Alejar la lógica empresarial de los modelos plantea un problema con un caso de uso muy común: ¿qué hacer con los estados del modelo?

Una factura puede estar pendiente o pagada, un pago puede fallar o tener éxito. Dependiendo del estado, un modelo debe comportarse de manera diferente; ¿Cómo acortamos esta brecha entre los modelos y la lógica empresarial?

Los estados y las transiciones entre ellos, son un caso de uso frecuente en grandes proyectos y de hecho son tan frecuentes que merecen un capítulo aparte.

## El patrón de estado

En esencia, el patrón de estado es un patrón simple, pero permite una funcionalidad muy poderosa. Tomemos nuevamente el ejemplo de las facturas: pueden estar pendientes o pagadas. Comencemos con un ejemplo simplificado porque quiero que entienda cómo el patrón de estado nos permite mucha flexibilidad.

Digamos que la descripción general de la factura debe mostrar una insignia que represente el estado de esa factura; mientras está pendiente, la insignia es de color naranja y verde si se paga.

Un enfoque de modelo gordo ingenuo haría algo como esto:

```php
class Invoice extends Model
{
    // ...

    public function getStateColour(): string
    {
        if ($this->state->equals(InvoiceState::PENDING())) {
            return 'orange';
        }

        if ($this->state->equals(InvoiceState::PAID())) {
            return 'green';
        }

        return 'grey';
    }
}
```
Dado que estamos usando algún tipo de clase de enumeración para representar el valor del estado, podríamos usar esa enumeración para encapsular el color correspondiente:

```php
/**
 * @method static self PENDING()
 * @method static self PAID()
 */
class InvoiceState extends Enum
{
    private const PENDING = 'pending';
    private const PAID = 'paid';

    public function getColour(): string
    {
        if ($this->value === self::PENDING) {
            return 'orange';
        }

        if ($this->value === self::PAID) {
            return 'green';
        }

        return 'grey';
    }
}
```
Se usaría así:
```php
class Invoice extends Model
{
    // ...

    public function getStateColour(): string
    {
        return $this->state->getColour();
    }
}
```
Podría escribir `InvoiceState::getColour` aún más corto, usando matrices:
```php
class InvoiceState extends Enum
{
    public function getColour(): string
    {
        return [
            self::PENDING => 'orange',
            self::PAID => 'green',
        ][$this->value] ?? 'grey';
    }
}
```
O usando match en PHP 8:
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
Cualquiera que sea el enfoque que prefiera, en esencia, está enumerando todas las opciones disponibles, verificando si una de ellas coincide con la actual y haciendo algo en función del resultado. Es una gran declaración if/else, cualquiera que sea el azúcar sintáctico que prefiera.

Con este enfoque, agregamos una responsabilidad, ya sea al modelo o a la clase enum, que es que debe saber qué debe hacer un estado específico y cómo funciona un estado. El patrón de estado invierte esto al revés: trata a _"un estado"_ como un ciudadano de primera clase de nuestra base de código. Cada estado está representado por una clase separada, y cada una de estas clases actúa sobre un sujeto.

Is that difficult to grasp? Let's take it step by step.
