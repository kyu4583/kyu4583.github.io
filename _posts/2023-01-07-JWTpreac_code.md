---
layout: single
title:  "JWT 스프링 시큐리티로 구현"

---

## 스프링 시큐리티 JWT 구현

### build.gradle 의존성 추가

```java
	// Spring Security
	implementation 'org.springframework.boot:spring-boot-starter-security'

	// JWT
	implementation 'io.jsonwebtoken:jjwt-api:0.11.5'
	implementation 'io.jsonwebtoken:jjwt-impl:0.11.5'
	implementation 'io.jsonwebtoken:jjwt-jackson:0.11.5'
	implementation 'org.projectlombok:lombok:1.18.26'
```



### JwtToken DTO

```java
package JWTpractice.JWTpracticespring.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;

@Builder
@Data
@AllArgsConstructor
public class JwtToken {
    private String grantType;
    private String accessToken;
    private String refreshToken;
}
```

여기서 grantType는 JWT에 대한 인증 타입이다. 그리고 액세스 토큰과 리프레시 토큰을 가진다.



### 암호 키 설정

```yml
jwt:
  secret: 05c93cbc6a7c59d6982ba7a38986071d5252e308cef93f7bd61daf8c19ba5035
```

application.yml에서 위와 같이 암호 키를 등록해준다. 키는 토큰 암호화/복호화에 사용되며, 랜덤한 32자 16진수 값을 가진다.(HS256을 사용하기 위함)



### JwtTokenProvider

기능:인증, 권한 부여 처리

주석으로 각 코드에 대한 설명을 붙여놨다.

```java
@Slf4j
@Component
public class JwtTokenProvider {
    private final Key key;

    // application.yml에서 secret 값 가져와서 key에 저장
    public JwtTokenProvider(@Value("${jwt.secret}"/*yml 파일 내 설정값*/) String secretKey) {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);    //16진수 시크릿키를 바이트 배열로 변환
        this.key = Keys.hmacShaKeyFor(keyBytes);    //바이트 배열로 HMAC SHA 키 생성
    }

    // Member 정보를 가지고 AccessToken, RefreshToken을 생성하는 메서드
    public JwtToken generateToken(Authentication authentication) {
        // 권한 가져오기
        String authorities = authentication.getAuthorities()    // 권한 목록 가져오기
                .stream()
                .map(GrantedAuthority::getAuthority)  // 권한을 각각 문자열로 변환
                                                    // GrantedAuthority(인터페이스) 객체를 받아 getAuthority 메서드 실행 결과를 사용
                .collect(Collectors.joining(","));  // 콤마를 껴서 문자열들을 하나의 문자열로 변환
                //사용자의 모든 권한이 쉼표로 구분된 하나의 문자열로 변환됨
```

프로바이더 인스턴스 생성 시 암호화/복호화에 쓸 키를 생성한다.

사용자 권한 객체를 받아, 객체 목록을 쉼표로 구분된 문자열 형태로 저장한다.

```java
        long now = (new Date()).getTime();  // 현재시간, 토큰 만료 시간 결정용

        // Access Token 생성
        Date accessTokenExpiresIn = new Date(now + 8640000);    // 2.4시간 후 만료
        String accessToken = Jwts.builder()   // Jwt 토큰 생성
                .setSubject(authentication.getName())   // 토큰 제목 설정(사용자 이름)
                .claim("auth", authorities)     // 사용자 권한 목록을 포함하는 커스텀 클레임 추가
                .setExpiration(accessTokenExpiresIn)    // 토큰 만료 시간 설정
                .signWith(key, SignatureAlgorithm.HS256)    // 토큰 암호화 알고리즘, 키 설정
                .compact(); // 토큰 생성(문자열 형태)

        // Refresh Token 생성
        String refreshToken = Jwts.builder()    // Jwt 토큰 생성
                .setExpiration(new Date(now + 86400000))    // 24시간 후 만ㄹ
                .signWith(key, SignatureAlgorithm.HS256)    // 토큰 암호화 알고리즘, 키 설정
                .compact(); // 토큰 생성(문자열 형태)

        return JwtToken.builder()   // Jwt"Token" 객체 생성
                .grantType("Bearer")    // 'Baerer' : '소지자', 이 토큰을 소지하면 권한을 가진 것과 같다.
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .build();
    }
```

액세스 토큰과 리프레시 토큰을 생성한다.

