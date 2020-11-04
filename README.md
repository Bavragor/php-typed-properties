# Typed properties, designing entities and the uninitialized state!

PHP 7.4 introduced a great new feature, typed properties.

    public string $string;
This allows us, for example when designing entities or value objects, to be even stricter.
Which will become even better with PHP 8 when union types are introduced `public string|false $string;`

## Uninitialized state

While typed properties where introduced, also a hidden feature got into PHP, the uninitialized state.
Assuming we would have an entity like this:

    class Car {
	    public string $model;
    }
And our logic would have looked like this:

    $car = new Car();
    $this->assertNull($car->model);
The accessing of the property model of class Car would fail with the error:
> Fatal error: Uncaught Error: Typed property Car::$model must not be accessed before initialization

Instead the check needs to be the following:

    $car = new Car();
    $this->assertTrue(isset($car->model));
This forces us to be very clear in what the property can hold, and if it actually should be nullable.

## Designing entities with typed properties

With the previous information in mind and the will to actually use typed properties the Car class could look like this:

    class Car {
	    public string $requiredProperty;
	    public ?string $optionalProperty = null;

		public function __construct(string $requiredProperty) {
			$this->requiredProperty = $requiredProperty;
		}
    }
This means that we have to enforce setting required properties which should not be nullable or unitialized from the start.
And optional properties should be nullable and initialized from the start, which is even more important when using getters and setters for it.
Assuming we would have a class Car like this:

     class Car {
	    private string $requiredProperty;
	    private ?string $optionalProperty;

		public function __construct(string $requiredProperty) {
			$this->requiredProperty = $requiredProperty;
		}

		public function getOptionalProperty(): ?string
		{
			return $this->optionalProperty;
		}
		
		public function setOptionalProperty(?string $optionalProperty): void
		{
			$this->optionalProperty = optionalProperty;
		}
    }
And our logic would have looked like this:

    $car = new Car();
    $this->assertNull($car->getOptionalProperty());
Using the function getOptionalProperty would fail with the error:
> Fatal error: Uncaught Error: Typed property Car::$optionalProperty must not be accessed before initialization

In this we would not be able to use isset and solving the error without changing the function getOptionalProperty to use isset, which would be an unessecary function call when the value should is optional and should be nullable.
Or we would have to rely on the user of the class Car calling setOptionalProperty before using getOptionalProperty.

But more importantly, the unitialized state forces us to ensure that not nullable properties are either required in the constructor or a static factory method and having a private/protected constructor.
Which results in making setters obsolete for required properties that cannot change after initialization.

This leads to the conclusion that a possible ideal entity with typed properties could look like this:

    class Car {
	    private string $model;
	    private string $bodyMaterial; // immutable
	    private string $color;
	    private ?string $rearSpoilerType = null;

		public function __construct (
			string $model,
			string $bodyMaterial,
			string $color,
		) {
			$this->model = $model;
			$this->bodyMaterial = $bodyMaterial;
			$this->color = $color;
		}

		public function setModel(string $model): void
		{
			$this->model = $model;		
		}

		public function setColor(string $color): void
		{
			$this->color = $color;		
		}

		public function setRearSpoilerType(string $rearSpoilerType): void
		{
			$this->rearSpoilerType = $rearSpoilerType;		
		}

		public function removeRearSpoilerType(): void
		{
			$this->rearSpoilerType = null;
		}
		
		public function getModel(): string
		{
			return $this->model;
		}
		
		public function getBodyMaterial(): string
		{
			return $this->bodyMaterial;
		}
		
		public function getColor(): string
		{
			return $this->color;
		}
		
		public function getRearSpoilerType(): string
		{
			return $this->rearSpoilerType;
		}
    }

Which could be used like this:

	// Creating a car from VW which has a body made of steel, painted black.
    $car = new Car('VW', 'steel', 'black');
    
    if ($car->getRearSpoilerType() === null) {
	    $this->logger->info('Our car doesn not have any spoiler');
    }
    
    $car->setRearSpoilerType('the big one'); // modified to have the biggest, spoiler.

	$car->setColor('pink'); // repaint our compensation to pink, more masculinity

	$car->removeRearSpoilerType();
	
	if ($car->getRearSpoilerType() === null) {
	    $this->logger->info('Too much masculinity, who needs a spoiler anway');
    }
This is why I think typed properties are great, and the unitialized state is an awesome addition, even though at first I hated it.
Because when using typed properties on existing entities for example, every check for null will break, and often you will have to restructure the usage of your getters.
But in the end it will be worth it, resulting in a clearer and stricter interface.

## Problems
### PHPUnit & Symfony Dependency Injection
PHPUnit has a setUp method which is called before running a test, when using dependency injection in this method for our test class from symfony container, the property in the test class has to be nullable, even though when actual test functions are executed the property will be the instance and not null.
