## Service Locator Pattern + DI

> 서비스 로케이터 패턴은 로케이터에 의해 객체의 초기화 방법을 등록하고, 해당 객체를 필요로 하는 곳에서 로케이터를 통해 객체를 제공받을 수 있도록 하는 패턴

- Static하게 관리를 해야한다면 해당 코드를 직접 작성해줘야 한다
- 동적으로 관리를 해야하므로 컴파일 시점에 어떤 문제가 있는 지 알 수 없어서 **로케이터에 등록되지 않은 타입의 객체를 요구할 경우 런타임에 에러가 발생**

## 제어의 역전(Inversion Of Control)

```kotlin
class MovieListener {
    fun moviesDirectedBy(director: String): List<Movie> {
        val allMovies = finder.findAll()
        return allMovies.filter { it.director == director }
    }
}
```

finder를 통해 현재 저장되어 있는 전체 영화를 조회해야하는 기능을  MovieListener라는 클래스 내에서 구현해야할 때, finder는 findAll라는 행동을 할 수 있다는 것이고 이는 인터페이스로 행위를 명세할 수 있다.

```kotlin
fun interface MovieFinder {
    fun findAll(): List<Movie>
}
```

그리고 이를 MovieListener 객체에서 초기화해줄 것이다.

```kotlin
class MovieListener(
    private val finder: MovieFinder = XMLMovieFinder("movies1.xml")
) {
    /* Logic */
}
```

영화 목록은 txt, xml,, db 등 다양한 형태로 저장될 수 있으므로 이 형식들을 지원하기 위해서는 다양한 Finder 클래스가 필요하다. 이런 형식들을 지원해주기 위해서는 Finder 객체를 설정할 때 사용자가 어떤 Finder를 사용할 지 모르는 상황임을 가정했을 때 컴파일 타임에 이를 알 수 없게 해둬야 한다. 이를 위해 Inversion Of Control 기법을 활용한다.

### 어떤 것이 역전된건데?

MovieListener를 MovieFinder를 구현한 클래스의 인스턴스를 제공만 해준다면 MovieListener를 만들 수 있다. 이를 위해서는 구현된 클래스를 제공해줄 수 있는 외부 모듈이 있어야 하고 이를 IoC라 칭하기에는 일반적인터라 이러한 모듈의 패턴(이름)을 Dependency Injection(DI)/Service Locator Pattern라 부르는 것이다.

### Dependency Injection Pattern

다음과 같은 방법으로 주입할 수 있다.

- Injection Interface

```kotlin
interface FinderInjector {
    fun inject(finder: MovieFinder)
}

class MovieListener: FinderInjector {
    lateinit var finder: Finder
    override fun inject(finder: MovieFinder) {
        this.finder = finder
    }
}
```

- Constructor Injection
- Setter Injection

Inject 과정

- 구현된 클래스와 연결해주기 위한 Configuration 클래스를 정의한다

```kotlin
class Tester {
    private lateinit var container: Container

    private fun configureContainer() {
        container = Container()
        registerComponents()
        registerInjectors()
        container.start()
    }
}
```

- 컴포넌트를 등록해서 어떤 컴포넌트를 제공하는 지 Look Up Table 역할을 하도록 한다.

```kotlin
class Tester {
    private lateinit var container: Container

    private fun configureContainer() {
        container = Container()
        registerComponents()
        registerInjectors()
        container.start()
    }

    private fun registerComponents() {
        with(container) {
            registerComponent("MovieListener", MovieListener::class.java)
            registerComponent("MovieFinder", CSVMovieFinder::class.java)
        }
    }
}
```

- 주입 가능한 클래스는 Injector를 구현한다

```kotlin
class Tester {
    private lateinit var container: Container

    private fun configureContainer() {
        container = Container()
        registerComponents()
        registerInjectors()
        container.start()
    }

    private fun registerComponents() {
        with(container) {
            registerComponent("MovieListener", MovieListener::class.java)
            registerComponent("MovieFinder", CSVMovieFinder::class.java)
        }
    }

    private fun registerInjectors() {
        with(container) {
            registerInjector(FinderInjector::class.java, container.lookup("MovieFinder"))
	            registerInjector(InjectFinderFilename::class.java, FinderFilenameInjector())
        }
    }
}

interface Injector {
    fun inject(target: Any)
}
```

