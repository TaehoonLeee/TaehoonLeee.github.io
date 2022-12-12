---
categories: [Kotlin]
tags: [memory visibility, code optimization, dclp]
---

코틀린을 사용하시는 분들은 흔히 사용하는 lazy property에 대해서 톺아보겠습니다.

먼저 lazy property는 아래와 같이 흔하게 사용됩니다.
```kotlin
val lazyValue: Any by lazy {
    --> Initialize block <--
    }
```
{: .nolineno }

하지만 이렇게 사용하면 lazy property에 있는 세 가지 모드 중
- LazyThreadSafetyMode.SYNCHRONIZED

이런 모드가 **default로 적용**됩니다.

먼저 말하면, 이 모드는 오로지 하나의 쓰레드에서만 접근이 가능하도록 설정되어 있으며 모든 쓰레드에서 접근해도 같은 값을 보게되는 모드입니다.

한 마디로 **여러 쓰레드의 동작 환경에서부터 안전하다**는 말이죠.

그런데 이러한 효과를 누릴 수 있도록 하는 **비용은 비싸**기 때문에 여러 쓰레드가 아닌 환경에서 이러한 쓰임새를 사용하게 되면 비효율적인 코드가 됩니다.

그래서 lazy 블럭 안의 초기화 코드가 하나의 쓰레드에서만 실행되는 것이 보장되어 있으면 다음과 같이 코드를 짜는 것이 바람직합니다.
```kotlin
val lazyValue: Any by lazy(LazyThreadSafetyMode.NONE) {
    ---> Initilize block <---
    }
```
{: .nolineno }

쓰임새를 대강 알아보았기 때문에, 본격적으로 lazy property에 대해 톺아보겠습니다.

앞서 말했듯이, lazy property에는 세 가지의 모드가 있습니다.
> - LazyThreadSafetyMode.SYNCHRONIZED
- LazyThreadSafetyMode.PUBLICATION
- LazyThreadSafetyMode.NONE


어떤 모드를 적용하냐에 따라 당연히 아래와 같이 초기화 부분이 나뉘게 됩니다.
```kotlin
public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }

```
{: .nolineno }

각각의 모드에 대한 동작도 뚜렷하게 다릅니다. 먼저, SYNCHRONIZED의 대해 알아보겠습니다.

## LazyThreadSafetyMode.SYNCHRONIZED
```kotlin
private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // final field is required to enable safe publication of constructed instance
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)
}

```
{: .nolineno }

SYNCHRONIZED 모드로 적용할 시 생성을 담당하는 위임 클래스입니다.

눈 여겨 봐야할 부분이 세 가지 있습니다.
> 
1. getter에 적용돼있는 DCLP(Double Checked Locking Pattern)
2. _value에 적용된 Volatile
3. typedValue 변수의 용도
{: .prompt-info }

먼저 _value에 적용한 Volatile이란 어노테이션의 뜻은 자바에서 부터 사용되던 용어로, **해당 변수를 읽어 들일 때 CPU 캐시가 아닌, 메인 메모리로부터 읽어들이겠다**는 뜻입니다.

이러한 개념의 필요성은 DCLP에서의 **memory visibility**를 해결하기 위해 있습니다.


그러면 DCLP는 어떤 것이길래 메인 메모리에서만 변수를 읽어들여야 할까요?

DCLP는 Singleton Pattern에서 **인스턴스가 하나만 생성되도록 보장해주는 패턴**입니다. 한 마디로 Singleton이 Singleton일 수 있도록 해주는 것이죠.
> DCLP의 자세한 설명은 인터넷에 좋은 자료가 많으니 검색해보시길 추천합니다.
{: .prompt-tip }

여기서 또 Singleton과 메인 메모리와 무슨 연관성이 있냐? 라고 생각하실 수 있습니다.

물론, DCLP가 적용된 코드 부분으로는 문제가 전혀 없습니다. 문제점이 있는 곳은 **CPU가 사용하는 메모리들**에 있습니다.

CPU들은 빠른 연산을 위해 **레지스터, 캐시, 램**과 같은 여러 종류의 메모리를 사용합니다.
> 메인 메모리에 접근하는 비용을 줄이기 위해 레지스터, 캐시를 사용합니다.
{: .prompt-info }