액세스 토큰에는 제목과 권한 목록을 설정하지만, 리프레시 토큰은 그러지 않는다.

```java
    // Jwt 토큰을 복호화하여 토큰에 들어있는 정보를 꺼내는 메서드
    public Authentication getAuthentication(String accessToken) {
        // Jwt 토큰 복호화
        Claims claims = parseClaims(accessToken);

        if (claims.get("auth") == null) {
            throw new RuntimeException("권한 정보가 없는 토큰입니다.");
        }

        // 클레임에서 권한 정보 가져오기
        Collection<? extends GrantedAuthority> authorities = Arrays.stream(claims.get("auth").toString().split(","))
                .map(SimpleGrantedAuthority::new)   // SimpleGrantedAuthority 클래스의 생성자를 참조해서 문자열을 객체로 변환
                .collect(Collectors.toList());  // 권한 목록을 리스트로 변환

        // UserDetails 객체를 만들어서 Authentication return
        // UserDetails: interface, User: UserDetails를 구현한 class
        UserDetails principal = new User(claims.getSubject()/*식별자(ex.이름)*/, "", authorities);   // 유저 식별자, 권한목록을 담은 유저 정보 객체
        return new UsernamePasswordAuthenticationToken(principal, "", authorities); // 유저 정보, 권한목록을 담은 객체 리턴
    }

```

복호화를 진행하고, 최종적으로 유저 정보, 권한 목록을 가진 *Authentication* 객체를 반환함.

```java
    // 토큰 정보를 검증하는 메서드
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                    .setSigningKey(key) // 검증에 쓸 키 설정
                    .build()
                    .parseClaimsJws(token); // 파스 및 검증
            return true;
        } catch (SecurityException | MalformedJwtException e) {
            log.info("Invalid JWT Token", e);
        } catch (ExpiredJwtException e) {
            log.info("Expired JWT Token", e);
        } catch (UnsupportedJwtException e) {
            log.info("Unsupported JWT Token", e);
        } catch (IllegalArgumentException e) {
            log.info("JWT claims string is empty.", e);
        }
        return false;
    }


    // accessToken 파싱
    private Claims parseClaims(String accessToken) {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(key)
                    .build()
                    .parseClaimsJws(accessToken)
                    .getBody(); // claims 객체 반환, claims: 토큰에 담긴 정보 조각들
        } catch (ExpiredJwtException e) {
            return e.getClaims();
        }
    }

}
```

토큰 유효성 검증, 액세스 토큰 -> Claims 파싱 메서드



### JwtAuthenticationFilter

기능: 요청에서 토큰 정보를 가져와서 `SecurityContext`에 저장

```java
@RequiredArgsConstructor    // final 필드를 인자값으로 하는 생성자를 자동으로 생성해줌. 토큰 프로바이더가 알아서 주입됨.
public class JwtAuthenticationFilter extends GenericFilterBean {
    private final JwtTokenProvider jwtTokenProvider;    // 자동 생성된 생성자로 알아서 주입


    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        // 1. Request Header에서 JWT 토큰 추출
        String token = resolveToken((HttpServletRequest) request);

        // 2. validateToken으로 토큰 유효성 검사
        if (token != null && jwtTokenProvider.validateToken(token)) {
            // 토큰이 유효할 경우 토큰에서 Authentication 객체를 가지고 와서 SecurityContext에 저장
            Authentication authentication = jwtTokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        chain.doFilter(request, response);
    }

    // Request Header에서 토큰 정보 추출
    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

`resolveToken()`은 HTTP 요청의 `Authorization` 헤더에서 값을 읽고 토큰 부분을 취해온다. 일반적으로 JWT 토큰은 `Authorization` 헤더에 `"Bearer {토큰}"` 형식으로 포함된다.



### SecurityConfig

기능: 스프링 시큐리티의 설정 관리(HttpSecurity를 구성)

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    private final JwtTokenProvider jwtTokenProvider;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                // REST API이므로 basic auth 및 csrf 보안을 사용하지 않음
                .httpBasic((httpBasic) ->
                        httpBasic.disable()
                )
                .csrf((csrf) ->
                        csrf.disable()
                )
                // JWT를 사용하기 때문에 세션을 사용하지 않음
                .sessionManagement((sessionManagement) ->
                        sessionManagement.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                )
                .authorizeHttpRequests((authorize) ->
                        authorize
                                // 해당 API에 대해서는 모든 요청을 허가
                                .requestMatchers("/members/sign-in").permitAll()
                                // USER 권한이 있어야 요청할 수 있음
                                .requestMatchers("/members/test").hasRole("USER")
                                // 이 밖에 모든 요청에 대해서 인증을 필요로 한다는 설정
                                .anyRequest().authenticated()
                )
                // JWT 인증을 위하여 직접 구현한 필터를 UsernamePasswordAuthenticationFilter 전에 실행
                .addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider), UsernamePasswordAuthenticationFilter.class);

        return httpSecurity.build();
    }


    @Bean
    public PasswordEncoder passwordEncoder() {
        // BCrypt Encoder 사용
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }


}
```

