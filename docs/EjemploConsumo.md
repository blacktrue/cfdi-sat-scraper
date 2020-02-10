# Ejemplo de consumo

Este ejemplo está documentado con las siguientes consideraciones:

- El RFC está en el entorno en la variable SAT_AUTH_RFC
- La clave CIEC está en el entorno en la variable SAT_AUTH_CIEC
- Desde donde se está llamando al código existen las carpetas `build/cookies/` y `build/cfdis/`
- Se está usando el `ConsoleCaptchaResolver` así que se espera que si se le solicita un captcha lo
  resuelva y escriba su contenido. El capta está en `captcha.png`

Y se espera que:

- Se pueda reutilizar la cookie si no ha expirado en el cliente o en el servidor y
  así no tener que volver a resolver un captcha.
- Se carge una lista de CFDI recibidos y vigentes entre 2019-12-01 y 2019-12-31.
- Se descarguen los XML correspondientes a dichos registros.

```php
<?php

declare(strict_types=1);

require __DIR__ . '/../vendor/autoload.php';

use GuzzleHttp\Client;
use GuzzleHttp\Cookie\FileCookieJar;
use PhpCfdi\CfdiSatScraper\Captcha\Resolvers\ConsoleCaptchaResolver;
use PhpCfdi\CfdiSatScraper\Filters\Options\DownloadTypesOption;
use PhpCfdi\CfdiSatScraper\Filters\Options\StatesVoucherOption;
use PhpCfdi\CfdiSatScraper\Query;
use PhpCfdi\CfdiSatScraper\SATScraper;

$rfc = strval(getenv('SAT_AUTH_RFC'));
$claveCiec = strval(getenv('SAT_AUTH_CIEC'));
$cookieJarPath = sprintf('%s/build/cookies/%s.json', getcwd(), $rfc);
$downloadsPath = sprintf('%s/build/cfdis/%s', getcwd(), $rfc);

$client = new Client();
$cookie = new FileCookieJar($cookieJarPath, true);
$captchaResolver = new ConsoleCaptchaResolver();

$satScraper = new SATScraper($rfc, $claveCiec, $client, $cookie, $captchaResolver);

$query = new Query(new DateTimeImmutable('2019-12-01'), new DateTimeImmutable('2019-12-31'));
$query->setDownloadType(DownloadTypesOption::recibidos()) // default: emitidos
    ->setStateVoucher(StatesVoucherOption::vigentes());   // default: todos

$list = $satScraper->downloadPeriod($query);
printf("\nSe encontraron %d registros", $list->count());

$satScraper->downloader()->setMetadataList($list)->saveTo($downloadsPath, true);
foreach ($list as $item) {
    echo PHP_EOL, $item->uuid(), ': ',
    var_export(file_exists(sprintf('%s/%s.xml', $downloadsPath, $item->uuid())), true);
}
echo PHP_EOL;
```