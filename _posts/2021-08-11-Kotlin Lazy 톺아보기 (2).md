---
categories: [Kotlin]
tags: [kotlin]
---

안녕하세요, 저번 시리즈에서는 lazy 프로퍼티의 **LazyThreadSafetyMode.SYNCHRONIZED**에 대해 알아보았습니다.
이번 시리즈에서는 나머지 모드에 대해 알아보겠습니다.

총 두 가지 모드가 남았습니다.

> - LazyThreadSafetyMode.PUBLICATION
- LazyThreadSafetyMode.NONE

간략히 말하면, PUBLICATION 모드로 설정하면, **여러 스레드에서 호출**될 수 있고, 다른 스레드에 의해 이미 초기화된 값이 있다면 그 값을 반환합니다. NONE 모드로 설정하게 되면, 단순히 초기화 여부만 판별하여 초기화를 진행합니다. NONE 모드는 **단일 스레드로서 환경이 보장**되어 있을 때만 사용해야합니다.

## LazyThreadSafetyMode.PUBLICATION

```kotlin
private class SafePublicationLazyImpl<out T>(initializer: () -> T) : Lazy<T>, Serializable {
    @Volatile private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // this final field is required to enable safe initialization of the constructed instance
    private val final: Any = UNINITIALIZED_VALUE

    override val value: T
        get() {
            val value = _value
            if (value !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return value as T
            }

            val initializerValue = initializer
            // if we see null in initializer here, it means that the value is already set by another thread
            if (initializerValue != null) {
                val newValue = initializerValue()
                if (valueUpdater.compareAndSet(this, UNINITIALIZED_VALUE, newValue)) {
                    initializer = null
                    return newValue
                }
            }
            @Suppress("UNCHECKED_CAST")
            return _value as T
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)

    companion object {
        private val valueUpdater = java.util.concurrent.atomic.AtomicReferenceFieldUpdater.newUpdater(
            SafePublicationLazyImpl::class.java,
            Any::class.java,
            "_value"
        )
    }
}
```
{: .nolineno }

PUBLICATION 모드의 코드입니다. SYNCHRONIZED 모드와 비슷한 모습을 띄고 있습니다.

SYNCHRONIZED 모드와의 차이점이라면, synchronized 키워드를 사용하지 않아 **여러 스레드에서의 동시 접근이 가능**합니다.

또, 다른 스레드에서의 초기화가 진행되었을 때 그 값을 반환한다는 것이 특징입니다.

## LazyThreadSafetyMode.NONE
```kotlin
internal class UnsafeLazyImpl<out T>(initializer: () -> T) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    private var _value: Any? = UNINITIALIZED_VALUE

    override val value: T
        get() {
            if (_value === UNINITIALIZED_VALUE) {
                _value = initializer!!()
                initializer = null
            }
            @Suppress("UNCHECKED_CAST")
            return _value as T
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)
}
```
{: .nolineno }

마지막으로, NONE 모드입니다. 코드를 보면 DCLP 알고리즘을 적용하지 않고 단순히 초기화를 진행해주고 있습니다.

그래서 단일 스레드 환경이 아닐 시, NPE가 발생할 우려가 있기 때문에 **단일 스레드의 환경이 보장되는 상황**에서만 사용해야합니다.