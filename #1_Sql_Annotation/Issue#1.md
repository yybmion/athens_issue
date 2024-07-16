## 테스트 코드에서의 @Sql이 의미하는 바

```markdown
📌 **As-is**
테스트코드(통합 테스트)를 작성하며 `@Sql` 어노테이션을 통해 값을 setting을 해야하는 상황이 있다.
해당 과정에서 겪은 여러가지 Issue들을 정리하려한다.

📌 **To-be**
`@Sql` 어노테이션을 통해 통합테스트를 수행하기 위한 사전 값들을 쉽게 setting할 수 있다.

📌 **기대 효과**
테스트 코드에 `@Sql`을 사용할 시 기대효과는 다음과 같다.

1. **테스트 데이터 설정**:
**테스트를 실행하기 전에 데이터베이스에 특정 데이터를 삽입**하거나 테이블 구조를 설정할 수 있다.
이를 통해 테스트에 필요한 초기 상태를 만들 수 있다.
   
2. **독립적인 테스트 환경**:
각 테스트마다 **독립적인 데이터 환경**을 구성할 수 있다.
이는 **테스트 간 데이터 의존성을 제거**하고 격리된 테스트를 가능하다.

📌**Todo**
 Job1: `@Sql`을 통해 통합테스트 로직 이해와 테스트 통과
```

### 문제상황 #1
- 아고라 통합테스트를 진행중에 있어 다음과 같은 에러가 반복해서 발생하였다.

```text
jakarta.servlet.servletexception: request processing failed: java.lang.nullpointerexception: **temporal**
```

- 먼저 문제가 되는 코드를 살펴보자

```java
@DisplayName("아고라 투표 API 통합 테스트")
public class AgoraVoteAuthApiIntegrationTest extends IntegrationTestSupport {

    @Test
    @DisplayName("투표 테스트")
    @Sql("/sql/enter-agora-members.sql")
    @Sql("/sql/agora-status-closed.sql")
    @WithMockCustomUser
    void vote() throws Exception {
        //given
        AgoraVoteRequest request = new AgoraVoteRequest(AgoraVoteType.PROS, true);

        objectMapper = new ObjectMapper();
        String jsonContent = objectMapper.writeValueAsString(request);

        //when
        final ResultActions result = mockMvc.perform(
                patch("/{prefix}/agoras/{agoraId}/vote", API_V1_AUTH, 1)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(jsonContent)
        );

        //then
        result.andExpect(status().isOk())
                .andExpectAll(
                        jsonPath("$.success").value(true),
                        jsonPath("$.response").exists(),
                        jsonPath("$.response.id").value(1),
                        jsonPath("$.response.voteType").value("PROS"),
                        jsonPath("$.error").doesNotExist()
                );
    }
}
```

- 해당 테스트 코드의 동작과정을 간략히 설명하자면 다음과 같다.
- `@Sql` 어노테이션을 통해 **초기 셋팅값을 설정**해주고
- `@WithMockCustomer`을 통해 **사용자 인증이 필요한 Api에 대해 인증작업**을 실행시켜준다.
- 해당 `Patch Request`는 AgoraVotRequest라는 `dto`를 전달하기 때문에 미리 객체를 만들어주고 JSON으로 변환시켜주었다.
- 그리고 `content`로 dto 요청도 보내고 `Pathvariable`또한 요청으로 보내고 나면
- `andExpect`로 값을 제대로 응답했는지 확인한다.

❗ 이러한 과정에서 나는 몇번이고 **NullpointerException:temporal**이라는 에러를 마주했다.

### 문제 해결 #1

`temporal` 이는 분명 시간과 관련된 필드에서 null값이 들어갔다는 의미이다.

- 처음에는 테스트 코드가 익숙치 않아 다른 이상한 곳에서 로그를 찍어보며 수정을 했었다.
- 하지만 처음부터 차근차근 다시 상황을 되짚어보니 service부분의 `agora.checkVoteTime()` 부분이 제대로 출력되질 않는다는 것을 깨달았다.
- `agora.checkVoteTime()` 코드를 보면 다음과 같다.

```java
public void checkVoteTime() {
        LocalDateTime now = LocalDateTime.now();
        if (Duration.between(now, this.endTime).getSeconds() > 20) {
            throw new VoteTimeOutException();
        }
    }
```
- 다음 코드를 보면 this.endTime으로 `endTime` 필드를 가져와야하는 코드가 존재했다.
- 따라서 이 값을 미리 `@Sql`에서 설정해주면 문제가 해결될 것이라 보았다.
- 처음에는 `@Sql`을 테스트 코드에서 왜 사용하고 어떤식으로 사용할지 감이 잘 잡히지 않았는데 직접 문제 상황을 겪어보니 이해했다.

내가 이후에 수정한 `@Sql문`은 다음과 같다.

```sql
UPDATE agora
SET status   = 'CLOSED',
    end_time = CURRENT_TIMESTAMP
WHERE agora_id = 1;
```

- end_time을 **현재 시간**으로 지정해주는 `CURRENT_TIMESTAMP`을 사용했다.
- 이후 오류는 사라지고 테스트에 통과하였다.

![img](https://github.com/user-attachments/assets/c3b4c879-593e-4dcd-bee3-f5ffb99bb3f9)






