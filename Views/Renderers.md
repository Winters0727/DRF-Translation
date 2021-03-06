# Renderers

```
템플릿 응답 인스턴스가 클라이언트에게 전달되기 전에 렌더링 과정이 필요하다. 렌더링 프로세스는 템플릿과 내용을 IR로 변경한 뒤, 최후에는 클라이언트가 수용할 수 있는 byte stream으로 변환된다.

- 장고 문서

*IR(Intermediate Representation) : 소스 코드를 표현하기 위해 컴파일러 또는 가상 시스템에서 내부적으로 사용하는 데이터 구조 또는 코드
```

DRF는 몇 개의 빌트인 렌더러 클래스를 제공하여 다양한 미디어 타입의 응답을 반환할 수 있게 해준다. 또한, 미디어 타입에 따라 유연하게 대처할 수 있도록 커스텀 렌더러를 정의하는 기능도 제공한다.



### How the renderer is determined

뷰에 사용될 렌더러의 구성은 클래스의 리스트로 구성된다. 뷰에 들어가면 DRF에서 들어온 요청에 대해 content negotiation 과정을 진행하고 요청에 적절한 렌더러를 결정한다.

content negotiation의 기본 과정은 요청의 `Accpet` 헤더를 검사하여 어떤 미디어 타입으로 응답이 반환되야할지 결정하는 것이다. 부가적으로, URL에 형태를 알려주는 접미사를 통해 데이터를 보여줄 형태를 명시할 수 있다. 예를 들어, `http://example.com/api/users_count.json`는 JSON data를 반환한다.



### Setting the renderers

전역에서 사용하는 기본 렌더러는 `DEFAULT_RENDERER_CLASSES` 설정에 정의되어 있다.

```python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ]
}
```

파서와 마찬가지로 클래스 기반 뷰는 `APIView`를 통해, 함수 기반 뷰는 `@api_view` 데코레이터를 이용하여 설정할 수 있다.

```python
from django.contrib.auth.models import User
from rest_framework.renderers import JSONRenderer
from rest_framework.response import Response
from rest_framework.views import APIView

class UserCountView(APIView):
    renderer_classes = [JSONRenderer]

    def get(self, request, format=None):
        user_count = User.objects.filter(active=True).count()
        content = {'user_count': user_count}
        return Response(content)
```

```python
@api_view(['GET'])
@renderer_classes([JSONRenderer])
def user_count_view(request, format=None):
    user_count = User.objects.filter(active=True).count()
    content = {'user_count': user_count}
    return Response(content)
```



### Ordering of renderer classes

각각의 미디어 타입이 갖는 우선순위에 따라 어떤 렌더러 클래스를 API에 적용할 것인지는 중요한 문제다. 만약 클라이언트에서 `Accpet: */*`나 `Accpet` 헤더를 포함하지 않고 요청을 보냈을 경우, DRF는 렌더러 클래스들의 리스트 중에서 첫번째 렌더러를 사용하여 응답을 보낸다.



### API Reference

##### JSONRenderer

utf-8 인코딩을 통해 요청 데이터를 `JSON` 형태로 렌더링한다.

기본 스타일은 유니코드 문자들을 공백이 없는 압축된 형태로 응답을 렌더링한다.

```json
{"unicode black star":"★","value":999}
```

들여쓰기가 적용된 `JSON`을 받기 위해 클라이언트에서 `indent` 미디어 타입 파라미터를 추가하여 보냈다면 그 값만큼 들여쓰기가 된 응답으로 렌더링된다. `Accept : application/json; indent=4`

```json
{
    "unicode black star": "★",
    "value": 999
}
```

기본적인 JSON 인코딩 스타일은 `UNICODE_JSON`과 `COMPACT_JSON` 설정 키로 대체될 수 있다.

**.media_type** : `application/json`

**.format** : `json`

**.charset** : `None`



##### TemplateHTMLRenderer

