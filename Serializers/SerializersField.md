# Serializers fields

```
폼 클래스의 각각의 필드는 데이터를 검증해야할 뿐만 아니라, 데이터를 정리하여 데이터를 일관된 형식으로 정규화합니다.
- 장고 문서
```

serializer 필드는 내부 데이터 타입과 파이썬 원시형 값 간의 변환을 다룹니다. 또한, 이들은 입력 값의 검증 과정과 부모 객체로부터 온 값을 설정하고 도출하는 역할도 합니다.

serializer 필드는 `fields.py`에 정의되어 있지만, 컨벤션을 따르기 위해 `from rest_framework import serializers`로 serializers를 임포트하고, `serializers.<FieldName>`으로 사용합니다.



### Core Arguments

각각의 serializer 필드 생성자는 적어도 다음 인자들을 받습니다. 몇몇 필드 클래스들은 추가적인, 필드에 특화된 인자를 받기도 하지만, 다음 인자들은 항상 적용됩니다.

- `read_only` : 읽기 전용 필드로 API의 결과물을 포함하지만, 생성과 수정 작업을 포함하지 않습니다. 읽기 전용 필드에 대한 입력값들은 모두 무시됩니다. 직렬화 과정으로 데이터를 보내고, 역직렬화 과정에서 인스턴스를 생성 및 수정하는 작업을 금지하려면 `True`로 설정합니다. 기본값은 `False`입니다.
- `write_only` : 읽기 전용과는 반대로 역직렬화 과정에서 인스턴스를 생성 및 수정하는 작업은 가능하지만, 직렬화 과정으로 데이터를 보내는 작업을 금지하려면 `True`로 설정합니다. 기본값은 `False`입니다.
- `required` : 역직렬화 과정에서 데이터가 비어있다면 에러를 발생시킵니다. 역직렬화 과정에서 필수적인 데이터가 아니면 `False`로 설정합니다. `False`로 설정한다면 인스턴스의 직렬화 과정에서 객체의 인자나 딕셔너리의 키를 생략합니다. 만약 키 값이 들어있지 않다면, 결과물에 포함되지 않습니다. 기본값은 `True`입니다.
- `default` : 설정한다면, 입력값으로 들어오지 않는 필드의 기본값으로 설정합니다. 기본값이 설정되어 있지 않다면 인자는 무시됩니다. 기본값은 부분 수정 작업에는 반영되지 않습니다. 부분 수정 작업에서는 들어온 입력값에 대해서만 검증과정을 거친 뒤, 결과물을 반환합니다. 함수 및 다른 호출 과정을 통해 사용될 때마다 값의 평가가 이루어져야할 경우, 이들이 인자로 `requires_context=True`를 가진다면 serializer 필드는 인자를 넘깁니다. 인스턴스의 직렬화 과정에서 기본값은 인스턴스에 제공되지 않은 인자와 딕셔너리 키의 값으로 사용됩니다. `default` 설정을 사용한다는 것은 이 필드가 필수적이지 않다는 것을 의미하므로, `default`와 `required` 설정을 함께 사용한다면 에러가 발생합니다.
- `allow_null` : 일반적으로 `None` 값이 serializer 필드에 들어오면 에러가 발생합니다. 이 키워드 인자를 `True`로 설정하면 `None` 값이 허용됩니다. 단, 이 키워드 인자를 사용한다는 것은 명시적인 `default` 설정이 없다하더라도 직렬화 과정에서 암시적으로 `default=null`을 의미하지만, 역직렬화 과정에서는 기본값을 의미하지 않습니다. 기본값은 `False`입니다.
- `source` : 필드를 채울 인자명을 의미합니다. 메서드의 경우에는 `URLField(source='get_absolute_url')`과 같이 `self` 인자만을 받습니다. 또는, 점 표기법으로 `EmailField(source='user.email')`과 같이 교차 인자를 사용할 수 있습니다. 점 표기법으로 직렬화할 때는, 인자의 교차 과정에서 비어있거나 없는 객체의 경우에 대비해 `default` 값을 설정해줘야 합니다. `source='*'`에는 특별한 뜻이 있는데, 필드를 통과하는 모든 객체를 암시하는데 사용합니다. 이는 내장 표현식을 생성하거나 결과 표현식을 결정하기 위한 완성 객체에 접근을 요청할 때 사용됩니다. 기본값은 필드명입니다.

