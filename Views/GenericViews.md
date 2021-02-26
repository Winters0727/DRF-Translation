# Generic Views

```
Django의 generic views은... 일반적인 사용 패턴으로 단축시키기 위해 개발되었지만... 반복없이 데이터의 뷰를 추상화시키고 뷰 개발의 일반적인 패턴등을 발견할 수 있게 했다.
- 장고 문서
```

Class-based view가 가지는 장점으로 재사용성이 뛰어나다는 점을 들 수 있다. DRF는 이를 이용하여 자주 사용되는 뷰의 패턴들을 사전에 만들어 제공하고 있다.

DRF에서 제공하는 generic views는 DB 모델에 근접한 API 뷰를 생성할 수 있게 도와준다.

만약 generic views가 API에 적합하지 않다면 원래의 `APIView` 클래스를 사용하거나 generic views를 조합하여 만든 mixins 또는 base 클래스를 재사용하면 된다.

일반적으로 generic views를 사용할 때는 일반 뷰와 몇가지 클래스 인자들을 오버라이드한다.

```python
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics
from rest_framework.permissions import IsAdminUser

class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAdminUser]
    
    def list(self, request):
        # self.queryset이 아닌 get_queryset()을 사용해야 하는 점에 주의하라
        queryset = self.get_queryset()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)
```

urls.py에 직접 적용할 수도 있다.

```python
urlpatterns = [
    path('users/', ListCreateAPIView.as_view(queryset=User.objects.all(),
                                            serializer_class=UserSerializer), name='user-list')
]
```



### GenericAPIView

DRF의 `APIView` 클래스를 확장한 것으로 표준 리스트와 디테일 뷰를 추가한 것이다. 명확한 generic views들은 이 `GenericAPIView`와 하나 이상의 mixin 클래스를 조합하여 제공된다.



#### Attributes

**기본 세팅**

- `queryset` : 뷰로부터 객체를 반환하는데 사용된다. 일반적으로 이 인자를 세팅하거나 `get_queryset()` 메서드를 오버라이딩 해야한다. 만약 뷰 메서드를 오버라이딩했을 경우 직접 프로퍼티에 접근하지말고 `get_queryset()`을 호출해야 한다. 그리하면 `queryset`은 캐싱되었다가 모든 요청에 대해 개별적으로 결과를 반환할 것이다.
- `serializer_class` : 직렬화 클래스는 값의 검증과 입력값의 반직렬화, 결과값의 직렬화에 사용된다. 일반적으로 이 인자를 세팅하거나 `get_serializer_class()` 메서드를 오버라이딩 해야한다.
- `lookup_field` : 각각의 모델 인스턴스들에 대한 객체를 구분하는데 사용되는 값이다. 기본 값은 `pk`다. 하이퍼링크 API를 사용할 때, 임의의 값을 사용하기 위해서는 API views와 직렬화 클래스 모두 lookup fields를 세팅해야 한다.
- `lookup_url_kwarg` : 객체 lookup에 사용되는 URL 키워드다. 값을 세팅하지 않았다면 기본적으로 `lookup_field`와 같은 값을 가진다.



**페이지네이션**

- `pagination_class` : 결과물 리스트를 페이지로 보여줄 때 사용된다. 기본값으로 `DEFAULT_PAGINATION_CLASS`를 가진다. (DRF의 `rest_framework.pagination.PageNumberPagination`) `None` 값으로 세팅하면 뷰에 페이지네이션을 없앤다.



**필터링**

- `filter_backends` : 필터 백엔드 클래스들의 리스트로 쿼리셋을 필터링하는데 사용된다. 기본값으로 `DEFAULT_FILTER_BACKENDS`를 갖는다.



#### Methods

**기본 메서드**

- `get_queryset(self)` : `queryset` 인자 값을 반환한다. `self.queryset`으로 프로퍼티에 직접 접근하기 보다는 이 메서드로 `queryset`에 접근하는 것을 권장한다. 동적인 작동을 위해 오버라이딩이 가능하다.

- `get_object(self)` : 디테일 뷰에 사용될 인스턴스 객체를 반환한다. `lookup_field` 파라미터로 기본 쿼리셋에서 필터링을한다. 이 또한 복잡한 작동을 위해 오버라이딩이 가능하다.

  ```python
  def get_object(self):
      queryset = self.get_queryset()
      filter = {}
      for field in self.multiple_lookup_fields:
          filter[field] = self.kwargs[field]
         
      obj = get_object_or_404(queryset, **filter)
      self.check_object_permissions(self.request, obj)
      
      return obj
  ```

  만약 API에 인증과정이 없다면 `self.check_object_permissions`는 무시해도 된다. `get_object_or_404`만을 반환해도 된다.

