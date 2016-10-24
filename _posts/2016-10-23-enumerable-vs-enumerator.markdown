---
title: enumerable과 enumerator
layout: post
date: 2016-10-23 17:09
image: /assets/images/markdown.jpg
headerImage: false
tag:
- ruby
- enumerator
- lazy
- each
blog: true
author: minjepark
description: enumerable과 enumerator의 개념과 차이점에 관하여 설명
---

> 왜 루비를 사용하세요?

모 스타트업 면접에서 받은 질문이었다. `Enumerator`의 사용이 굉장히 편리하다는, 그다지 매력적이지 않는 답변을 했던 것 같다. 당시에는 루비를 배운 지가 얼마 안되서 사실 루비를 잘 알지도 못했거니와, 레일즈와 루비의 구분도 모호했었다. 그런 상황에서 루비를 '왜' 사용하는지 질문을 받으면 크게 답할 것이 없었다. 다만, 배운 지 얼마 안된 뉴비에게도 루비의 `Enumerator`는 굉장히 신기한 기능이었고, 다른 언어와 비교해서 가장 돋보이는 기능이었다.

심지어는 '이런 기능을 굳이 만들어야 했나'라는 정도의 기능까지 있을만큼 굉장히 다양한 기능을 Ruby의 `Enumerator`는 제공하고 있었다. 정확하게 말하자면, 내가 루비에서 마음을 들어했던 다양한 이터레이션 기능들은 `Enumerator`에서 온 것이 아니라 `Enumerable`이라 할 것이다. 이 둘은 무엇이 다른 것일까? `Enumerator`<U>는 이터레이션을 가능하게 하는 클래스</U>이고, `Enumerable`<U>은 이터레이션을 위한 검색, 정렬 등의 메소드를 모아 놓은 모듈</U>이다.

> Enumerator : 이터레이션을 가능하게 하는 클래스<br>
    Enumerable : 이터레이션을 위한 검색, 정렬 등의 메소드를 모아 놓은 모듈

`Enumerable` 모듈의 기능을 사용하기 위해 `Enumerator` 객체가 필요한 것이다. `Array`나 `Range`도 이터레이션을 위해 `Enumerable` 모듈을 참조하고 있다. Enumerable 모듈의 메소드를 사용하면 Enumerator 객체를 만드는 것을 알 수 있다. `Enumerable`의 메소드를 블록없이 사용하면 `Enumerator` 객체를 생성하는 것을 알 수 있다. 아래의 코드에서 Array와 Range에 블록 없이 `each`를 실행해서 `Enumerator`를 생성하는 지 확인하고, `Array`, `Range`가 `Enumerable` 모듈을 참조하고 있는 지를 확인해보자.

    # Array
    [1, 2, 3, 4].each
    => #<Enumerator: [1, 2, 3, 4]:each>

    # Range
    (1..10).each
    => #<Enumerator: 1..10:each>

    # 각 클래스가 enumerable 모듈을 참조하고 있는 지 확인해보자

    Array.included_modules
    # Output: => [Enumerable, Kernel]

    Hash.included_modules
    # Output: => [Enumerable, Kernel]

    Range.included_modules
    # Output: => [Enumerable, Kernel]

이와 같이 <U>데이터 콜렉션</U>과 사용할 <U>Enumerable 메소드의 정보</U>가 `Enumerator` 객체에 저장된다. 위에서 `Enumerator` 객체가 "이터레이션을 가능하게 하는 클래스"라는 모호한 설명을 하였는데, 보다 정확해졌다. 이 객체는 <U>콜렉션을 어떻게 이터레이션 할 지에 관한 정보를 갖고 있는 객체</U>라고 하는 것이 정확할 것 같다. 루비에 이 코드가 구현된 Array.c 파일을 열어, `each`의 코드를 살펴보면 더욱 명확해 진다.

    rb_ary_each(VALUE array)
    {
        long i;
        volatile VALUE ary = array;

        RETURN_SIZED_ENUMERATOR(ary, 0, 0, ary_enum_length);
        for (i=0; i<RARRAY_LEN(ary); i++) {
            rb_yield(RARRAY_AREF(ary, i));
        }
        return ary;
    }

    # https://github.com/ruby/ruby/blob/5b3b8554c9f851a958c03fc707bd69e6ad6c5c13/array.c

`Array` 객체는 `RETURN_SIZED_ENUMERATOR`을 통해서 `Enumerator` 객체를 생성하는 것을 알 수 있다.

    Array.instance_method(:each)
    # Output: => #<UnboundMethod: Array#each>

    Array.instanece_method(:sort)
    # Output: => #<UnboundMethod: Array(Enumerable)#sort>

    Array.instanece_method(:each_with_index)
    # Output: => #<UnboundMethod: Array(Enumerable)#each_with_index>

가만, 위에서는 이터레이션을 위해 `Enumerable` 모듈을 사용한다고 했는데, 왜 `each`는 array.c에 구현이 되어 있을까? each와 같은 `Enumerable` 모듈에 구현된 메소드들이, `Array`나 `Hash`와 같은 데이터 콜렉션에서 자체적으로 구현을 하고 있는 경우도 있다. 모듈 속의 메소드들은 클래스 메소드를 오버라이팅 하지 않기 때문에, 모듈 보다는 클래스의 메소드가 우선하게 되고, 클래스의 메소드가 없을 경우에만 모듈 속의 메소드가 작동한다. 이것은 `each`와 같은 경우에 각 객체마다의 특성이 있기 떄문에 다르게 구현하고 있는 것이다. `each` 외에 `each_with_index`나 `find`, `sort`와 같은 메소드들은 `Enumerable`에서 구현한 메소드를 사용하고 있다.


### Enumerable과 Enumerator 차이 정리
> Enumerator : 이터레이션 하기 위한 데이터 콜렉션과, 이터레이션 메소드를 갖고 있는 클래스<br>
    Enumerable : 이터레이션 메소드를 갖고 있는 모듈


## 읽어보면 좋은 글
* [Ruby Iterators, Enumerators, Enumerable, and Loops](http://www.zenruby.info/2016/06/ruby-iterators-enumerators-enumerable.html?m=1)
* [Ruby 2.0 Works Hard So You Can Be Lazy](http://patshaughnessy.net/2013/4/3/ruby-2-0-works-hard-so-you-can-be-lazy)