# Acciones

Ahora que podemos trabajar con datos de forma segura y transparente, debemos comenzar a hacer algo con ellos.

Al igual que no queremos trabajar con arreglos aleatorios llenos de datos, tampoco queremos que la parte más crítica de nuestro proyecto, la funcionalidad comercial, se extienda a través de funciones y clases aleatorias.

Aquí hay un ejemplo: una de las historias de usuario en su proyecto podría ser para "un administrador para crear una factura". Esto significa que guardaremos una factura en la base de datos, pero hay más que hacer que eso.

Calcule el precio de cada línea de factura individual y el precio total

- Generar un número de factura
- Guardar la factura en la base de datos
- Crear un pago a través del proveedor de pago
- Crear un PDF con toda la información relevante
- Enviar este PDF al cliente

Una práctica común en Laravel es crear "modelos gordos" que manejarán toda esta funcionalidad. En este capítulo, veremos otro enfoque para agregar este comportamiento a nuestra base de código.

En lugar de mezclar funcionalidad en modelos o controladores, trataremos estas historias de usuario como ciudadanos de primera clase del proyecto. Tiendo a llamar a estas “acciones”.

## Terminología

Antes de ver su uso, necesitamos discutir cómo se estructuran las acciones. Para empezar, viven en el dominio.

En segundo lugar, son clases simples sin abstracciones ni interfaces. Una acción es una clase que toma entrada, hace algo y da salida. Es por eso que una acción generalmente solo tiene un método público y, a veces, un constructor.

Como convención en nuestros proyectos, decidimos sufijar todas nuestras clases. Sin duda, `CreateInvoice` suena bien, pero tan pronto como esté tratando con varios cientos o miles de clases, querrá asegurarse de que no se produzcan colisiones de nombres. Verá, `CreateInvoice` también podría ser el nombre de un controlador invocable, un comando, un trabajo o una solicitud. Preferimos eliminar tanta confusión como sea posible, por lo tanto, `CreateInvoiceAction` será el nombre.

Evidentemente, esto significa que los nombres de las clases se vuelven más largos. La realidad es que si está trabajando en proyectos más grandes, no puede evitar elegir nombres más largos para asegurarse de que no haya confusión posible. Aquí hay un ejemplo extremo de uno de nuestros proyectos (no estoy bromeando):
`CreateOrUpdateHabitantContractUnitPackageAction`.

Odiamos este nombre al principio. Intentamos desesperadamente encontrar uno más corto. Sin embargo, al final, tuvimos que admitir que la claridad de lo que es una clase es lo más importante. El autocompletado de nuestro IDE se encargará de los inconvenientes de los nombres largos de todos modos.

Cuando nos decidimos por un nombre de clase, el siguiente obstáculo a superar es nombrar el método público para usar nuestra acción. Una opción es hacerlo invocable, así:

```php
class CreateInvoiceAction
{
    public function __invoke(InvoiceData $invoiceData): Invoice
    {
        // ...
    }
}
```
Sin embargo, hay un problema práctico con este enfoque. Para entenderlo, necesito mencionar algo que veremos más adelante en este capítulo y es que, a veces, creamos acciones a partir de otras acciones. Se vería algo como esto:

```php
class CreateInvoiceAction
{
    private CreateInvoiceLineAction $createInvoiceLineAction;

    public function __construct(
        CreateInvoiceLineAction $createInvoiceLineAction
    ) { /* ... */ }

    public function __invoke(InvoiceData $invoiceData): Invoice
    {
        foreach ($invoiceData->lines as $lineData) {
            $invoice->addLine(
                ($this->createInvoiceLineAction)($lineData)
            );
        }
    }
}
```
¿Puedes ver el problema? PHP no le permite invocar directamente una propiedad invocable en una clase, ya que en su lugar está buscando un método de clase. Es por eso que tendrás que envolver la acción entre paréntesis antes de llamarla.

Si no quieres usar esa sintaxis rara, hay alternativas: podríamos, por ejemplo, usar `handle`, que Laravel suele usar como nombre predeterminado en este tipo de casos. Una vez más, hay un problema con él, específicamente porque Laravel lo usa.

