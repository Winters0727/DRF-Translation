# ViewSets

```
요청을 처리하는 컨트롤러를 위한 라우트가 결정된 후, 컨트롤러는 요청에 대한 적절한 결과물을 생성할 수 있어야 한다.
- 루비 온 레일즈 문서
```

DRF는 `ViewSet`라는 단일 클래스를 제공하는데, 이는 뷰에 관련된 로직들을 조합할 수 있게 한다. 다른 프레임워크에서 언급하는 'Controllers'나 'Resources'와 유사하다.

`ViewSet` 클래스는 `.get()`, `.post()`와 같은 어떤 메서드 핸들러도 제공하지 않는 class-based View다. 대신 `.list()`, '`create()`와 같은 액션들을 제공한다.

`ViewSet`을 위한 메서드 핸들러는 뷰를 완성하는 시점에서 `.as_view()` 메서드로

일반적으로 urlconf에 viewset의 views를 명시적으로 등록하기보다는 viewset을 router 클래스에 등록하여 자동으로 urlconf가 결정되도록 하는 것이 좋다.

```python
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapp.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):
    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)
    
    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```

필요에 따라 다음과 같이 viewset을 두 개의 view로 분리할 수 있다.

```python
user_list = UserViewSet.as_view({'get':'list'})
user_detail = UserViewSet.as_view({'get':'retrieve'})
```

일반적으로 이렇게 viewset을 분리하지는 않는다. 대신 라우터에 등록하여 자동으로 urlconf을 생성한다.

```python
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet, basename='user')
urlpatterns = router.urls
```

다음과 같이 viewset이 아닌 모델을 그대로 사용할 수도 있다.

```python
class UserViewSet(viewsets.ModelViewSet):
    serializer_class = UserSerializer
    queryset = User.objects.all()
```

`View` 클래스보다 `ViewSet`을 사용하는 것이 2가지 면에서 장점이 있다.

- 반복되는 로직을 단일 클래스로 조합할 수 있다. 상기 예처럼 `queryset`을 한번 특정하면 여러 뷰에서 재사용이 가능하다.
- 라우터를 사용함으로써 더이상 URL conf를 신경쓸 필요가 없다.

`View`든 `ViewSet`든 트레이드 오프가 존재한다. 일반 view와 URL conf를 사용하는 것은 명시적이고 작업에 더 자세하게 관여할 수 있다. viewset은 빠르게 또는 큰 API를 구축할 때 도움이 되며, 일관적인 URL conf를 강제할 때도 유용하다.



### ViewSet actions

DRF에서 제공하는 기본 라우터에는 다음과 같이 기본적인 CRUD 라우트를 제공한다.

```python
class UserViewSet(viewsets.ViewSet):
    """
    비어있는 viewset은 router 클래스에 의해 다루어지는 기본 액션이다.
    만약에 포맷 접미어를 사용 중이라면 'format=None' 키워드 인자를 각각의 액션에 추가해야한다.
    """

    def list(self, request):
        pass

    def create(self, request):
        pass

    def retrieve(self, request, pk=None):
        pass

    def update(self, request, pk=None):
        pass

    def partial_update(self, request, pk=None):
        pass

    def destroy(self, request, pk=None):
        pass
```



### Introspecting ViewSet actions

- `basename` : 생성된 URL의 기본 이름
- `action` : 현재 액션의 이름(create, list)
- `detail` : 현재 액션이 리스트 또는 디테일인지 여부
- `suffix` : viewset의 타입을 보여주는 접미어
- `name` : viewset의 이름
- `description` : viewset의 개별 뷰에 대한 설명

```python
def get_permission(self):
    if self.action == 'list':
        permission_classes = [IsAuthenticated]
    else:
        permission_classes = [IsAdmin]
    return [permission() for permission in permission_classes]
```



### Marking extra action for routing

만약 라우트를 설정할 필요가 있는 애드훅 메서드가 있다면, `@action` 데코레이터를 사용할 수 있다. 일단 액션과 마찬가지로, 추가적인 액션들도 하나의 객체 또는 전체 콜렉션에 대해 적용이 가능할 것이다. 이를 명확히 해주려면 `detail` 인자를 `True` 또는 `False`로 설정하면 된다. 이에 따라 라우터는 URL 패턴을 생성한다.

```python
from django.contrib.auth.models import User
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from myapp.serializers import UserSerializer, PasswordSerializer

class UserViewSet(viewsets.ModelViewSet):
    """
    A viewset that provides the standard actions
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer

    @action(detail=True, methods=['post'])
    def set_password(self, request, pk=None):
        user = self.get_object()
        serializer = PasswordSerializer(data=request.data)
        if serializer.is_valid():
            user.set_password(serializer.data['password'])
            user.save()
            return Response({'status': 'password set'})
        else:
            return Response(serializer.errors,
                            status=status.HTTP_400_BAD_REQUEST)

    @action(detail=False)
    def recent_users(self, request):
        recent_users = User.objects.all().order_by('-last_login')

        page = self.paginate_queryset(recent_users)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(recent_users, many=True)
        return Response(serializer.data)
```

`action` 데코레이터는 기본적으로 `GET` 라우트를 받는다. 하지만 다른 HTTP 메서드들도 받는다면 `methods` 인자를 설정하면 된다.

```python
@action(detail=True, methods=['post', 'delete'])
def unset_password(self, request, pk=None):
    ...
```

또한, 데코레이터는 viewset-단계의 설정에 대한 오버라이딩이 가능하다.(`permission_classes`, `serializer_class`, `filter_backends` ...)

```python
@action(detail=True, methods=['post'], permission_classes=[IsAdminOrIsSelf])
def set_password(self, request):
    ...
```

