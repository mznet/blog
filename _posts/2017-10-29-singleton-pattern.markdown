---
title: singleton pattern
layout: post
date: 2017-10-29 16:33
image: /assets/images/markdown.jpg
headerImage: false
tag:
- ruby
- pattern
- singleton pattern
blog: true
author: minjepark
description: singleton pattern
---

싱글톤 패턴의 개념 - 하나의 클래스 인스턴스만이 존재할 수 있음. 어플리케이션의 설정을 담은 클래스를 만드는 경우에 많이 사용됨. 어플리케이션의 설정을 담은 클래스는 하나만 만들어져서 전 어플리케이션에 사용되면 되고, 오히려 여러개가 생성되면 문제가 생길 소지가 있음
루비에서 제공하는 singleton을 사용하면 간편하게 싱글톤 클래스를 만들 수 있음
이것을 사용하면 단 한번만 만들어짐을 보장함

왜 싱글톤 모듈은 new 대신에 instance 메소드를 사용하여 인스턴스화 시키나?

if you try to instantiate this class as you normally would a regular class, a NoMethodError exception is raised.
The constructor is made private to prevent other instances from being accidentally created:

First, the class << foo syntax opens up foo's singleton class (eigenclass)
Now, to answer the question: class << self opens up self's singleton class, so that methods can be redefined for the current self object (which inside a class or module body is the class or module itself).
Usually, this is used to define class/module ("static") methods:

dup vs clone?

As you’ve seen, the most common type of singleton method is the class method—a method added to a Class object on an individual basis:
 
There is no difference. There are no class methods in Ruby. "Class method" is just a name that we humans call a singleton method if the object happens to be an instance of the Class class.

While it's true that class << something is the syntax for a singleton class, as someone else said,
it's most often used to define class methods within a class definition. But these two usages are consistent. Here's how.
(http://stackoverflow.com/questions/6182628/ruby-class-inheritance-what-is-double-less-than)


 ## 읽어보면 좋은 글
 * [Understanding class_eval and instance_eval](http://web.stanford.edu/~ouster/cgi-bin/cs142-winter15/classEval.php)
 * [Design Patterns in Ruby: Observer, Singleton](https://www.sitepoint.com/design-patterns-in-ruby-observer-singleton/)
 * [Ruby Metaprogramming: Part I](https://www.sitepoint.com/ruby-metaprogramming-part-i/)