Siempre que Laravel le permita usar `handle` en, por ejemplo, trabajos o comandos, también proporcionará inyección de método desde el contenedor de dependencia. En nuestras acciones, solo queremos que el constructor tenga capacidades de inyección de dependencia. Una vez más, veremos de cerca las razones detrás de esto más adelante en este capítulo.

Entonces el mango también está fuera. Cuando comenzamos a usar acciones, en realidad pensamos mucho en este enigma de nombres. Al final nos decidimos por ejecutar. Sin embargo, tenga en cuenta que es libre de crear sus propias convenciones de nomenclatura porque el punto aquí es más sobre el patrón de uso de acciones que sobre sus nombres.

Así que `handle` también está fuera. Cuando comenzamos a usar acciones, en realidad pensamos mucho en este enigma de nombres. Al final nos decidimos por `execute`. Sin embargo, tenga en cuenta que es libre de crear sus propias convenciones de nomenclatura porque el punto aquí es más sobre el patrón de uso de acciones que sobre sus nombres.

## En practica

Con toda la terminología fuera del camino, hablemos de por qué las acciones son útiles y cómo usarlas realmente.

Primero hablemos de la reutilización. El truco cuando se usan acciones es dividirlas en partes lo suficientemente pequeñas para que algunas cosas sean reutilizables, mientras se mantienen lo suficientemente grandes para no terminar con una sobrecarga de clases. Tome nuestro ejemplo de factura: generar un PDF a partir de una factura es algo que probablemente suceda desde varios contextos en nuestra aplicación. Claro, está el PDF que se genera cuando se crea realmente una factura, pero es posible que un administrador también desee ver una vista previa o un borrador antes de enviarlo.

Estas dos historias de usuario: _"creación de una factura"_ y _"vista previa de una factura"_ obviamente requieren dos puntos de entrada y, por lo tanto, dos controladores. Sin embargo, por otro lado, generar el PDF a partir de la factura es algo que se hace en ambos casos.

Cuando comience a pensar en lo que la aplicación realmente hará, notará que hay muchas acciones que se pueden reutilizar. Por supuesto, también debemos tener cuidado de no abstraer demasiado nuestro código. A menudo es mejor copiar y pegar un poco de código que hacer abstracciones prematuras.

Una buena regla general al hacer abstracciones es pensar en la funcionalidad en lugar de las propiedades técnicas del código. Cuando dos acciones pueden hacer cosas similares, pero se usan en contextos completamente diferentes, debe tener cuidado de no comenzar a abstraerlas demasiado pronto.

Por otro lado, hay casos en los que las abstracciones pueden ser útiles. Tome nuevamente nuestro ejemplo de PDF de factura: es probable que necesite generar más PDF que solo para facturas, al menos ese es el caso en nuestros proyectos. Podría tener sentido tener una `GeneratePdfAction` general que acepte una interfaz `toPDF`, que luego implementa `Invoice`. Esto significa que la funcionalidad para generar archivos PDF ahora se puede reutilizar en todo el proyecto.

Pero, seamos honestos; lo más probable es que la mayoría de nuestras acciones sean bastante específicas para sus historias de usuario y no sean reutilizables. Podría pensar que las acciones, en estos casos, son una sobrecarga innecesaria. Sin embargo, espere, porque la reutilización no es la única razón para usarlos. En realidad, la razón más importante no tiene nada que ver con los beneficios técnicos: las acciones permiten al programador pensar de forma más cercana al mundo real, en lugar del código.

Digamos que necesita hacer cambios en la forma en que se crean las facturas. Una aplicación típica de Laravel probablemente tendrá esta lógica de creación de facturas distribuida en un controlador y un modelo, tal vez un trabajo que genera el PDF y, finalmente, un detector de eventos para enviar el correo de la factura. Son muchos lugares que debes conocer. Una vez más, nuestro código se distribuye por la base de código, agrupado por sus propiedades técnicas, en lugar de su significado.

Las acciones reducen la carga cognitiva que introduce dicho sistema. Si necesita trabajar en cómo se crean las facturas, simplemente puede ir a la clase de acción y comenzar desde allí.

