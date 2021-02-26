# Response

```
기본 HttpResponse 객체와는 다르게 TemplateResponse 객체는 응답을 계산하기 위해 뷰에서 제공하는 내용의 디테일을 유지한다. 응답의 최종 결과물은 필요할 때까지(응답 프로세스보다 더 늦게) 계산되지 않는다.
- 장고 문서
```

DRF는 사용자의 요청에 따라 다중 컨텐츠 타입으로 렌더링될 수 있는 컨텐츠를 반환하는 `Response` 클래스를 제공하는 HTTP content negotiation 기능을 지원한다.

`Response` 클래스는 Django의 `SimpleTemplateResponse` 클래스의 서브클래스다. `Response` 객체는 파이썬의 원시형으로 이루어진 데이터로 초기화된다. 그 후에 DRF는 상기에 언급한 HTTP content negotiation 기능을 통해 최종 응답 컨텐츠를 어떻게 렌더링할 지를 결정한다.

`Response` 클래스를 반드시 사용할 필요는 없다. 당신의 뷰의 필요에 따라 `HttpResponse` 또는 `StreamingHttpResponse` 객체를 사용해도 된다. 단지 `Response` 클래스를 사용하면 다중 포맷을 렌더링할 수 있는 content-negotiated Web API 응답을 간단하게 제공할 수 있다.

어떤 이유들로 인해 DRF를 커스터마이징할 필요가 없다면, 당신은 반드시 `APIView` 클래스 또는 `@api_view` 데코레이터를 뷰에 응답 객체를 반환하는데 사용해야 한다. 그렇게 함으로써 뷰는 바로 데이터를 보여주지 않고, content negotiation 기능을 사용하여 응답을 적절하게 렌더링하여 데이터를 보여준다.



### Creating Responses

#### Response()

```python
Response(data, status=None, template_name=None, headers=None, content_type=None)
```

- `data` : 응답을 위한 직렬화된 데이터
- `status` : 상태 코드로 기본 값은 200이다.
- `template_name` : `HTMLRenderer`가 선택되었을 경우 사용할 템플릿 이름
- `headers` : 응답에 사용할 딕셔너리 형태의 HTTP headers
- `content_type` : 응답의 컨텐츠 타입. 일반적으로 렌더러의 content negotiation에 의해 자동으로 정해지지만, 사용자가 명시적으로 컨텐츠 타입을 입력해야하는 경우가 있을 수 있다.

기존 `HttpResponse` 객체와는 다르게 렌더링된 컨텐츠를 `Response` 객체로 인스턴스화 시킬 필요는 없다. 그 대신 파이썬 원시형으로 이루어진 렌더링 되지 않은 데이터를 넘겨주면 된다.

`Response` 클래스가 사용하는 렌더러는 Django 모델 인스턴스와 같이 복잡한 데이터 타입을 다룰 수 없다. 따라서, `Response` 객체를 생성하기 전에 파이썬 원시형으로 데이터를 직렬화 해야한다.

직렬화를 위해서는 DRF에서 제공하는 `Serializer` 클래스를 사용할 수도 있고, 커스텀 직렬화를 사용할 수도 있다.



### Attributes

- `response.data` : 렌더링 되지 않은, 직렬화된 응답 데이터
- `response.status_code` : HTTP 응답 상태 코드
- `response.content` : 렌더링된 응답의 컨텐츠. 프로퍼티에 접근하기 전에 렌더러의 `.render()` 메서드가 실행되어야 한다.
- `response.template_name` : `HTMLRenderer` 또는 다른 커스텀 템플릿 렌더러가 응답을 렌더링할 렌더러로 선택되었을 때 사용
- `response.accpeted_rendere` : 응답의 렌더러로 사용될 렌더러 인스턴스. 뷰로 데이터가 넘어가기 전에 `APIView` 또는 `@api_view`에 의해 자동으로 세팅된다.
- `response.accepted_media_type` : content negotiation 단계에서 선택되는 미디어 타입. 이 역시 뷰로 데이터가 넘어가기 전에 `APIView` 또는 `@api_view`에 의해 자동으로 세팅된다.
- `response.renderer_context` : 렌더러의 `.render()` 메서드로 넘겨질 딕셔너리 형태의 추가적은 정보. 이 역시 뷰로 데이터가 넘어가기 전에 `APIView` 또는 `@api_view`에 의해 자동으로 세팅된다.



### Standard HttpResponse attributes

`Response` 클래스는 `SimpleTemplateResponse` 클래스를 확장한 것으로, `SimpleTemplateResponse` 클래스에서 제공하는 모든 인자와 메서드를 사용할 수 있다.

- `.render()` : 어떤 `TemplateResponse`든 간에 최종 응답 컨텐츠로 직렬화된 데이터를 보내기 위해서는 이 메서드가 호출되어야 한다. `.render()`가 호출되면 응답 컨텐츠는`accpeted_renderer` 인스턴스의 `.render(data, accpeted_media_type, renderer_context)` 메서드 호출의 결과물로 세팅된다. Django의 기본 응답 싸이클을 고려한다면 당신이 직접 `.render()`을 호출할 필요는 없다.