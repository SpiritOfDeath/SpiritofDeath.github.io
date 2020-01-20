---
title: "Welcome!"
date: 2020-01-20 
categories: Company Common 
---
### Apache Sorl Jennifer 연동(2017.08.29)

현상
    Search Service 내의 Sorl 서버의 서비스 지연이 발생
 
원인
    Sorl Server Thread Dump

    "httpShardExecutor-4-thread-1450-processing-x:kr11st-payload.20170829_shard7_replica1 r:core_node7 http:////172.21.12.197:11100//solr//kr11st-payload.20170829_shard2_replica1// n:172.21.12.150:11100_solr s:shard7 c:kr11st-payload.20170829 [http:////172.21.12.197:11100//solr//kr11st-payload.20170829_shard2_replica1//]" #2605 prio=5 os_prio=0 tid=0x00007fb868012000 nid=0x6db2 waiting for monitor entry [0x00007fb7ee4f2000]

    java.lang.Thread.State: BLOCKED (on object monitor)

        at jennifer.control.e.c$1$1.a(SourceFile:39)

        - locked <0x00000001006e16d0> (a jennifer.control.e.c$1$1)

        at jennifer.control.e.c$1$1.takeNewData(SourceFile:92)

        at jennifer.nio.ArrayPartitioningAdapterWithManagingSendQueue$1$1.enque(SourceFile:26)

        at jennifer.nio.ByteArrayRespondingToExchangingAdapter$1$1.enque(SourceFile:32)

        at jennifer.nio.CommandToByteArrayRespondingAdapter$1$1.enque(SourceFile:26)

        at jennifer.control.d.a.a(SourceFile:50)

        at jennifer.control.d.a.enqueueSendingRawDataAndGetResult(SourceFile:39)

        at jennifer.base.AgentDataSenderKeeper.enqueueSendingRawDataAndGetResult(AgentDataSenderKeeper.java:28)

        at jennifer.runtime.e.a(SourceFile:21)

        at jennifer.runtime.e.do(SourceFile:15)

        at jennifer.runtime.tracer.g.a(SourceFile:17)

        at jennifer.runtime.tracer.ActiveObject.popProfile(SourceFile:209)

        at jennifer.runtime.tracer.a.g.a(SourceFile:159)

        at jennifer.runtime.tracer.a.e.a(SourceFile:205)

        at jennifer.runtime.tracer.b.c.end(SourceFile:308)

        at jennifer.base.profile.ProfileService.end(ProfileService.java:106)

        at org.apache.solr.handler.component.ShardResponse.setShardRequest(ShardResponse.java:67)

        at org.apache.solr.handler.component.HttpShardHandler.lambda$submit$0(HttpShardHandler.java:136)

        at org.apache.solr.handler.component.HttpShardHandler$$Lambda$416/861860030.call(Unknown Source)

        at java.util.concurrent.FutureTask.run(FutureTask.java:266)

        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)

        at java.util.concurrent.FutureTask.run(FutureTask.java:266)

        at com.codahale.metrics.InstrumentedExecutorService$InstrumentedRunnable.run(InstrumentedExecutorService.java:176)

        at org.apache.solr.common.util.ExecutorUtil$MDCAwareThreadPoolExecutor.lambda$execute$0(ExecutorUtil.java:229)

        at org.apache.solr.common.util.ExecutorUtil$MDCAwareThreadPoolExecutor$$Lambda$15/1862027464.run(Unknown Source)

        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)

        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)

        at java.lang.Thread.run(Thread.java:745)



분석 내용

    jennifer control Package 내에서의 처리중 Thread Lock 으로 추정
 
작업 진행 순서

    Jennifer Agent Version Upgrade
    Jennifer Agent Configuration
    Sorl Service restart
    Application Launch Point Configuration
    Launch point Package appointed
        Package Name : org.apache.sorl.*
    Application Launch Point Configuration remove
    create thread dump for sorl service 
    Sorl Service restart
    SAS Group A 트래픽 차단
    Sorl Service 재기동
    Error SR 진행
 
SR 진행 첨부 자료

    sorl service thread dump
    Application profiling data
    Jennifer data server log
    Jennifer agent Service dump
 
Application 시작 지점 패키지 선정 이유

    기존 Configuration 유지
 
추후 반영 작업 시 환경

    Backup sorl 서버 Jennifer Agent 구성
    Jennifer trace log 설정
    SR 진행 결과 내용의 장애시 내용 전달을 위한 설정
    일부 트래픽 유입
    서비스 모니터링
 
추후 작업 방향

    SR 진행 답변 확인후 세부 계획 수립
 
서비스엔지니어링2팀 요청 사항

    어플리케이션 인입 class 확인 필요
    Jennifer 연동 테스트 및 모니터링을 위한 Backup Server 트래픽 유입 필요
 
##SR 답변

1차(2017-08-31)

    원인 : 애플리케이션 시작점을 패키지로 설정하면서 운영중인 시스템에 redefine 이 일어나서 에이전트 네트워크 쓰레드가 BLOCK 가 걸리는 현상
         에이전트 로그에 보면 redefine 이 일어난 시점 부터 큐 사이즈가 full 이 발생

    자체 테스트 결과 : solr 최신 버전(6.6.0)을 가지고 테스트를 해봤는데 트랜잭션 수집이 되지 않아서 서블릿 필터를 시작점으로 추가하여 설정후 테스트 해보았 으나 상황 재현 안됨
         ex) org.apache.solr.servlet.SolrDispatchFilter, org.apache.solr.*

    Previous cases : 트랜잭션이 발생하거나 동적으로 redefine 이 일어 날 경우 환경에 따라서 정상적인 서비스가 되지 않았던 경우가 있습니다.
    (보통 패키지 제품을 포함하거나 다른 에이전트가 있거나 I/O 관련 작업이 있는 부분에 프로파일 등 BCI 작업을 설정할 경우 발생)
            해당 경우 옵션 적용 하여 서비스 영향 문제 해결 

    중간 결과 : 
    실제 redefine 으로 인해서 동기화 문제가 발생 할 수 있는지에 대한 내용은 파악 중
    이전 사례 및 좀더 심도 영향도 파악을 위해 서비스 덤프 분석 중

2차(2017-09-04)

    결과 : 
        제니퍼에서 어플리케이션 감시 시작점을 패키지 명으로 잡아 Jennifer Agent log를 전송하는데 다수의 thread 가 Lock이 잡혀 발생

    제조사 제안
        어플리케이션 감시 시작점을 패키지명으로 잡지 말고 해당 클래스 로 설정 후 등록

요청사항

    Service ENG 2 → 개발팀
    어플리케이션 감시 시작점으로 잡아야할 클래스 확인 
    Service ENG 2 → 제조사
    추후 다른 문제 발생시 여러번 작업 하기 힘든 상황이므로 문제 발생시 필요한 데이터를 수집 할수 있는 방법 및 옵션 요청

작업 방향

    Common
    기존 구성햇던 Config Data 적용 안함
    기존의 구성 데이터 기반으로 구성시 문제가 발생
 
문제 발생시 추출할수 있는 데이터 수집방안 확인

해당 옵션이 서비스에 큰 지장안 안준다는 제조사 답변 얻은 이후 작업 일자 결정