No se equivoque: las acciones pueden funcionar muy bien junto con, por ejemplo, trabajos asincrónicos y detectores de eventos; aunque estos trabajos y oyentes simplemente proporcionan la infraestructura para que las acciones funcionen, y no la lógica empresarial en sí. Este es un buen ejemplo de por qué necesitamos dividir las capas de dominio y aplicación: cada una tiene su propio propósito.

Así que ahora tenemos reutilización y una reducción de la carga cognitiva, ¡pero aún hay más!

Debido a que las acciones son pequeñas piezas de software que viven casi por sí mismas, es muy fácil probarlas unitariamente. En sus pruebas, no tiene que preocuparse por enviar solicitudes HTTP falsas, hacer fachadas, etc. Simplemente puede realizar una nueva acción, tal vez proporcionar algunas dependencias simuladas, pasarle los datos de entrada requeridos y hacer afirmaciones en su salida.

Por ejemplo, tomemos `CreateInvoiceLineAction`. Va a:
- tomar datos sobre qué producto se facturará
- recibir una cantidad más un período
- calcular el precio total y luego,
- calcular precios con y sin IVA

Estas son cosas para las que puede escribir pruebas unitarias robustas, pero simples.

Si todas sus acciones se prueban correctamente, puede estar seguro de que la mayor parte de la funcionalidad que debe proporcionar la aplicación realmente funciona según lo previsto. Ahora solo es cuestión de usar estas acciones de manera que tengan sentido para el usuario final y escribir algunas pruebas de integración para esas piezas.

## Acciones de composición

Una característica importante de las acciones que ya mencioné antes brevemente, es cómo usan la inyección de dependencia. Dado que estamos usando el constructor para pasar datos del contenedor y el método `execute` para pasar datos relacionados con el contexto, somos libres de componer acciones a partir de acciones a partir de acciones a partir de...

Entiendes la idea. Sin embargo, aclaremos que una cadena de dependencia profunda es algo que desea evitar (hace que el código sea complejo y altamente dependiente entre sí), sin embargo, hay varios casos en los que tener DI es muy beneficioso.

Tomemos de nuevo el ejemplo de CreateInvoiceLineAction que tiene que calcular los precios del IVA. Ahora, dependiendo del contexto, una línea de factura puede tener un precio que incluya o no IVA. Calcular los precios del IVA es algo trivial, pero no queremos que `CreateInvoiceLineAction` se preocupe por los detalles.

Así que imagine que tenemos una clase `VatCalculator` simple, que es algo que podría vivir en el espacio de nombres `\Support`, podría inyectarse así:

```php
class CreateInvoiceLineAction
{
    private VatCalculator $vatCalculator;

    public function __construct(VatCalculator $vatCalculator)
    {
        $this->vatCalculator = $vatCalculator;
    }

    public function execute(
        InvoiceLineData $invoiceLineData
    ): InvoiceLine {
        // ...
    }
}
```
Y lo usarías así:
```php
public function execute(
    InvoiceLineData $invoiceLineData
): InvoiceLine {
    $item = $invoiceLineData->item;

    if ($item->vatIncluded()) {
        [$priceIncVat, $priceExclVat] =
            $this->vatCalculator->vatIncluded(
                $item->getPrice(),
                $item->getVatPercentage()
            );
    } else {
        [$priceIncVat, $priceExclVat] =
            $this->vatCalculator->vatExcluded(
                $item->getPrice(),
                $item->getVatPercentage()
            );
    }
    
    $amount = $invoiceLineData->item_amount;

    return new InvoiceLine([
         'item_price' => $item->getPrice(),
         'total_price' => $amount * $priceIncVat,
         'total_price_excluding_vat' => $amount * $priceExclVat,
    ]);
}
```
`CreateInvoiceLineAction`, a su vez, se inyectaría en `CreateInvoiceAction`. Y este nuevamente tiene otras dependencias: `CreatePdfAction` y `SendMailAction`, por ejemplo.

Puede ver cómo la composición puede ayudarlo a mantener pequeñas las acciones individuales y, al mismo tiempo, permitir que la funcionalidad comercial compleja se codifique de una manera clara y fácil de mantener.

## Alternatives to actions
