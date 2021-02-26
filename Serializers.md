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