- `filter_queryset(self, queryset)` : 백엔드에서 사용할 필터에 대한 쿼리셋을 입력하여 새로운 쿼리셋을 반환한다.

  ```python
  def filter_queryset(self, queryset):
      filter_backends = [CategoryFilter]
      
      if 'geo_route' in self.request.query_params:
          filter_backends = [GeoRouteFilter, CategoryFilter]
      elif 'geo_point' in self.request.query_params:
          filter_backends = [GeoPointFilter, CategoryFilter]
          
      for backend in list(filter_backends):
          queryset = backend().filter_queryset(self.request, request, view=self)
          
      return queryset
  ```

- `get_serializer_class(self)` : 직렬화 클래스를 반환한다. 기본값은 `serializer_class` 인자와 같다. 다음과 같이 오버라이딩하여 유저 및 HTTP 메서드에 따라 다른 직렬화 클래스를 반환할 수 있다.

  ```python
  def get_serializer_class(self):
      if self.request.user.is_staff:
          return FullAcccountSerializer
      return BasicAccountSerializer
  ```



**저장 및 삭제 훅**

다음 메서드들은 mixin 클래스들로부터 제공되며, 오버라이딩을 통해 쉽게 변형이 가능하다.

- `perform_create(self, serializer)` : `CreateModelMixin`에서 호출되며, 새로운 인스턴스 객체를 저장할 때 호출된다.
- `perform_update(self, serializer)` : `UpdateModelMixin`에서 호출되며, 존재하는 인스턴스 객체를 업데이트할 때 호출된다.
- `perform_destroy(self, serializer)` : `DestroyModelMixin`에서 호출되며, 인스턴스 객체를 삭제할 때 호출된다.

이 훅들은 요청 데이터의 일부가 아니라 요청에 암묵적으로 세팅된 인자들로 부분적으로 유용하다.

```python
def perform_create(self, serializer):
    serializer.save(user=self.request.user)
```

```python
def perform_update(self, serializer):
    instance = serializer.save()
    send_email_confirmation(user=self.request.user, modified=instance)
```

```python
def perform_create(self, serializer):
    queryset = SignupRequest.objects.filter(user=self.request.user)
    if queryset.exists():
        raise ValidationError('You have already signed up')
    serializer.save(user=self.request.user)
```



**다른 메서드**

일반적으로 다음 메서드들은 오버라이딩 할 필요가 없다. (이는 커스텀 뷰를 사용할 때도 마찬가지다.)

- `get_serializer_context(self)` : 직렬화 클래스에 대한 추가적인 정보를 딕셔너리 형태로 반환한다. 기본키로 `request`, `view`, `format`을 가진다.
- `get_serializer(self, instance=None, data=None, many=False, partial=False)` : 직렬화 인스턴스를 반환한다.
- `get_paginated_response(self, data)` : `Response` 객체를 페이지네이션 스타일로 반환한다.
- `paginate_queryset(self, queryset)` : 쿼리셋을 페이지네이션하여 페이지 객체를 반환한다. 페이지네이션이 뷰에 적합하지 않다면 `None`을 반환한다.
- `filter_queryset(self, queryset)` : 필터를 통해 새로운 쿼리셋을 생성하여 반환한다.



### Mixins

mixin 클래스들은 기본 뷰에 사용되는 액션들을 제공한다. 이 클래스들은 `rest_framework.mixins`에서 임포트할 수 있다.



#### ListModelMixin

`.list(request, *args, **kwargs)` 메서드를 제공하며, 쿼리셋의 리스트를 보여준다. 쿼리셋이 만들어진 뒤에 `200 OK` 응답이 반환된다.

#### CreateModelMixin

`.create(request, *args, **kwargs)` 메서드를 제공하며, 모델의 인스턴스를 생성하고 저장하는데 적용된다. 객체가 정상적으로 생성되었다면 `201 Created` 응답이 반환되고, 요청 데이터가 유효하지 않아 객체가 생성되지 않았다면 `400 Bad Request` 응답이 반환된다.

#### RetrieveModelMixin

