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

## Scaling down models
