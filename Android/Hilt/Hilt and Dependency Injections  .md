# Hilt and Dependency Injections

이번 게시글에서 다룰 것은 의존성 주입(이하 DI(Dependency Injection의 약자))의 개념과 Android Jetpack에서 제공하는 DI Framework인 Hilt에 대해서 다뤄보고자 한다.

본 게시글은 [Hilt and Dependency Injections](https://youtu.be/1Zt6aIqZnqU)와 [Dependency injection with Hilt](https://developer.android.com/training/dependency-injection/hilt-android#application-class)를 번역한 것이다.

## Why Dependency Injection is Important For Good App Architecture?

이 부분은 영상의 내용만으로는 IoC와 DI의 개념을 이해하기가 힘들 수 있을 것 같아서  필자가 이해한 내용을 추가적으로 작성하고자 한다.

아래의 텍스트 자동 변환 클래스를 보도록 하자

```kotlin
class TextTransformer {
  private var parser: XMLParser = XMLParser()
  
  fun execute() {
    parser.execute()
    /* etc */
  }
}
```

위 클래스를 그대로 구현했을 때에는 어떠한 문제점도 없을 수 있다. 그러나 이 클래스만을 활용할 시에는 XML형식이 아닌 다른 형식(txt, json 등)의 데이터를 파싱할 수 없을 것이다. 

또한 이 동작을 검증하기 위한 테스트 코드를 작성하고자 할 때, 파싱 엔진이라는 외부 요인에 의존하여 동작이 되기에 [Test Double](https://en.wikipedia.org/wiki/Test_double) 테크닉을 활용하여 검증을 하는 것이 좋은데, 이런 구조에서는 테스트 코드에서 Mock 객체를 이 객체에게 제공을 하기가 굉장히 힘들다.

이러한 단점을 해소하기 위해서 parser의 의존성을 사용자가 직접 설정해주는 것이 아닌 외부 클래스에서 생성된 parser를 생성인자로 넣어서 TextTransformer 객체를 만들어서 TextTransformer 클래스의 코드는 그대로 둔 채 parser만 바꿀 수 있어 재사용성이 올라간다. 또한 테스트 시 MockParser 객체를 주입할 수 있어서 원하는 테스트 조건을 만들 수 있다.

```kotlin
class TextTransformer(
  private val parser: Parser
){ 
  fun execute() {
    parser.execute()
    /* etc */
  }
}
```

## Manual DI is not appropriate approach

하지만 이런 DI를 개발자 손수 설정하는 것은 보일러 플레이트만 증가시킬 수 있다.

```kotlin
class FeedViewModel(
    private val loadCurrentMomentUseCase: LoadCurrentMomentUseCase,
    loadAnnouncementsUseCase: LoadAnnouncementsUseCase,
    private val loadStarredAndReservedSessionsUseCase: LoadStarredAndReservedSessionsUseCase,
    getTimeZoneUseCase: GetTimeZoneUseCase,
    getConferenceStateUseCase: GetConferenceStateUseCase,
    private val timeProvider: TimeProvider,
    private val analyticsHelper: AnalyticsHelper,
    private val signInViewModelDelegate: SignInViewModelDelegate,
    themedActivityDelegate: ThemedActivityDelegate,
    private val snackbarMessageManager: SnackbarMessageManager
) : ViewModel()
```

위의 코드는 [iosched](https://github.com/google/iosched/blob/main/mobile/src/main/java/com/google/samples/apps/iosched/ui/feed/FeedViewModel.kt)의 코드를 일부 발췌한 것이다. 단적으로, 이 클래스의 dependency를 본인이 적어본다고 가정을 해보자. 아 물론 지금 보고 있는 코드뿐만 아니라 각각의 클래스의 의존성 모두 다 파악하고 적어야 한다. 이 모든 의존성을 하나도 틀리지 않고 완벽히 다 적을 수 있다고 장담할 수 있는가? 아마 그렇지 못할 것이다.

하지만, DI 라이브러리를 사용한다면 라이브러리에서 모든 의존성을 파악하고 자체에서 코드를 만들기 때문에 DI 패턴의 장점을 그대로 활용한 채 위의 문제를 해결할 수 있다.

## Hilt is Recommended Android DI Framework

구글에서 개발한 DI 프레임워크인 **Hilt**는 이미 구글에서 만든 Dagger를 기반으로 만들었고 어노테이션 프로세싱을 활용하여 컴파일 타임에 의존성 관련 코드 생성을 하기에 런타임에서 뛰어난 성능을 보여준다.

또한 Hilt는 안드로이드 Jetpack의 전폭적인 지지를 받아서 나온 라이브러리여서 다른 Jetpack 라이브러리와의 호환도 잘 이뤄지고 있다. 이제 Hilt를 어떻게 앱에 적용하는 지 보도록 하자.

## Quick Start Guide

### @HiltAndroidApp

Hilt를 사용하고자 하는 모든 앱은 @HiltAndroidApp 어노테이션을 포함한 Application 클래스를 가지고 있어야 한다. 이 어노테이션은 application 레벨의 Dependency Container 역할을 하는 Base Application를 만들고 Hilt에게 코드 생성을 하도록 트리거를 만든다.

```kotlin
@HiltAndroidApp
class MusicApp: Application()
```

생성된 Hilt 컴포넌트는 이 Application 객체의 생명주기와 연결되고 dependency 객체들을 제공한다.  또한 이는 Application의 가장 상위 컴포넌트이기에 다른 컴포넌트들이 제공하는 dependency에도 접근할 수 있다.

### @AndroidEntryPoint

이렇게 만든 dependency들을 Activity에 주입을 받기 위해서, 주입이 필요한 Activity에 @AndroidEntryPoint 어노테이션을 기입해야한다.

```kotlin
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() { ... }
```

참고로 Hilt를 통해 주입을 받으려면 Activity는 최소한 ComponentActivity 혹은 AppCompatActivity를 상속받고 있어야한다.

### @Inject

```kotlin
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() {

  @Inject lateinit var analytics: AnalyticsAdapter
  ...
}
```

Activity 내에서 주입받고 싶은 객체에 대해 @Inject 어노테이션을 붙여야 한다. 또한 Hilt로 주입받는 모든 변수들은 ``super.onCreate()``가 호출된 후에 사용할 수 있다.

@Inject는 클래스의 ``constructor`` 옆에 붙을 경우 Hilt에게 이 클래스가 어떻게 구성되는 지(어떻게 만들 수 있는 지)를 알려준다.

```kotlin
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```

빌드 시간에 Hilt는 안드로이드에서 제공된 클래스와 관련된 Dagger Component를 만들고 Dagger는 다음과 같은 순서로 코드를 생성한다

- 의존성 그래프를 만들면서 의존이 순환되고 있는 지 혹은 제공된 코드 중에서 미작성된(완성되지 않은) 의존성이 있는 지 검증한다
- 런타임에서 실제 객체와 의존성을 만들 수 있는 클래스를 생성한다

## Complicated Example: What If Injecting Other Types?

다음과 같은 상황을 가정해보자.

```kotlin
class MusicPlayer @Inject constructor(
  // Room Database
  private val db: MusicDatabase
) {
  fun play(id: String) { .. }
}
```

MusicPlayer는 MusicDatabase라는 Room 클래스(안드로이드에서 제공하는 내부 데이터베이스)를 의존하고 있다. @Inject로 MusicPlayer의 구현형태를 알려줄 수는 있을 것이다. 그러나 MusicDatabase는 어떻게 만들어졌는지 어떻게 알려줄 수 있을까?

### @Module, @Provides

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DataModule {

  @Provides
  fun provideMusicDB(@ApplicationContext context: Context): MusicDatabase {
    return Room.databaseBuilder(
      context, MusicDatabase::class.java, "music.db"
    )
  }
}
```

Room은 추상 클래스로 만들어지기에 의존성을 제공할 때 다른 방법으로 제공할 수 있어야한다. Hilt에는 위와 같이 Hilt Module에 특정 타입의 의존성을 제공하는 방법을 함수로 정의하면 특정 타입 객체가 필요할 시 이와 같은 방법으로 제공할 수 있도록 한다.

이 Module은 @Module 어노테이션이 클래스에 기재되어있어야 하며 제공 방법(Hilt에서는 binding이라고 한다)을 정의한 함수에는 @Provides 어노테이션이 기재되어있어야한다.

이 함수는 패러미터에 어떤 의존성이 있는 지, 리턴 값에는 어떤 타입을 제공할 것인지 적혀있어야 한다. 위의 예시에서 @ApplicationContext는 이미 Hilt에서 제공할 수 있는 Application Context 객체를 제공받기 위해 적은 것이다. 

그렇다면 저 @InstallIn 어노테이션과 SingletonComponent는 무엇일까?

## Component in Hilt

<img src="https://developer.android.com/images/training/dependency-injection/hilt-hierarchy.svg" width="80%"/>

컴포넌트는 지정된 타입에 맞는 객체를 제공해주는 역할을 가진 클래스이다. 컴파일 타임에서 Hilt는 Application의 의존성 그래프를 역전시키고 모든 타입의 transitive 의존성을 제공할 수 있는 코드를 생성한다.

각 컴포넌트의 binding은 위에 그려진 컴포넌트 계층을 따라서 전파된다. 예를 들어서 MusicDatabase 같은 경우 SingletonComponent과 대응되는 Application 클래스 에서 사용할 수 있는 경우 다른 컴포넌트에서도 사용할 수 있다.

컴포넌트 클래스들은 컴파일 타임에 Hilt를 통해 생성된다.

### @InstallIn

@InstallIn 어노테이션은 이 binding들을 사용할 수 있는 클래스들과 컴포넌트 내의 다른 binding들을 컨트롤하는 데 이용된다.

### @Singleton

Hilt는 기본적으로 주입 요청이 들어올 때마다 객체를 binding에 맞춰서 새롭게 제공한다. 만약에 특정 타입의 객체는 Application 전역에서 동일하게 사용하고 싶다면 어떻게 해야할까? 

이 역시 어노테이션으로 해결가능하다.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DataModule {

  @Singleton
  @Provides
  fun provideMusicDB(@ApplicationContext context: Context): MusicDatabase {
    return Room.databaseBuilder(
      context, MusicDatabase::class.java, "music.db"
    )
  }
}
```

binding에 @Singleton 어노테이션을 붙이면 Hilt에게 해당 컴포넌트에서 항상 동일한 객체를 제공하라고 알려줄 수 있다.

여기서 @Singleton은 Component와 연관된 Scope 어노테이션이다. 또한, 각각의 Component에도 이와 같은 Scope 어노테이션이 있는데 이는 위 사진에서 @ScopeAnnotation 해당하는 위치에 있는 것들이다.

만약에 Activity의 생명주기에 맞춰서 특정 객체를 사용하고 싶을 때, @ActivityScoped를 클래스나 binding에 부착하면 해당 생명주기에 맞춰서 한 번만 생성된다.

## Hil's Jetpack Integration

위에서 봤던 FeedViewModel을 DI를 통해 제공하고자할 때 어떻게 만들어야할까? Hilt는 Jetpack에서 제공해주는 클래스를 편하게 사용할 수 있는 기능을 제공해 주기에 특정 어노테이션만으로 쉽게 DI를 설정할 수 있다.

```kotlin
@HiltViewModel
class FeedViewModel @Inject constructor(
    private val loadCurrentMomentUseCase: LoadCurrentMomentUseCase,
    loadAnnouncementsUseCase: LoadAnnouncementsUseCase,
    private val loadStarredAndReservedSessionsUseCase: LoadStarredAndReservedSessionsUseCase,
    getTimeZoneUseCase: GetTimeZoneUseCase,
    getConferenceStateUseCase: GetConferenceStateUseCase,
    private val timeProvider: TimeProvider,
    private val analyticsHelper: AnalyticsHelper,
    private val signInViewModelDelegate: SignInViewModelDelegate,
    themedActivityDelegate: ThemedActivityDelegate,
    private val snackbarMessageManager: SnackbarMessageManager
) : ViewModel() { .. }

@AndroidEntryPoint
class FeedFragment : Fragment() {
  private val viewModel by viewModels<FeedViewModel>()
}
```

위와 같이 @HiltViewModel 어노테이션만 ViewModel 클래스 위에 붙이면 ViewModelFactory를 굳이 정의할 필요 없이 생성자 패러미터가 있는 ViewModel을 제공할 수 있다.

또한 Fragment에서도 @AndroidEntryPoint만 달면 lifecycle-ktx에서 제공하는 ViewModel 위임 생성함수를 활용하여 ViewModel을 제공받을 수 있다.

```kotlin
@Composable
fun FeedScreen(viewModel = viewModel()) { .. }
```

Jetpack Compose에서도 위와 거의 동일한 방식으로 함수만을 호출하여 ViewModel을 제공받을 수 있다.