장고의 기본 템플릿 렌더링을 이용하여 데이터를 HTML로 렌더링한다. 다른 렌더러와는 다르게 `Response`로 보내지는 데이터는 직렬화될 필요가 없으며, `Response`를 생성할 때 `template_name` 인자를 포함할 수 있다.

TemplateHTMLRenderer는 `response.data`를 컨텍스트 딕셔너리로 사용하여 `RequestContext`를 생성하고, 컨텍스트를 렌더링할 때 사용할 템플릿명을 결정한다.

템플릿명은 다음과 같은 순서대로 결정된다.

1. 응답에 명시적으로 `template_name` 인자를 넣었는가
2. 명시적으로 클래스 인자 `.template_name`이 설정되었는가
3. `view.get_template_name()`의 호출 결과를 반환

```python
class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    renderer_classes = [TemplateHTMLRenderer]

    def get(self, request, *args, **kwargs):
        self.object = self.get_object()
        return Response({'user': self.object}, template_name='user_detail.html')
```

`TemplateHTMLRenderer`를 사용하여 REST framework로 사용할 HTML 페이지들을 반환하거나 HTML과 API 응답을 하나의 엔드포인트로 반환할 수 있다.

만약 `TemplateHTMLRenderer`와 다른 렌더러 클래스들을 함께 사용하여 웹사이트를 만든다면, 렌더러 클래스 리스트의 첫번째 값으로 `TemplateHTMLRenderer`를 설정하여 우선순위를 지정한다면 요청에 `Accept` 헤더가 포함되지 않더라도 요청에 대한 렌더링 처리가 가능하다.

**.media_type** : `text/html`

**.format** : `html`

**.charset** : `utf-8`



##### StaticHTMLRenderer

사전에 렌더링된 HTML을 반환하는 단순한 렌더러다. 다른 렌더러들과는 다르게 응답 객체를 통해 반환되는 데이터는 문자열이어야 한다.

```python
@api_view(['GET'])
@renderer_classes([StaticHTMLRenderer])
def simple_html_view(request):
    data = '<html><body><h1>Hello, world</h1></body></html>'
    return Response(data)
```

`StaticHTMLRenderer`를 사용하여 REST framework로 사용할 HTML 페이지들을 반환하거나 HTML과 API 응답을 하나의 엔드포인트로 반환할 수 있다.

**.media_type** : `text/html`

**.format** : `html`

**.charset** : `utf-8`



##### BrowsableAPIRenderer

Browsable API를 위한 HTML로 데이터를 렌더링한다.

이 렌더러는 어떤 렌더러 클래스가 가장 높은 우선순위를 가지는지 결정하고, 이를 사용하여 API 스타일의 응답을 HTML 페이지로 보여준다.

**.media_type** : `text/html`

**.format** : `api`

**.charset** : `utf-8`

**.template** : `rest_framework/api.html`



##### Customizing BrowsableAPIRenderer

`BrowsableAPIRenderer`의 렌더러 결정 우선순위를 변경하고 싶다면 해당 렌더러를 상속받은 새로운 렌더러에 `get_default_renderer()` 메서드를 오버라이딩하면 된다.

```python
class CustomBrowsableAPIRenderer(BrowsableAPIRenderer):
    def get_default_renderer(self, view):
        return JSONRenderer()
```



##### AdminRenderer

admin 관리창과 비슷한 HTML로 데이터를 렌더링한다.

이 렌더러는 CRUD-스타일의 웹 API에 적합하며, 데이터를 다룰 때 사용자 친화적인 인터페이스를 제공한다.

단, 내장 또는 리스트 serializer를 입력값으로 받는 뷰에서는 HTML 폼이 지원하지 않으므로 `AdminRenderer`를 사용하기에 적절하지 않다.

`AdminRenderer`는 데이터의 세부 페이지로 이동하는 링크인 `URL_FIELD_NAME`(기본값으로 `url`) 인자를 유일하게 가질 수 있다. `HyperlinkedModelSerializer`도 유사하게 가능하나, `ModelSerializer`나 일반 `serializer`는 필드에 명시적으로 설정해줘야한다.

