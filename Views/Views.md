# Class-based Views

```
Django의 Class-based views는 old-style views로부터의 훌륭한 출발이다.
- Reinout van Rees
```

DRF는 Django의 `View` 클래스의 서브클래스인 `APIView` 클래스를 제공한다.

`APIView` 클래스는 `View` 클래스와 다음과 같은 차이점을 갖는다.

- 핸들러 메서드로 넘겨지는 요청 객체는 DRF의 `Request` 인스턴스지, Django의 `HttpRequest` 인스턴스가 아니다.
- 핸들러 메서드는 Django의 `HttpResponse` 인스턴스가 아닌 DRF의 `Response` 인스턴스를 반환한다. 이로 인해 뷰에서는 content negotiation이 이루어지고 응답에 적절한 렌더러를 세팅한다.
- 어떤 `APIException` 예외들도 적절한 응답으로 넘겨진다.
- 핸들러 메서드에 의해 요청이 처리되기 전에 들어온 요청에 대한 적절한 인증과 허가, 쓰로틀링 체크가 이루어진다.

`APIView` 클래스는 기존의 `View` 클래스를 사용하는 것처럼 들어온 요청을 `.get()`, `.post()`로 핸들러 메서드에 넘겨주는 것과 같다. 추가적으로, API 정책을 다양한 측면에서 관리하는 클래스에는 적절한 수의 인자들이 세팅된다.

```python
from rest_framework.views import APIView
from rest_framework.resposne import Response
from rest_framework import authentication, permissions
from django.contrib.auth.models import User

class ListUsers(APIView):
    """
    모든 유저들의 리스트를 보여주는 뷰
    * 토큰 인증이 필요
    * 오직 어드민 유저만이 이 뷰에 접근할 수 있음
    """
    authentication_classes = [authentication.TokenAuthentication]
    permission_classes = [permissions.IsAdminUser]
    
    def get(self, request, format=None):
        """
        모든 유저들의 리스트를 반환
        """
        usernames = [user.username for user in User.objects.all()]
        return Response(usernames)
```

DRF에서 제공하는 `APIView`, `GenericAPIView`, 다양한 `Mixins`, 그리고 `Viewsets`들은 처음부터 복잡할 수 있으니 문서를 참조할 것.



### API policy attributes

- `.renderer_classes`
- `.parser_classes`
- `.authentication_classes`
- `.throttle_classes`
- `.permission_classes`
- `content_negotiation_class`



### API policy instantiation methods

- `.get_renderers(self)`
- `.get_parsers(self)`
- `.get_authenticators(self)`
- `.get_throttles(self)`
- `.get_permissions(self)`
- `.get_content_negotiator(self)`
- `get_exception_handler(self)`



### API policy implementation methods

- `.check_permissions(self, request)`
- `.check_throttles(self, request)`
- `.perform_content_negotiation(self, request, force=False)`



### Dispatch methods

다음의 메서드들은 뷰의 `.dispatch()` 메서드로부터 직접 호출되는 메서드들이다. 이는 핸들러 메서드들이 호출되기 전 및 호출된 후에 이루어질 연산을 수행한다.

`.get()`, `.post()`, `put()`, `patch()`, `delete()`

- `.initial(self, request, *args, **kwargs)` : 핸들러 메서드가 호출되기 전에 이루어질 연산을 수행한다. 주로 content negotiation을 수행하거나 쓰로틀링 및 허가를 강제할 때 사용된다. 일반적으로 이 메서드를 오버라이딩할 필요는 없다.
- `.handle_exception(self, exc)` : 핸들러 메서드에의해 발생하는 어떤 예외든 간에 이 메서드를 통과하는데, 각각 `Response` 인스턴스 또는 예외를 반환한다.
- `.initialize_request(self, request, *args, **kwargs)` : DRF에서 핸들러 메서드를 통과하는 요청은 Django의 `HttpRequest`가 아닌 DRF의 `Request`의 인스턴스임을 확인시켜준다. 일반적으로 이 메서드를 오버라이드할 필요는 없다.
- `.finalize_response(self, request, response, *args, **kwargs)` : 핸들러 메서드로부터 반환된 `Response` 객체가 적절한 컨텐츠 타입으로 렌더링됨을 확인시켜준다. 이 역시 메서드를 오버라이드할 필요는 없다.



### Function Based Views

```
class-based views가 항상 더 나은 선택지라고 말하는 것은 실수다.
- Nick Coghlan
```

DRF는 function based views을 사용하는 것도 가능하다. 몇가지의 간단한 데코레이터를 제공하는데, 이를 사용하여 적절한 `Request` 인스턴스를 받고, `Response` 인스턴스를 반환하는 것이 가능하다. 또한, 어떻게 요청이 이루어지는지 확인할 수 있다.

#### @api_view()

`@api_view(http_method_names=['GET', 'POST', 'PUT', 'UPDATE', 'DELETE'])`

function based views의 핵심은 `api_view` 데코레이터로, 어떤 HTTP 메서드에 반응하는지 HTTP 메서드 리스트를 갖는다. 예를 들어, 다음과 같이 HTTP 메서드 분기를 구성할 수 있다.

```python
from rest_framework.decorators import api_view

@api_view(['GET', 'POST'])
def hello_wolrd(request):
    if request.method == 'POST': # HTTP 메서드가 POST라면
        return Response({"message" : "Got some data!", "data" : request.data})
    return Response({"message" : "Hello, world!"}) # 그 이외의 경우, HTTP 메서드가 GET이라면
```

만약 `@api_view` 데코레이터에 명시되지 않은 HTTP 메서드로 요청을 받을 경우에는 `405 Method Not Allowed` 에러가 발생한다.



#### API policy decorators

기본 세팅을 오버라이드하기 위해서 DRF에서는 뷰에 추가할 수 있는 데코레이터들을 제공한다. 이 데코레이터들은 반드시 `@api_view` 데코레이터 이후에 추가해야 한다. 예를 들어, 다음과 같이 `throttle` 기능을 구현하는데 `@throttle_classes` 데코레이터를 사용한다.

```python
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
    rate = '1/day'
    
@api_view(['GET'])
@throttle_classes([OncePerDayUserThrottle])
def view(request):
    return Response({"message" : "Hello for today! See you tomorrow!"})
```

이 데코레이터들은 `APIView` 서브클래스들에 지정된 인자들과 상응한다.

- `@renderer_classes(...)`
- `@parser_classes(...)`
- `@authentication_classes(...)`
- `@throttle_classes(...)`
- `@permission_classes(...)`

각각의 데코레이터들은 클래스들로 이루어진 튜플 또는 리스트인 한 개의 인자를 가진다.



### View schema decorator

function based view의 기본 스키마 생성을 오버라이드하기 위해 `@schema` 데코레이터를 사용해야한다. 이 데코레이터 역시 `@api_view` 데코레이터 이후에 와야한다.

```python
from rest_framework.decorators import api_view, schema
from rest_framework.schemas import AutoSchema

class CustomAutoSchema(AutoSchema):
    def get_link(self, path, method, base_url):
        # override view introspection here...

@api_view(['GET'])
@schema(CustomAutoSchema())
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

이 데코레이터는 단일 `AutoSchema` 인스턴스를 받는데, 이 인스턴스는 `AutoSchema` 서브클래스 인스턴스 또는 `ManualSchema` 인스턴스다. 스키마 생성에서 뷰를 제외하려면 `None` 값을 넣어주면 된다.

```python
@api_view(['GET'])
@schema(None)
def view(request):
    return Response({"message": "Will not appear in schema!"})
```