- Injector 구현 클래스는 inject 메서드를 통해 실제 데이터를 inject 해준다.

```kotlin
class ColonMovieFinder: Injector {
    override fun inject(target: Any) {
        (target as FinderInjector).inject(this)
    }
}

class Tester {
    object FinderFilenameInjector : Injector {
        override fun inject(target: Any) {
            (target as InjectFinderFileName).injectFilename("movies1.txt")
        }
    }
}
```

- container에 등록했던 injector 인터페이스를 사용하여 종속성을 파악하고 Injector를 사용하여 주입한다

### Service Locator Pattern

- ServiceLocator에서 객체를 가져온다

```kotlin
class ServiceLocator {
    private var movieFinder: MovieFinder? = null
    companion object {
        private var soleInstance: ServiceLocator? = null
        fun movieFinder() = soleInstance?.movieFinder
    }
}
```

- Configuration

```kotlin
class Tester {
    private fun configure() {
        ServiceLocator.load(ServiceLocator(ColonMovieFinder("movies1.txt"))
    }
}

class ServiceLocator(
    private var movieFinder: MovieFinder? = null
) {
    companion object {
        private var soleInstance: ServiceLocator? = null
        fun movieFinder() = soleInstance?.movieFinder
        fun load(arg: ServiceLocator) { soleInstance = arg }
    }
}
```

- UseCase

```kotlin
class Tester {
    fun testSimple() {
        configure()
        val finder = MovieFinder()
        movies = finder.moviesDirectedBy("Sergio Leone")
    }
}
```

이를 각각의 인터페이스로 분리하여 로케이터 간의 역할 분리도 해낼 수 있다

```kotlin
interface MovieFinderLocator {
    fun movieFinder(): MovieFinder
}
```

- 동적 서비스 로케이터: 각 서비스에 대한 프로퍼티 대신 맵을 사용하고 주입할 객체까지 미리 맵에 담아놓는 방법으로 구성한다

```kotlin
class ServiceLocator(
    private val services = hashMapOf<String, Any>()
) {
    companion object {
        private var soleInstance: ServiceLocator? = null
        fun load(arg: ServiceLocator){ soleInstance = arg }
        fun getService(key: String) = soleInstance?.services?.get(key)
        fun loadService(key: String, service: Any) = services.put(key, service)
    }
}
```

## Service Locator vs DI

- 서비스 로케이터는
  - 서비스를 사용하는 모든 부분에서 로케이터에 대한 의존성을 가진다.
  - 로케이터 호출에 대한 소스코드를 알아야 종속성을 파악할 수 있다
- DI는
  - 생성자와 같은 Injection 방법을 찾고 종속성을 알 수 있다

### 왜 서비스 로케이터는 안티패틴일까?

- 인터페이스 분리원칙(ISP) 위반
  - 사용하지 않는 메서드에 대한 의존이 강제되면 안된다
  - 서비스 로케이터 패턴에서 동적으로 의존성을 등록하는 경우 의존성에 관련된 여러 메서드를 노출할 수 있다
- 캡슐화 위반
  - 세부적인 구현에 대한 이해를 덜고 인터페이스만으로 상호작용할 수 있는 것을 추구한다
  - 동적 서비스 로케이터의 구현에서 볼 수 있듯이 MovieListener에서 MovieFinder가 등록되어 있다는 것을 가정하고 로케이터를 사용한다, 즉 MovieFinder가 등록되어있다는 것을 컴파일 타임에서 보장할 수 없기에 캡슐화 되었다고 보기 어렵다.

### TL;DR

- **IoC: Class that is using the dependency shouldn’t create it**
- **Dependency Injection:** Pattern that shows how the dependencies should be provided to fulfill the IoC principle(제어 역전 원칙을 지키면서 의존 객체가 제공되는 패턴) → Mock 테스트가 쉬워짐
- Dependency Injection 패턴에서는 객체가 어디서 와야하는 지를 몰라야 하는데 코인과 같은 로케이터 패턴에서는 로케이터 객체를 통해 제공받는다.
- Activity, Fragment는? by inject(lazy { get() })나 field injection(setter injection을 활용할 수 있는 의존성 객체를 만든다)