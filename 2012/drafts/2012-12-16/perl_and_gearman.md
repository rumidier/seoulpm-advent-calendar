# gearman worker 운영 #

용빈님의 gearman기초덕에 요점만 쓸수 있게 되어서 기쁩니다^^ 용빈님++.

# Gearman 
Worker
기어맨을 본격적으로 도입하셨나요?
손쉽고 깔끔하게 RPC와 맵리듀스를 구현한 뒤에 찾아오는 워커관리 지옥을 겪어보셨나요?
저는 그 지옥을 보았습니다. 그래서 그 지옥에서 탈출하는 방법을 만들었고 공유드리고자 합니다.

## 기어맨 워커 운영 시 겪는 워커 문제들
   - 너무 많은 워커
   - 너무 적은 워커
   - 중복되는 코드
   - 메모리릭
   - 워커 프로세스의 종료

## Gearman::SlotManager
위 문제들을 한꺼번에 해결하기 위해
[Gearman::SlotManager](https://metacpan.org/module/Gearman::SlotManager) 을 작성하였습니다.
슬롯을 미리 확보해놓고 해당 슬롯의 워커를 spawn하는 방식으로 작동합니다.
최소슬롯수만큼 워커를 띄우고 띄워진 모든 워커가 작업중이라면 워커를 최대슬롯수만큼 띄우게 끔 했습니다. 노는 워커가 있다면 최소슬롯수 까지 하나씩 줄입니다.

  * Flexible한 워커풀의 관리.
  * 워커의 Time to live 관리.
  * AnyEvent 루프내장.

### 모듈 설치

    cpan install Gearman::SlotManager
    또는
    cpanm Gearman::SlotManager

로 설치하시면 됩니다.

### 워커의 작성

Gearman::SlotManager는 [Gearman::SlotWorker](https://metacpan.org/module/Gearman::SlotWorker)를 이용하도록 되어 있습니다.

    package TestWorker; # 패키지명을 꼭 명시
    use Any::Moose; # Any::Moose를 이용합니다.
    extends 'Gearman::SlotWorker'; # Gearman::SlotWorker를 상속
     
    sub reverse{ # 'TestWorker::reverse' 로 Gearman에 등록됩니다.
        my $self = shift;
        my $data = shift;
        return reverse($data);
    }
     
    sub _private{  # '_' 로 시작하면 등록되지 않습니다.
        my $self = shift;
        my $data = shift;
        return $data;
    }
     
    sub NOTVISIBLE{ # 전체대문자 역시 등록되지 않습니다.
        #...
    }

데이터를 받아서 reverse함수의 리턴값을 되돌려주는 크게 쓸모는 없는 워커를 만들었습니다.

메소드를 추가하면 PACKAGE_NAME::METHOD_NAME 으로 Gearman Daemon에 등록이 되구요.

SlotManager를 작성해서 모듈만 명시해주면 바로 사용할 수 있게 됩니다.

### gearmand 의 기동

이 모듈을 성공적으로 설치했다면, perl로 구현된 gearmand 가 설치되어 있을 것입니다.

    gearmand -p 9955 -d

와 같이 실행을 해서 gearman 데몬을 하나 띄워줍니다.

물론 [binary gearmand](http://gearman.org/doku.php) 도 사용가능 합니다.

### SlotManager 의 작성 및 기동

[Gearman::SlotManager](https://metacpan.org/module/Gearman::SlotManager) 를 이용하여 시동스크립트를 작성해봅시다.

https://metacpan.org/source/KHS/Gearman-SlotManager-0.3/testManager.pl 를 참고하여 작성해 봅시다.

    #!/usr/bin/perl
    
    use AnyEvent;
    use Gearman::SlotManager;
       
    my $cv = AE::cv;
     
    my $sig = AE::signal 'INT'=> sub{ # 데몬종료를 위한 SIGNAL 처리
        $cv->send;
    };
     
    my $slotman = Gearman::SlotManager->new(
        config=>
        {
            slots=>{
                'TestWorker'=>{ # 워커의 패키지명
                job_servers=>["localhost:9955"], # gearmand 의 port
                libs=>['.'], # worker가 있는 경로
                min=>2, # child 최소갯수, 기본값 1
                max=>5, # child 최대갯수, 기본값 2
                workleft=>10, # child 당 처리 횟수, 기본값 0.
                }
            }
        },
        port=>55522, # child 상태조회용 port. 여러개의 SlotManager를 띄울경우 겹치지 않게 설정.
    );
     
    $slotman->start; # 데몬 시작 준비 (child process 생성)
    
    my $res = $cv->recv; # 이벤트루프 시작(데몬시작)
    
    $slotman->stop; # SlotManager 중지 (child process 제거)
    undef($slotman); # SlotManager 삭제
    undef($sig); #시그널 핸들러 삭제

좀 길어 보이죠? 
그렇지만 잘 뜯어보면 아주 간략합니다.

부분부분 살펴봅시다.

#### 모듈 로드 ####

    use AnyEvent;
    use Gearman::SlotManager;

SlotManager는 AnyEvent를 이용하므로 AnyEvent를 use해줘야합니다.

#### 종료 시그널 처리 ####

    my $cv = AE::cv;
     
    my $sig = AE::signal 'INT'=> sub{ # 데몬종료를 위한 SIGNAL 처리
        $cv->send;
    };

Ctrl+C 를 받으면 $cv, AnyEvent의 CondVar에 메세지를 던져서 루프를 종료시키게끔 해놨습니다.

#### SlotManager의 생성 ####

    my $slotman = Gearman::SlotManager->new(
        config=>
        {
            slots=>{
                'TestWorker'=>{ # 워커의 패키지명
                job_servers=>["localhost:9955"], # gearmand 의 port
                libs=>['.'], # worker가 있는 경로
                min=>2, # child 최소갯수, 기본값 1
                max=>5, # child 최대갯수, 기본값 2
                workleft=>10, # child 당 처리 횟수, 기본값 0.
                }
            }
        },
        port=>55995, # child 상태조회용 port. 생략가능. 여러개의 SlotManager를 띄울경우 겹치지 않게 설정.
    );

여긴 좀 복잡하지만~ 

  * TestWorker.pm 파일이 같은 디렉토리('.')에 있고
  * localhost:9955 에 떠있는 gearmand에 접속을 할 것이며,
  * 시작시 2개의 워커를 띄우고,
  * 15초에 한번씩 검사하여 2워커가 모두 busy 상태이면, 5개까지 늘리도록 하고,
  * 각 워커는 10번의 호출을 처리하고 나면 respawn을 하도록 해라.

라는 설정입니다.

여러개의 워커는?

          slots=>{
              'TestWorker'=>{ # 워커의 패키지명
              job_servers=>["localhost:9955"], # gearmand 의 port
              libs=>['.'], # worker가 있는 경로
              min=>2, # child 최소갯수, 기본값 1
              max=>5, # child 최대갯수, 기본값 2
              workleft=>10, # child 당 처리 횟수, 기본값 0.
              },
              'SecondWorker'=>{
              job_servers=>["localhost:9955"],
              libs=>['../second'],
              min=>2,
              max=>5,
              workleft=>10,
              }
          }

이렇게 여러개를 등록해줄 수 있습니다.
반복되는 설정항목들이 보이죠?

          slots=>{
              global=>{
                job_servers=>["localhost:9955"],
                libs=>['.'],
                min=>2,
                max=>5,
                workleft=>10,
              },
              'TestWorker'=>{
                max=>50,
              },
              'SecondWorker'=>{
                workleft=>20,
              }
          }

이렇게 global 항목에 기본값들을 넣어주시면 깔끔해집니다.
이런 방법으로 여러 서비스에 사용되는 워커들을 통합관리할수 있게 됩니다.

### 기동

    perl testManager.pl

위와 같이 해주시면 됩니다.

프로세스 목록을 보면 child프로세스들이 후루룩 떠있는걸 목격하실텐데,
testManger.pl 의 프로세스만 ctrl+c로 종료시켜주면 샤라락 사라지니 걱정않으셔도 됩니다.

### SlotWorker 에서의 AnyEvent

SlotWorker는 Slot클래스에 의해 직접 별도의 프로세스로 실행됩니다.
fork로 프로세스를 생성하지 않으므로 AnyEvent의 부작용이 없습니다.

SlotWorker프로세스는 독립적인 AnyEvent Loop를 가지고 있어서 메소드안에서 마음껏 $cv->recv()를 이용하여 비동기 처리를 해줄 수 있습니다.

TestWorker 에다가 아래와 같이 써서, "TestWorker::wait"를 호출해도 잘 작동하는 거죠~

    sub wait{
      my $cv = AE::cv;
      my $timer = AE::timer 10,0,sub{$cv->send}; # 10초 딜레이
      
      $cv->recv;
      return "OK";
    }

게다가 workleft를 설정해놓으면, Worker 프로세스를 완전히 중지시키고 새 Worker프로세스를 시작시키기 때문에, 사용하는 모듈에 Memory-Leak이 있다고 하더라도 마음놓고 돌릴 수 있답니다. ^^

# 마치며 #

요즘 시간에 여유가 없어서, 마음 급한채로 작성하다보니 보다 풍부한 예를 들지 못한 점이 아쉽네요.

Gearman 워커를 다수 운영하며, 여러가지 아쉬웠던 점들을 보완하기 위해 만든 모듈이니 만큼, 같은 입장에 놓이신 여러분들께 많은 도움이 되리라 기대합니다.

그럼 이만~