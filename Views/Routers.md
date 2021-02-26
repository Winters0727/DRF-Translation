# Routers

```
"Resource 라우팅은 주어진 resource한 컨트롤러로부터 빠르게 일반 라우트들을 생성할 수 있게 해준다. 각 인덱스에 대해 별개의 라우트를 선언하는 것 보다... 단 한줄의 코드만으로 resource한 라우트를 생성할 수 있다."
- 루비 온 레일즈 문서
```

레일즈와 같은 몇몇 웹 프레임워크들은 어떻게 해야 URL을 들어온 요청들을 다루는 로직에 자동으로 매핑할 수 있는지에 관한 기능들을 제공한다.

REST 프레임워크 역시 장고에 자동으로 간편하고 빠르게 URL 라우트를 생성하는 기능을 제공한다.



### Usage

`SimpleRouter`를 활용한 예제다.

```python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)
urlpatterns = router.urls
```

`register()` 메서드를 사용하기 위해서는 2개의 인자를 필수적으로 입력해야한다.

- `prefix` - 라우트를 설정하기 위한 URL 접두사
- `viewset` - viewset 클래스

부가적으로, 다음과 같은 인자들을 설정할 수 있다.

- `basename` - 생성된 URL이 기본적으로 사용할 이름이다. 입력하지 않을 경우, viewset의 `queryset` 인자로 자동으로 설정된다. 만약 viewset이 `queryset` 인자를 가지지 않을 경우, 명시적으로 `basename`을 등록해야한다.

상기의 예시는 다음과 같은 URL을 생성한다.

- URL Pattern : `^users/$` Name : `user-list`
- URL Pattern : `^users/{pk}/$` Name : `user-detail`
- URL Pattern : `^accounts/$` Name : `account-list`
- URL Pattern : `^accounts/{pk}/$` Name : `account-detail`

`basename` 인자를 입력하지 않았기 때문에 viewset의 `queryset` 인자로부터 각각 `user`와 `account`를 가져왔다. 만약, `get_queryset` 메서드와 같이 오버라이딩에 의해 `.queryset` 인자가 지정되지 않았다면 `basename` 인자를 반드시 명시적으로 설정해줘야한다.



### Using `include` with routers

라우터 인스턴스의 `.urls` 인자는 URL 패턴들의 기본 리스트다. 이 url들을 장고에 반영하는 몇 가지 다른 방법들이 있다.

```python
from rest_framework import routers

router = router.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', ACcountViewSet)

urlpatterns = [
    path('forgot-password/', ForgotPasswordFormView.as_view()),
]

urlpatterns += router.urls
```

다른 대안으로 장고의 `include` 함수를 사용할 수 있다.

```python
urlpatterns = [
    path('forgot-password/', ForgotPasswordFormView.as_view()),
    path('', include(router.urls)),
]
```

애플리케이션의 네임스페이스와 함께 `include` 함수를 사용할 수도 있다.

```python
urlpatterns = [
    path('forgot-password/', ForgotPasswordFormView.as_view()),
    path('api/', include((router.urls, 'app_name'))),
]
```

또는, 애플리케이션과 인스턴스 네임스페이스를 둘 다 사용할 수도 있다.

```python
urlpatterns = [
    path('forgot-password/', ForgotPasswordFormView.as_view()),
    path('api/', include((router.urls, 'app_name'), namespace='instance_name')),
]
```



### Routing for extra actions

`@action` 데코레이터를 사용하여 viewset이 정의한 추가 액션에 대한 라우트를 설정할 수 있다. 여기서 추가된 액션들은 생성된 라우트에 추가된다. 예를 들어,

```python
from myapp.permissions import isAdminOrSelf
from rest_framework.decorators import action

class UserViewSet(ModelViewSet):
    ...
    @action(methods=['post'], detail=True, permissio_classes=[IsAdminOrSelf])
    def set_password(self, request, pk=None):
        ...
```

`set_password`에 대한 라우트는 다음과 같이 생성된다.

- URL pattern : `^users/{pk}/set_password/$`
- URL name : `'user-set-password'`

기본적으로 URL pattern은 메서드명을 기반으로 생성되며, URL name은 `ViewSet.basename`과 하이픈된 메서드명의 조합으로 생성된다. 자동으로 생성되는 URL pattern과 URL name을 변경하고 싶다면 `@action` 데코레이터 내부에 인자로서 정의하면 된다.

```python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import action

class UserViewSet(ModelViewSet):
    ...
    @action(methods=['post'], detail=True, permission_classes=[IsAdminOrIsSelf],
            url_path='change-password', url_name='change_password')
    def set_password(self, request, pk=None):
        ...
```

그러면 라우트는 다음과 같이 생성된다.

- URL pattern : `^users/{pk}/change-password/$`
- URL name : `'user-change_password'`



### API Guide

### Simple Router

Simple Router는 기본적으로 라우트를 위한 기본 세트로 `list`, `create`, `retrieve`, `update`, `partial_update`, `destroy` 액션들을 제공한다. 또한, viewset에 대해 `@action` 데코레이터로 정의된 추가 액션들에 대한 라우트를 추가할 수 있다.

| **URL Style**                 | **HTTP Method**                | **Action**                                | **URL Name**          |
| ----------------------------- | ------------------------------ | ----------------------------------------- | --------------------- |
| {prefix}/                     | GET                            | list                                      | {basename}-list       |
|                               | POST                           | create                                    | {basename}-list       |
| {prefix}/{url_path}/          | GET, 또는 인자로 특정된 메서드 | `@action(detail=False)` 데코레이팅 메서드 | {basename}-{url_name} |
| {prefix}/{lookup}/            | GET                            | retrieve                                  | {basename}-detail     |
|                               | PUT                            | update                                    | {basename}-detail     |
|                               | PATCH                          | partial_update                            | {basename}-detail     |
|                               | DELETE                         | destroy                                   | {basename}-detail     |
| {prefix}/{lookup}/{url_path}/ | GET, 또는 인자로 특정된 메서드 | `@action(detail=True)` 데코레이팅 메서드  | {basename}-{url_name} |