설정된 두 액션들은 `^users/{pk}/set_password/$`와 `^users/{pk}/unset_password/$` URL로 사용이 가능하다. URL을 변경하고 싶다면 `url_path` 그리고 `url_name` 파라미터를 변경하면 된다.

모든 추가 액션들을 보고 싶다면, `get_extra_actions()` 메서드를 호출하면 된다.



### Routing additional HTTP methods for extra actions

추가 액션들을 통해 `ViewSet` 메서드들 분리하기 위해 추가적인 HTTP 메서드들을 매핑하는 것이 가능하다.

예를 들어, 상기의 `set_password`와 `unset_password`를 하나의 라우트로 합치기 위해 다음과 같이 매핑하면된다. 단, 이 경우에 추가된 매핑은 추가적인 인자를 허용하지 않는다.

```python
@action(detail=True, methods=['put'], name='Change Password')
    def password(self, request, pk=None):
        """Update the user's password."""
        ...

@password.mapping.delete
    def delete_password(self, request, pk=None):
        """Delete the user's password."""
        ...
```



### Reversing action URLs

액션의 URL 값이 필요하다면, `.reverse_action()` 메서드를 사용하자. 이것은 `reverse()`에 유용한 래퍼로, `.basename` 인자를 가지는 `url_name`을 앞에 추가하고, 뷰의 `request` 객체를 자동으로 통과시킨다.

여기서 `basename`은 `ViewSet` 등록 시에 제공된다. 만약 라우터를 사용하지 않는다면, `.as_view()` 메서드에 `basename` 인자를 반드시 제공해야한다.

```python
view.reverse_action('set-password', args=['1'])
'http://localhost:8000/api/users/1/set_password'
```

대안으로, `@action` 데코레이터의 `url_name` 인자를 사용할 수 있다.

```python
view.reverse_action(view.set_password.url_name, args=['1'])
'http://localhost:8000/api/users/1/set_password'
```

`.reverse_action()`에 사용할 `url_name` 인자는 `@action`의 인자와 일치해야 한다. 또한, 이 메서드는 reverse의 기본 액션들도 사용할 수 있다.(`list`, `create`)



### ViewSet

`ViewSet` 클래스는 `APIView` 클래스를 상속받는다. viewset에 API 정책을 관리하기 위해 `permission_classes`, `authentication_classes`와 같은 기본 인자들을 사용할 수 있다.

`ViewSet` 클래스는 그 어떤 액션도 제공하지 않는다. `ViewSet`을 사용하기 위해서는 클래스를 오버라이딩하거나 명시적으로 액션을 정의해야한다.



### GenericViewSet

`GenericViewSet` 클래스는 `GenericAPIView` 클래스를 상속받으며, generic view 기반의 메서드나 `get_object`, `get_queryset`과 같은 기본 메서드를 제공한다. 하지만, 액션들은 제공되지 않는다.

`GenericViewSet` 클래스를 사용하기 위해서는 다른 클래스나 믹스인(믹스인 클래스를 상속받은)을 오버라이딩하거나 명시적으로 액션을 정의해야한다.



### ModelViewSet

`ModelViewSet` 클래스는 `GenericAPIView` 클래스를 상속받으며, 다양한 액션들을 포함한다. (믹스인 클래스들의 조합에 의한)

`ModelViewSet`에서 제공되는 액션들은 `.list()`, `.retrieve()`, `.create()`, `.update()`, `.partial_update()` 그리고 `.destroy()`가 있다.

먼저 `ModelViewSet`은 `GenericAPIView`을 확장한 것이므로 최소한 `query_set`과 `serializer_class` 인자를 제공해야한다.

```python
class AccountViewSet(viewsets.ModelViewSet):
    """
    계정을 다루는 기본 ViewSet
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]
```

이 때, `GenericAPIView`에 있는 기본 인자나 메서드를 오버라이드하여 사용할 수 있다는 사실을 명심하자.

```python
class AccountViewSet(viewsets.ModelViewSet):
    """
    유저와 관련된 계정을 다루는 기본 ViewSet
    """
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]

    def get_queryset(self):
        return self.request.user.accounts.all()
```

`ViewSet`에서 `queryset` 프로퍼티를 제거한다면 라우터는 모델의 `basename`을 자동으로 유추할 수 없게 된다. 이 경우에는 `basename` 키워드 인자를 특정해줘야 한다. 또한, 기본적으로 제공하는 CRUD 기능들도 기본 permission classes를 사용하여 제약을 걸 수 있다.



### ReadOnlyModelViewSet

`ReadOnlyModelViewSet`은 `ModelViewSet`과 전반적으로 비슷하지만, 이름 그대로 '읽기 전용'이므로 `.list()`와 `.retrieve()` 메서드만 사용이 가능하다.

```python
class AccountViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
```

그 외의 기능은 `ModelViewSet`과 같으므로 `GenericAPIView`의 프로퍼티와 메서드를 오버라이딩할 수 있다.



### Custom ViewSet

`ModelViewSet`에서 제공하는 기본 CRUD 기능 중 일부만을 사용하려면 커스텀 `ViewSet`을 만들어 사용하면된다.

```python
from rest_framework import mixins

class CreateListRetrieveViewSet(mixins.CreateModelMixin,
                                mixins.ListModelMixin,
                                mixins.RetrieveModelMixin,
                                viewsets.GenericViewSet):
    """
    이 viewset은 'create', 'retrieve', 'list' 액션을 가진다.
    viewset을 사용하기 위해서는 'queryset'과 'serializer_class'를 정의하여 오버라이딩해야한다.
    """
    pass
```

