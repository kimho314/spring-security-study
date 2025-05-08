# 8장 권한 부여 구성: 제한 적용

## enpoint에 인가 적용하는 법

- Spring Security에서 endpoint에 권한마다 인가를 적용하고 싶으면 requestMathers를 사용하면 된다.
- /enpoint uri에 "USER"라는 권한을 가진 유저만 접근할 수 잇도록 하려면 아래와 같이 설정하면 된다.

```java
@Bean
public SecurityFilterChain web(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests((authorize) -> authorize
	    .requestMatchers("/endpoint").hasAuthority("USER")
            .anyRequest().authenticated()
        )
        // ...

    return http.build();
}
```

- 유저에 Authority가 아니라 Role를 사용하여 권한을 제어하려먼 hasAuthority() 대신 hasRole()을 사용하면 된다.
- "USER" 권한을 가지지 못한 유저가 "/endpoint"로 접근하면 http status == 403(forbidden) 을 응답한다.
- .anyRequest().authenticated() 설정없이 새로운 api가 추가되면 해당 api에 대한 접근은 기본적으로 불가능하다.
- 실무에서 모호함이 없어야 하므로 .anyRequest() 설정을 꼭 하는게 좋다.
- .anyRequest()에서는 permitAll(), denyAll(), authenticated()를 사용할 수 있고, 각각의 의미는 "모두 허용", "모두 거부", "인증된 사용자만 허용" 이다.

### Spring Security 6 이상에서 authorization 방법의 변화

- 스터디에서 사용하는 Spring Security in Action의 경우 Spring Security 5.x 버전을 사용하고 있다. 해당 버전에서는 세 자기 유형의 mathers가 있는데, mvcMatchers(), antMatchers(), regexMatchers()이다.
- 하지만 Spring Security 6 이상부터는 requestMatchers()로 통합되었다.
- requestMatchers()는 Spring MVC를 사용하면 MvcRequestMatcher 를 적용하고, Spring MVC가 없으면 AntPathRequestMatcher 를 사용하도록 동작한다.
- Spring Security 5.x 버전에서 6.x버전으로 업그레이드 하는 경우 해당 사항을 유의해서 적용해야 한다.

before Spring Security 6

```java
@Configuration
@EnableWebSecurity
@EnableWebMvc
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .mvcMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        return http.build();
    }

}
```

after Spring Security 6

```java
@Configuration
@EnableWebSecurity
@EnableWebMvc
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        return http.build();
    }

}
```

### uri와 Http Method에 따라 authorization 방법을 다르게 설정하는 방법

- 같은 uri에 대해 http method 따라서 다른 인가 설정을 하고 싶으면 `public C requestMatchers(HttpMethod method, String... patterns)` method를 사용하면 된다.

```java
@Bean
    @Order(2)
    public SecurityFilterChain configure2(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .httpBasic(Customizer.withDefaults())
            .securityMatcher("/a/**")
            .authorizeHttpRequests((auth) -> auth
                    .requestMatchers(HttpMethod.POST, "/a")
                    .permitAll() // POST /a 호출은 모든 유저 접근 허용
                    .requestMatchers(HttpMethod.GET, "/a")
                    .authenticated() // GET /a 호출은 인증 통과한 유저만 접근 허용
                    .anyRequest()
                    .denyAll()
            )
        ;

        return http.build();
    }
```

- 위 코드를 보면 `/a` endpoint에 대해서 http method GET은 인증된 유저만 접근 허용하고, http method POST는 모든 유저에 대해서 접근 허용한 것이다.
- `/a/b`, `/a/b/c` endpoint들에 대해서 모두 접근 허용하고 싶으면 정규식을 requestMathers에 사용하면 된다.

```java
@Bean
    @Order(2)
    public SecurityFilterChain configure2(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .httpBasic(Customizer.withDefaults())
            .securityMatcher("/a/**")
            .authorizeHttpRequests((auth) -> auth
                    .requestMatchers("/a/b/**")
                    .authenticated() // /a/b 로 시작하는 모든 url은 인증 통과한 튜어만이 접근 허용
                    .anyRequest()
                    .denyAll()
            )
        ;

        return http.build();
    }
```
