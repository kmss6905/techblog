---
layout: post
title: "[Spring] @RestControllerAdvice에서 body 로깅으로 남기기"
description: "@RestControllerAdvice에서 예외 발생 시 HttpServletRequest의 body를 로깅하려 할 때 발생하는 문제를 다룹니다. InputStream이 한 번만 읽을 수 있는 이유를 설명하고, ContentCachingRequestWrapper와 커스텀 필터를 사용하여 이 문제를 해결하는 방법을 제시합니다."
categories:
- "Spring"
tags:
- "spring"
- "spring boot"
- "restcontrolleradvice"
- "filter"
- "contentcachingrequestwrapper"
- "logging"
- "exception handling"
date: "2024-05-11 00:00:00 +0900"
toc: true
---

# 2024.05.11 -  🌱 @RestControllerAdvice 에서 body 로깅으로 남기기.md

## 사건의 발단
현재 진행하고 있는 프로젝트에서는 @RestControllerAdvice 를 이용하여 예외를 전역으로 처리하고 있다.  
예외가 났을 때 어떤 요청값이 들어와서 예외가 발생했는 지 로그로 남기고 싶어서 request 의 body 를 읽고 싶었으나 아무런 값이 없는 문제가 발생했다.
```java

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(HotelkingException.class)
  public ResponseEntity<ApiResponse<ErrorContent>> handleHotelkingException(
      HotelkingException e,
      HttpServletRequest request
  ){
    ErrorCode ec = e.getErrorCode();
String body = StreamUtils.copyToString(request.getInputStream(), StandardCharsets.UTF_8);
log.error("body = {}", body); // 슬프게도 아무것도 안나온다.
    return ResponseEntity.status(ec.getHttpStatus()).body(ApiResponse.error(ErrorContent.from(ec)));
  }

  @ExceptionHandler(HttpMessageNotReadableException.class)
  public ResponseEntity<ApiResponse<ErrorContent>> handleHttpMessageNotReadableException(HttpMessageNotReadableException e, HttpServletRequest request) {
    ErrorCode ec = ErrorCode.NOT_READABLE;
    printLog(request, ec);
    return ResponseEntity
        .status(ErrorCode.NOT_READABLE.getHttpStatus())
        .body(ApiResponse.error(ErrorContent.from(ec)));
  }

}
```

HttpServletRequest 에서 다시 한번 InputStream 을 이용하여 body 를 읽으려고 했지만 아무것도 읽을 수 없었다.


## HttpServletRequest 에서 값을 읽을 때는 내부적으로 InputStream 을 사용한다.

> The spring-web module contains the HttpMessageConverter interface for reading and writing the body of HTTP requests and responses through InputStream and OutputStream. HttpMessageConverter instances are used on the client side (for concurrency, in the RestClient) and on the server side (for concurrency, in Spring MVC REST controllers).

![](https://velog.velcdn.com/images/kmss6905/post/bc215933-2cc6-43a4-9a7f-36dc13766dee/image.png)


Spring MVC 는 Servlet API 기반으로 만들어졌다. Spring MVC 에서는 Dispatcher Servlet 이 그 역할을 담당하고 있다.
Http 요청을 다룰 때 HttpServletRequest 에서 요청 body 를 읽을 때 `getInputStream()` 그리고 `getReader()` 메서드를 제공한다.
이러한 각 메소드는 동일한 InputStream 을 사용하기 때문에 InputStream을 한 번 읽으면 다시 읽을 수 없는 문제가 있다.


## ContentCachingRequestWrapper
`ContentCachingRequestWrapper` 는 생성자로 받은 HttpServeltRequest 를 input stream 과 reader 로 부터 모든 HttpServletRequest 컨텐츠를 캐시하는 HttpServlerRequest Wrapper 클래스이다.

캐시한 content 는 byte array 형태로 다시 얻을 수 있다.

얻을 때는 `getContentAsByteArray()` 를 통해 다시 얻을 수 있다.

중요한 건 요청 컨텐츠가 consumed 되지 않았다면, 컨텐츠는 캐시되지 않는다.

```java

public class ContentCachingRequestWrapper extends HttpServletRequestWrapper {

private final FastByteArrayOutputStream cachedContent;

public ContentCachingRequestWrapper(HttpServletRequest request) {
		super(request);
		
		// 캐시 하는 부분
		int contentLength = request.getContentLength();
		
		// content 가 있는 경우 캐시한다.
		this.cachedContent = (contentLength > 0) ? new FastByteArrayOutputStream(contentLength) : new FastByteArrayOutputStream();
		this.contentCacheLimit = null;
	}
	
	...
	
	// 캐시한 컨텐츠를 가져온다.
	public byte[] getContentAsByteArray() {
		return this.cachedContent.toByteArray();
	}
}
```

엄청 특이한 건 없다.

다만 `FastByteArrayOutputStream` 이라고 하는 OutputStream 을 extend 하여 새롭게 만들어 사용하고 있다. 문서에서는 ByteArrayOutputStream 의 대안으로 나왔다고 한다.

참고로 `AbstractRequestLoggingFilter` 에서도 위의 `ContentCachingRequestWrapper` 가 사용된다.

![](https://velog.velcdn.com/images/kmss6905/post/8eb4fafc-4f37-4855-9bb4-8a8bc838a0d5/image.png)

## 적용

Custom Filter 를 만들어 기존 HttpServletRequest 를 캐싱할 수 있도록 들어온 HttpServletRequest 를 이용하여 ContentCachingRequestWrapper 객체를 만든 후 doFilter를 호출합니다.

```java
@Component
public class CachingFilter extends OncePerRequestFilter {

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    filterChain.doFilter(new ContentCachingRequestWrapper(request), response);
  }
}
```

### 참고
* [REST Clients :: Spring Framework](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-message-conversion)