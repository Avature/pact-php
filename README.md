# Pact PHP

[![AppVeyor](https://img.shields.io/appveyor/ci/mattermack/pact-php.svg?style=flat-square&logo=appveyor)](https://ci.appveyor.com/project/mattermack/pact-php/branch/master)
[![Travis](https://img.shields.io/travis/pact-foundation/pact-php.svg?style=flat-square&logo=travis)](https://travis-ci.org/pact-foundation/pact-php)
[![Packagist](https://img.shields.io/packagist/v/mattersight/phppact.svg?style=flat-square&label=stable)](https://packagist.org/packages/mattersight/phppact)
[![Packagist Pre Release](https://img.shields.io/packagist/vpre/mattersight/phppact.svg?style=flat-square&label=unstable)](https://packagist.org/packages/mattersight/phppact)

[![Downloads](https://img.shields.io/packagist/dt/mattersight/phppact.svg?style=flat-square)](https://packagist.org/packages/mattersight/phppact)
[![Downloads This Month](https://img.shields.io/packagist/dm/mattersight/phppact.svg?style=flat-square)](https://packagist.org/packages/mattersight/phppact)
[![Downloads Today](https://img.shields.io/packagist/dd/mattersight/phppact.svg?style=flat-square)](https://packagist.org/packages/mattersight/phppact)

PHP version of [Pact](https://pact.io). Enables consumer driven contract testing. Please read the [Pact.io](https://pact.io) for specific information about PACT.

## Installation

Install the latest version with:

```bash
$ composer require mattersight/phppact --dev
```

## Basic Consumer Usage

All of the following code will be used exclusively for the Consumer.

### 1. Configure PHPUnit to start the Mock Service

This library contains a wrapper for the [Ruby Standalone Mock Service](https://github.com/pact-foundation/pact-mock_service).

The easiest way to configure this is to use a [PHPUnit Listener](https://phpunit.de/manual/current/en/appendixes.configuration.html#appendixes.configuration.test-listeners). A default listener is included in this project, see [PactTestListener.php](/src/PhpPact/Consumer/Listener/PactTestListener.php). This utilizes environmental variables for configurations. These env variables can either be added to the system or to the phpunit.xml configuration file. Here is an example [phpunit.xml](/example/phpunit.consumer.xml) file configured to use the default. Keep in mind that both the test suite and the arguments array must be the same value.

##### How To Start and Stop the Service

```php
<?php
    use PhpPact\Standalone\MockService\MockServer;
    use PhpPact\Standalone\MockService\MockServerConfig;

    // Create your basic configuration. The host and port will need to match
    // whatever your Http Service will be using to access the providers data.
    $config = new MockServerConfig('localhost', 7200, 'SomeConsumer', 'SomeProvider');
    
    // Instantiate the mock server object with the config. This can be any
    // instance of MockServerConfigInterface.
    $server = new MockServer($this->mockServerConfig);
    
    // Create the process.
    $server->start();
    
    // Stop the process
    $server->stop();
```

### 2. Create Your Unit Tests

[Click here](/example/tests/Consumer/Service/ConsumerServiceHelloTest.php) to see the full sample file.

##### 1. Create a phpunit test class

```php
<?php

namespace Consumer\Service;

use PHPUnit\Framework\TestCase;

class ConsumerServiceTest extends TestCase { }
```

##### 2. Create your test function

```php
public function testGetHelloString() { }
```

##### 3. Set up mock request

This will define what the expected request coming from your http service will look like.

```php
$request = new ConsumerRequest();
$request
    ->setMethod('GET')
    ->setPath('/hello/Bob')
    ->addHeader('Content-Type', 'application/json');
```

You can also create a body just like you will see in the provider example.

##### 4. Set up mock response

This will define what the response from the provider should look like.

```php
$response = new ProviderResponse();
$response
    ->setStatus(200)
    ->addHeader('Content-Type', 'application/json;charset=utf-8')
    ->setBody([
        'message' => new RegexMatcher('Hello, Bob', '(Hello, )[A-Za-z]')
    ]);
```

In this example, we are using matchers. This allows us to add flexible rules when matching the expectation with the actual value. In the example, you will see Regex is used to validate that the response is valid.

Matcher | Explanation | Parameters | Example
---|---|---|---
PhpPact\Consumer\Matcher\RegexMatcher | Match a value against a regex pattern. | Value, Regex Pattern | new RegexMatcher('Hello, Bob', '(Hello, )[A-Za-z]')
PhpPact\Consumer\Matcher\TypeMatcher | Match a value against its data type. | Value, Min (Optional), Max (Optional) | new TypeMatcher(12, 0, 100)

##### 5. Build the interaction

Now that we have the request and response, we need to build the interaction and ship it over to the mock server.

```php
// Create a configuration that reflects the server that was started. You can 
// create a custom MockServerConfigInterface if needed. This configuration
// is the same that is used via the PactTestListener and uses environment variables.
$config      = new MockServerEnvConfig();
$mockService = new InteractionBuilder($config);
$mockService
    ->given('Get Hello')
    ->uponReceiving('A get request to /hello/{name}')
    ->with($request)
    ->willRespondWith($response); // This has to be last. This is what makes an API request to the Mock Server to set the interaction.
```

##### 6. Make the request

```php
$service = new HttpService($config->getBaseUri()); // Pass in the URL to the Mock Server.
$result  = $service->getHelloString('Bob'); // Make the real API request against the Mock Server.
```

##### 7. Make assertions

```php
$this->assertEquals('Hello, Bob', $result); // Make your assertions.
```

## Basic Provider Usage

All of the following code will be used exclusively for Providers. This will run the Pacts against the real Provider and either verify or fail validation on the Pact Broker.

There are two ways to handle verifications.

1. Create a test for each Consumer of this Provider.
2. Create a test and run all tests for this Provider.

Option 1 is the most likely use case but we can start with option 2.

### Example of Running One Consumer at a Time

##### 1. Create a PHPUnit test per Consumer

##### 2. Spin up a copy of your API

Get an instance of the API up and running. [Click here](#starting-your-api) for some tips.

##### 3. Create your test

```php
$config = new VerifierConfig();
$config
    ->setProviderName('SomeProvider') // Providers name to fetch.
    ->setProviderVersion('1.0.0') // Providers version.
    ->setProviderBaseUrl(new Uri('http://localhost:58000')) // URL of the Provider.
    ->setBrokerUri(new Uri('http://localhost')) // URL of the Pact Broker to publish results.
    ->setPublishResults(true); // Flag the verifier service to publish the results to the Pact Broker.

// Verify that the Consumer 'SomeConsumer' that is tagged with 'master' is valid.
$verifier = new Verifier($config, new BrokerHttpService(new GuzzleClient(), $config->getBrokerUri()));
$verifier->verify('SomeConsumer', 'master');

// This will not be reached if the PACT verifier throws an error, otherwise it was successful.
$this->assertTrue(true, 'Pact Verification has failed.');
```

### Example of Running All Consumers at Once

##### 1. Create a single PHPUnit TestCase

##### 2. Spin up a copy of your API

Get an instance of the API up and running. [Click here](#starting-your-api-asyncronously) for some tips.

##### 3. Create your test
```php
public function testPactVerifyAll()
{
    $config = new VerifierConfig();
    $config
        ->setProviderName('SomeProvider') // Providers name to fetch.
        ->setProviderVersion('1.0.0') // Providers version.
        ->setProviderBaseUrl(new Uri('http://localhost:58000')) // URL of the Provider.
        ->setBrokerUri(new Uri('http://localhost')) // URL of the Pact Broker to publish results.
        ->setPublishResults(true); // Flag the verifier service to publish the results to the Pact Broker.

    // Verify that all consumers of 'SomeProvider' are valid.
    $verifier = new Verifier($config, new BrokerHttpService(new GuzzleClient(), $config->getBrokerUri()), new InstallManager());
    $verifier->verifyAll();

    // This will not be reached if the PACT verifier throws an error, otherwise it was successful.
    $this->assertTrue(true, 'Pact Verification has failed.');
}
```

## Tips

### Starting your API Asyncronously

You can use the built in PHP server to accomplish this during your tests setUp function. The Symfony Process library can be used to run the process asynchronous.

[PHP Server](http://php.net/manual/en/features.commandline.webserver.php)

[Symfony Process](https://symfony.com/doc/current/components/process.html)