기본적인 DCLP 코드를 예시로 들겠습니다.
```kotlin
fun getInstance(): Singleton {
    if (instance == null) {
        synchronized(this) {
            if (instance == null) {
                instance = Singleton()
            }
        }
     
     return instance
 }
```
{: .nolineno }

만약 A 쓰레드에서 널을 체크해서 인스턴스의 정보를 레지스터에 생성하고, 아직 메인 메모리에 보내지지 않은 시점을 생각해보도록 합시다.

A 쓰레드에서 생성한 인스턴스가 메인 메모리에 보내지지 않았고 A 쓰레드가 synchronized 블록을 벗어났을 때 B 쓰레드에서 synchronized 블록에 접근합니다.

B 쓰레드는 해당 인스턴스의 정보를 메인 메모리에서 읽어 들일 것이고, 아직 A 쓰레드에서 생성한 인스턴스의 정보는 메인 메모리에 보내지지 않았기 때문에 B 쓰레드는 널을 읽을 것입니다.

B 쓰레드는 인스턴스를 하나 더 생성할 수 밖에 없게 되고, 결국은 A, B 쓰레드 둘 다 인스턴스를 생성하여 최종적으로 두 개의 인스턴스가 만들어졌습니다. 

**하나의 인스턴스만 생성하려는 목적을 벗어난 코드가 된 것**입니다.

>이렇게 한 쓰레드에서 변경한 어떤 메모리 값이 다른 쓰레드에서 제대로 읽어지지 않을 때 발생하는 문제점을 memory visibility라고 합니다.
{: .prompt-info }

이러한 memory visibility의 문제점을 해결하기 위해 메인 메모리에서만 값을 읽어들이기 위해 Volatile을 적용한 것입니다.

자 그러면 Volatile을 적용한 이유는 알겠고, getter에 typedValue의 용도는 또 무언고.. 라는 생각이 드실겁니다.


거두절미하고 본론만 말씀드리면, 이 변수의 용도는 memory reordering의 문제점을 해결하기 위한 변수입니다.

우리의 컴파일러는 여전히 또 최적화를 위해 memory ordering이라는 기법을 수행합니다.

이런 최적화가 싱글톤 인스턴스를 생성할 때 일어난다고 생각을 해봅시다.

인스턴스 생성을 위한 new operator는 다음과 같은 순서로 동작을 합니다.

1. 객체를 위한 메모리 할당
2. 객체의 생성자 실행
3. 객체의 reference 리턴

이러한 순서를 컴파일러는 모종의 이유로 다음과 같이 **재배치**할 수 있습니다.

1. 객체를 위한 메모리 할당
2. 객체의 reference 리턴
3. 객체의 생성자 실행

> 위와 같은 최적화의 이유는 객체의 메모리 주소를 바로 인스턴스에 할당해주게 되면 메모리 주소를 저장하는 변수를 하나 더 만들지 않아도 되기 때문입니다.
{: .prompt-info }

위와 같이 Instruction reordering이 수행되면 아직 초기화가 되지 않아 다른 쓰레드에서 인스턴스를 읽어들일 때 **null을 가져올 수 있어 인스턴스를 하나 더 생성**할 수 있게 됩니다.

똑같이 **하나의 인스턴스만 생성하려는 목적을 벗어난 코드가 된 것**입니다.

```kotlin
val typedValue = initializer!!()
_value = typedValue
initializer = null
typedValue
```
{: .nolineno }

이러한 문제점을 해결하기 위해 typedValue라는 변수를 따로 두어 _value가 **null**이거나 **완전히 initializer로 생성된 객체**를 참조한다는 것을 확인하도록 했습니다.

lazy property의 기본 모드인 LazyThreadSafetyMode.SYNCHRONIZED 에는 이러한 개념들이 녹아있습니다.

다음 포스팅에서 설명해드릴 LazyThreadSafetyMode.NONE 과 비교해서 조금 더 비용이 드는 연산을 수행합니다.

그래서 lazy property를 적용하는 변수가 오로지 하나의 쓰레드에서만 접근되는 것이 보장되면 LazySafetyMode.NONE 을 사용하면 위와 같은 연산들을 줄일 수 있습니다.

___
## References

1. https://m.blog.naver.com/jjoommnn/130036635345
2. http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html
3. https://gampol.tistory.com/entry/Double-checked-locking과-Singleton-패턴
4. https://bladecoder.medium.com/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4
5. https://assylias.wordpress.com/2013/02/01/java-memory-model-and-reordering/