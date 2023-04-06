---
title: 'ArrayList는 동적 배열인가'
categories:
    - Java
tags:
    - Java
    - Collection
toc: true
toc_sticky: true
---

연속된 데이터를 저장할 필요가 있을 때는 배열 대신 Collection의 ArrayList를 사용합니다.  
ArrayList와 배열의 가장 큰 차이는 크기가 동적으로 변한다는 것인데 이를 어떻게 구현했는지 알아보겠습니다.


## 배열

![](https://user-images.githubusercontent.com/40778768/230308443-2aa5a9f6-53b6-4a9d-a7b3-ac63496ad83d.png){: width="400" .align-center}

배열은 연속된 메모리를 할당받아 데이터를 저장하는 자료구조입니다.  
번호와 번호에 해당하는 데이터를 저장하는 방식으로 데이터를 관리하며 처음 초기화 시 크기를 지정해야 합니다.

```java
String[] strArr = new String[5];
int[] intArr = {1, 2, 3, 4, 5};

intArr[5] = 6; // ArrayIndexOutOfBoundsException 발생
```

자바에서는 위처럼 배열을 선언하고 초기화할 수 있는데 이때 배열의 크기는 불변입니다.   
배열의 데이터를 변경할 때는 인덱스를 통해 접근해야 하는데 고정 크기의 자료구조이기 때문에 배열의 크기를 넘어서는 인덱스에 접근하면 `ArrayIndexOutOfBoundsException` 예외가 발생합니다.

때문에 배열에 데이터를 더 추가하기 위해서는 새로운 배열을 생성하고 기존 배열의 값들을 복사하는 과정을 거쳐야 합니다.    
자바에서는 배열을 복사하는 여러가지 방법을 제공하지만 이를 자세히 살펴보진 않겠습니다.

## ArrayList

`ArrayList`는 자바에서 제공하는 `Collection` 인터페이스의 구현체 중 하나입니다.   
배열과 유사하게 연속된 데이터를 저장하는 자료구조이며 배열과의 가장 큰 차이는 크기가 동적으로 변한다는 것입니다.

### ArrayList의 필드

```java
private static final int DEFAULT_CAPACITY = 10

private static final Object[] EMPTY_ELEMENTDATA = {};

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

transient Object[] elementData; // non-private to simplify nested class access

private int size;
```

`ArrayList`의 필드입니다.   
- `DEFAULT_CAPACITY` : 기본 초기 용량을 나타내며 기본값으로 10이 저장되어 있습니다.
- `EMPTY_ELEMENTDATA` : 빈 배열을 나타내며 `elementData`가 비어있을 때 사용됩니다.
- `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` : 기본 초기 용량의 빈 배열을 나타내며 `elementData`가 비어있을 때 사용됩니다.
- `elementData` : 실제 데이터를 저장하는 배열입니다.
- `size` : `ArrayList`에 저장된 데이터의 개수입니다.

`ArrayList`의 용량은 `elementData`의 길이로 나타내고 있으며 여기서 중요한 점은 `ArrayList`도 결국 내부에서 고정 크기의 배열을 사용한다는 것을 알 수 있습니다.    
그럼 `ArrayList`는 데이터를 추가할 때 어떻게 동적으로 크기를 조절할까요?

### ArrayList의 크기 변경

우선 데이터를 추가하는 과정을 보겠습니다.

```java
public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}

private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}
```

`add()` 메서드는 `ArrayList`에 데이터를 추가하는 메서드이며 동일한 이름의 private 메서드가 존재합니다.   
여기서 `modCount` 변수는 `AbstractList`라는 상위 클래스에서 상속받은 변수로 `ArrayList`의 구조가 변경되었음을 나타냅니다.    

`add()` 메서드 호출 시 저장할 데이터와 `elementData` 배열, `size`를 인자로 전달하고 있습니다.    
여기서 `elementData` 배열의 길이와 `size`가 같다면 `grow()` 메서드를 호출합니다.
`elementData` 배열의 길이는 `ArrayList`의 용량을 가리키므로 이 값이 `size`와 같다면 배열에 더 이상 데이터를 저장할 수 없다는 뜻을 나타냅니다.

```java
private Object[] grow(int minCapacity) {
    return elementData = Arrays.copyOf(elementData, newCapacity(minCapacity));
}

private Object[] grow() {
    return grow(size + 1);
}

private int newCapacity(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity <= 0) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return minCapacity;
    }
    return (newCapacity - MAX_ARRAY_SIZE <= 0)
        ? newCapacity
        : hugeCapacity(minCapacity);
}
```

`grow()`는 내부에서 사용하는 private 메서드로 현재 `size`에 1을 더한 값을 오버로딩한 다른 `grow()` 메서드에 전달합니다.  
`grow(int minCapacity)` 메서드는 `minCapacity`를 인자로 전달받아 새로운 배열을 생성하기 위해 `Arrays.copyOf()` 메서드를 호출합니다. 
`Arrays.copyOf()` 메서드는 배열을 복사하는 유틸 메서드로 깊은 복사를 수행한 완전 새로운 배열을 생성하는데 용량을 늘리기 위해 두번째 인자로 배열의 크기를 전달받습니다.
이 때 `newCapacity(int minCapacity)`를 호출하는데 이 메서드는 기존의 용량에서 1.5배를 더한 값을 새로운 용량으로 반환하며 기존의 `elementData`에 새롭게 할당하게 됩니다.

```java
@HotSpotIntrinsicCandidate
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

@HotSpotIntrinsicCandidate
public static native void arraycopy(Object src,  int  srcPos,
        Object dest, int destPos,
        int length);
```

위 메서드는 `Arrays.copyOf()`의 코드인데 내부에서 native 코드로 작성된 `System.arraycopy()` 메서드를 호출하는 것을 확인할 수 있습니다. 

결국 `ArrayList`도 배열을 사용하고 있으며 개발자가 편하게 사용할 수 있도록 크기를 동적으로 조절하는 기능이 구현되어 있음을 알 수 있습니다.    
배열을 복사하는 것은 native 코드로 구현되어 있으며 자세한 확인은 어렵지만 초기 용량을 적절하게 설정하지 않으면 성능에 영향을 줄 수 있다고 생각됩니다.    