`.retrieve(request, *args, **kwargs)` 메서드를 제공하며, 존재하는 모델 인스턴스 객체를 반환한다. 객체가 정상적으로 반환되면 `200 OK` 응답이 반환되고, 인스턴스 객체가 존재하지 않는다면 `404 Not Found` 응답이 반환된다.

#### UpdateModelMixin

`.update(request, *args, **kwargs)` 메서드를 제공하며, 존재하는 인스턴스 객체를 업데이트 후에 저장한다. 또한, `.partial_update(request, *args, **kwargs)` 메서드를 제공하는데 이는 `update` 메서드와 비슷하나 모든 필드가 아닌 필드를 선택적으로 업데이트할 수 있다. 업데이트에 성공하면 `200 OK` 응답이 반환되며, 유효하지 않은 요청이 들어오면 `400 Bad Request` 응답이 반환된다.

#### DestroyModelMixin

`.destroy(request, *args, **kwargs)` 메서드를 제공하며, 존재하는 모델 인스턴스 객체를 삭제한다. 객체가 삭제되었다면 `204 No Content` 응답이 반환되고, 객체가 존재하지 않는다면 `404 Not Found` 응답이 반환된다.



### Concrete View Classes

다음 클래스들은 명확한 generic views로 수정을 많이 하지 않는 이상 명시된 액션에 대해서만 작업을 수행한다. 이 뷰 클래스들은 `rest_framework.generics`에서 임포트할 수 있다.

#### CreateAPIView

**생성**만 가능한 엔드포인트에 사용된다. `post` 메서드 핸들러를 제공한다.

`GenericAPIView`, `CreateModelMixin`으로부터 확장되었다.

#### ListAPIView

**읽기**만 가능한 엔드포인트에 사용되며, 모델 인스턴스들의 집합이 반환된다. `get` 메서드 핸들러를 제공한다.

`GenericAPIView`, `ListModelMixin`으로부터 확장되었다.

#### RetrieveAPIView

**읽기**만 가능한 엔드포인트에 사용되며, 단일 모델 인스턴스를 반환한다. `get` 메서드 핸들러를 제공한다.

`GenericAPIView`, `RetrieveModelMixin`으로부터 확장되었다.

#### DestroyAPIView

**삭제**만 가능한 엔드포인트에 사용된다. `delete` 메서드 핸들러를 제공한다.

`GenericAPIView`, `DestroyModelMixin`으로부터 확장되었다.

#### UpdateAPIView

**업데이트**만 가능한 엔드포인트에 사용된다. `put`과 `patch` 메서드 핸들러를 제공한다.

`GenericAPIView`, `UpdateModelMixin`으로부터 확장되었다.



### Customizing the generic views

```python
class MultipleFieldLookupMixin:
    """
    Apply this mixin to any view or viewset to get multiple field filtering
    based on a `lookup_fields` attribute, instead of the default single field filtering.
    """
    def get_object(self):
        queryset = self.get_queryset()             # Get the base queryset
        queryset = self.filter_queryset(queryset)  # Apply any filter backends
        filter = {}
        for field in self.lookup_fields:
            if self.kwargs[field]: # Ignore empty fields.
                filter[field] = self.kwargs[field]
        obj = get_object_or_404(queryset, **filter)  # Lookup the object
        self.check_object_permissions(self.request, obj)
        return obj
```

```python
class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_fields = ['account', 'username']
```



### Creating custom base classes

```python
class BaseRetrieveView(MultipleFieldLookupMixin,
                       generics.RetrieveAPIView):
    pass

class BaseRetrieveUpdateDestroyView(MultipleFieldLookupMixin,
                                    generics.RetrieveUpdateDestroyAPIView):
    pass
```



### PUT as create

DRF v3.0부터는 `PUT` 또한 객체의 존재 여부에 따라 업데이트 뿐만 아니라 생성 작업을 다루게 된다. `PUT`에 생성 작업을 수행하는 것은 문제가 있을 수 있지만, 객체의 존재 여부에 대한 정보를 외부에 노출시킬 필요가 있다. 또한, 삭제된 인스턴스를 재생성하는 과정을 기본 설정으로 두는 것이 단순히 `404` 응답을 반환하는 것보다 낫다. 만약 `PUT-as-create`를 사용하겠다면 뷰의 mixin에 `AllowPUTAsCreateMixin`을 추가하면 된다.