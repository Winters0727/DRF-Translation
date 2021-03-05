# Serializers

```
serializers의 유용함을 확장시키는 것이 우리가 강조하고 싶은 점이다. 하지만, 이는 단순한 문제가 아니며 신중히 작업해야할 일이다.
- Russell Keith-Magee, 장고 유저 그룹
```

Serializers는 쿼리셋이나 모델 인스턴스와 같은 복잡한 데이터를 `JSON`, `XML` 또는 다른 컨텐츠 타입으로 쉽게 렌더링 될 수 있는 파이썬 순수 데이터타입으로 변환하는 일을 한다. Serializers는 역직렬화(deserialization)를 지원하여 직렬화를 통해 파싱된 데이터를 다시 직렬화 이전에 넣은 복잡한 데이터의 형태로 바꿀 수 있다.

DRF의 serializers는 장고의 `Form`과 `ModelForm` 클래스와 유사하게 작동한다. 이미 `Serializer` 클래스를 제공할 뿐만 아니라, `ModelSerializer`를 통해 모델 인스터스와 쿼리셋을 다루는 serializer를 즉시 생성할 수 있다.



### Declaring Serializers

```python
from datetime import datetime

class Comment:
    def __init__(self, email, content, created=None):
        self.email = email
        self.content = content
        self.created = created or datetime.now()
        
comment = Comment(email='winters@example.com', content='foo bar')
```

생성된 `Comment` 객체를 직렬화 및 역직렬화 해줄 serializer를 선언해보자.

serializer 선언 과정은 form과 비슷하다.

```python
from rest_framework import serializers

class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```



### Serializing objects

```python
serializer = CommentSerializer(comment)
serializer.data
# {'email' : 'winters@example.com', 'content' : 'foo bar', 'created' : '2016-01-27T15:17:10.375877'}
```

comment 데이터가 serializer에 의해 모델 인스턴스에서 파이썬 순수 데이터타입으로 변환되었다. 이를 `JSON` 에 맞게 변경하자.

```python
from rest_framework.renderers import JSONRenderer

json = JSONRenderer().render(serializer.data)
json
# b'{'email' : 'winters@example.com', 'content' : 'foo bar', 'created' : '2016-01-27T15:17:10.375877'}'
```

 

### Deserializing objects

역직렬화도 비슷하다. 먼저 스트림 타입을 파이썬 순수 데이터타입으로 파싱하자.

```python
import io
from rest_framework.parsers import JSONParser

stream = io.ByteIO(json)
data = JSONParser().parse(stream)
```

그 후에, 파이썬 순수 데이터타입을 다시 딕셔너리 형태의 인증된 데이터로 복구한다.

```python
serializer = CommentSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# {'email' : 'winters@example.com', 'content' : 'foo bar', 'created' : '2016-01-27T15:17:10.375877'}
```



### Saving Instances

인증된 데이터를 다시 모델 인스턴스로 변환하기 위해서는 `.create()`나 `.update()` 메서드를 사용해야 한다.

```python
class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
    
    def create(self, validated_data):
        return Comment(**validated_data)
    
    def update(self, instance, validated_data):
        instance.email = validated_data.get('email', instance.email)
        instance.content = validated_data.get('content', instance.content)
        instance.created = validated_data.get('created', instance.created)
        return instance
```

여기서 사용되는 객체가 장고모델이라면 코드는 다음과 같이 바뀌어야 한다.

```python
def create(self, validated_data):
    return Comment.objects.create(**validated_data)

def update(self, instance, validated_data):
    instance.email = validated_data.get('email', instance.email)
    instance.content = validated_data.get('content', instance.content)
    instance.created = validated_data.get('created', instance.created)
    instance.save()
    return instance
```
데이터를 역직렬화 할 때는 검증된 데이터를 기반으로 생성된 객체 인스턴스를 반환하는 `.save()`를 호출하면 된다.

```python
comment = serializer.save()
```

`.save()`를 호출하면 기존 객체 인스턴스의 유무에 따라 `.create()`가 호출될지, `update()`가 호출될지 결정된다.

```python
# .save()는 새로운 인스턴스를 생성한다.
serializer = CommentSerializer(data=data)

# .save()는 이미 존재하는 'comment' 인스턴스를 업데이트한다.
serializer = CommentSerializer(comment, data=data)
```

`.create()`와 `.update()`는 옵션이므로 필요에 따라 사용하면 된다.



##### passing additional attribute to `.save()`

때때로 인스턴스 저장 과정에서 추가적인 정보를 넣어주고 싶을 때가 있을 것이다. 그 경우에는 `.save()`에 키워드 인자로 넘겨주면 된다.

```python
serializer.save(owner=request.user)
```

여기서 추가되는 키워드 인자는 `.create()`와 `.update()`가 호출하는 `validated_data`의 인자로 추가된다.