```python
class AccountSerializer(serializers.ModelSerializer):
    url = serializers.CharField(source='get_absolute_url', read_only=True)

    class Meta:
        model = Account
```

**.media_type** : `text/html`

**.format** : `admin`

**.charset** : `utf-8`

**.template** : `rest_framework/admin.html`



##### HTMLFormRenderer

HTML form 형태로 데이터를 렌더링한다. 렌더링의 결과값은 `<form>`의 닫힌 태그나 숨겨진 CSRF 값이나 제출 버튼 등을 포함하지 않는다.

이 렌더러는 직접적으로 사용하는 것이 아니라, `renderer_form` 템플릿 태그에 serializer 인스턴스를 넘겨주는 것으로 사용할 수 있다.

```html
{% load rest_framework %}

<form action="/submit-report/" method="post">
    {% csrf_token %}
    {% render_form serializer %}
    <input type="submit" value="Save" />
</form>
```

**.media_type** : `text/html`

**.format** : `form`

**.charset** : `utf-8`

**.template** : `rest_framework/horizontal/form.html`



##### MultiPartRenderer

HTML multipart 폼 데이터를 렌더링하는데 사용된다. **응답 렌더러로는 부적합하나**. 테스트 요청 등을 생성하는데 사용된다.

**.media_type** : `multipart/form-data; boundary=BoUnDaRyStRiNg`

**.format** : `multipart`

**.charset** : `utf-8`



##### Custom renderers

커스텀 렌더러를 적용하기 위해 `BaseRenderer`를 오버라이딩하고, `.media_type`과 `.format` 프로퍼티를 설정한 뒤, `.render(self, data, media_type=None, renderer_context=None)` 메서드를 적용합니다.

메서드의 반환값은 HTTP 응답의 바디에 사용될 bytestring이어야 합니다.

`.render()`의 인자는 다음과 같습니다.

- `data` : `Response()` 인스턴스 생성에 사용할 요청 데이터
- `media_type=None` : 클라이언트의 요청의 `Accpet` 헤더에 따라 결정되는 미디어 타입 파라미터로 렌더러의 `media_type` 인자보다 더 특정적입니다.
- `renderer_context=None` : 옵션으로, 설정되었다면 뷰에 제공할 추가 정보들을 딕셔너리 형태로 설정합니다. 이 프로퍼티는 기본적으로 다음 기본 키값을 가집니다. - `view`, `request`, `response`, `args`, `kwargs`



### Setting the character set

렌더러 클래스의 인코딩에 사용되는 기본 문자열은 `utf-8`입니다. 다른 인코딩 방법을 사용하고 싶다면, 렌더러의 `charset` 인자의 값을 설정하면 됩니다.

```python
class PlainTextRenderer(renderers.BaseRenderer):
    media_type = 'text/plain'
    format = 'txt'
    charset = 'iso-8859-1'

    def render(self, data, media_type=None, renderer_context=None):
        return data.encode(self.charset)
```

만약 렌더러 클래스가 유니코드 문자열을 반환한다면, 응답 컨텐츠는 `Response` 클래스에 의해 렌더러 클래스에서 인코딩하기로 설정한 `charset` 인자 설정을 활용하여 bytestring으로 강제변환됩니다.

만약 렌더러 클래스가 순수 이진 컨텐츠를 bytestring으로 반환한다면, `charset`을 `None`으로 설정하여 응답의 `Content-Type` 헤더가 `charset` 설정 값을 가지지 못하게 해야합니다.

`render_style` 인자의 값을 `'binary'`로 설정하고 싶을 때가 있을겁니다. 그렇게 함으로써 브라우저 API는 이진 컨텐츠를 문자열로 보여주지 않게 됩니다.

```python
class JPEGRenderer(renderers.BaseRenderer):
    media_type = 'image/jpeg'
    format = 'jpg'
    charset = None
    render_style = 'binary'

    def render(self, data, media_type=None, renderer_context=None):
        return data
```



### Advanced renderer usage

DRF의 렌더러 클래스를 활용하면 몇가지 유연한 트릭이 가능하다.