`SimpleRouter`에 의해 생성되는 라우터들은 기본적으로 트레일링 슬래시를 가진다. 이를 없애고 싶다면 라우터 인스턴스를 생성할 때 `trailing_slash` 인자의 값을 `False`로 설정해주면 된다.

트레일링 슬래시는 장고의 컨벤션이지만, 레일즈와 같은 다른 프레임워크들에서는 사용되지 않는다. 어떤 스타일을 선택할지는 개인의 선호지만, 몇몇 자바스크립트 프레임워크들은 특정 라우팅 스타일을 사용하는 것을 고려해야한다.

경우에 따라 정규표현식을 사용하여 라우트를 제한(또는 완화)하고 싶다면 `lookup_value_regex` 인자를 설정해주면 된다. 예를 들어, 허용된 UUID의 범위를 제한하고 싶다면

```python
class MyModelViewSet(mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    lookup_field = 'my_model_id'
    lookup_value_regex = '[0-9a-f]{32}'
```



### DefaultRouter

상기의 `SimpleRouter`와 유사하지만, 기본 API root view를 가진다는 점에서 차이가 있다. 여기서 기본 API root view는 뷰 리스트들의 하이퍼링크를 가지는 뷰를 의미한다.

| **URL Style**                          | **HTTP Method**                | **Action**                                | **URL Name**          |
| -------------------------------------- | ------------------------------ | ----------------------------------------- | --------------------- |
| [.format]                              | GET                            | 자동으로 생성되는 root view               | api-root              |
| {prefix}/[.format]                     | GET                            | list                                      | {basename}-list       |
|                                        | POST                           | create                                    | {basename}-list       |
| {prefix}/{url_path}/[.format]          | GET, 또는 인자로 특정된 메서드 | `@action(detail=False)` 데코레이팅 메서드 | {basename}-{url_name} |
| {prefix}/{lookup}/[.format]            | GET                            | retrieve                                  | {basename}-detail     |
|                                        | PUT                            | update                                    | {basename}-detail     |
|                                        | PATCH                          | partial_update                            | {basename}-detail     |
|                                        | DELETE                         | destroy                                   | {basename}-detail     |
| {prefix}/{lookup}/{url_path}/[.format] | GET, 또는 인자로 특정된 메서드 | `@action(detail=True)` 데코레이팅 메서드  | {basename}-{url_name} |

`SimpleRouter`와 마찬가지로 `trailing_slash` 인자를 `False`로 설정하여 트레일링 슬래시를 제거할 수 있다.



### Custom Routers

커스텀 라우터를 사용하는 것은 자주 있는 일은 아니지만, API에 대한 URL에 특정 요구사항을 반영할 때 유용하다. 그렇게 함으로써 재사용성이 있는 URL 구조를 숨길 수 있고, 새로운 뷰에 대한 URL pattern을 명시적으로 작성할 필요가 없다.

커스텀 라우터를 생성하는 가장 쉬운 방법은 기존에 있는 라우터 클래스의 서브클래스를 생성하는 것이다. `.routes` 인자는 각각의 viewset에 매핑된 URL 패턴들 작성하는데 사용된다. `.routes` 인자는 네임드 튜플인 `Route`의 리스트로 구성되어있다.

**url** : 라우트된 URL을 가리키는 문자열. 다음과 같은 포맷 문자열들을 가진다.

- `{prefix}` : 라우트 구성에 사용될 URL 접두사
- `{lookup}` : 단일 인스턴스에 대응되는 필드
- `{trailing_slash}` : `trailing_slash` 인자 값에 따라 '/' 또는 빈 문자열

**mapping** : 뷰 메서드에 매핑된 HTTP 메서드명들

**name** : `reverse` 함수가 호출할 때 사용되는 URL 이름.

- `{basename}` : 생성된 URL 이름의 기본명

**initkwargs** : 뷰의 인스턴스를 생성할 때 넣어준 추가 인자들의 딕셔너리. `detail`, `basename`, `suffix`

Note that the `detail`, `basename`, and `suffix` arguments are reserved for viewset introspection and are also used by the browsable API to generate the view name and breadcrumb links.



### Customizing dynamic routes

`@action` 데코레이터 라우트 또한 커스터마이징이 가능하다. `.routes` 리스트에 포함된 `DynamicRoute` 네임드 튜플을 포함해서, 적절한 `detail` 인자를 설정하면 된다.

**url** : 라우트된 URL을 가리키는 문자열. `Route`와 같은 문자열을 가지며, 여기에 `{url_path}` 문자열이 추가된다.

**name** : `reverse` 함수가 호출할 때 사용되는 URL 이름.

- `{basename}` : 생성된 URL 이름의 기본명
- `{url_name}` : `@action`이 제공하는 `url_name`

**initkwargs** : 뷰의 인스턴스를 생성할 때 넣어준 추가 인자들의 딕셔너리.



### Advance custom routers

만약에 모든 라우트를 직접 커스텀하고 싶다면 `BaseRouter` 클래스와 `get_urls(self)` 메서드를 오버라이딩하면 된다. 이 메서드는 등록된 viewsets을 탐색하여 URL pattern을 반환한다. 등록된 접두사, viewset, 기본명 튜플은 `self.registery` 인자로 접근이 가능하다.