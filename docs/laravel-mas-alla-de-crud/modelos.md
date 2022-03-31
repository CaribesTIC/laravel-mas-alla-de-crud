# Modelos

En los capítulos anteriores, hablé sobre dos de los tres componentes básicos de cada aplicación: DTO y acciones: datos y funcionalidad. En este capítulo, veremos la última pieza que considero parte de este núcleo: exponer datos que persisten en un almacén de datos; en otras palabras: modelos.

Ahora, los modelos son un tema complicado. Laravel proporciona una gran cantidad de funciones en sus clases de modelo Eloquent, lo que significa que no solo representan los datos en un almacén de datos, sino que también le permiten crear consultas, cargar y guardar datos, tener un sistema de eventos incorporado y más.

En este capítulo, no le diré que se deshaga de toda la funcionalidad del modelo que proporciona Laravel; es bastante útil. Sin embargo, nombraré algunas trampas que debe tener en cuenta y las soluciones para ellas, de modo que incluso en proyectos grandes, los modelos no sean la causa de un mantenimiento difícil.

Mi punto de vista es que deberíamos abrazar el marco en lugar de tratar de luchar contra él. Aunque deberíamos aceptarlo de tal manera que los grandes proyectos se mantengan.

## Modelos ≠ lógica de negocios

El primer escollo en el que caen muchos desarrolladores es que piensan en los modelos como el lugar ideal cuando se trata de lógica empresarial. Ya enumeré algunas responsabilidades de los modelos que están integrados en Laravel, y diría que hay que tener cuidado de no agregar más.

Suena muy atractivo al principio, poder hacer algo como `$invoiceLine->price_including_vat` o `$invoice->total_price`; y seguro que lo hace. De hecho, creo que las facturas y las líneas de factura deberían tener estos métodos. Sin embargo, hay una distinción importante que hacer: estos métodos no deberían calcular nada. Echemos un vistazo a lo que no se debe hacer:

Aquí hay un descriptor de acceso `total_price` en nuestro modelo de factura, recorriendo todas las líneas de factura y haciendo la suma de su precio total.

```php
class Invoice extends Model
{
    public function getTotalPriceAttribute(): int
    {
        return $this->invoiceLines
            ->reduce(
                fn (int $totalPrice, InvoiceLine $invoiceLine) =>
                    $totalPrice + $invoiceLine->total_price,
                0
            );
    }
}
```

Y así es como se calcula el precio total por línea:

```php
class InvoiceLine extends Model
{
    public function getTotalPriceAttribute(): int
    {
        $vatCalculator = app(VatCalculator::class);

        $price = $this->item_amount * $this->item_price;

        if ($this->price_excluding_vat) {
            $price = $vatCalculator->totalPrice(
                $price,
                $this->vat_percentage
            );
        }

        return $price;
    }
}
```

Ya que leyó el capítulo anterior sobre acciones, puede adivinar qué haría yo en su lugar: calcular el precio total de una factura es una historia de usuario que debe estar representada por una acción.

Los modelos `Invoice` y `InvoiceLine` podrían tener las propiedades simples `total_price` y `price_including_vat`, pero primero se calculan mediante acciones y luego se almacenan en la base de datos. Cuando usa `$invoice->total_price`, simplemente está leyendo datos que ya se calcularon antes.

Hay algunas ventajas de este enfoque. Primero lo obvio: el rendimiento. Solo está haciendo los cálculos una vez, no cada vez que necesita los datos. Segundo: puede consultar los datos calculados directamente. Y tercero: no tiene que preocuparse por los efectos secundarios.

Ahora, podríamos iniciar un debate purista sobre cómo la responsabilidad única ayuda a mantener sus clases pequeñas, mejor mantenibles y fácilmente comprobables y cómo la inyección de dependencia es superior a la ubicación del servicio, pero prefiero decir lo obvio en lugar de tener largos debates teóricos donde sé que simplemente hay dos lados que no estarán de acuerdo.

Entonces, lo obvio: aunque le gustaría poder hacer `$invoice->send()` o `$invoice->toPdf()`, el código del modelo está creciendo y creciendo. Esto es algo que sucede con el tiempo y no parece ser un gran problema al principio. De hecho, `$invoice->toPdf()` en realidad podría ser solo una o dos líneas de código para empezar.

Sin embargo, por experiencia, estas una o dos líneas se suman. Una o dos líneas no es el problema, pero cien veces una o dos líneas sí lo es. La realidad es que las clases modelo crecen con el tiempo y pueden crecer bastante.

Incluso si no está de acuerdo conmigo sobre las ventajas que trae la responsabilidad única y la inyección de dependencia, hay poco en lo que estar de acuerdo: una clase modelo con cientos de líneas de código no se podrá mantener.

Para resumir, piense en los modelos y su propósito como si solo le proporcionaran datos a usted y deje que otra persona se ocupe de asegurarse de que los datos se calculen correctamente.

## Reducción de modelos

Si nuestro objetivo es mantener las clases de modelos razonablemente pequeñas — lo suficientemente pequeñas como para poder entenderlas simplemente abriendo su archivo — necesitamos mover algunas cosas más. Idealmente, solo queremos mantener los datos leídos de la base de datos, accesos simples para cosas que no podemos calcular de antemano, conversiones y relaciones.

Otras responsabilidades deben trasladarse a otras clases. Un ejemplo son los ámbitos de consulta. Podríamos moverlos fácilmente a clases de creación de consultas dedicadas.