- 요청 미디어 타입에 따라 동일한 엔드포인트로부터 일반 또는 내장형 표현식을 모두 제공할 수 있습니다.
- 동일한 엔드포인트로 JSON 기반의 API 응답과 HTML 웹페이지 응답 모두 가능합니다.
- 클라이언트가 사용하는 API의 표현식의 타입들을 특정할 수 있습니다.
- 렌더러의 미디어 타입을 `media_type='image/*'` 임의로 특정짓고 싶다면 `Accept` 헤더를 사용하여 응답의 인코딩 방법을 다양하게 사용할 수 있습니다.



**Varying behaviour by media type**

경우에 따라 미디어 타입에 따라 다른 직렬화 스타일을 뷰에 적용시키고 싶을 때가 있을 것입니다. 그렇게 할 필요가 있다면 `request.accepted_renderer`로 응답에 사용할 렌더러를 결정하면 됩니다.

```python
@api_view(['GET'])
@renderer_classes([TemplateHTMLRenderer, JSONRenderer])
def list_users(request):
    """
    JSON과 HTML 표현식을 가지는 뷰
    """
    queryset = Users.objects.filter(active=True)

    if request.accepted_renderer.format == 'html':
        # TemplateHTMLRenderer는 딕셔너리형 컨텍스트를 받고,
        # 추가적으로 'template_name' 값을 받는다..
        # 직렬화 과정을 필요로 하지 않는다.
        data = {'users': queryset}
        return Response(data, template_name='list_users.html')

    # JSONRenderer는 직렬화 과정이 필요하다.
    serializer = UserSerializer(instance=queryset)
    data = serializer.data
    return Response(data)
```



**Underspecifying the media type**

경우에 따라 어떤 미디어 타입의 범위를 다루는 렌더러를 사용하고 싶을 때가 있을 것이다. 이 경우에는 `media_type=image/*` 또는 `media_type = */*`와 같이 설정하면 된다.

단, 미디어 타입을 정확하게 특정하지 않은 경우에는 응답 객체에 `content_type`을 명시적으로 설정해줘야 한다.

```python
return Response(data, content_type='image/png')
```



**Designing your media types**

웹 API로 사용할 목적이라면, 하이퍼링크 관계를 가지는 `JSON` 응답만으로도 충분할 것이다. 만약 완전히 RESTful한 디자인을 원한다면 미디어 타입의 디자인과 사용과 관련하여 세부적인 설정을 고려할 필요가 있다.

Roy Fielding의 말에 의하면,

```
"A REST API should spend almost all of its descriptive effort in defining the media type(s) used for representing resources and driving application state, or in defining extended relation names and/or hypertext-enabled mark-up for existing standard media types."

"REST API는 현존하는 기준 미디어 타입을 위한 확장된 관계명 또는 하이퍼텍스트한 마크업을 정의하거나 애플리케이션의 상태와 자원표시에 사용할 미디어 타입을 정의하는데 대부분의 노력을 쏟아부어야 한다."
```



**HTML error views**

일반적으로 렌더러는 `APIException`의 서브클래스나 `Http404` 또는 `PermissionDenied`와 같은 응답 예외나 일반 응답에 상관없이 행동한다.

`TemplateHTMLRenderer` 또는 `StaticHTMLRenderer`를 사용할 때는 예외가 발생했을 때 조금 다른 방식으로 행동하며, [장고의 기본 에러 뷰 처리 방식](https://docs.djangoproject.com/en/stable/topics/http/views/#customizing-error-views)을 따른다.

HTML 렌더러는 발생한 예외를 다음과 같은 순서로, 다음 중 하나의 방법으로 처리한다.

- `{static_code}.html` 템플릿을 불러오고 렌더링한다.
- `api_exception.html` 템플릿을 불러오고 렌더링한다.
- HTTP 상태 코드와 문자를 렌더링한다. `404 Not Found`

템플릿은 `status_code`와 `details` 키를 포함하는 `RequestContext`를 렌더링한다. 