`filterChain`을 메서드 체이닝 방식으로 구현한게 인터넷에 많았는데, **스프링 시큐리티 6.1부터는 그런 방식으로 쓰면 오류를 일으킨다.** 그래서 위와 같이 람다식을 이용해 함수 형식으로 구현해야 한다.



### Member

```java
@Entity //(Lombok)JPA 엔티티임을 나타냄
@Getter //(Lombok)필드의 getter 메서드를 자동으로 생성
@NoArgsConstructor(access = AccessLevel.PROTECTED)  //(Lombok)기본 생성자(파라미터 안받는)를 자동으로 추가. AccessLevel.PROTECTED로 했으니 외부에서는 생성자를 사용할 수 없음.
@AllArgsConstructor //(Lombok)모든 필드 값을 파라미터로 받는 생성자를 자동으로 생성
@Builder    //(Lombok)빌더 패턴 클래스를 자동으로 생성(선택적 매개변수를 가진 생성자를 만들어줌 등)
@EqualsAndHashCode(of = "id")   //equals 메서드와 hashCode 메서드를 자동으로 생성. of = "id"로 지정해주면 id 값만 같으면 같은 객체로 판단하도록 함.
public class Member implements UserDetails {
    @Id
    @GeneratedValue
    @Column(name = "member_id", unique = true, nullable = false)
    private Long id;    //PK, DB 식별용

    @Column(nullable = false)
    private String password;
    private String username;    //ID, 로그인 식별용
    private String nickname;    //닉네임, 후보키가 아님, 중복 가능

    public void setId(Long id) {
        this.id = id;
    }

    public void setName(String name) {
        this.username = name;
    }

    @ElementCollection(fetch = FetchType.EAGER) //얘가 있어서 List<String>을 필드로 가질 수 있음(JPA).
    @Builder.Default    //@Builder를 사용할 때(Lombok의 빌더를 쓸 때) 기본값을 설정해줌.
    private List<String> roles = new ArrayList<>();
    /*
    이 클래스 내에서 rolse를 채우는 코드는 없지만, 외부에서 Lombok 빌더를 사용할 때 채워진다고 가정함.
    예) Member member = Member.builder()
                      .username("user")
                      .password("password")
                      .roles(Arrays.asList("ROLE_USER", "ROLE_ADMIN"))
                      .build();
     */
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.roles.stream()
                .map(SimpleGrantedAuthority::new) /*스트림의 각 요소(권한을 나타내는 문자열)를 SimpleGrantedAuthority 객체로 변환
                                                    SimpleGrantedAuthority가 문자열을 통해 생성되는 생성자를 가졌기에 가능한 코드*/
                .collect(Collectors.toList()); //변환된 객체들을 리스트로 변환
    }


    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

클래스 상단에 줄줄이 붙인 어노테이션은 대부분 Lombok의 기능을 사용하기 위함임. 

`roles`와 `getAuthorities`가 기능적으로 핵심인 부분이고, 주석을 통해 자세히 설명함.



### MemberRepository

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    //JPA 리포지토리에서 기본적으로 구현되는 다양한 메서드들이 뒤로 존재함.
    Optional<Member> findByUsername(String username); // 얘는 직접 선언까진 해줘야함. 필드 이름까진 미리 모르니까. 기능은 유추해서 짜줌.
    List<Member> findByNickname(String nickname);
}
```

스프링 데이터 JPA를 이용한 리포지토리.

내가 이름지은 필드들에 대해서만 메서드를 선언까지 해줌. 기능 구현은 이름을 통해 유추돼서 자동 구현됨. 그 외의 메서드들은 이미 뒤로 자동 구현 되어있음.



