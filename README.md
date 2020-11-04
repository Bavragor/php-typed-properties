---


---

<h1 id="typed-properties-designing-entities-and-the-uninitialized-state">Typed properties, designing entities and the uninitialized state!</h1>
<p>PHP 7.4 introduced a great new feature, typed properties.</p>
<pre><code>public string $string;
</code></pre>
<p>This allows us, for example when designing entities or value objects, to be even stricter.<br>
Which will become even better with PHP 8 when union types are introduced <code>public string|false $string;</code></p>
<h2 id="uninitialized-state">Uninitialized state</h2>
<p>While typed properties where introduced, also a hidden feature got into PHP, the uninitialized state.<br>
Assuming we would have an entity like this:</p>
<pre><code>class Car {
    public string $model;
}
</code></pre>
<p>And our logic would have looked like this:</p>
<pre><code>$car = new Car();
$this-&gt;assertNull($car-&gt;model);
</code></pre>
<p>The accessing of the property model of class Car would fail with the error:</p>
<blockquote>
<p>Fatal error: Uncaught Error: Typed property Car::$model must not be accessed before initialization</p>
</blockquote>
<p>Instead the check needs to be the following:</p>
<pre><code>$car = new Car();
$this-&gt;assertTrue(isset($car-&gt;model));
</code></pre>
<p>This forces us to be very clear in what the property can hold, and if it actually should be nullable.</p>
<h2 id="designing-entities-with-typed-properties">Designing entities with typed properties</h2>
<p>With the previous information in mind and the will to actually use typed properties the Car class could look like this:</p>
<pre><code>class Car {
    public string $requiredProperty;
    public ?string $optionalProperty = null;

	public function __construct(string $requiredProperty) {
		$this-&gt;requiredProperty = $requiredProperty;
	}
}
</code></pre>
<p>This means that we have to enforce setting required properties which should not be nullable or unitialized from the start.<br>
And optional properties should be nullable and initialized from the start, which is even more important when using getters and setters for it.<br>
Assuming we would have a class Car like this:</p>
<pre><code> class Car {
    private string $requiredProperty;
    private ?string $optionalProperty;

	public function __construct(string $requiredProperty) {
		$this-&gt;requiredProperty = $requiredProperty;
	}

	public function getOptionalProperty(): ?string
	{
		return $this-&gt;optionalProperty;
	}
	
	public function setOptionalProperty(?string $optionalProperty): void
	{
		$this-&gt;optionalProperty = optionalProperty;
	}
}
</code></pre>
<p>And our logic would have looked like this:</p>
<pre><code>$car = new Car();
$this-&gt;assertNull($car-&gt;getOptionalProperty());
</code></pre>
<p>Using the function getOptionalProperty would fail with the error:</p>
<blockquote>
<p>Fatal error: Uncaught Error: Typed property Car::$optionalProperty must not be accessed before initialization</p>
</blockquote>
<p>In this we would not be able to use isset and solving the error without changing the function getOptionalProperty to use isset, which would be an unessecary function call when the value should is optional and should be nullable.<br>
Or we would have to rely on the user of the class Car calling setOptionalProperty before using getOptionalProperty.</p>
<p>But more importantly, the unitialized state forces us to ensure that not nullable properties are either required in the constructor or a static factory method and having a private/protected constructor.<br>
Which results in making setters obsolete for required properties that cannot change after initialization.</p>
<p>This leads to the conclusion that a possible ideal entity with typed properties could look like this:</p>
<pre><code>class Car {
    private string $model;
    private string $bodyMaterial; // immutable
    private string $color;
    private ?string $rearSpoilerType = null;

	public function __construct (
		string $model,
		string $bodyMaterial,
		string $color,
	) {
		$this-&gt;model = $model;
		$this-&gt;bodyMaterial = $bodyMaterial;
		$this-&gt;color = $color;
	}

	public function setModel(string $model): void
	{
		$this-&gt;model = $model;		
	}

	public function setColor(string $color): void
	{
		$this-&gt;color = $color;		
	}

	public function setRearSpoilerType(string $rearSpoilerType): void
	{
		$this-&gt;rearSpoilerType = $rearSpoilerType;		
	}

	public function removeRearSpoilerType(): void
	{
		$this-&gt;rearSpoilerType = null;
	}
	
	public function getModel(): string
	{
		return $this-&gt;model;
	}
	
	public function getBodyMaterial(): string
	{
		return $this-&gt;bodyMaterial;
	}
	
	public function getColor(): string
	{
		return $this-&gt;color;
	}
	
	public function getRearSpoilerType(): string
	{
		return $this-&gt;rearSpoilerType;
	}
}
</code></pre>
<p>Which could be used like this:</p>
<pre><code>// Creating a car from VW which has a body made of steel, painted black.
$car = new Car('VW', 'steel', 'black');

if ($car-&gt;getRearSpoilerType() === null) {
    $this-&gt;logger-&gt;info('Our car doesn not have any spoiler');
}

$car-&gt;setRearSpoilerType('the big one'); // modified to have the biggest, spoiler.

$car-&gt;setColor('pink'); // repaint our compensation to pink, more masculinity

$car-&gt;removeRearSpoilerType();

if ($car-&gt;getRearSpoilerType() === null) {
    $this-&gt;logger-&gt;info('Too much masculinity, who needs a spoiler anway');
}
</code></pre>
<p>This is why I think typed properties are great, and the unitialized state is an awesome addition, even though at first I hated it.<br>
Because when using typed properties on existing entities for example, every check for null will break, and often you will have to restructure the usage of your getters.<br>
But in the end it will be worth it, resulting in a clearer and stricter interface.</p>
<h2 id="problems">Problems</h2>
<h3 id="phpunit">PHPUnit</h3>
<p>PHPUnit has a setUp method which is called before running a test, when using dependency injection in this method for our test class from symfony container, the property in the test class has to be nullable, even though when actual test functions are executed the property will be the instance and not null.</p>