##### Overriding `.save()` directly

경우에 따라 `.create()`나 `.update()` 메서드는 무의미할 수도 있다. 그런 경우에는 `.save()` 메서드를 직접 오버라이딩하여 사용할 수 있다.

```python
class ContactForm(serializers.Serializer):
    email = serializers.EmailField()
    message = serializers.CharField()

    def save(self): # DB에 데이터를 저장하는게 무의미하기 때문에
        email = self.validated_data['email']
        message = self.validated_data['message']
        send_email(from=email, message=message) # 들어온 데이터로 이메일을 보낸다.
```

위와 같은 경우에는 serializer의 `.validated_data` 프로퍼티에 직접 접근한다.



### Validation

데이터의 역직렬화 과정에서 객체 인스턴스를 저장하거나 검증된 데이터로 접근하기 전에 항상 `is_valid()` 메서드가 호출한다. 어떤 검증 평가에서 에러가 발생한다면, `.error` 프로퍼티에 딕셔너리 형태로 에러 메시지가 저장된다.

```python
serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'email': ['Enter a valid e-mail address.'], 'created': ['This field is required.']}
```

여기서 딕셔너리의 키는 각각 필드명이 되고, 값은 필드에 상응하는 에러메시지가 된다. `non_field_errors` 키 또한 존재하는데 이는 일반 평가 에러들의 리스트를 보여준다. `non_field_errors` 키의 이름은 DRF 세팅에서 `NON_FIELD_ERRORS_KEY`로 변경할 수 있다.



##### Raising an exception on invalid data

`.is_valid()` 메서드는 `raise_exception`이라는 부가적인 플래그를 가지는데, 이 플래그는 평가 에러가 존재하면 `serializers.ValidationError`를 발생시킨다

여기서 다루는 예외는 DRF가 제공하는 기본 예외 핸들러에 의해 자동으로 다루어지며, `HTTP 400 Bad Request` 응답을 기본으로 반환한다.

```python
# 데이터에 에러가 존재하면 400 Bad Request 에러를 발생시킨다.
serializer.is_valid(raise_exception=True)
```



##### Field-level validation

`Serializer` 서브클래스에 `.validate_<field_name>` 메서드를 추가함으로써 필드 단계에서 검증 단계를 생성할 수 있다. 이는 장고에서 사용했던 장고 폼의 `.clean_<field name>` 메서드와 유사하다.

이 메서드들은 하나의 인자를 받는데, 이 인자는 검증 과정에 사용할 필드 값이다.

검증 과정에서 에러가 발생한다면 `serializers.ValidationError`를 반환한다.

```python
from rest_framework import serializers

class BlogPostSerializer(serializers.Serializer):
    title = serializers.CharField(max_length=100)
    content = serializers.CharField()

    def validate_title(self, value):
        """
        포스트 제목의 'django'가 포함되어 있는지 검증
        """
        if 'django' not in value.lower():
            raise serializers.ValidationError("Blog post is not about Django")
        return value
```

serializer에 등록된 `<field_name>`의 파라미터로 `required=False`가 있다면 검증 단계를 거치지 않는다.



##### Object-level validation

여러 필드에 동시에 접근해 검증 단계를 처리하고 싶다면 `Serializer` 서브클래스에 `.validate()` 메서드를 추가하면 된다. 이 메서드는 하나의 인자를 받으며, 이 인자는 필드명과 필드값의 키-값 페어를 이루는 딕셔너리다. 검증 과정에 에러가 발생하면 `serializers.ValidationError`를 반환한다.

```python
from rest_framework import serializers

class EventSerializer(serializers.Serializer):
    description = serializers.CharField(max_length=100)
    start = serializers.DateTimeField()
    finish = serializers.DateTimeField()

    def validate(self, data):
        """
        Check that start is before finish.
        """
        if data['start'] > data['finish']:
            raise serializers.ValidationError("finish must occur after start")
        return data
```



##### validator

serializer의 각각의 필드는 필드 인스턴스에 validator 선언하여 validator를 가질 수 있다.

```python
def multiple_of_ten(value):
    if value % 10 != 0:
        raise serializers.ValidationError('Not a multiple of ten')

class GameRecord(serializers.Serializer):
    score = IntegerField(validators=[multiple_of_ten])
    ...
```

serializer 클래스들 역시 재사용 가능한 validator를 포함하여 필드 데이터 셋에 적용시킬 수 있다. 이 validator들은 내부의 `Meta` 클래스에서 선언된다.

```python
class EventSerializer(serializers.Serializer):
    name = serializers.CharField()
    room_number = serializers.IntegerField(choices=[101, 102, 103, 201])
    date = serializers.DateField()

    class Meta:
        # 각각의 방은 하루에 하나의 이벤트를 가진다.
        validators = [
            UniqueTogetherValidator(
                queryset=Event.objects.all(),
                fields=['room_number', 'date']
            )
        ]
```



