# Request

```
만약 당신이 REST 기반의 웹 서비스를 제공하고 있다면... 당신은 request.POST를 무시해야 한다.
- Malcom Tredinnick, 장고 개발자 그룹
```

DRF(Django Rest Framework)에서 `Request`는 `HttpRequest` 클래스를 확장한 것으로(상속 X), DRF에서의 파싱과 인증 기능을 추가한 것이다.



### Request parsing

DRF의 Request 객체는 JSON 및 다른 미디어 타입의 데이터들에 대해 동일한 방법으로 폼 데이터를 다룰 수 있도록 유연한 파싱 기능을 제공한다.

- `request.data` : Request body의 내용을 파싱하여 반환한다. 이는 `request.POST`와 `request.FILES`와 비슷하지만 다음 사항에 대해서 차이점을 보인다.
  - 파일 뿐만 아니라 비파일 입력에 대해서도 파싱한다.
  - `POST`를 제외한 다른 HTTP 메서드에 대해서도 파싱 기능을 지원한다. 즉, `PUT`, `PATCH`의 컨텐츠에도 적용이 가능하다.
  - 단순히 폼 데이터를 지원하는 기능을 넘어서 유연한 처리를 가능하게 해준다. 예를 들어, 폼 데이터 형태를 다루는 방법과 같은 방법으로 JSON 데이터를 다룰 수 있다.
- `request.query_params` :  `request.GET`에 상응한다.
  - 코드의 명확성을 위해 Django의 `request.GET`보다는 `request.query_params`를 사용할 것을 권장한다.
  - 이는 `GET` 요청에 대해서 쿼리 파라미터의 포함 여부를 구분하기 위해서다.
- `request.parsers` : `APIView` 클래스 또는 `@api_view` 데코레이터는 이 프로퍼티를 `Parser` 인스턴스 리스트로 자동으로 세팅한다. 여기서 `Parser` 인스턴스는 뷰의 `parser_classes` 또는 세팅의 `DEFAULT_PARSER_CLASSES`를 기반으로 생성된 인스턴스다. 일반적으로 이 프로퍼티에 접근할 필요는 없을 것이다.
  - 만약, 사용자가 잘못된 형태의 데이터를 포함하여 요청을 보냈다면 `request.data`는 `parserError`를 일으킬 것이다. 기본적으로 DRF에서는 `APIView` 클래스와 `@api_view` 데코레이터는 에러를 인지하고 `400 Bad Request` 응답을 반환한다.



### Content negotiation

요청 객체는 content negotiation 단계의 결과물을 사용자가 정할 수 있도록 몇가지 프로퍼티들을 노출시켜 놓았다. 이는 사용자가 각각 서로 다른 미디어 타입들이 다른 직렬화 스키마를 가질 수 있게 한다.

- `request.accepted_renderer` : content negotiation 단계에서 선택된 렌더러 인스턴스
- `request.accepted_media_type` : content negotiation 단계에서 허용된 미디어 타입을 나타내는 문자열



### Authentication

DRF에서는 각각의 인증 요청에 대해 유연한 기능을 제공한다. 이러한 기능들은

\- API에서 서로 다른 파트에 대해 다른 인증 정책을 사용할 수 있다.

\- 다중 인증 정책을 지원한다.

\- 들어온 요청에 대해 유저와 토큰 정보 둘 모두를 제공한다.

- `request.user` : 사용되는 인증 정책에 따라 다르지만, 일반적으로 `django.contrib.auth.models.User` 인스턴스를 반환한다. 만약 요청에 대해 인증이 이루어지지 않았다면 `django.contrib.auth.models.AnonymousUser` 인스턴스를 반환한다.
- `request.auth` : 추가적인 인증 내용을 반환한다. `request.auth`의 값은 인증 정책에 따라 다르지만, 일반적으로 요청을 인증하는 토큰 인스턴스를 갖는다. 만약 요청이 인증되지 않았거나 추가적은 인증 내용을 가지지 않는다면 `None` 값을 반환한다.
- `request.authenticators` : `APIView` 클래스 또는 `@api_view` 데코레이터는 이 프로퍼티를 `Authentication` 인스턴스 리스트로 자동으로 세팅한다. 여기서 `Authentication` 인스턴스는 뷰의 `authentication_classes` 또는 세팅의 `DEFAULT_AUTHENTICATORS`를 기반으로 생성된 인스턴스다. 일반적으로 이 프로퍼티에 접근할 필요는 없을 것이다.
  - `request.user` 혹은 `request.auth`를 호출하는 과정에서 `WrappedAttributeError`가 발생할 수 있다. 이 에러는 기본 에러인 `AttributeError`로부터 유래된 것이지만, 외부에서 프로퍼티를 접근하기 위해 다른 예외 타입으로 처리할 필요가 있다.



### Browser enhancements

DRF는 브라우저 기반의 `PUT`, `PATCH` 그리고 `DELETE` 폼들과 같이 몇가지 브라우저 기능을 제공한다.

- `request.method` : HTTP 요청 메서드 문자열을 **대문자**로 반환한다.
- `request.contet_type` : HTTP 요청 바디의 미디어 타입 문자열을 반환한다. 미디어 타입이 제공되지 않는다면 빈 문자열을 반환한다.
  - DRF에서 제공하는 기본 파싱 기능을 생각하면 이 프로퍼티에 접근할 필요가 없을 것이다.
  - 만약에 이 프로퍼티에 접근을 할 필요가 있다면 `request.META.get('HTTP_COTENT_TYPE')`보다는 `request.content_type`를 통해 에 접근할 것을 권장한다.
- `request.stream` : 요청 바디의 내용을 갖는 스트림을 반환한다. 이 역시 DRF에서 제공하는 기본 파싱 기능을 생각하면 이 프로퍼티에 접근할 필요가 없을 것이다.



### Standard HttpRequest attributes

앞서 언급했듯이 DRF의 `Request`는 Django의 `HttpRequest`를 확장한 것으로, `HttpRequest`에서 제공하는 모든 인자와 메서드를 사용할 수 있다. (ex : `request.META`, `request.session`)

`Request` 클래스는 `HttpRequest` 클래스를 상속한 것이 아니라 컴포지션을 통해 확장했다는 사실을 명심해야한다.