- `validators` : 입력값에 적용할 검증 함수들의 리스트입니다. 검증 과정에서 문제가 생기면 `serializers.ValidationError`를 반환하지만, 장고의 빌트인 `ValidationError` 역시 서드파티 장고 패키지 또는 장고 데이터베이스에 정의된 validators의 적합성을 지원합니다.

- `error_message` : 에러 코드와 에러 메시지를 가지는 딕셔너리

- `label` : HTML 폼 필드나 다른 서술 엘리먼트에 사용될 짧은 문자열

- `help_text` : HTML 폼 필드나 다른 서술 엘리먼트에 대한 설명 문자열

- `initial` : HTML 폼 필드에 사전에 사용될 값

  ```python
  import datetime
  from rest_framework import serializers
  class ExampleSerializer(serializers.Serializer):
      day = serializers.DateField(initial=datetime.date.today)
  ```

- `style` : 렌더러가 필드를 렌더링할 때 사용할 옵션을 가지는 키-값 페어의 딕셔너리

  ```python
  # 입력에 <input type="password">을 적용
  password = serializers.CharField(
      style={'input_type': 'password'}
  )
  
  # 선택 입력이 아닌 라디오  사용
  color_channel = serializers.ChoiceField(
      choices=['red', 'green', 'blue'],
      style={'base_template': 'radio.html'}
  )
  ```

  

### Boolean fields

##### BooleanField

HTML 인코딩 폼을 사용할 때 입력값이 생략되었다면 `default=True`로 설정하더라도 필드값은 항상 `False`로 저장됐다. 이는 체크박스 입력 표현에 대해 값을 생략하는 미체크 상태로 인식하기 때문이다. 그래서 DRF는 값을 생략하면 빈 체크박스 입력으로 인식한다.

장고 2.1부터는 `models.BooleanField`의 `blank` 키워드 인자가 제거되고 항상 `blank=True` 옵션을 가지게 되었다. 따라서, `serializers.BooleanField`의 인스턴스는 자동으로 `required` 키워드 인자 없이 생성되며, 이전 버전의 장고에서는 `required=False` 옵션을 가진다. 이 설정을 의도적으로 다루고 싶다면 명시적으로 serializer 클래스에 `BooleanField`를 선언하거나 `extra_kwargs`옵션으로 `required`를 설정하면 된다.

**사용법 : `BooleanField()`**



##### NullBooleanField

`None`을 허용하는 BooleanField다.

**사용법 : `NullBooleanField()`**



### String fields

##### CharField

문자 표현식으로, 부가적으로 `min_length`, `max_length`에 대한 검증을 거친다.

**사용법 : `CharField(max_length=None, min_length=None, allow_blank=False, trim_whitespace=True)`**

- `max_length` : 문자의 최대 길이
- `min_length` : 문자의 최소 길이
- `allow_blank` : `True`로 설정하면 빈 문자열을 허용. `False`로 설정하면 빈 문자열에 대해 `ValidationError` 발생
- `trim_whitespace` : `True`로 설정하면 앞 뒤의 공백을 제거한다.

`allow_null` 설정도 있지만 `allow_blank` 때문에 잘 사용되지 않는다. `allow_null`과 `allow_blank` 모두 `True`로 설정하는 것이 가능은 하지만, 빈 값에 대한 표현식(빈 공백, null)을 허용하는 것은 데이터의 일관성을 해치고 버그를 유발할 수 있다.



##### EmailField

e-mail 주소 형태의 문자 표현식

**사용법 : `EmailField(max_length=None, min_length=None, allow_blank=False)`**



##### RegexField

정규표현식에 일치하는 문자 표현식

**사용법 : `RegexField(regex, max_length=None, min_length=None, allow_blank=False)`**