### Accessing the initial data and instance

serializer 인스턴스에 초기화 된 객체나 쿼리셋을 보낼 때, `.instance` 프로퍼티로 객체에 접근이 가능하다. 만약 초기화되지 않은 객체가 보내질 때는 `.instance` 프로퍼티는 `None`이 된다.

serializer 인스턴스에 데이터를 보낼 때, 수정이 되지 않은 원본 데이터는 `.initial_data` 프로퍼티로 접근이 가능하다. `data` 키워드 인자를 받지 않았을 경우, `.initial_data` 프로퍼티는 존재하지 않는다.



### Partial Updates

기본적으로 serializer에는 모든 필드의 값을 보내야하며, 그렇지 않으면 validation errors가 발생한다. 데이터를 부분적으로 보내 수정할 경우에는 `partial` 인자의 값을 `True`로 넘겨줘야한다.

```python
serializer = CommentSerializer(comment, data={'content': 'foo bar'}, partial=True)
```



### Dealing with nested objects

이전의 예들은 단순한 데이터 타입을 다루는데는 괜찮지만, 객체의 인자가 단순한 데이터 타입이 아닌 문자열, 날짜, 정수와 같은 복잡한 데이터 타입을 다뤄야하는 경우가 있을 수 있다.

이런 경우에는 `serializer` 클래스는 자체적으로 `field`의 타입을 가지며, 이를 통해 내장된 객체들을 다룰 수 있다.

```python
class UserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    username = serializers.CharField(max_length=100)

class CommentSerializer(serializers.Serializer):
    user = UserSerializer()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

필드로써 사용되는 serializer가 `None` 값을 가질 수 있으면 `required=False`, 아이템 리스트를 가질 때는 `many=True` 플래그를 인자로 넣어줘야 한다.

```python
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)
    edits = EditItemSerializer(many=True)
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```



### Writable nested representations

내장된 객체를 포함하는 데이터에 대한 역직렬화 데이터를 다룰 때, 내장된 객체의 값에 문제가 있으면 에러가 발생한다. 이는 앞에서 다루었던 `.validated_data` 프로퍼티를 내장된 객체도 가진다는 것을 의미한다.



##### writing `.create()` methods for nested representations

내장된 객체를 다룬 뒤에 인스턴스를 생성하고 싶을 때, `.create()`나 `.update()` 메서드를 오버라이딩하면 된다.

```python
class UserSerializer(serializers.ModelSerializer):
    profile = ProfileSerializer()

    class Meta:
        model = User
        fields = ['username', 'email', 'profile']

    def create(self, validated_data):
        profile_data = validated_data.pop('profile')
        user = User.objects.create(**validated_data) # 유저 인스턴스를 생성하고
        Profile.objects.create(user=user, **profile_data) # 유저 인스턴스의 프로파일 내장 객체를 생성한다.
        return user # 반환값은 프로파일 내장 객체를 가지는 유저 인스턴스다.
```



##### writing `.update()` methods for nested representations

만약 데이터 간의 관계가 `None`이거나 제공되지 않았다면, 어떻게 처리해야할까?

- 데이터베이스에서의 관계를 `NULL`로 설정
- 연관된 인스턴스를 제거
- 데이터를 무시하고 인스턴스를 그대로 둠
- 검증 에러를 발생

앞에서 사용한 `UserSerializer` 클래스에 예시를 참고하자.

```python
def update(self, instance, validated_data):
        profile = instance.profile

        instance.username = validated_data.get('username', instance.username)
        instance.email = validated_data.get('email', instance.email)
        instance.save()

        profile.is_premium_member = profile_data.get(
            'is_premium_member',
            profile.is_premium_member
        )
        profile.has_support_contract = profile_data.get(
            'has_support_contract',
            profile.has_support_contract
         )
        profile.save()

        return instance
```

내장된 객체에 대한 생성과 업데이트 액션이 애매모호하기 때문에, 이러한 복잡한 모델 관계를 다룰 때 DRF는 명시적으로 사용할 메서드를 요구한다. 내장된 객체를 작성하는데 `ModelSerializer.create()`와 `ModelSerializer.update()`를 기본적으로 지원하지 않기 때문이다.

이러한 경우를 다루기 위해 DRF의 써드 파티 패키지가 존재하니 해당 패키지를 사용하도록 하자.



### Model Serializer

`ModelSerializer` 클래스는 모델 필드에 상응하는 `serializer` 클래스를 즉시 생성해준다.

**`ModelSerializer` 클래스는 일반 `serializer` 클래스와 유사하지만, 다음과 같은 점에서 차이를 가진다.**

- 모델을 기반으로 필드를 자동으로 생성해준다.
- unique_together validators와 같은 serializer의 validator을 자동으로 생성해준다.
- 기본적으로 `.create()`와 `.update()` 메서드를 적용할 수 있다.

`ModelSerializer`는 다음과 같이 선언한다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']
```

