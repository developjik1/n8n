# 🎯 Cursor와 TDD로 만드는 Swift Macro

> 🕒 **발행일:** Thu, 17 Apr 2025 06:13:12 GMT  
> 🔗 **원본 링크:** [👉 바로 가기](https://medium.com/daangn/cursor%EC%99%80-tdd%EB%A1%9C-%EB%A7%8C%EB%93%9C%EB%8A%94-swift-macro-0e4a245caee2?source=rss----4505f82a2dbd---4)

---

## 📌 **핵심 요약**  
📖 안녕하세요. 모바일실 iOS팀에서 iOS Engineer로 일하고 있는 Elon이에요.

제가 속한 모바일실은 당근에 전사적으로 필요한 기능을 개발해요. CI/CD, 애널리틱스, 실험 플랫폼, 딥링크 시스템 등을 직접 개발 및 관리하며, 앱 개발에 필요한 플랫폼 엔지니어링을 담당하고 있어요. 이러한 플랫폼 엔지니어링에는 iOS 엔지니어분들의 개발 생산성을 높이기 위한 Swift Macro를 개발하는 것도 포함되는데요.

이 글에서는 Curosr와 함께 TDD를 통해 Swift Macro를 구현해 보면서, 실제 프로덕션에 적용할 수 있는 신뢰도 높은 코드를 작성하는 방법에 대해 이야기하려고 해요.

### TDD에 들어가기에 앞서

당근 앱의 토대를 쌓아 올리셨던 iOS 엔지니어분들은 당근 초기부터 모두 TDD(Test Driven Development)를 진행했어요. 그래서 단순 View를 제외하면 현재까지도 iOS 당근 앱에선 테스트 코드가 없는 곳을 찾아보기 어려워요. 기존 개발 환경이 이러하다 보니 iOS 챕터에 합류한 엔지니어분들도 모두 테스트 코드를 작성하는 걸 당연하게 생각해요. 테스트 코드가 없으면 Pull Reqeust를 올리지 않을 정도예요. 전 당근에 입사하기 전까지는 iOS 앱 개발에서는 테스트 코드를 작성해 본 경험이 없었는데요. 그랬던 저도 지금은 테스트 코드를 수월하게 작성해요. Swift Macro와 같이 처음 접하는 기술을 도입할 때도 자연스럽게 TDD로 시작해보고 있죠.

Swift Macro는 Swift로 작성된 코드를 SwiftSyntax로 파싱한 후, Swift Syntax Tree에서 원하는 코드를 탐색해 가져와 사용하는 방식이에요. 그래서 매크로를 만들기 위해서는 먼저 원하는 형태의 매크로 인터페이스와 매크로 적용 후 생성될 코드 형태를 미리 설계해야 하는데요. 테스트 작성 시 일반적으로 사용하는 패턴인 _Given-When-Then_ 패턴에서 매크로 인터페이스는 _Given,_ 매크로가 적용되어 생성된 코드는 _Then_에 해당돼요. 따라서 자연스럽게 테스트코드를 먼저 작성하고 구현을 개발하는 TDD 방식에 적합하다고 볼 수 있어요.

TDD는 아래와 같은 세 단계를 하나의 사이클로 반복하면서 기능을 완성해 나가요. Swift Macro의 경우, 일반적으로 iOS 앱 개발에서 접할 일이 거의 없는 SwiftSyntax API를 사용해야 해요. 그래서 SwiftSyntax 레퍼런스 문서와 Swift AST Explorer 사이트를 오가며 필요한 Syntax를 찾는데 많은 시간을 소비하게 되는데, 이 과정을 Cursor와 함께 한다면 많은 시간을 절약할 수 있어요.

* 🔴 Red: 원하는 기능의 테스트 코드를 작성하고 테스트를 실행하여 실패하는 것을 확인해요.
* 🟢 Green: 테스트를 통과할 수 있도록 최대한 빠르게 동작할 수 있는 코드를 구현해요.
* 🟡 Refactor: 빠르게 작성한 코드에서 중복된 코드 등을 제거하고 유지보수하기 쉽도록 가독성 좋게 리팩토링해요.

이번 글에서는 예시로 Swift에서 JSON 파싱을 위해 Codable의 CodingKeys 를 자동으로 추가해 주는 Swift 매크로를 만들어 볼게요. 코드 작성은 Cursor에서 진행하고, 컴파일과 테스트 코드 실행은 Xcode를 사용해요.

💡 아래 예시는 Cursor를 사용하여 TDD를 하는 것을 목적으로 하기 때문에 Swift Macro 패키지를 만들기 위한 모든 내용을 다루지 않아요.

* 전체 코드들은 [예제 레포지토리](https://github.com/ElonPark/Make-SwiftMacro-using-Cousor-Example)에서 보실 수 있어요.

### 🔴 Red

1. 먼저 매크로의 인터페이스를 설계하여 사용될 예시 코드를 정의해요.

@CustomCodable  
struct Person {  
  let name: String  
  @CodableKey("user_age") let age: Int  
}

2\. 실제 매크로가 적용되었을 때 추가될 코드의 모습을 정의해요.

struct Person {  
  let name: String  
  let age: Int  
  
  enum CodingKeys: String, CodingKey {  
    case name  
    case age = "user_age"  
  }  
}

3\. 해당 코드들을 매크로 패키지에 테스트코드로 정의해요.

func testExpansionWithCodableKeyAddsCustomCodingKeys() {  
  assertMacroExpansion(  
    """  
    @CustomCodable  
    struct Person {  
      let name: String  
      @CodableKey("user_age") let age: Int  
    }  
    """,  
    expandedSource: """  
      struct Person {  
        let name: String  
        @CodableKey("user_age") let age: Int  
  
        enum CodingKeys: String, CodingKey {  
          case name  
          case age = "user_age"  
        }  
      }  
      """,  
    macros: macros,  
    indentationWidth: .spaces(2)  
  )  
}

4\. 테스트를 실행하여 테스트에 통과하지 못하고 실패하는 것을 확인해요.

![](https://cdn-images-1.medium.com/max/1024/1*Je28RFdhR2h61aOX2-Ae3g.png)

### 🟢 Green

테스트에 실패한 것을 확인했으니 동작하는 코드를 구현할 차례예요.

1\. 작성한 테스트 코드를 포함한 프롬프트를 Cursor Composer를 통해 LLM에게 전달하여 실제 구현 코드를 생성해요.

> Tip: SwiftSyntax 버전에 따라 사용할 수 있는 API가 달라요. 따라서 프롬프트 Swift 버전을 명시하여 Swift 버전에 맞는 SwiftSyntax 코드를 생성할 수 있도록 해요.

아래와 같은 테스트 코드를 통과하는 Swift Macro를 작성해 줘  
Swift 버전은 5.10을 사용하고 swift-syntax에 맞게 구현하여야 해  
Tuple이 필요하다면 struct 또는 enum을 새로 정의하여 사용해 줘  
  
```swift  
/* 작성한 테스트 코드 */  
```

2\. 생성된 코드를 리뷰하고 문제가 있는 부분을 수정 요청해요. 코드를 커밋해도 될 만큼 코드가 개선되었다고 판단되면, 해당 코드들을 수락하여 패키지에 반영해요.

3\. Xcode에서 패키지를 열고 테스트를 실행해요. LLM이 생성한 코드가 한 번에 테스트를 통과하면 좋겠지만, 대부분은 컴파일 단계에서 실패하는 코드가 생성돼요.

* 여기서는 컴파일러 경고와 에러가 하나씩 발생했네요.

![](https://cdn-images-1.medium.com/max/1024/1*71T5a3VQtOb5AXlkBiH_KQ.png)

* 컴파일러 경고 메시지는 Fix 버튼을 눌러서 간단하게 수정 가능하기 때문에 직접 수정해요.

![](https://cdn-images-1.medium.com/max/1024/1*7-Rse83urN4pswD5qqSiGQ.jpeg)

4\. 컴파일러 에러를 수정할 차례예요. 테스트가 통과할 수 있도록 에러를 수정해 달라고 요청해볼게요.

다음과 같은 에러와 경고가 발생하였어 테스트가 통과할 수 있게 수정해 줘  
- 에러 메시지: `CustomCodableMacros/CustomCodableMacro.swift:39:15 Initializer for conditional binding must have Optional type, not 'AttributeListSyntax'`

![](https://cdn-images-1.medium.com/max/1024/1*CRP8pY6Tetc5EeOyNwUesw.jpeg)

Cursor에서 실행한 결과

5\. 다시 Xcode로 돌아와서 테스트 코드를 실행해 봅니다. 이번에는 다행히 컴파일에 성공할 수 있도록 코드를 수정해 주었네요. 하지만 아쉽게도 아직 테스트는 통과하지 못했어요.

![](https://cdn-images-1.medium.com/max/1024/1*3H2fr4WKNF7wLsKdYmmR9g.jpeg)

6\. 다시 한번 에러 메시지를 전달하여 수정을 요청하여 볼게요.

![](https://cdn-images-1.medium.com/max/1024/1*O0RuJRA_thBWyx5ok46KrA.jpeg)

7\. 수정된 코드를 리뷰하여 문제가 있다면 다시 수정 요청하기를 반복해요. 수정된 코드들에 문제가 없다고 판단되면 적용하고, 다시 Xcode로 돌아와 테스트를 실행해 봅니다. 저는 이번에도 테스트가 실패했는데요.

오류 메시지를 확인해 보니 생성된 코드와 테스트에서 기대하는 코드의 인덴트가 다르기 때문에 테스트에 통과하지 못했네요.

> Tip: Swift Macro는 매크로가 생성한 코드와 테스트의 결괏값이 정확히 일치하는 경우에만 성공하기 때문에, 문법 상에 문제가 없는 경우에도 테스트에 실패할 수 있어요.

![](https://cdn-images-1.medium.com/max/1024/1*aWHkHSnnBWG3DH9wwjZvHA.jpeg)

8\. 단순히 인덴트만 수정하면 되기 때문에 이번에는 직접 수정해요.

![](https://cdn-images-1.medium.com/max/896/1*0uqU-NqZ3hIXBG2s49AJBw.jpeg)

9\. 다시 테스트를 실행시켜 테스트에 통과하는 것을 확인해요.

![](https://cdn-images-1.medium.com/max/1024/1*JxBPjJ8NFEBKix0OciNFaQ.jpeg)

### 🟡 Refactor

테스트에 통과하는 것을 확인했으니 이제 리팩토링을 진행할 차례예요.

> Tip: 리팩토링 요청 시에는 원하는 코드 스타일이나 메소드 추출 기법 등 리팩토링 방법들을 프롬프트에 구체적으로 명시하면 더욱 좋은 결과물을 얻을 수 있어요.

1. 리팩토링 또한 LLM에게 요청하여 진행해 봅니다 😉

이제 @CustomCodableMacro.swift를 가독성 좋고 이해기 쉬운 단위로 메소드로 추출하여  
리팩토링해줘 필요하다면 파일을 여러 개로 분리해도 괜찮아 하지만 기존 테스트는 통과할 수 있도록  
구현에는 문제가 없어야 해

* Green 단계에서 진행한 것과 동일하게 수정된 코드를 리뷰해 보고 문제가 없다면 적용해요. 문제가 있다면 해당 내용을 지적하여 다시 리팩토링을 요청해요.

2\. 이제 리팩토링된 코드가 문제가 없는지 다시 한번 테스트를 실행해 봅니다.

* 리팩토링 과정에서 이전과 동일하게 인덴트가 스페이스 4칸으로 변경되어 테스트가 실패했네요.

![](https://cdn-images-1.medium.com/max/1024/1*MrtNB4tr8N6jDB30YIWSoA.jpeg)

3\. 이번에도 단순 인덴트 차이가 원인이기 때문에 직접 수정해요.

![](https://cdn-images-1.medium.com/max/1024/1*13m1ECHJEo2uG9EZx8HDzA.jpeg)

4\. 다시 테스트를 실행하여 테스트가 통과하는 것을 확인해요.

![](https://cdn-images-1.medium.com/max/1024/1*3rYyaWL7IfZz-kXArvBG-w.jpeg)

5\. 필요에 따라 다시 리팩토링을 진행하거나 작업을 마무리해요.

> Tip: LLM은 Context 크기에 제한이 있기 때문에 한 번에 여러 작업을 진행하기보다는 하나씩 작업을 진행하고, 많은 양의 코드를 LLM에 전달하여야 하는 큰 작업이 있다면 여러 개의 작은 단위로 쪼개서 작업하는 것이 좋아요.

### TDD에 LLM을 활용한다면?

이것으로 TDD의 3가지 스텝을 차례대로 진행해 보면서 TDD의 한 사이클을 돌아보았는데요. 이 과정에서 LLM을 사용하여 TDD를 진행했을 때의 두 가지 장점을 찾을 수 있었어요.

첫 번째 장점은 많은 시간을 절약할 수 있다는 점이에요. Swift Macro를 만들기 위해서는 Green 단계에서 Swift Syntax Tree를 탐색하고 필요한 Syntax를 가져오는 데 시간을 소모해야 하는데요. 이 작업을 LLM이 대신 수행하므로 많은 시간을 절약할 수 있었어요.

두 번째 장점은 코드의 문제점을 빠르게 파악하고 수정할 수 있다는 점이에요. 테스트를 먼저 작성한 후, 테스트를 통해 LLM이 생성한 코드를 검증하기 때문인데요. 테스트가 통과한 이후 리팩토링을 진행할 때에도 테스트를 통해 문제를 감지하고 바르게 수정할 수 있어요.

다만, 주의점은 LLM을 활용해 TDD를 하더라도, LLM이 생성한 코드를 리뷰하고 오류를 수정하는 것은 엔지니어의 역할이에요. 따라서 SwiftSyntax와 같이 구현에 사용되는 기술에 대한 지식을 반드시 갖추고 진행하여야 해요.

### 나가며

ChatGPT 등장 이후 LLM 기반의 생성형 AI가 빠르게 발전해 나가면서 세상을 바꾸고 있는데요. 당근에서는 LLM을 활용하여 사용자 경험이나 동료의 생산성을 높이는 도구를 함께 만들어 갈 iOS 엔지니어를 찾고 있어요. 저희의 여정에 함께하고 싶으시다면 아래 공고를 통해 지원하실 수 있어요! 😃

[Software Engineer, iOS | 당근 팀 채용](https://team.daangn.com/jobs/5282170003/)

![](https://medium.com/_/stat?event=post.clientViewed&referrerSource=full_rss&postId=0e4a245caee2)

---

[Cursor와 TDD로 만드는 Swift Macro](https://medium.com/daangn/cursor%EC%99%80-tdd%EB%A1%9C-%EB%A7%8C%EB%93%9C%EB%8A%94-swift-macro-0e4a245caee2) was originally published in [당근 테크 블로그](https://medium.com/daangn) on Medium, where people are continuing the conversation by highlighting and responding to this story.

---

## 📋 **추가 정보**  
🔹 **게시 날짜:** Thu, 17 Apr 2025 06:13:12 GMT  
🔹 **출처:** [원본 링크](https://medium.com/daangn/cursor%EC%99%80-tdd%EB%A1%9C-%EB%A7%8C%EB%93%9C%EB%8A%94-swift-macro-0e4a245caee2?source=rss----4505f82a2dbd---4)  
🔹 **관련 태그:** #RSS #자동화 #n8n  

---

> ✨ _이 문서는 자동 생성되었습니다. 🚀 Powered by n8n_
