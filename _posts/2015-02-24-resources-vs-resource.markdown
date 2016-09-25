---
title: "routes에서 resources와 resource의 차이"
layout: post
date: 2016-08-13 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- routes
- resource
- resources
blog: true
author: minjepark
description: routes.rb에서 resources와 resource의 차이
---

최근 옆자리 동료에게 `routes.rb`에서 `resources`와 `resource`의 차이점에 관한 질문을 받았다. 보통은 `resources`를 쓰기 때문에, `resource`가 있다는 생각을 전혀 하지 못했다. 둘의 차이는 `index` 액션의 유무라는 stackoverflow에서 본 답변으로 대신했다. 하지만, 사실 두가지 기능이 존재하는 이유는 분명 이유가 있을 것이기 때문에, 차이점을 찾아보기로 하였다.

## resources

레일즈에서 컨트롤러의 URI를 등록하기 위해서 routes를 통해 등록한다. 레일즈에서는 CRUD와 관련된 공통적인 액션 7개를 `resources` 명령을 통하면 간단하게 등록할 수 있도록 하고 있다. 예를 들어 posts 컨트롤러가 있다면, routes에 `resoruces :posts`라고 적어주는 것만으로 CRUD에 관한 액션을 등록할 수 있다.

    resources :posts

    Prefix Verb         URI Pattern               Controller#Action
    posts       GET     /posts(.:format)          posts#index
                POST    /posts(.:format)          posts#create
    new_post    GET     /posts/new(.:format)      posts#new
    edit_post   GET     /posts/:id/edit(.:format) posts#edit
    post        GET     /posts/:id(.:format)      posts#show
                PATCH   /posts/:id(.:format)      posts#update
                PUT     /posts/:id(.:format)      posts#update
                DELETE  /posts/:id(.:format)      posts#destroy

CRUD 액션은 7개이지만, `update`에 해당하는 http method가 `PATCH`, `PUT` 메소드 두개이기 떄문에, update 액션은 2개를 정의하여 8개의 URI가 등록된다.
애초에, [레일즈의 update에 해당하는 http method는 PUT이 기본이었지만 PATCH으로 변경하였다](http://weblog.rubyonrails.org/2012/2/26/edge-rails-patch-is-the-new-primary-http-method-for-updates/)


`resources`로 등록한 URI는 URI를 통해서 접근하고자 하는 자원이 복수(plural)의 형태 즉, collection이기를 가정하고, 복수의 형태로 URI를 만든다. `index` 액션에서는 post 자원의 전체 리스트를 살피고, `show`나 `edit`에서는 하나의 element를 보거나, 수정하는 액션을 가정한다.

즉 `index` 액션의 경우 `/posts`의 형태를 갖게 되는데, RESTful API에서 이는 전체 포스트를 보여 달라는 요청이다. `show`의 액션은 `/posts/1`의 형태를 갖게 되는데, 전체 포스트 중에서 1번 id의 포스트 element를 보여 달라는 요청이다. 즉, `resources`는 collection을 위한 url helper이다.

하지만, 우리가 URI로 등록하고자 하는 컨트롤러가 복수가 아니라 단수(singular) 형태의, element라면 어떨까? 그럴 때는 `resources` 대신에 `resource`를 사용한다. 그리고 등록하는 심볼도 단수형으로 써줘야 URI도 단수로 나오기 때문에 단수로 써주는 것이 관례(Convention)이다.

## resource

    resource :post

    Prefix Verb         URI Pattern                 Controller#Action
    posts       POST    /post(.:format)             posts#create
    new_posts   GET     /post/new(.:format)         posts#new
    edit_posts  GET     /post/edit(.:format)        posts#edit
                GET     /post(.:format)             posts#show
                PATCH   /post(.:format)             posts#update
                PUT     /post(.:format)             posts#update
                DELETE  /post(.:format)             posts#destroy

`resources`로 등로했을 때와의 차이점이 확연하다. 일단은 `index` 액션이 빠졌다. RESTful API에서 element는 단수의 자원을 가리키므로, `index`과 같이 컬렉션 즉 자원의 리스트에 대한 요청은 필요가 없으므로 보여주지 않는다. `resource`에서도 `resource :posts`라고 사용할 수는 있으나, URI상에 collection을 의미하는 복수형 posts이 붙게 되므로, 단수형인 `post`를 이용하는 것이 RESTful API에 더 적합하다.

## 읽어보면 좋을 글
* [RESTful이란 무엇인가?](http://blog.remotty.com/blog/2014/01/28/lets-study-rest/#crud)
* [REST 아키텍처를 훌륭하게 적용하기 위한 몇 가지 디자인 팁](https://spoqa.github.io/2012/02/27/rest-introduction.html)