---
title: 객체 검증 어노테이션 비교 @Valid, @Validated
description:
date: 2024-06-05 +0900
categories: [Workspace]
tags: [valid, validated]
---

> Spring 에서 Request 에 대한 객체 검증시 @Valid, @Validated 어노테이션을 사용한다. 이번에 platform-api 에도 도입하면서 두 개의 차이점을 기억하기 위해 작성 
{: .prompt-info }
> 리플렉션을 통해 값을 가져오기때문에 @Getter 선언이 필수
{: .prompt-tip }

## **@Valid 어노테이션**
- `jakarta.validation` 패키지에서 구현
- JSR-303 표준에 대한 구현으로(자바 진영 스펙) 구현체는 Hibernate 
- Dispatcher 를 통해 전달받은 request 를 자바 객체로 변환하는 과정에서 객체에 대한 검증을 처리함
  - Dispatcher 의 동작 위치는 스프링 컨테이너가 시작하는 지점이기 때문에 Controller 에서만 동작(서블릿 컨테이너-> 스프링 컨테이너(*))
  - 자바 객체에 대한 검증처리이므로 PathVariable, RequestParam 은 해당X
- 예외 발생 시 MethodArgumentNotValidationException 발생

### **주의할 점**
RequestBody 내에 중첩 객체가 있다면 해당 객체에는 @Valid 를 사용해야 함
  ```java
  @Getter
  @NoArgumentsConstructor
  @AllAgrumentsConstructor
  public class UserRequest {
  
    @Email
    private String email;
  
    @NotBlank
    private String password;
      
    @Valid // working!
    @NotNull // not working!
    private Address address;
  }
  
  // other class
  @Getter
  @NoArgumentsConstructor
  @AllAgrumentsConstructor
  public class Address {
  
    @NotBlank
    private String city;
  
    @NotBlank
    private String zipCode;
  }
``` 
<br>

## **@Validated 어노테이션**
- `org.springframework.validation.annotation` 패키지에서 구현
- JSR-303 표준 기반의 구현체는 아니고 스프링 프레임워크에서 제공하는 기능
- AOP 기반으로 동작
  - Controller 이외의 영역이나 PathVariable, RequestParam 에 대한 검증이 가능
  - `self-invocation 주의`
- 제약 조건에 대한 그룹화 가능
  - 하나의 DTO 를 공유해서 사용할 경우에 유용!!
- 클래스에는 @Validated, 메소드에는 @Valid 를 붙여서 사용
- 예외 발생 시 ConstraintViolationException 발생

### **조건 그룹 지정**
- 1.그룹으로 사용할 인터페이스 생성
  - CardGroup
  - BankGroup
- 2.프로퍼티에 검증 그룹 추가
  ```java
  @Getter
  public class CommonRequest {
    @NotNull(groups = {CardGroup.class, BankGroup.class})
    Integer conditionId;
    
    @NotBlank(groups = {CardGroup.class, BankGroup.class})
    String clientName;
    
    @NotBlank(groups = {CardGroup.class, BankGroup.class})
    String clientNumber;
    
    @NotBlank(groups = {CardGroup.class})
    String cardId;
    
    @NotBlank(groups = {BankGroup.class})
    String bankId;
  }
  ```
- 3.Controller 의 @validated 에 그룹명 추가
  ```java
	@PostMapping("")
	public Mono<BaseApiResponse<List<CashItem>>> getAvailableItem (
    @Validated(CardGroup.class) CommonRequest commonRequest) {
		return itemService.makeItem(commonRequest)
			.collectList()
			.map(BaseApiResponse::new);
  }
  ```
  
### **주의할 점**
- 어노테이션의 위치 확인
  ```java
  @RestController
  @RequiredArgsConstructor
  @RequestMapping("/items")
  @Validated // AOP 사용
  public class ItemController {
  
    private final ItemService itemService;
  
    //메소드 내에 제약조건 추가
    @GetMapping("")
    public Mono<BaseApiResponse<List<CashItem>>> getAvailableItem(
      @NotNull @RequestParam YearMonth start_month,
      @Max(value = 100000000) @RequestParam int money
    ) {
      return itemService.getItems(start_month, money)
        .collectList()
        .map(BaseApiResponse::new);
    }
  ```
<br>

## **예외 처리**
- @valid, @validated 가 던지는 예외가 달라 각각 처리
  - error 파싱에 사용되는 메소드가 다르다
  ```java
  // from @Valid
  @ExceptionHandler(WebExchangeBindException.class)
	public ResponseEntity<BaseApiResponse<Set<String>>> handValidException(WebExchangeBindException e) {
		Set<String> erorrs = e.getBindingResult()
			.getFieldErrors()
			.stream()
			.map(
				fieldError -> fieldError.getField() + " : " + fieldError.getDefaultMessage()
			).collect(Collectors.toSet());
		log.error(e.toString());
		return new ResponseEntity<>(
			new BaseApiResponse<>(erorrs, false, "입력값을 확인해주세요"), HttpStatus.BAD_REQUEST);
	}
  //from @validated
	@ExceptionHandler(ConstraintViolationException.class)
	public ResponseEntity<BaseApiResponse<Set<String>>> handleValidatedException(ConstraintViolationException e) {
		Set<String> errors = e.getConstraintViolations()
			.stream()
			.map(
				error -> error.getPropertyPath().toString() + " : " + error.getMessage()
			).collect(Collectors.toSet());

		return new ResponseEntity<>(
			new BaseApiResponse<>(errors, false, "입력값을 확인해주세요"), HttpStatus.BAD_REQUEST);
	}
  ```

