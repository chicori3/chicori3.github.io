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

자바의 컬렉션 중 가장 쉽게 사용되는 것은 List 인터페이스를 구현한 ArrayList가 아닐까합니다.  
여러 블로그의 포스팅을 보면 배열과 함께 언급이 되는데 배열과 ArrayList의 차이와 ArrayList의 내부 구조가 어떻게 구현되어 있는지 알아보려고 합니다.


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
배열의 데이터를 변경할 때는 인덱스를 통해 접근해야 하는데 고정 크기의 자료구조이기 때문에 배열의 크기를 넘어서는 인덱스에 접근하면 ArrayIndexOutOfBoundsException` 예외가 발생합니다.

때문에 배열에 데이터를 더 추가하기 위해서는 새로운 배열을 생성하고 기존 배열의 값들을 복사하는 과정을 거쳐야 합니다.    
자바에서는 배열을 복사하는 여러가지 방법을 제공하지만 이를 자세히 살펴보진 않겠습니다.

## ArrayList

ArrayList는 자바에서 제공하는 `Collection` 인터페이스의 구현체 중 하나입니다.     
배열과 유사하게 연속된 데이터를 저장하는 자료구조이며 배열과의 가장 큰 차이는 크기가 동적으로 변한다는 것입니다.     
ArrayList는 어떻게 크기를 동적으로 변경할 수 있을까요?

### ArrayList의 생성자

```java
public ArrayList() {...};
public ArrayList(int initialCapacity) {...};
public ArrayList(Collection<? extends E> c) {...};
```

ArrayList는 3개의 생성자를 제공하며 우리가 확인할 것은 기본 생성자와 정수를 받는 생성자입니다.  
ArrayList는 내부에 `elementData`라는 이름의 배열을 사용하고 있으며 생성될 때 기본값 혹은 인자로 받은 값으로 배열의 크기를 초기화합니다.      

여기서 중요한 점은 ArrayList도 결국 내부에서 고정 크기의 배열을 사용한다는 것을 알 수 있습니다.         
그럼 ArrayList는 데이터를 추가할 때 어떻게 동적으로 크기를 조절할까요?

### ArrayList가 데이터를 저장할 때

우선 데이터를 추가하는 과정을 보겠습니다.

```java
// 단순한 데이터 저장
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

// 데이터를 지정한 인덱스에 저장
public void add(int index, E element) {
        rangeCheckForAdd(index);
        modCount++;
        final int s;
        Object[] elementData;
        if ((s = size) == (elementData = this.elementData).length)
            elementData = grow();
        System.arraycopy(elementData, index, elementData, index + 1, s - index);
        elementData[index] = element;
        size = s + 1;
}
```

ArrayList는 데이터를 저장하는 add 메서드를 제공합니다.

저장할 데이터를 add 메서드의 인자로 전달하면 modCount를 증가시킨 후 저장할 데이터와 실제 데이터를 저장하는 배열, size를 오버로딩한 private add 메서드에 인자로 전달하고 있습니다.

> 여기서 modCount 변수는 AbstractList라는 상위 클래스에서 상속받은 변수로 ArrayList의 구조가 변경되었음을 나타냅니다.            
> modCount 변수는 fail-fast 방식을 구현하기 위해 사용되는데 Iterator를 사용한 순회(for-each) 중에 자료구조의 변형이 발생하면 ConcurrentModificationException을 발생시킵니다.

데이터를 추가하는 과정은 마치 배열에 값을 추가하듯이 현재 용량을 포인터로 사용하여 인덱스에 직접 저장하는 것을 확인할 수 있었습니다.      
위와 같은 방법을 사용하기 때문에 값을 추가할 때 아까 살펴본 ArrayIndexOfBoundsException 예외를 방지할 수 있게 되었습니다.

다만 인덱스를 통해 데이터를 저장할 때는 rangeCheckForAdd 메서드를 통해 인덱스의 유효성을 검사합니다.    
이 때는 인덱스를 통해 직접 저장을 수행하기 때문에 IndexOutOfBoundsException 예외가 발생할 수 있습니다.

> ArrayList는 IndexOutOfBoundsException 예외를 사용합니다.        

근데 add 메서드 중간에 `s == elementData.length` 조건식으로 분기를 나누는 코드가 존재합니다.       
s는 size, elementData.length는 현재 배열의 크기를 의미하며 size와 배열의 크기가 같다면 배열에 더 이상 저장할 공간이 없기 때문에 배열의 크기를 늘려야 합니다.   
grow 메서드가 그 책임을 수행하는데 이 메서드는 어떻게 동작할까요?

### ArrayList의 grow 메서드

```java
private Object[] grow() {
    return grow(size + 1);
}

private Object[] grow(int minCapacity) {
    return elementData = Arrays.copyOf(elementData, newCapacity(minCapacity));
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

grow는 내부에서 사용하는 private 메서드로 현재 size에 1을 더한 값을 오버로딩 된 grow 메서드에 전달합니다.               
minCapacity는 배열이 최소한으로 늘어날 수 있는 크기이며, newCapacity 메서드를 통해 새로운 배열의 크기를 계산합니다.    
이 과정에서 minCapacity가 0보다 작다면 OutOfMemoryError를 발생시키고, newCapacity가 MAX_ARRAY_SIZE보다 크다면 hugeCapacity 메서드를 호출합니다.     

이렇게 반환된 배열의 크기는 Arrays.copyOf 메서드로 전달되는데요.      
Arrays.copyOf 메서드는 배열을 복사하는 메서드로 깊은 복사를 수행하여 기존의 배열에 담긴 데이터를 새로운 배열에 옮겨 담습니다.                     

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

좀 더 들어가보면 Arrays.copyOf 메서드는 native 코드로 작성된 System.arraycopy 메서드를 호출하는 것을 확인할 수 있습니다.       
배열을 복사하는 것은 native 코드로 구현되어 있으며 자세한 확인은 어렵지만 배열을 복사하는 과정이 쉽지만은 않은 것 같습니다.   

결국 ArrayList도 고정 배열을 사용하고 있으며 개발자가 편하게 사용할 수 있도록 크기를 동적으로 조절하는 기능이 구현되어 있음을 알 수 있습니다.        
적절한 초기 용량을 설정해서 불필요한 오버헤드를 방지할 수 있지 않을까 생각됩니다.            
