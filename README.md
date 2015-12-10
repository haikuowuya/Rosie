![Karumi logo][karumilogo]Rosie
======

> The only way to make the deadline—the only way to go fast—is to keep the code as clean as possible at all times.

> &mdash; <cite> Robert C. Martin in [Clean Code: A Handbook of Agile Software Craftsmanship](http://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)</cite>

Introduction
------------

Rosie is an Android framework to create applications following the Clean Architecture [[1] [clean]].

Rosie divides your application in three layers, **view**, **domain** and **repository**. For each layer, Rosie provides plenty of classes that will make defining and separating these concerns much easier.

* **View**: It contains all your presentation logic, implemented through the Model-View-Presenter pattern [[2] [mvp]]. Rosie provides classes to represent the main components of this layer like ``RosieActivity``, ``RosieFragment`` and ``RosiePresenter``.
* **Domain**: Holding all your business logic, its main component is ``RosieUseCase`` that gives you an easy way to define your application use cases and execute them in a background thread using the command pattern [[3] [com]].
* **Repository**: The repository layer gives you an abstraction of how to retrieve and store data in your application following the pattern with the same name [[4] [rep]]. ``RosieRepository`` and the multiple ``DataSource`` classes gives you the base to start building your own repositories.

The framework comes with Dagger to solve Dependency inversion through Dependency Injection [[5] [di]]

Screenshots
-----------

#~TODO Real example gif~

![Demo screenshot][sample]

Usage
-----

First thing you will need to do is to make your own ``Application`` instance extend ``RosieApplication`` in order to provide your global dependencies module to Dagger:

```java
public class SampleApplication extends RosieApplication {
	@Override protected List<Object> getApplicationModules() {
		return Arrays.asList((Object) new SampleGlobalModule());
	}
}
```


Rosie provides several base classes to start implementing your architecture separated in three layers, **view**, **domain** and **repository**. Let's explore them in detail.

###View
The view package contains all the classes needed to implement your presentation logic following the MVP pattern. To use the view package, make your ``Activity`` extend from ``RosieActivity`` or your ``Fragment`` from ``RosieFragment`` and specify the layout that Rosie will automatically inflate for you:

```java
public class SampleActivity extends RosieActivity {
	@Override protected int getLayoutId() {return R.layout.sample_activity;}
	/*...*/
}
```

####Butter Knife

You will have access in your activities and fragments to [ButterKnife] [butterknife] annotations to inject your views:

```java
public class SampleActivity extends RosieActivity {
	@Bind(R.id.sample_view) TextView sampleView;
	/*...*/
}
```

####Dagger

Besides, you can define the [Dagger] [dagger] module that will contain the dependencies for your activity by overriding the ``getActivityScopeModules`` method:

```java
public class SampleActivity extends RosieActivity {
	@Override protected List<Object> getActivityScopeModules() {
		return Arrays.asList((Object) new SampleModule());
	}
	/*...*/
}
```

There is also two useful annotations to provide ``Context`` dependencies into your classes, one for your ``Application`` context and one for your current ``Activity``:

```java
public class InjectedClass {
	@Inject public InjectedClass(@ForApplication Context context, @ForActivity Context context) {
		/*...*/
	}
}
```

####Presenter
To follow the MVP pattern, Rosie also provides a ``RosiePresenter`` class that will be responsible for all your presentation logic. Rosie will take care of linking your view (a ``RosieActivity`` or ``RosieFragment`` implementation) with your presenter and subscribing it to its lifecycle. In order to do that, create a ``RosiePresenter`` and link it to your view:

```java
public class SamplePresenter extends RosiePresenter<SamplePresenter.View> {
	public interface View extends RosiePresenter.View {
		/*...*/
	}
}
```

```java
public class SampleActivity extends RosieActivity implements SamplePresenter.View {
	@Inject @Presenter SamplePresenter presenter;
	@Override protected void onPreparePresenter() {/*...*/}
}
```

Once both, view and presenter, are linked you can react to your view lifecycle directly from the presenter. The ``onPreparePresenter`` method gives you the opportunity to configure your presenter before any of these methods are called (e.g. with your intent parameters). You will be also able to call your view easily from the presenter:

```java
public class SamplePresenter extends RosiePresenter<SamplePresenter.View> {
	protected void initialize() {/*...*/}
	protected void update() {/*...*/}
	protected void pause() {/*...*/}
	protected void destroy() {/*...*/}
	
	private void sampleMethod() {
		View view = getView(); // Get the view linked to the presenter
		view.foo();
	}
	
	public interface View extends RosiePresenter.View {
		void foo();
	}
}
```

To understand when the lifecycle methods are called take a look at the following table:

| RosiePresenter  | Activity       | Fragment           |
| --------------- |----------------| -------------------|
| ``initialize``  | ``onCreate``   | ``onViewCreated``  |
| ``update``      | ``onResume``   | ``onResume``       |
| ``pause``       | ``onPause``    | ``onPause``        |
| ``destroy``     | ``onDestroy``  | ``onDestroy``      |

####Renderers

Finally, Rosie includes the [Renderers] [renderers] library to simplify your ``RecyclerView`` handling code. If you decide to use Renderers, remember to extend directly from ``RosieRenderer<T>`` to have ButterKnife injections for free in your renderer views.

###Domain

The domain package is meant to contain all your business logic that will change from app to app. For that reason, Rosie only provides a single ``RosieUseCase`` class that will help you execute your use cases in background following the command pattern.

To start using the Rosie domain package, create your use cases extending ``RosieUseCase`` and define the method containing your use case logic. Remember to call ``notifySuccess`` or ``notifyError`` to get results back to your presenter:

```java
public class DoSomething extends RosieUseCase {
	@UseCase public void doSomething(Object arg) {
		Object response = getData();
		if (response.isOk()) {
			notifySuccess(response.getContent());
		} else {
			notifyError(response.getError());
		}
	}
}
```

#~TODO Example of use cases usage with callback~

####Named use cases

Sometimes you need to specify use cases that are very similar to each other. To avoid creating multiple classes representing every use case configuration, you can create a single ``RosieUseCase`` class with multiple methods. Rosie will be able to identify the use case being called by matching its input parameters. If you create two methods with the ``@UseCase`` annotation that have the same input parameters, you can provide a name to each of them in order to identify them:

```java
public class DoSomething extends RosieUseCase {
	public static final String USE_CASE_NAME = "UseCaseName";
	public static final String OTHER_USE_CASE_NAME = "OtherUseCaseName";
	@UseCase(name = USE_CASE_NAME) public void doSomething(Object arg) {/*...*/}
	@UseCase(name = OTHER_USE_CASE_NAME) public void doSomethingElse(Object arg) {/*...*/}
}
```

Even though using names is not mandatory when the method input parameters are different, it's highly recommended to use them just to make its usage more readable.

To call one of those use cases you will need to provide the name in your call:

#~TODO Example of use cases usage with names~

###Error handling

Errors can be either manually reported or implicitly notified when an exception is thrown from your use case context. To handle errors, there is a capturing event system that iterates over all your registered ``OnErrorCallback`` implementations and notifies them of the issue. Every callback needs to return a boolean value to inform whether the error has been handled and needs no further management. Callbacks are always called in specific order:

1. Use case specific callback; the ones you register before calling a use case.
2. Global callbacks; the ones you can register from your presenter constructor.

With this approach you can create an error callback in your presenter that shows a generic error message to your user and create multiple other listeners for specific use cases and/or errors.

To register global callbacks from your presenter, call ``registerOnErrorCallback`` in the constructor:

```java
public class SamplePresenter extends RosiePresenter<SamplePresenter.View> implements OnErrorCallback {
	public SamplePresenter(UseCaseHandler useCaseHandler) {
		super(useCaseHandler);
		registerOnErrorCallback(this);
	}

	@Override public boolean onError(Error error) {
		getView().showGenericError();
		return true;
	}
}
```

#~TODO Example of registering an error listener in a use case~

###Repository

The third layer is meant to encapsulate your data sources. To start using it just extend ``RosieRepository`` and configure its data sources in its constructor:

```java
public class SampleRepository extends RosieRepository<Key, Value> {
	@Inject public SampleRepository(SampleApiDataSource sampleApiDataSource,
			SampleCacheDataSource sampleCacheDataSource) {
		addReadableDataSources(sampleApiDataSource);
		addCacheDataSources(sampleCacheDataSource);
	}
}
```

There are three different types of data sources and each one has its own interface that you can implement to use it in your repository:

* ``ReadableDataSource<K, V>``: Defines data sources where you can read values by a given key or retrieving them all.
* ``WriteableDataSource<K, V>``: Defines data sources where you can persist data with operations to add, update or delete values.
* ``CacheDataSource<K, V>``: Defines a mix of readable and writeable data sources to speed up access to your values.

There are empty implementations of each data source to simplify your own subclasses by overriding only the methods that make sense in your context. Besides, there is a generic ``InMemoryCacheDataSource`` to store recent values up to a configurable time.

```java
public class SampleRepository extends RosieRepository<Key, Value> {
	@Inject public SampleRepository() {
	PaginatedCacheDataSource<Key, Value> inMemoryCacheDataSource =
		new InMemoryCacheDataSource<>(new TimeProvider(), MINUTES.toMillis(5));
	addCacheDataSources(inMemoryCacheDataSource);
  }
}
```

Users of the repositories can retrieve or modify data by using one of the multiple methods available for that matter. There are multiple ways to retrieve and store data using repositories, the following snippet shows some of the most useful:

```java
// Initialization
Key key = /*...*/; Value value; Collection<Value> values;
RosieRepository<Key, Value> repository = /*...*/;

// Get a value by its key
value = repository.getByKey(key);
// Get a value by its key using only the defined cache
value = repository.getByKey(key, ReadPolicy.CACHE_ONLY);
// Get all the available values
values = repository.getAll();
// Get all the available values using only readable sources
values = repository.getAll(ReadPolicy.READABLE_ONLY);
// Add a new value
repository.addOrUpdate(value);
// Add a new value only to the first writeable data source that works
repository.addOrUpdate(value, WritePolicy.WRITE_ONCE);
// Delete a value by its key
repository.deleteByKey(key);
// Delete all values stored in the repository
repository.deleteAll();
```

####Paginated repositories

Finally, Rosie gives you support for pagination in repositories. If your data is paginated, just extend ``PaginatedRosieRepository`` instead of ``RosieRepository``. You will also need to implement your paginated data sources, ``PaginatedReadableDataSource<V>`` and ``PaginatedCacheDataSource<K, V>``. Once your paginated repository is completely defined and configured you will be able to use it just as a regular repository with additional pagination-related methods:

```java
// Initialization
Key key = /*...*/; PaginatedCollection<Value> page;
PaginatedRosieRepository<Key, Value> repository = /*...*/;

// Get a value by its key just as with a regular repoitory
Value value = repository.getByKey(key);
// Get a page
page = repository.getPage(offset, limit);
// Get a page using only the cache data source
page = repository.getPage(offset, limit, ReadPolicy.CACHE_ONLY);
```


Add it to your project
----------------------

Include the library in your ``build.gradle``

```groovy
dependencies{
    compile 'com.karumi:rosie:1.0.0'
}
```

or to your ``pom.xml`` if you are using Maven

```xml
<dependency>
    <groupId>com.karumi</groupId>
    <artifactId>rosie</artifactId>
    <version>1.0.0</version>
    <type>aar</type>
</dependency>

```

Do you want to contribute?
--------------------------

Feel free to report us or add any useful feature to the library, we will be glad to improve it with your help.

Keep in mind that your PRs **must** be validated by Travis-CI. Please, run a local build with ``./gradlew checkstyle build`` before submiting your code.


Libraries used in this project
------------------------------

* [JUnit] [junit]
* [Mockito] [mockito]
* [Robolectric] [robolectric]
* [ButterKnife] [butterknife]
* [Dagger] [dagger]
* [Renderers] [renderers]
* [Android Priority Job Queue] [jobqueue]

License
-------

    Copyright 2015 Karumi

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

[clean]: https://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html
[di]: http://martinfowler.com/articles/injection.html
[com]: http://www.oodesign.com/command-pattern.html
[rep]: https://msdn.microsoft.com/en-us/library/ff649690.aspx
[mvp]: http://martinfowler.com/eaaDev/uiArchs.html#Model-view-presentermvp
[sample]: ./art/sample.png
[karumilogo]: ./art/karumilogo.png
[junit]: https://github.com/junit-team/junit
[mockito]: https://github.com/mockito/mockito
[robolectric]: https://github.com/robolectric/robolectric
[dagger]: https://github.com/square/dagger
[butterknife]: https://github.com/JakeWharton/butterknife
[renderers]: https://github.com/pedrovgs/renderers
[jobqueue]: https://github.com/yigit/android-priority-jobqueue