`regex` 인자는 문자열이거나 파이썬 정규 표현식 객체다.

검증 과정에서 `django.core.validators.RegexValidator`를 사용한다.



##### SlugField

`[a-zA-Z0-9_-]+` 패턴을 검증하는 `RegexField` 

**사용법 : `SlugField(max_length=50, min_length=None, allow_blank=False)`**



##### URLField

URL 매칭 패턴을 검증하는 `RegexField`

URL 패턴인 `http://<host>/<path>` 값을 예상한다.

**사용법 : `URLField(max_length=200, min_length=None, allow_blank=False)`**



##### UUIDField

유효한 UUID 문자열을 입력값으로 받는 필드

`to_internal_value` 메서드는 `uuid.UUID` 인스턴스를 반환한다. 필드에 저장되는 결과값은 다음과 같이 하이픈된 형태의 문자열이다.

```python
"de305d54-75b4-431b-adb2-eb6b9e546013"
```

**사용법 : UUIDField(format='hex_verbose')**

- `format` : uuid 값의 표현식을 결정한다.
  - `hex_verbose` : 하이픈으로 구분된 16진수 표현식, `5ce0e9a5-5ffa-654b-cee0-1238041fb31a`
  - `hex` : 16진수 표현식, `5ce0e9a55ffa654bcee01238041fb31a`
  - `int` : 정수 표현식, `123456789012312313134124512351145145114`
  - `urn` : RFC 4122 URN 표현식, `urn:uuid:5ce0e9a5-5ffa-654b-cee0-1238041fb31a`

`format` 설정은 표현식의 값에만 영향을 미친다.



##### FilePathField

파일 시스템에 의존하는 파일명과 파일 디렉토리로 제한되는 필드

**사용법 : `FilePathField(path, match=None, recursive=False, allow_files=True, allow_folders=False, required=None, **kwargs)`**

- `path` : `FilePathField`가 가져야할 절대적인 파일시스템 경로(디렉토리)
- `match` : 파일명 필터에 사용할 정규 표현식 문자열
- `recursive` : 경로의 서브디렉토리들의 포함 여부
- `allow_files` : 특정 위치에 있으며 반드시 포함되어야 하는 파일을 특정한다. 기본값은 `True`
- `allow_folders` : 특정 위치에 있으며 반드시 포함되어야 하는 폴더를 특정한다. 기본값은 `False`

`allow_files`와 `allow_folders` 둘 중 하나는 반드시 `True`여야 한다.



##### IPAddressField

IPv4 또는 IPv6 문자열

**사용법 : `IPAddressField(protocol='both', unpack_ipv4=False, **options)`**

- `protocol` : 특정 프로토콜의 입력값으로 제한. 'both', 'IPv4', 'IPv6' 등
- `unpack_ipv4` : `protocol='both'`인 경우에만 사용 가능. 매핑된 IPv4 주소를 언패킹한다. (127.0.0.1)



### Numeric fields

##### IntegerField

정수 표현식

**사용법 : `IntegerField(max_value=None, min_value=None)`**



##### FloatField

부동소수점 표현식

**사용법 : `FloatField(max_value=None, min_value=None)`**



##### DecimalField

파이썬에서 `Decimal` 인스턴스에 해당하는 십진수 표현식

**사용법 : `DecimalField(max_digits, decimal_places, coerce_to_string=None, max_value=None, min_value=None)`**

- `max_digits` : 허용되는 숫자의 최대값. 이 값은 `None`이거나 `decimal_places`보다 크거나 같아야한다.
- `decimal_places` : 숫자를 저장할 십진수의 위치
- `coerce_to_string` : `True`로 설정하면 표현식을 문자열로 반환한다. `False`이면 십진수 객체를 반환한다. 기본적으로 settings key의 `COERCE_DECIMAL_TO_STRING`과 동일한 값을 가지며, 오버라이딩하지 않은 이상 `True`다. `localize` 설정 사용 시 `True`가 된다.
- `localize` : 
- `rounding` : 