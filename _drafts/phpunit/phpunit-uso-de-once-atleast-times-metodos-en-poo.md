---
layout: post
title:  "PHPUnit: Uso de atLeast, once y times en POO"
date:   2018-07-26 12:47:20 -0600
categories: phpunit, testing
---
## Introducción

Muchas veces en las pruebas únitarias y funcionales existen dudas y discuciones acerca de que probar y que no. En esta entrada muestro cuando usar los métodos `atLeast`, `once` y `times` para hacer pruebas siempre desde un punto de vista orientado a objetos.

## Probando 

En una prueba estos métodos deberían ser usados cuando necesitas garantizar que un método dentro del código en prueba es ejecutado, cero, una o más veces, y el código en prueba siempre regresa el mismo valor sin importar si este método se ejecuta o no. Por ejemplo

```php

public function charge(Payment $payment)
{
	// Other code

	if ($payment->isWithCreditCard()) {
		$this->transactionRepository->save($payment)
	}

	return true;
}
```

Para probar el código dentro del `if` necesitamos garantizar que el método `save` de `TransactionRepository` es llamado. Notemos que una vez que la ejecución llega al `if` no importa si entra o no el método `charge` regresara siempre `true` y el resulto del método `save` es ignorado totalmente por el método. Por estos motivos, es necesario garantizar la llamada al método `save`.

```php

public function testChargeWithCreditCard() 
{
	$transactionRepository = Mockery::mock('trans_repo', \App\Payment\Repositories\TransactionInterface::class)

	$transactionRepository
		->shouldReceive('save')
		->once(1);

	$paymentService = new \App\Payment\PaymentService(
		$transactionRepository
	);

	$creditCardPayment = new \App\Payment\DataTypes\Payment();
	$creditCardPayment->setCard(Payment::CREDIT_CARD);
	self::assertTrue($this->chage($creditCardPayment()));
}
```

En esta prueba, primero creamos una imitación de `TransactionRepository`, dándole, en el primer parámetro, un nombre y en el segundo una interfaz que debe cumplir. Hacer que nuestra imitación implemente una cierta interface hace que nuestra prueba sea más simple debido que podemos omitir chequeos de tipos. Esto lo analizaremos en otra entrada.

Luego, le decimos a la imitación como comportarse, es decir, definimos su comportamiento. Le decimos que debe *recibir* una llamada al método `save` exactamente 1 vez. Si el método no es llamado. Notese que no estamos usando el método `with` del imitador para indicar el parámetro que recibirá el método `save` gracias a que indicamos que el imitador usa una interfaz, ponerlo sería redundante. Tampoco estamos usando `andReturn` porque al método en prueba no le interesa el valor que regresa el método `save`, solo le interesa que el método `save` sea llamado con éxito. 

Por ultimo, creamos la clase a probar y ponemos una afirmación.

Veamos como quedaría la prueba para el caso de tarjeta  de débito

```php
public function testChargeWithDebitCard() 
{
	$transactionRepository = Mockery::mock('trans_repo', \App\Payment\Repositories\TransactionInterface::class)

	$transactionRepository
		->shouldNotReceive('save');

	$paymentService = new \App\Payment\PaymentService(
		$transactionRepository
	);

	$debitCardPayment = new \App\Payment\DataTypes\Payment();
	$debitCardPayment->setCard(Payment::DEBIT_CARD);
	self::assertTrue($this->chage($debitCardPayment()));
}
```

El código de esta prueba, es practicamente un *copy-and-paste* de la prueba anterior. La primera diferencia es en la definición del comportamiento del imitador. En vez de decirle que *espere* una llamada al método `save` le decimos que no espere ninguna llamada a este método, es decir, no queremos que el código dentro del `if` se ejecute.

En este ejemplo ya hemos visto cuando usar `once` y nos mustra de manera implicita donde usar los otros dos métodos: `least` y `times`. Sin embargo, daremos un ejemplo más explicito de cuando usarlos

```php

public function notify(Payment $payment)
{
	$attempts = 0;

	while ($email->isSuccess() && $email->maxAttempts() > $attempts {
		$email->send();
		$attempts++;
	}
}
```
Una de las pruebas del método `notify` es garantizar que ejecuta `$email->send()` exactamente `$email->maxAttempts()`. Así que la prueba quedaría de la siguiente forma

```php
public function testMaxAttemptReached() 
{
	$mailer = Mockery::mock('trans_repo', \App\Mail\Mailer::class)

	$mailer
		->shouldRecieve('send')
		->times(3);
	$mailer
		->shouldReceive('isSuccess')
		->andReturn(false);
	$mailer
		->shouldReceive('maxAttempts')
		->andReturn(3);

	$notifer = new \App\Notifier();
	$notifier->notify();

	self::assertTrue(true);
}
```

En nuestra prueba indicamos que el método `send` debe ser usado exactamente 3 veces y si ningún error ocurre entonces le decimos manualmente con una afirmación a PHPUnit que la prueba pasa exitosamente. La notificación manual a PHPUnit es necesaria por que el método no regresa nada pero sin embargo paso nuestra pruebas.


## Concluciones

El mayor beneficio de estos 3 operadores se obtiene cuando el método que se esta provando no regresa nada o regresa el mismo valor sin importar si se ejecuto o algún método de las dependencias que creamos.


## Keywords

> dependency, injection, poo, types, datatypes, php7, php