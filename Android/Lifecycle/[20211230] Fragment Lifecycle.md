## Fragment Lifecycle

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FcDyVCU%2Fbtq9CtTEtoA%2FkpOuUqYRAw8aVmbyKT7jpk%2Fimg.png" width="60%"/>

### 전체 생명주기 콜백(헷갈리니 Activity도)

```
// Fragment
onAttach()
onCreate()
onCreateView()
onViewCreated()
onViewStateRestored()
onStart()
onResume()
onPause()
onStop()
onSaveInstanceState()
onDestroyView()
onDestroy()
onDetach()

// Activity
onCreate()
onStart()
(onRestoreInstanceState())
onResume()
onPause()
(onSavedInstanceState())
onStop()
onDestroy()
```

특이점: 객체의 생명주기와 View의 생명주기가 다르다

-> Fragment의 Lifecycle이 변하면(Callback 함수가 호출되면) 해당 콜백 종료 시점에서 View의 Lifecycle에 이벤트 전달

### onCreate(bundle: Bundle)

- Fragment만 CREATED 된 상황 -> FragmentManager에 add가 되었을 때 호출됨

- bundle: onSavedInstanceState에서 저장된 번들 값, 이 값은 Fragment가 처음 생성되었을 때에만 null, onSavedInstanceState 정의안했어도 그 이후부터는 non-null

### onCreateView(), onViewCreated()

onCreateView에서 정상적인 Fragment View를 inflate하고 return 해야 onViewCreated에서 패러미터로 이 View를 전달받음 

viewLifeCycleOwner도 onAttach~onCreateView에서 생성됨(performCreateView)

이때부터 View의 Lifecycle이 INITIALIZED 상태로 변함

- View 초기값/LiveData observing, Adapter 세팅은 여기서

### onViewStateRestored()

viewLifecycleOwner: INITIALIZED -> CREATED

SparseArray<Parcelable>타입의 mSavedViewState에 View 상태를 저장한다. (onStop 이후 onDestroyView 이전에 저장)

### onStart()

- Fragment가 사용자에게 보여질 수 있을 때
- 이때부터 childFragmentManager를 통해 FragmentTransaction을 안전하게 수행 가능
- Fragment/View의 Lifecycle 모두 STARTED로 변환

### onResume()

- 모든 Animator/Transition 상태가 종료 + 사용자와 Interaction 가능할 때
- 입력을 시도하거나 포커스를 설정하는 것과 같은 작업을 이 시점 전에 하는 것은 안된다

### onPause()

- Fragment/View Lifecycle이 **STARTED**가 됨
- 사용자에겐 여전히 visible하지만 Fragment에서 떠나기 시작할 때

### onStop()

- 더 이상 화면에 보여지지 않을 때
- Fragment/View Lifecycle이 **CREATED**가 됨
- API 28 기점으로 생명주기 호출순서가 살짝 달라짐

```
// Before API 28
ON_STOP -> onSavedInstanceState() -> onStop()

// API 28+
ON_STOP -> onStop() -> onSavedInstanceState()
=> FragmentTransaction을 안전하게 수행할 수 있는 마지막 지점
```

### onDestroyView()

- Exit Animation/Transition이 완료되고 Fragment가 화면으로부터 벗어날 경우 ViewLifecycle은 DESTORYED가 되고 onDestroy 호출
- getViewLifecycleOwnerLiveData() 리턴 시 null
- GC에 의해 수거될 수 있도록 Fragment View에 대한 모든 참조가 제거되어야 한다

### onDestroy()

- Fragment가 제거 or FragmentManager destroy -> Fragment Lifecycle이 DESTORYED가 됨
- 이후 onDetach() 호출

### onAttach/onDetach

- Fragment-Activity 연결관계를 등록/해제한다

### FragmentManager에서 Fragment를 Add하는 경우

-> 뒤에 있는 백그라운드 Fragment는 View/객체 모두 Destroy하지 않는다( 뒤에 보여질 수 있다)

### Replace하는 경우

-> 뒤에 있는 백그라운드 Fragment는 View/객체 모두 Destroy한다.

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FnUK4s%2Fbtq9yNS8YVJ%2F06EdlfhuuhvgPNg8PyXCdK%2Fimg.png" width="80%"/>

onStop을 먼저하고 프래그먼트는 onStart까지 실행한 다음에 뒤의 뷰/객체 떼고 onResume() -> 뒤의 View/객체를 전부 떼어아 사용자와 상호작용이 가능하다

왜 액티비티와 다른가? -> 화면 교체 방식이 좀 다르기 때문에?

액티비티는 슬라이드하는 형식으로 화면이 바뀌는데 프래그먼트는 컨테이너 자체에서 뷰가 바뀌어서 onStop까지 단번에 호출되는건가

<img src="https://miro.medium.com/max/838/1*NKRcmHa4XDeFyELAZ8U7GQ.png" />

Activity와 비교 필수

### addToBackStack까지 하면서 Replace를 한다면?

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbhQ1n0%2Fbtq9Eh6i97O%2FBcJURGOaokOcKB5Ch9tnnk%2Fimg.png"/>

onDestroy와 onDetach는 호출 안함

### HOME 버튼으로 밖으로 빠져나갈 때

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fcc6if4%2Fbtq9yL8YdbR%2FQ0yx8KW9jlQg51lBV6ipzk%2Fimg.png"/>

onStop -> onSavedInstanceState까지 => onDestroyView는 안찍힘

다만 메모리 부족 시 그냥 destory가 찍힐 수 있음