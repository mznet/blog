---
title: 저사양 인스턴스에서 카피스트라노로 배포하기
layout: post
date: 2016-12-10 17:09
image: /assets/images/markdown.jpg
headerImage: false
tag:
- ruby
- capistrano
blog: true
author: minjepark
description: 저사양 인스턴스 환경에서 카피스트라노로 배포할 때 메모리 부족 문제 해결
---
{: .center}
![capistrano](/assets/images/capistrano.png)
<p class="center image-note">카피스트라노는 훌륭한 툴이지만, 때때로 내 속을 썩인다</p>
<br>

AWS EC2의 small이나 nano 인스턴스에 배포를 할 경우 메모리 부족 문제로 배포 시에 에러가 발생할 수 있다. 
인스턴스 크기가 아니더라도, `assets.rb` 파일에 `config.assets.compile = true`이 켜져, assets 파일들이 압축되어 배포될 경우, 압축해야 할 assets가 많거나, 파일 크기가 클 경우에도 메모리 부족으로 에러가 발생할 여지가 있다.
EC2 인스턴스를 업그레이드 할 수 도 있지만, `rsync`를 이용하면, 배포 시에 메모리 부족으로 인한 에러를 막을 수 있다.
카피스트라노를 통해 배포할 경우에, 원격 서버에 배포한 후, 원격 서버에서 `precompile`을 진행하기 때문에, 서버에 메모리가 부족할 경우에 에러를 일으킬 수 있다.

<br>

{: .center}
![rsync](/assets/images/rsync.png)
<p class="center image-note">rsync는 구세주가 될 수 있을 것인가?</p>

### rsync?
rsync는 Remote Sync의 줄임말로, 말 그대로 두 컴퓨터 시스템의 파일을 동기화하기 위한 도구이다. scp도 파일을 송수신할 수 있지만, rsync를 이용하면 아래와 같은 장점이 있다.

* 데이터를 압축해서 송/수신하기 때문에 더 빠르게 복사할 수 있다.
* remote-update 프로토콜을 이용해서 차이가 있는 파일만 복사하기 때문에, 더 빠르게 복사할 수 있다.
* `Quick Check` 알고리즘을 통해 파일크기와 파일 변경 시간을 비교하여, 차이가 날 경우에만 파일을 전송하게 된다.

그럼 간단하게 `rsync`의 사용법을 알아보자

    rsync [options] [source] [destination]

[options]에는 rsync의 옵션들, source에는 동기화 원본, destination에는 동기화 대상을 입력한다. 간단하게 작성해보면,

    rsync -z application.rb mznet@52.176.1.1:/home/mznet/app

위와 같은 방법으로 동기화를 빠르게 진행할 수 있다. 옵션은 [https://linux.die.net/man/1/rsync](https://linux.die.net/man/1/rsync) 링크를 참조하자.

### capistrano에 rsync를 넣어보자

`deploy.rb` 파일을 열어서 아래의 코드를 넣자. 자신이 사용하는 ssh 키의 위치와 rsync 옵션을 자신이 원하는 것으로 적절하게 세팅하도록 하자.

{% highlight ruby %}

# 이미 정의된 compile_assets 명령 때문에, precompile이 중복될 수 있으므로 기존의 compile_asssets를 동작하지 않도록 한다
Rake::Task['deploy:compile_assets'].clear

task :compile_assets do
  # compile assets locally
  run_locally do
    rails_env = fetch(:rails_env)
    if system('which rvm')
      with rails_env: rails_env do
        execute 'bundle exec rake assets:precompile'
      end
    else
      execute "RAILS_ENV=#{rails_env} bundle exec rake assets:precompile"
    end
  end

  # rsync to each server
  local_dir = './public/assets/'
  on roles(:web) do
    # this needs to be done outside run_locally in order for host to exist
    remote_dir = "#{host.user}@#{host.hostname}:#{shared_path}/public/assets/"
    run_locally { execute "rsync -av --rsh \"ssh -i ssh키 위치" --delete #{local_dir} #{remote_dir}" }
  end

  # clean up
  run_locally { execute "rm -rf #{local_dir}" }
end
{% endhighlight %}

rsync를 통해서 assets 배포를 진행할 경우, assets 파일의 `precompile`을 로컬 상에서 진행한 다음, 원격 서버에 `rsync`를 통해 동기화하는 것이다.

정의한 `compile_assets`의 실행 시점을 정해야 한다. 아래의 코드를 `deploy.rb`에 넣도록 하자.

{% highlight ruby %}
after  :finishing,    :compile_assets
{% endhighlight %}

after hook을 넣어서, 모든 파일의 배포가 끝난 뒤에 `compile_assets`를 통해 `precompile`과 동기화를 진행한다.
hook에 관하여 더 알고 싶다면, [http://capistranorb.com/documentation/getting-started/flow/](http://capistranorb.com/documentation/getting-started/flow/)와 [http://capistranorb.com/documentation/getting-started/before-after/](http://capistranorb.com/documentation/getting-started/before-after/)를 살펴보자