# AwsSignatureMiddleware
Kind of connector to use AWS Elasticsearch Service with elastic/elasticsearch-php client

## Installation

Add private repository

```json
{
  ...
  "repositories": [
    {
      "type": "vcs",
      "url": "git@github.com:atlastechnol/package-php-AwsSignatureMiddleware.git"
    }
  ]
}
```

Require this package with composer

```shell
composer require atlas/aws-signature-middleware -n
```

## Usage
Example with elasticsearch client

```php
<?php

namespace App\Providers\Aws;

use Aws\Credentials\CredentialProvider;
use Aws\Signature\SignatureV4;
use Atlas\AwsSignatureMiddleware\AwsSignatureMiddleware;
use App\Services\Aws\Contracts\SearchMetricsServiceContract;
use App\Services\Aws\SearchService;
use atlas\LaravelCircuitBreaker\Store\CircuitBreakerStoreInterface;
use Elasticsearch\ClientBuilder;
use Illuminate\Support\ServiceProvider;

class SearchMetricsServiceProvider extends ServiceProvider
{
    protected $defer = true;

    /**
     * Register the application services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->bind(
            SearchMetricsServiceContract::class,
            function ($app) {
                return new SearchService(
                    ClientBuilder::create()
                        ->setSSLVerification(false)
                        ->setConnectionPool(config('metrics.elastic.elastic_connection'))
                        ->setHosts(config('metrics.elastic.elastic_hosts'))
                        ->setHandler($this->getElasticHandler()),
                    $app[CircuitBreakerStoreInterface::class]
                );
            }
        );
    }

    /**
     * Get the services provided by the provider.
     *
     * @return array
     */
    public function provides(): array
    {
       return [
           SearchMetricsServiceContract::class
       ];
    }

    private function getElasticHandler()
    {
        $defaultHandler = ClientBuilder::defaultHandler();

        if (in_array(config('app.env'), ['local', 'testing'])) {
            return $defaultHandler;
        }

        $provider = CredentialProvider::defaultProvider();
        $credentials = $provider()->wait();
        $signature = new SignatureV4('es', config('metrics.elastic.aws_region'));

        $middleware = new AwsSignatureMiddleware($credentials, $signature);
        return $middleware($defaultHandler);
    }
}
```
