👋 앞서 통합테스트 작성하는 도중에 하나의 의문이 들어 해당 글을 작성한다.

통합테스트란 본디 여러 구성 요소가 함께 작동하는 방식, 즉 시스템의 다양한 부분이 올바르게 상호작용하는지 검증하는 것이다.

나는 현재 토큰 재발급 api를 검증하기 위한 통합테스트를 작성중인데 코드는 다음과 같다.

## Issue 1

### 토큰 통합 테스트

```java
@DisplayName("토큰 통합 테스트")
public class TokenApiIntegrationTest extends IntegrationTestSupport {

    @MockBean
    private AuthService authService;

    @BeforeEach
    void setUp() {
        // Mockito Mock 객체 초기화
        MockitoAnnotations.openMocks(this);
    }

    @Test
    @DisplayName("토큰 재발급 테스트")
    void reissue() throws Exception {
        //given
        final String oldAccessToken = "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6IjIiLCJyb2xlIjoiUk9MRV9URU1QX1VTRVIiLCJpYXQiOjE3MjE3OTA4MDAsImV4cCI6MTcyMTg3NzIwMCwic3ViIjoicmVmcmVzaC10b2tlbiJ9.6HKNtahwgOXvjrWnyBvYUPSE3rgsxuQ4fNq3BVLIZfw";
        final String newAccessToken = "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6IjIiLCJyb2xlIjoiUk9MRV9URU1QX1VTRVIiLCJpYXQiOjE3MjE3OTA4MDAsImV4cCI6MTcyMTc5MjAwMCwic3ViIjoiYWNjZXNzLXRva2VuIn0.kx6UtcOUhs42OekvwPZPixQZjQAMVTKJEOlzOI__6Tg";
        final String refreshToken = "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6IjIiLCJyb2xlIjoiUk9MRV9URU1QX1VTRVIiLCJpYXQiOjE3MjE3OTA2OTksImV4cCI6MTcyMTc5MTg5OSwic3ViIjoiYWNjZXNzLXRva2VuIn0.PiUaYpSs6d7xi7nv9ufPw-UmbrQSySTjLJ7DJMISD6Y";

        doReturn(newAccessToken).when(authService).reissueRefreshToken(any(HttpServletRequest.class), any(HttpServletResponse.class));

        final ResultActions result = mockMvc.perform(
                post("/{prefix}/reissue", API_V1_AUTH)
                        .contentType(MediaType.APPLICATION_JSON)
                        .header("Authorization", "Bearer " + refreshToken)
                        .cookie(new Cookie("Refresh-Token", oldAccessToken))
        );

        //then
        result.andExpect(status().isOk())
                .andExpectAll(
                        jsonPath("$.success").value(true),
                        jsonPath("$.response").value(newAccessToken),
                        jsonPath("$.error").doesNotExist()
                );
    }
}
```

보면 `AuthService`에서 `Mock`을 사용했다.
왜냐하면 나의 처음 생각은 해당 테스트의 반환값인 토큰이 `동적`이기 때문에 `Mokito`의 `doReturn`을 사용하여 **토큰 재발급의 결과값을 미리 정의**해주기 위해서다.

하지만 통합테스트는 여러 시스템의 상호작용을 보기 위한 작업이라 생각하는데 **그러면 미리 doReturn을 통해 결과값을 정해두면 상호작용을 하지 않아도 테스트가 통과할거라 생각**했다.

그래서 이후 AuthService를 `@Autowired`를 통해 **실제 객체**를 불러와 실행하려했지만 이건 **당연히 jsonPath를 통해 동적으로 결과값이 변하는 토큰값을 테스트할 수 없다.**

그래서 생각한것이 반환값이 동적으로 변하는 경우, jsonPath로 검증할 때 **직접적인 값을 비교하는 대신**, 특정 조건이나 패턴을 검증하는 방법을 사용할 수 있어서 `형식을 검증할까`, `값의 존재 여부만 검사할까` 고민을 많이 했다.

이 부분에 대해서는 추후 결론을 내려 작성을 해볼 생각이다.

___

## Issue 1 sol

먼저 **1번 문제**에 대한 나만의 테스트코드를 작성했다.

```java
@DisplayName("토큰 통합 테스트")
public class RefreshTokenApiIntegrationTest extends IntegrationTestSupport {

    @Autowired
    private JwtUtils jwtUtils;

    @Autowired
    AuthService authService;

    private SecretKey secretKey;

    @Test
    @DisplayName("토큰 재발급 테스트")
    void 성공_토큰재발급_유효한파라미터전달() throws Exception {
        //given
        String refreshToken = jwtUtils.createJwtToken(REFRESH_TOKEN, 10L, "ROLE_TEMP_USER");

        //when
        final ResultActions result = mockMvc.perform(
                post("/{prefix}/reissue", API_V1_AUTH)
                        .contentType(MediaType.APPLICATION_JSON)
                        .cookie(new Cookie("Refresh-Token", refreshToken))
        );

        //then
        result.andExpect(status().isOk())
                .andExpectAll(
                        jsonPath("$.success").value(true),
                        jsonPath("$.response").value(Matchers.matchesRegex("^[\\w-]+\\.[\\w-]+\\.[\\w-]+$")),
                        jsonPath("$.error").doesNotExist()
                );
    }
```

실제객체를 `@AutoWired`를 하여 의존성을 주입하였고, 임의의 **refreshToken을 생성**하였다.

이후에 `mockMvc`로 api를 호출하므로 **token은 동적으로 생성**될 것이다.
그러므로 우리는 우리가 생성한 `refreshToken`을 가지고 `accessToken`이 잘 생성되는 것만 확인하면 되므로 **정확한 값이 아닌
accessToken의 형식을 확인**하면 될것이라 생각했다.

따라서
```java
 jsonPath("$.response").value(Matchers.matchesRegex("^[\\w-]+\\.[\\w-]+\\.[\\w-]+$"))
 ```

다음과 같이 **accessToken의 형식을 검사**하였다.