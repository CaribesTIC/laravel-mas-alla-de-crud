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

¿Es eso difícil de entender? Vamos a hacerlo paso a paso.

Comenzamos con una clase abstracta `InvoiceState`, esta clase describe toda la funcionalidad que pueden proporcionar estados de factura concretos. En nuestro caso solo queremos que aporten un color:

```php
abstract class InvoiceState
{
    abstract public function colour(): string;
}
```
A continuación, creamos dos clases, cada una de las cuales representa un estado concreto:
```php
class PendingInvoiceState extends InvoiceState
{
    public function colour(): string
    {
        return 'orange';
    }
}

class PaidInvoiceState extends InvoiceState
{
    public function colour(): string
    {
        return 'green';
    }
}
```
Lo primero que debe notar es que cada una de estas clases puede probarse fácilmente por separado. Echale un vistazo a éste ejemplo:
```php
class InvoiceStateTest extends TestCase
{
    /** @test */
    public function the_colour_of_pending_is_orange
    {
        $state = new PendingInvoiceState();

        $this->assertEquals('orange', $state->colour());
    }
}
```
En segundo lugar, debe tener en cuenta que _"colours"_ es un ejemplo ingenuo que se usa para explicar el patrón. También podría tener una lógica comercial más compleja encapsulada por un estado. Toma esta: ¿debe pagarse una factura? Esto, por supuesto, depende del estado y de si ya se pagó o no, pero también podría depender del tipo de factura con la que estemos tratando. Digamos que nuestro sistema admite notas de crédito que no tienen que pagarse, o permite facturas con un precio de 0. Esta lógica empresarial puede encapsularse en estas clases de estado.

Sin embargo, falta una cosa para que esas reglas complejas funcionen: necesitamos poder ver el modelo desde dentro de nuestra clase estado, si vamos a decidir si esa factura debe pagarse o no. En otras palabras, necesitaremos inyectar nuestro modelo `Invoice` en la clase estado.

Podríamos manejar esa configuración repetitiva en nuestra clase principal abstracta `InvoiceState`:
```php
abstract class InvoiceState
{
    /** @var Invoice */
    protected $invoice;

    public function __construct(Invoice $invoice) { /* ... */ }

    abstract public function mustBePaid(): bool;

    // ...
}
```
Y luego implemente `mustBePaid` en cada estado concreto:
```php
class PendingInvoiceState extends InvoiceState
{
    public function mustBePaid(): bool
    {
        return $this->invoice->total_price > 0
            && $this->invoice->type->equals(InvoiceType::DEBIT());
    }

    // ...
}

class PaidInvoiceState extends InvoiceState
{
    public function mustBePaid(): bool
    {
        return false;
    }

    // ...
}
```
Nuevamente, podemos escribir pruebas unitarias simples para cada estado, y nuestro modelo `Invoice` simplemente puede hacer algo como esto:
```php
class Invoice extends Model
{
    public function getStateAttribute(): InvoiceState
    {
        return new $this->state_class($this);
    }

    public function mustBePaid(): bool
    {
        return $this->state->mustBePaid();
    }
}
```
En la base de datos podemos guardar la clase de estado del modelo concreto en el campo `state_class` y listo. Obviamente, hacer este mapeo manualmente (guardar y cargar desde y hacia la base de datos) se vuelve tedioso muy rápido. Es por eso que escribí el paquete `spatie/laravel-model-states` que se encarga de todo el trabajo duro por ti.

Sin embargo, el comportamiento específico del estado, en otras palabras, _"el patrón del estado"_, es solo la mitad de la solución; todavía tenemos que manejar la transición del estado de la factura de uno a otro, y asegurarnos de que solo estados específicos puedan hacer la transición a otros. Así que echemos un vistazo a las transiciones de estado.

## Transiciones

¿Recuerdas que hablé sobre alejar la lógica comercial de los modelos y permitirles solo proporcionar datos de una manera viable desde la base de datos? El mismo pensamiento se puede aplicar a estados y transiciones.

