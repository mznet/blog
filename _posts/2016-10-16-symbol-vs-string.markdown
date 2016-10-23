---
title: "symbol vs string"
layout: post
date: 2016-10-16 16:50
image: /assets/images/markdown.jpg
headerImage: false
tag:
- ruby
- hash
- string
blog: true
author: minjepark
description: hash와 symbol의 차이
---

[<Agile Web Development With Rails 4>](http://www.insightbook.co.kr/도서-목록/programming-insight/레일스와-함께-하는-애자일-웹-개발-개정판/)라는 책으로 레일즈와 루비에 관해 처음 접하게 되었는데, `symbol`이라는 다른 언어에서 보지 못한 개념이 등장하여 꽤나 혼란스러웠던 기억이 난다. 그 책에는 사실 루비 문법이나 용례에 관한 설명이 없기에 구글링을 통해 개념을 찾아 봤지만, 머리로는 이해할 뿐 '왜'라는 질문은 지울 수가 없었다. 처음에는 익숙하지 못한 개념이라, symbol이 필요할 때 string을 대신 썼다. 레일즈에 설정을 바꾸거나 추가할 때는 꼭 symbol이 필요했는데, 왜 그렇게 레일즈의 설정은 symbol을 사랑하는지 의문스러웠다. 하지만, 레일즈와 루비를 점점 쓰면서 그 차이는 몸으로 머리로 이해하게 되었고, hash와 symbol의 차이에 대해 내가 아는 내용을 조금이나마 정리하고자 한다.

# symbol vs string
symbol와 string의 차이는 매우 간단하다. 변하는 지(mutable), 변하지 않는 지(immutable). string은 같은 문자열이라도 매번 부를 때 마다 새롭게 만들어지는 객체(루비에서 string은 객체)이고, symbol은 같은 내용이라면 다시 만들지 않는 객체이다. 그 차이를 콘솔을 열어 코드를 몇 줄 적어보면 간단해진다.

    2.2.3 :001 > 'test'.object_id
     => 70342731412040
    2.2.3 :002 > 'test'.object_id
     => 70342731380880
    2.2.3 :003 > 'test'.object_id
     => 70342731348200
    2.2.3 :004 > 'test'.object_id
     => 70342731331000

string의 경우 "test"라는 같은 내용이라도 object_id가 계속 달라지는 것을 볼 수 있다. 내용은 같지만 다른 객체이며, "test"를 부를 때 마다, 새롭게 생성한다는 것을 알 수 있다. 그렇다면 symbol은 어떨까? 똑같이 적어보자.

    2.2.3 :001 > :test.object_id
     => 350108
    2.2.3 :002 > :test.object_id
     => 350108
    2.2.3 :003 > :test.object_id
     => 350108
    2.2.3 :004 > :test.object_id
     => 350108

symobl은 string과는 다르게 object_id가 항상 일정하다. 즉 같은 내용이라면 객체를 새롭게 생성하지 않는 것이다. 같은 객체를 계속 생성하지 않는 symbol이 string 보다 메모리 낭비가 적은 것이다. 다만, symbol은 가비지 컬렉팅이 되지 않기 때문에, symbol이 너무 남용되지 않게 주의해야 한다.

이이 정도까지가 인터넷 상에서 흔히 찾을 수 있는 내용이었다. symbol과 string의 차이를 더 깊이 들어가보자.

symbol이 가비지 컬렉팅의 대상이 되지 않는 것은 ruby 2.2 버전 이전의 일이고, 2.2부터는 symbol도 가비지 컬렉팅의 대상이 된다. symbol은 메모리 낭비가 적기 때문에, 설정을 작성할 때 많이 사용된다. 레일즈 프레임워크에서 `application.rb`나 `production.rb`와 같이 빈번하게 설정은 프로덕트 상에서 몇 번이고 불러오게 되는데, 이 때마다 string을 사용한다면, 객체를 매번 생성하기 때문에 메모리 낭비를 줄이기 위해 string 대신에 symbol을 사용한다.

ruby 2.2 이전의 경우 symbol이 가비지 컬렉팅의 대상이 되지 않았었다는 것은 위에서 언급하였다. 그래서 symbol은 남용하면 좋지 않고, 특히 iterator를 통해서 동적으로 심볼을 만드는 것을 금지시 하였다. 그리고 [동적으로 symbol 만드는 것이 크게 문제](https://www.ruby-lang.org/en/news/2013/02/22/json-dos-cve-2013-0269/)가 된 적이 있었다. 루비에 포함되어 있는 JSON gem의 1.7 대 버전에서 json을 파싱할 때 동적으로 symbol을 만드는 것을 이용해 DoS를 할 수 있는 것이었다.

# hash에서 symbol과 string의 쓰임과 퍼포먼스 차이
hash에서의 symbol과 string의 차이를 알아보면 둘의 차이가 더 명확해진다. 레일즈에서 설정의 경우 hash인 경우가 많기 때문에, symbol과 string의 차이는 사실상 hash에서의 차이라고 볼 수 있다. 처음에는 hash에서 key로 string을 사용하였지만, symbol과의 차이를 보고 난 뒤에 다시는 쓰지 않게 되었다. 그 차이를 보기 위해 Benchmark를 작성해 보자.

    hash = {
      test: 'test',
      'test' => 'test'
    }

    Benchmark.bm do |x|
      x.report('string') { 6.times { |t| hash['test'] }}
      x.report('hash') { 6.times { |t| hash[:test] }}
    end

    # 벤치 마크 결과
    [#<Benchmark::Tms:0x007f83b20c3a68 @label="string", real=0.0852638639998986, cstime=0.0, cutime=0.0, stime=0.0, utime=0.08000000000000007, total=0.08000000000000007>,
    #<Benchmark::Tms:0x007f83b20c32c0 @label="string", real=0.06605425199995807, cstime=0.0, cutime=0.0, stime=0.009999999999999995, utime=0.05999999999999994, total=0.06999999999999994>]

hash에서 string과 symbol의 차이가 꽤 나는 것을 알 수 있다. 만약 우리가 사용하고 있는 프로덕트에서 기존의 해시 로직이 string으로 대거 만들어져 있다면, 어떻게 할 것인가? 무턱대고 symbol로 바꾸기엔 무리가 따른다. 로직에서 타입을 체크하거나, 타입을 변환하는 일도 있을테니 큰 문제가 없을 것이라 보장할 수는 없다. 그 때 사용하면 좋은 것이 `freeze` 메소드이다.

# freeze

[`freeze` 메소드](https://ruby-doc.org/core-2.3.1/Object.html) 객체의 수정을 금지하는 기능을 한다. 만약에, 수정을 시도하면 exception을 발생시킬 것이다.

    2.2.3 :070 > test = {test: 1}
     => {:test=>1}
    2.2.3 :071 > test.freeze!
    NoMethodError: undefined method `freeze!' for {:test=>1}:Hash
      from (irb):71
      from /Users/mznet08/.rvm/rubies/ruby-2.2.1/bin/irb:11:in `<main>'
    2.2.3 :072 > test = test.freeze
     => {:test=>1}
    2.2.3 :073 > test << {test2: 'test'}
    NoMethodError: undefined method `<<' for {:test=>1}:Hash
      from (irb):73
      from /Users/mznet08/.rvm/rubies/ruby-2.2.1/bin/irb:11:in `<main>'

freeze를 사용하면 객체의 수정을 금지하므로, 새롭게 만들지도 않을 것이다. 따라서, `string.freeze`의 형식으로 만든 객체는 항상 같은 object.id를 가질 것이다.

    2.2.3 :076 > "string".freeze.object_id
     => 70101815872820
    2.2.3 :077 > "string".freeze.object_id
     => 70101815872820
    2.2.3 :078 > "string".freeze.object_id
     => 70101815872820
    2.2.3 :079 > "string".freeze.object_id
     => 70101815872820

행여나 기존에 작성한 해시의 퍼포먼스를 올리고 싶다면 `freeze`를 사용해보자.

ruby 2.2 기준으로 string으로 만든 key는 자동으로 freeze를 사용하여 만들 기 때문에 이전 버전의 hash 보다 더 나은 퍼포먼스를 보여준다. 다만 key에 freeze하는 시간이 더해지기 때문에, symbol의 속도보다는 느리다. 새롭게 hash를 만들어야 한다면 key는 반드시 symbol로 만드는 것이 좋을 것이다.


## 읽어보면 좋은 글
* [Unraveling String Key Performance in Ruby 2.2](https://www.sitepoint.com/unraveling-string-key-performance-ruby-2-2/)
* [Ruby Symbols](http://railsdevs.blogspot.kr/2012/12/ruby-symbols.html)
* [Symbol DOS](https://www.ruby-lang.org/en/news/2013/02/22/json-dos-cve-2013-0269/)
* [freeze](https://ruby-doc.org/core-2.3.1/Object.html)


