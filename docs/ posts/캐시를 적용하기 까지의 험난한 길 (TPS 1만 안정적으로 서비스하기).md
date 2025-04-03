# 🎯 캐시를 적용하기 까지의 험난한 길 (TPS 1만 안정적으로 서비스하기)

> 🕒 **발행일:** Mon, 31 Mar 2025 02:32:00 GMT  
> 🔗 **원본 링크:** [👉 바로 가기](https://toss.tech/article/34481)

---

## 📌 **핵심 요약**  
📖 안녕하세요, 토스뱅크 Server Developer 김경윤 입니다.

토스 커뮤니티, 그리고 토스뱅크에서는 하루에 수 백번의 라이브 배포가 일어나고 있으며 다양한 개선점과 신규 제품들이 빠르게 출시되고 있어요. 이렇게 많은 배포를 기반으로 토스뱅크가 성장하면서 토스뱅크를 이용하는 사용자도 많아졌는데요. 이로 인해서 TPS가 평균 1만, 최대 2만까지 늘어난 약관(terms) 서버에 안정적인 서비스 제공을 위해 캐시를 적용한 이야기를 들려드리려고 해요.

## Database: 더 이상 버틸 수 없다!

약관 서버는 사용자가 동의 또는 철회한 약관 및 동의서 동의 여부를 기록하고 조회할 수 있는 서비스예요. 사용자가 동의 그리고 철회하는 경우는 그렇게 많지 않지만, 동의 여부에 따라서 개인 정보를 제3자에게 제공해도 되는지, 토스뱅크가 사용해도 되는지 등 다양한 비지니스에서 약관 및 동의서 동의 여부를 확인해요. 

(앞으로는 ‘약관 및 동의서’를 간단히 ‘약관’이라고 부를게요.)

![](https://static.toss.im/ipd-tcs/toss_core/live/7feb8944-5dee-4996-903c-d9db84387ab7/image.png)

어느 날, 토스뱅크에서 새로운 서비스를 배포하면서 TPS가 급증했고, 이로 인해 DB 조회량이 급증하여 DB 부하가 심각해졌었어요. 이 트래픽이 유지될 경우 다른 서비스에도 문제가 생길 수 있다고 판단했고, 급히 롤백을 진행하며 DB 부하를 줄일 방안을 고민하게 되었어요. 그 결과, 많은 분들이 알고 계신 ‘캐시’를 적용하기로 했지만, 막상 시도해 보니 생각보다 간단치 않았어요.

## 캐시 적용은 보다 신중하게

이 글을 읽으면서, “어? 그럼 Replication된 Database를 두고 부하를 분산하면 되는거 아니야?” 라고 생각할 수 있어요. 약관은 토스뱅크 고객의 개인정보와 밀접한 연관성이 있어요. 약관 동의 여부는 사용자의 정보를 특정 목적으로 이용해도 되는지, 제3자에게 공유가 되도 되는지 결정하는 기준이기 때문이에요. 

그래서 약관 동의 여부는 값이 DB에 Commit 되는 순간, 바로 다음 요청에 DB에 저장된 값이 정확하게 응답되어야해요. 이를 Strong Consistency라고 부르며, 데이터의 무결성을 보장하기 위해 필수적이에요. 만약, 잘못된 응답을 한 번이라도 발생하면, 고객이 약관을 철회한 이후에도 개인정보가 공유되는 등의 보안 문제가 발생할 수 있어요. 따라서 데이터 신뢰성과 정확성을 보장하기 위해 Replication Database는 사용하지 않기로 했어요. Replication 구조에서는 복제 지연(Replication Delay)이 발생할 가능성이 있기 때문이에요. 대신, Strong Consistency를 유지하면서도 빠르게 조회가 가능한 Redis Cache를 활용해 고객의 약관 동의 상태를 안전하게 관리하기로 결정했어요.

#### 복제 지연(Replication Delay)?주 DB(Primary)와 보조 DB(Replica) 간의 데이터 동기화가 지연되는 현상

### 이러한 방식으로 캐시를 적용했어요!

![](https://static.toss.im/ipd-tcs/toss_core/live/cb4390cc-0d43-4d98-998f-ff21a1d7102d/image.png)

약관 서버는 대부분 조회성으로 캐시를 접근하므로, DB의 정보가 자주 변경되지 않고, 대부분의 케이스는 Cache Hit를 하기 때문에, Look-aside 전략을 사용했어요. Look-aside 전략은 캐시에 원하는 데이터가 있는지 먼저 확인하고, 데이터가 없다면 DB에 접근한 후에 캐시에 저장하고 응답하는 방식이예요. 만약 DB 정보가 자주 변경된다면 다른 방식을 고민해야해요.

캐시 만료처리는 Spring의 `@EntityListener`와 `@TransactionalEventListener` 를 사용해서, DB에 Commit된 이후에 캐시가 만료되도록 했어요. DB에 Commit 전에 만료처리를 한다면, 다른 요청에서 Commit 전 데이터를 다시 캐시에 적재하는 아래와 같은 케이스가 발생할 수 있기 때문이에요.

### \[과거 버전이 다시 적재되는 케이스\]

1. A Thread: 약관 동의 여부가 변경되어, Entity Version이 1에서 2로 올라감
2. A Thread: Entity Version 1 캐시가 Evict 처리 됨 (아직 Commit 전)
3. B Thread: Cache Miss로 Database에서 ⚠️ Entity Version 1을 다시 Cache에 적재함 ⚠️
4. A Thread: Entity Version 2 Commit
5. C Thread: Cache Hit로 ⚠️ 잘못된 Entity Version 1을 응답함 ⚠️

이렇게 보면, 완벽한 것 같지만, `@TransactionalEventListener` 가 `AFTER_COMMIT` 으로 설정되어 있어, Cache Evict 처리가 실패해도, 이미 Commit 된 상태이기 때문에 Transaction이 Rollback 되지 않는 문제가 있어요. 그러면, 과거 캐시 버전이 지속적으로 응답되는 문제가 있죠. 

그러한 문제를 해결하기 위해서, Circuit Breaker를 활용했어요. 만약, Cache Evict을 실패하면, Circuit을 Force Open 해서 자동으로 닫히지 않도록 하고, Cache Get할 때 Circuit이 Open 되어있다면, 모든 트래픽은 바로 Database를 조회하도록 하여, 잘못된 캐시가 응답되지 않도록 방어해두었어요. 이러한 방법은 Database에 큰 부하가 부담될 수 있지만, 잘못된 약관 동의 상태가 응답되어 문제가 발생되는 것보다, Database에 부하를 주고 빠르게 캐시에 대한 문제를 파악 및 해결하는 것이 맞다고 생각해서에요. 아직 이러한 경우를 만난 적이 없지만, 문제 발생 시 빠르게 확인, 해결할 수 있도록 상시 모니터링 중입니다!

#### Circuit Breaker?주로 MSA 구조로된 서비스에서 서비스 간의 장애 전파를 미리 차단하고자 사용하는 기술 Circuit Breaker에 대해 [자세히 설명되어 있는 글](https://techblog.woowahan.com/15694/)을 소개드려요.

## 이렇게 열심히 생각했지만 현실은…?

그럼 Strong Consistency를 잘 지키고 있는지 확인을 해야 하는데요. 그러기 위해서 서버에는 캐시를 적용했지만, 응답은 항상 DB에 저장된 데이터를 응답 하도록하고 DB와 캐시 데이터를 비교해서 확인했어요. 열심히 준비했기 때문에 한 번에 테스트를 통과할 줄 알았지만 언제나 그렇듯, 아래와 같은 문제를 만났어요.

### 0.003초 차이로 발생하는 불일치 케이스

\[문제 과정\]

* A Thread - 약관 동의 발생
* A Thread - `TermsAgreement` 값이 변경되어서, ApplicationEvent 발행
* A Thread - 약관 동의 여부 변경 Kafka Event Produce
* A Thread - `TermsAgreement`가 DB에 Commit 됨
* B Thread - 약관 동의 여부 변경 Kafka Event Consume
* B Thread - 약관 동의 여부 조회시 캐시로부터 ⚠️ 이전 Entity Version 조회 됨 ⚠️
* A Thread - `AFTER_COMMIT` 설정이 되어있어, 이때 Cache Evict 됨

A Thread에서 Commit 이후, Cache를 Evict하는 그 0.003초 사이에 Kafka Event를 Consume한 곳에서 잘못된 캐시를 조회한 문제를 발견했어요. 이 케이스는 생각보다 간단히 해결할 수 있었어요. Kafka Event를 Cache Evict 처리 이후에 발행하도록 하면 되죠.

Cache Evict과 비슷한 방식으로, Database Commit 이후에 처리되도록 `TransactionSynchronizationManager`를 활용했어요. 이때, Cache Evict과 Kafka Event Produce가 Commit 이후에 발생하도록 되어있으므로, 반드시 Cache Evict 이후에 Kafka Event Produce가 되도록 `@Order`와 `TransactionSynchronization` Interface의 `getOrder`메서드를 Override 하여 순서를 지정하여 처리 순서를 보장되도록 했어요. 자, 이제 그러면 아래와 같이 처리되므로 문제가 해결될거예요.

\[문제 해결 후 과정\]

* A Thread - 약관 동의 발생
* A Thread - `TermsAgreement` 값이 변경되어서, ApplicationEvent 발행
* A Thread - `TermsAgreement`가 DB에 Commit 됨
* A Thread - `AFTER_COMMIT` 설정이 되어있어, 이때 Cache Evict 됨
* A Thread - 약관 동의 여부 변경 Kafka Event Produce
* B Thread - 약관 동의 여부 변경 Kafka Event Consume
* B Thread - 약관 동의 여부 조회시 캐시로부터 변경 후 Entity Version이 조회 됨

### 그래도 Commit 이후 Cache Evict 처리 되기전에 조회하면…?

Kafka Event가 아니더라도, 발생할 확률은 엄청나게 낮지만, 여전히 Cache Evict 되기 전에 사용자의 요청이 들어온다면, 발생할 수 있어요. 이 경우는 코드로 어떻게든 해결할 수 있지만, 정책으로 해결했어요. “정책??” 이라는 생각이 잠깐 드실 거예요. 다시 처음으로 가보면, 우리는 아래와 같은 목표를 가졌어요.

> 약관 동의 여부는 값이 DB에 Commit 되는 순간, 바로 다음 요청에 DB에 저장된 값이 응답되어야 한다.

중요한 포인트는 “바로 다음 요청에 DB에 저장된 값이 응답되어야 한다” 입니다. `@TransactionalEventListener` 로 전달된 이벤트가 모두 처리되기 전에는 아직 API 응답을 하기 전 상태예요. 즉, 약관 동의 또는 철회 요청이 아직 완벽히 처리가 되기 전인 상태입니다. 그래서, Cache Evict 처리 되기 전은 아직 API가 처리 중인 상태인거죠! 그래서, 약관 서버는 아래와 같은 정책을 최종적으로 가지게 됩니다.

> 약관 동의 여부는 약관 동의 또는 철회 요청 API 처리가 완료된 순간, 바로 다음 요청에 DB에 저장된 값이 응답되어야 한다.

이러면, 제가 생각했던 문제가 더 이상 문제가 아니게 되었고, 코드를 작성하는 저와 제품을 담당하는 PO 모두 행복하게 약관을 서비스할 수 있게 되었어요!

## 캐시를 적용하면서 느낀점

### Execution Over Perfection (완벽해지려 하기보다 실행에 집중하라)

토스 커뮤니티의 여러 Core Value 중 하나인데요. 사실 매우 중요한 시스템 중 하나이기 때문에, 캐시를 적용하기 전에 완벽해지기 위해 Legal, Compliance 등 다양한 부분들을 먼저 검토하고 어떤 문제가 발생할지에 대해 다양한 방면을 고민하느라 캐시를 적용하는데 많은 시간이 들었던 것 같아요. 만약 Legal, Compliance 검토 이후 정책을 바로 세우고 캐시를 적용하여 모니터링해가며 만났던 문제들을 하나씩 해결해 나간다면 좀 더 빠르게 적용할 수 있었지 않을까 생각이 들었어요.

### 코드로 해결할 수 있지만, 정책을 조금만 틀면 더 쉽게 해결할 수 있다

프로그래머는 모든 것을 코드로 해결하려고 하면 안된다고 개인적으로 생각해요. 다양한 문제들이 존재할 때, 코드보다 더 쉬운 방법이 어딘가에는 존재할거에요. 이번에 말씀드린 문제를 해결한 과정은 정말 짧아 보이지만, 그 전에 수 많은 코드를 작성하고, 지웠고, 방법들을 고민했었어요. 결국 정책을 수정하여 코드보다 간단히 해결할 수 있었어요.

### 마무리

지금은 DB에 큰 부하 없이, 최대 2만 TPS 트래픽을 든든하게 버텨주는 서비스가 되었어요. 최대 2만 TPS의 트래픽은 사실, 한 번 사용자의 요청에 MSA 서비스들이 여러번 호출해서 수치가 높게 보이는 부분도 있는데요? 이런 불필요한 트래픽을 줄이기 위해, Netflix의 Passport와 비슷한 개념인, TermsPassport 와 같은 기술도 고민하고 있어요. 아직은 트래픽이 그렇게 높지 않아 적용을 하지 않지만, 만약 적용한다면 추후에 다시 글로 찾아뵐께요! 

---

## 📋 **추가 정보**  
🔹 **게시 날짜:** Mon, 31 Mar 2025 02:32:00 GMT  
🔹 **출처:** [원본 링크](https://toss.tech/article/34481)  
🔹 **관련 태그:** #RSS #자동화 #n8n  

---

> ✨ _이 문서는 자동 생성되었습니다. 🚀 Powered by n8n_