Debemos evitar los efectos secundarios al usar estados: cosas como hacer cambios en la base de datos, enviar correos, etc. Los estados deben usarse para leer o proporcionar datos. Las transiciones, por otro lado, no proporcionan nada. Más bien, se aseguran de que el estado de nuestro modelo pase correctamente de uno a otro, lo que lleva a
efectos secundarios aceptables.

Dividir estas dos preocupaciones en clases separadas nos brinda las mismas ventajas sobre las que escribí una y otra vez: mejor capacidad de prueba y carga cognitiva reducida. Permitir que una clase solo tenga una responsabilidad hace que sea más fácil dividir un problema complejo en varias partes-fáciles-de-entender.

Entonces, transiciones: clases que tomarán un modelo, una factura en nuestro caso, y cambiarán el estado de esa factura cuando se permita a otro. En algunos casos, puede haber pequeños efectos secundarios, como escribir un mensaje de registro o enviar una notificación sobre la transición de estado.

Una implementación ingenua podría verse así:
```php
class PendingToPaidTransition
{
    public function __invoke(Invoice $invoice): Invoice
    {
        if (! $invoice->mustBePaid()) {
            throw new InvalidTransitionException(self::class, $invoice);
        }

        $invoice->status_class = PaidInvoiceState::class;
        $invoice->save();

        History::log($invoice, "Pending to Paid");
    }
}
```
De nuevo, hay muchas cosas que puedes hacer con este patrón básico:

- Definir todas las transiciones permitidas en el modelo
- Transición de un estado directamente a otro, mediante el uso de una clase de transición bajo el capó
- Determinar automáticamente a qué estado hacer la transición en función de un conjunto de parámetros

El paquete que mencioné antes agrega soporte para transiciones, así como administración básica de transiciones. Sin embargo, si desea máquinas de estado complejas, es posible que desee ver otros paquetes; Discutiré un ejemplo en las notas al pie al final de este libro.

## Estados sin transiciones

Cuando pensamos en _"estado"_, a menudo pensamos que no pueden existir sin transiciones. Sin embargo, eso no es cierto: un objeto puede tener un estado que nunca cambie y no se requieren transiciones para aplicar el patrón de estado. ¿Porque es esto importante? Bueno, eche un vistazo de nuevo a nuestra implementación `PendingInvoiceState::mustBePaid`:
```php
class PendingInvoiceState extends InvoiceState
{
    public function mustBePaid(): bool
    {
        return $this->invoice->total_price > 0
            && $this->invoice->type->equals(InvoiceType::DEBIT());
    }
}
```
Ya que queremos usar el patrón de estado para reducir los frágiles bloques if/else en nuestro código, ¿puedes adivinar a dónde voy con esto? ¿Ha considerado que `$this->invoice->type->equals(InvoiceType::DEBIT())` es de hecho una declaración if disfrazada?

De hecho, `InvoiceType` también podría muy bien aplicar el patrón de estado. Es simplemente un estado que probablemente nunca cambiará para un objeto determinado. Mira eso:
```php
abstract class InvoiceType
{
    protected Invoice $invoice;

    // ...

    abstract public function mustBePaid(): bool;
}

class CreditInvoiceType extends InvoiceType
{
    public function mustBePaid(): bool
    {
        return false;
    }
}

class DebitInvoiceType extends InvoiceType
{
    public function mustBePaid(): bool
    {
        return true;
    }
}
```
Ahora podemos refactorizar nuestro `PendingInvoiceState::mustBePaid` así:
```php
class PendingInvoiceState extends InvoiceState
{
    public function mustBePaid(): bool
    {
        return $this->invoice->total_price > 0
            && $this->invoice->type->mustBePaid();
    }
}
```
Reducir las declaraciones `if/else` en nuestro código permite que ese código sea más lineal, lo que a su vez es más fácil de razonar.

El patrón de estado es, en mi opinión, asombroso. Nunca más tendrá que volver a escribir grandes declaraciones `if/else` — en la vida real, a menudo hay más de dos estados de factura — y permite un código limpio y comprobable.

Es un patrón que puede introducir gradualmente en sus bases de código existentes, y estoy seguro de que será de gran ayuda para mantener el proyecto a largo plazo.
