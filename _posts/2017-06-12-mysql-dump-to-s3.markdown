---
title: mysql dump를 s3로 저장하기
layout: post
date: 2017-06-12 01:11
image: /assets/images/markdown.jpg
headerImage: false
tag:
- ruby
- rails
- mysql
- aws
- s3
blog: true
author: minjepark
description: mysql dump를 s3로 저장하기
---

{: .center}
![capistrano](/assets/images/aws_rds.png)
<p class="center image-note">RDS는 정말 편리한 서비스다</p>
<br>

AWS에서 mysql을 사용할 때 RDS를 사용하는 이유는 크게, EC2의 인스턴스의 컨디션에 구애 받지 않고, 사용할 수 있다는 점, 복제, 다양한 스케일링 옵션, 장애 복구, 백업 등 다양한 이유가 있고, RDS를 적극적으로 사용하기를 추천한다. 하지만, 프로젝트의 초창기에 극도로 빈곤한 재정 상태일 경우, freetier 계정 조차 얻지 못한 극한의 상황인 경우를 가정하자. 물론, 이런 상황은 좀처럼 일어나지 않는다. 그럴 때는, mySQL을 EC2 인스턴스에 같이 붙이는 수 밖에 없을 것이다.

그럴 때 가장 문제가 되는 것은, MySQL의 데이터를 백업할 방법이 없다는 것이다. 중요한 데이터들이 데이터베이스에 저장되어 있는데, 한 순간의 실수로 모든 데이터를 잃어버리게 된다면, 프로젝트의 운명은 그날로 끝나는 것이다. 주기적으로 MySQL 덤프를 떠서, S3 상에 저장시켜 놓는다면 어떨까. 덤프 시점부터 복구 시점까지의 데이터가 유실될 수도 있지만, 나쁘지는 않은 선택이다.

일단, 레일즈 프로젝트라고 가정하고. AWS에서 제공하는 `aws-sdk`을 설치한다. 이 젬을 설치하면, 레일즈와 루비 상에서 AWS 서비스에 접근할 수 있다.

{% highlight ruby %}
  gem 'aws-sdk'
{% endhighlight %}

AWS의 접근할 수 있는 계정 정보를 `config/aws.yml`에 기록한다.

{% highlight ruby %}
development:
  access_key_id: 
  secret_access_key:
  region:

stage:
  access_key_id:
  secret_access_key:
  region:

production:
  access_key_id:
  secret_access_key:
  region:
{% endhighlight %}

실제로 덤프를 수행하고, S3에 업로드하기 위해 `lib/tasks/backup_db_to_s3.rake` 파일을 만든다.

{% highlight ruby %}
require 'aws-sdk'
require 'YAML'

namespace :backup_db_to_s3 do
  task run: :environment do
    path = ""
    time = (Time.current - 1.day).strftime("%Y%m%d")
    file_name = "dump-#{time}.sql" # 덤프가 저장될 파일명

    # 로컬과 서버 상에서의 경로, DB설정이 다를 수가 있으니 분기를 한다
    if Rails.env.development?
      path = "/Users/mznet/#{file_name}" # dump가 저장될 경로를 적어주자
      `mysqldump -u username db_name > #{path}`
    else
      path = "/home/ubuntu/#{file_name}" # dump가 저장될 경로를 적어주자
      `mysqldump -u username -p password db_name > #{path}`
    end

    p "#{file_name} file has been made"

    yaml = YAML.load_file('config/aws.yml') # 위에서 작성한 AWS 계정 정보를 불러온다
    aws_config = yaml[Rails.env] # 불러온 yaml에서 레일즈의 환경에 따라 값을 찾는다

    s3 = Aws::S3::Resource.new(
      credentials: Aws::Credentials.new(aws_config['access_key_id'], aws_config['secret_access_key']),
      region: aws_config['region']
    )

    # bucket에 데이터가 저장될 S3의 bucket명을 적는다.
    obj = s3.bucket('bucket').object(file_name)
    obj.upload_file(path)

    p "#{file_name} has been successfully uploaded"
  end
end
{% endhighlight %}

테스트를 하기 전에 S3에 bucket을 만드는 것을 잊지 말자. 그리고 rake를 돌려보면 작업이 정상적으로 수행되는 것을 알 수 있다. 주기적으로 rake 작업을 돌리기 위해 whenever에 등록하여 cronjob으로 동작하도록 하면 더욱 좋다.

AWS의 RDS의 t2.small 인스턴스 한달 사용료가 $38 정도 밖에 안한다. 저렴하고, 빠르고, 장애 대응에도 더욱 좋다. 왠만하면 RDS를 사용하는 것이 낫다.