기본적으로 모델 클래스의 필드는 serializer의 필드에 매핑된다. 외래키의 경우에는 `PrimaryKeyRelatedField`를 통해 따로 지정해주어야 하는데, 이는 serializer에서 명시적으로 특정하지 않은 이상 기본적으로 역 관계를 성립시키지 않기 때문이다.



##### inspecting a `ModelSerializer`

`ModelSerializer`의 `__repr__` 특별 메서드는 `ModelSerializer`의 개요를 보여준다. 따라서, `repr()` 내장 함수를 통해 `ModelSerializer`의 형태를 쉽게 파악할 수 있다.



### Specifying which fields to include

모델의 기본 필드들 중 일부만을 사용하고 싶을 경우, `fields`와 `exclude` 옵션을 통해 장고의 `Form`과 마찬가지로 필드들을 특정할 수 있다. 보통 `fields`를 통해 사용할 필드의 셋을 명시적으로 지정하는 것을 권장한다.

또한, `fields`의 값을 `'__all__'`로 설정하면 모든 필드를 사용할 것임을 명시적으로 보여줄 수 있다.

`exclude`는 필드 중에서 제외할 필드를 설정할 수 있다.



### Specifying nested serialization

기본적으로 `ModelSerializer`는 관계 정의를 위해 Primary key를 사용하지만, `depth` 옵션을 설정하여 쉽게 nested representations를 생성할 수 있다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']
        depth = 1
```

`depth` 옵션은 정수 값으로 지정되어야 한다.



### Sepcifying fields explicity

`ModelSerializer`에 추가적인 필드를 생성하거나 `serializer`에 정의된 필드를 활용하여 클래스에 선언된 기본 필드를 오버라이딩할 수 있다.

```python
class AccountSerializer(serializers.ModelSerializer):
    url = serializers.CharField(source='get_absolute_url', read_only=True)
    groups = serializers.PrimaryKeyRelatedField(many=True)

    class Meta:
        model = Account
```



### Specifying read only fields

다중의 필드를 읽기 전용으로 사용하고 싶은 경우가 있을 것이다. 이 경우에는 명시적으로 `read_only=True` 인자를 설정하거나, `Meta` 옵션에 `read_only_fields`를 정의하면 된다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']
        read_only_fields = ['account_name']
```

`editable=False`로 설정된 필드나 `AutoField`는 기본적으로 읽기 전용이다. 따라서, `read_only_fields`와 같은 옵션을 적용시키지 않아도 된다.

모델 단계에서 `unique_together` 제약의 일부에 해당하는 읽기 전용 필드와 같은 특수 케이스가 있다. 이는 필드는 serializer에 정의된 제약 검증 단계를 거치지만, 유저에 의한 수정은 불가능한 경우다.

이러한 경우를 올바르게 다루는 방법으로 serializer에 있는 해당 필드를 명시적으로 `read_only=True`와 `default=...` 키워드 인자를 설정해주면 된다.

```python
user = serializers.PrimaryKeyRelatedField(read_only=True, default=serializers.CurrentUserDefault())
```



### Additional keyword arguments

`extra_kwargs` 옵션을 사용하여 임의의 키워드 인자를 필드에 추가시킬 수 있는 방법이 있다. 여기서 `read_only_fields`의 경우, serializer에 명시적으로 필드를 선언할 필요가 없다.

```python
class CreateUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['email', 'username', 'password']
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User(
            email=validated_data['email'],
            username=validated_data['username']
        )
        user.set_password(validated_data['password'])
        user.save()
        return user
```

여기서 명심해야할 것은 serializer 클래스에 이미 명시적으로 선언된 필드라면 `extra_kwargs` 옵션은 무시된다는 것이다.



### Relational fields

모델 인스턴스들을 직렬화할 때, 모델 인스턴스 간의 관계를 표현하는 몇가지 다른 방법이 있다. `ModelSerializer`의 기본 표현식은 관계가 있는 인스턴스들의 `primary key`를 사용하는 것이다.

다른 대체 방법으로는 하이퍼링크를 이용한 직렬화, 완전한 내장 표현식을 이용한 직렬화, 또는 커스텀 표현식을 이용한 직렬화가 있다.



### Customizing field mappings

`ModelSerializer` 클래스는 serializer 초기화 과정에서 자동으로 결정된 필드를 대체하기 위해 오버라이딩이 가능한 API를 가진다. 만약 `ModelSerializer`에 의해 자동으로 필드가 생성되는 것을 원치 않는다면 클래스에 명시적으로 정의하든가, 아니면 기존의 `Serializer` 클래스를 사용하면 된다.