Lo crea o no, las clases de creación de consultas son en realidad la forma normal de usar Eloquent; los ámbitos son simplemente azúcar sintáctico encima de ellos. Este es el aspecto que podría tener una clase de generador de consultas.

```php
namespace Domain\Invoices\QueryBuilders;

use Domain\Invoices\States\Paid;
use Illuminate\Database\Eloquent\Builder;

class InvoiceQueryBuilder extends Builder
{
    public function wherePaid(): self
    {
        return $this->whereState('status', Paid::class);
    }
}
```
A continuación, anulamos el método `newEloquentBuilder` en nuestro modelo y devolvemos nuestra clase personalizada. Laravel lo usará a partir de ahora.

```php
namespace Domain\Invoices\Models;

use Domain\Invoices\QueryBuilders\InvoiceQueryBuilder;

class Invoice extends Model
{
    public function newEloquentBuilder($query): InvoiceQueryBuilder
    {
        return new InvoiceQueryBuilder($query);
    }
}
```
Esto es lo que quise decir con adoptar el marco: no es necesario introducir nuevos patrones como repositorios per se; puede construir sobre lo que proporciona Laravel. Pensándolo un poco, logramos el equilibrio perfecto entre el uso de los productos proporcionados por el marco y la prevención de que nuestro código crezca demasiado en lugares específicos.

Usando esta mentalidad, también podemos proporcionar clases de colección personalizadas para las relaciones. Laravel tiene un excelente soporte de colección, aunque a menudo terminas con largas cadenas de funciones de colección, ya sea en el modelo o en la capa de la aplicación. Nuevamente, esto no es ideal y, afortunadamente, Laravel nos brinda los enlaces necesarios para agrupar la lógica de la colección en una clase dedicada.

Aquí hay un ejemplo de una clase de colección personalizada y tenga en cuenta que es completamente posible combinar varios métodos en otros nuevos, evitando largas cadenas de funciones en otros lugares.

```php
namespace Domain\Invoices\Collections;

use Domain\Invoices\Models\InvoiceLines;
use Illuminate\Database\Eloquent\Collection;

class InvoiceLineCollection extends Collection
{
    public function creditLines(): self
    {
        return $this->filter(fn (InvoiceLine $invoiceLine) =>
            $invoiceLine->isCreditLine()
        );
    }
}
```
Así es como vincula una clase de cobro a un modelo — `InvoiceLine` — en este caso:

```php
namespace Domain\Invoices\Models;

use Domain\Invoices\Collection\InvoiceLineCollection;

class InvoiceLine extends Model
{
    public function newCollection(
        array $models = []
    ): InvoiceLineCollection {
        return new InvoiceLineCollection($models);
    }

    public function isCreditLine(): bool
    {
        return $this->price < 0.0;
    }
}
```
Cada modelo que tenga una relación `HasMany` con `InvoiceLine`, ahora usará nuestra clase de colección en su lugar.

```php
$invoice
    ->invoiceLines
    ->creditLines()
    ->map(function (InvoiceLine $invoiceLine) {
        // ...
    });
```
Intente mantener sus modelos limpios y orientados a datos, en lugar de que proporcionen lógica de negocios. Hay mejores lugares para manejarlo.

## Modelos basados en eventos

En el capítulo 3, ya mencioné los sistemas controlados por eventos y cómo ofrecen más flexibilidad, a costa de la complejidad.

Es posible que prefiera un enfoque basado en eventos, por lo que creo que hay una nota importante que hacer sobre los eventos relacionados con el modelo. Laravel emitirá eventos de modelos genéricos de forma predeterminada y espera que uses un observador de modelos o configures oyentes en el propio modelo.

También hay otro enfoque, uno que es más flexible y robusto. Puede reasignar eventos de modelos genéricos a clases de eventos específicas, según el modelo, y usar suscriptores de eventos dedicados para ellos.

En un modelo, se vería así:

```php
class Invoice
{
    protected $dispatchesEvents = [
        'saving' => InvoiceSavingEvent::class,
        'deleting' => InvoiceDeletingEvent::class,
    ];
}
```
A su vez, una clase de evento tan específica sería similar a:

```php
class InvoiceSavingEvent
{
    public Invoice $invoice;

    public function __construct(Invoice $invoice)
    {
        $this->invoice = $invoice;
    }
}
```

Y finalmente, el suscriptor podría implementarse como:

```php
use Illuminate\Events\Dispatcher;

class InvoiceSubscriber
{
    private CalculateTotalPriceAction $calculateTotalPriceAction;
    
    public function __construct(
        CalculateTotalPriceAction $calculateTotalPriceAction
    ) {/* ... */}

    public function saving(InvoiceSavingEvent $event): void
    {
        $invoice = $event->invoice;

        $invoice->total_price =
        ($this->calculateTotalPriceAction)($invoice);
    }
    
    public function subscribe(Dispatcher $dispatcher): void
    {
        $dispatcher->listen(
            InvoiceSavingEvent::class,
            self::class . '@saving'
        );
    }
}
```

El cual quedaría registrado en el `EventServiceProvider`, así:

```php
class EventServiceProvider extends ServiceProvider
{
    protected $subscribe = [
        InvoiceSubscriber::class,
    ];
}
```
Este enfoque también le brinda la flexibilidad de conectar sus propios eventos de modelo personalizados y manejarlos de la misma manera que se manejan los eventos elocuentes. Y una vez más, las clases de suscripción permiten que nuestros modelos se mantengan pequeños y fáciles de mantener.

## Empty bags of nothingness
