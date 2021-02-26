# Parsers

```
기계와 상호작용하는 웹 서비스는 간단한 폼으로 복잡한 데이터를 보내게 된 이래로 폼의 형식에 맞춘 것보다 더 구체화된 형태로 데이터를 보내는 경향이 있습니다.
- Malcom Tredinnick, 장고 개발자 그룹
```

DRF는 다양한 빌트인 파서 클래스들을 제공하며, 다양한 미디어 타입의 요구에 맞춰져 있습니다. 또한, API가 받아들이는 미디어 타입에 유연하게 대처하기 위해 커스텀 파서를 정의하는 기능도 지원하고 있습니다.



### How the parser is determined

뷰를 위한 유효한 파서의 구성은 클래스들의 리스트로 정의됩니다. `request.data`가 들어오면 DRF는 요청이 들어온 데이터의 `Content-Type` 헤더를 검사하고, 이에 요청 컨텐츠를 적절하게 파싱할 수 있는 Parser를 결정합니다.

클라이언트 애플리케이션을 개발하는 개발자는 반드시 HTTP 요청을 보낼 시 `Content-Type`을 설정할 것을 고려해야 합니다. 이를 명시하지 않으면 대부분의 클라이언트들이 `Content-Type`을 기본적으로 `application/x-www-form-urlencoded`로 설정됩니다. 대표적인 예로 `json`을 통한 ajax 통신에 대한 코드를 작성하려면 `ContentType : application/json`으로 설정해야합니다.



### Setting the parsers

전역적으로 사용되는 파서의 기본 구성은 `DEFAULT_PARSER_CLASS`입니다. 예를 들어, 폼 데이터나 JSON이 아닌 DRF에서 제공하는 파서를 통해 `JSON` 데이터를 다룬다면

```python
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
    ]
}
```

또한, 각각의 view, viewset에 대하여 `APIView` 클래스 기반 뷰를 통해 파서를 설정할 수 있습니다.

```python
from rest_framework.parsers import JSONParser
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    """
    JSON 컨텐츠를 POST 요청으로 받는 뷰
    """
    parser_classes = [JSONParser] # JSON 파서를 사용

    def post(self, request, format=None):
        return Response({'received data': request.data})
```

또한, 함수 기반 뷰에서는 `@api_view` 데코레이터를 사용합니다.

```python
from rest_framework.decorators import api_view
from rest_framework.decorators import parser_classes
from rest_framework.parsers import JSONParser

@api_view(['POST'])
@parser_classes([JSONParser])
def example_view(request, format=None):
    """
    A view that can accept POST requests with JSON content.
    """
    return Response({'received data': request.data})
```



### API Reference

##### JSONParser

`JSON` 컨텐츠 요청을 파싱합니다.

**.media_type** : `application/json`



##### FormParser

HTML form 컨텐츠를 파싱합니다. `request.data`는 데이터의 `QueryDict`로 파싱됩니다.

HTML form에서 다루는 모든 데이터를 지원하기 위해서 `FormParser`와 `MultiPartParser`를 함께 사용할 수 있습니다.

**.media_type** : `application/x-www-form-urlencoded`



##### MultiPartParser

파일 업로드와 관련된 multipart HTML form 컨텐츠를 파싱합니다. 마찬가지로 `request.data`는 데이터의 `QueryDict`로 파싱됩니다.

HTML form에서 다루는 모든 데이터를 지원하기 위해서 `FormParser`와 `MultiPartParser`를 함께 사용할 수 있습니다.

**.media_type** : `multipart/form-data`



##### FileUploadParser

일반 파일 컨텐츠를 업로드할 때 사용하는 파서입니다. `request.data` 프로퍼티는 업로드되는 파일이 담겨져 있으며 단일 키 값 `file`을 갖는 딕셔너리로 변환됩니다.

만약 뷰에서 `filename` URL 키워드 인자와 함께 호출된 `FileUploadParser`를 사용한다면 인자의 값은 파일 이름으로 사용됩니다.

`filename` URL 키워드 인자 없이 호출되었다면, 클라이언트는 `Content-Disposition` HTTP 헤더를 통해 파일명을 설정해야합니다. 예를 들어, `Content-Disposition: attachment; filename=upload.jpg`

**.media_type** : `*/*`

- `FileUploadParser`는 원래의 데이터 그대로 파일을 업로드하는 순수 클라이언트를 위해 사용됩니다. 웹 기반의 업로드 또는 여러 종류의 파일 업로드를 지원하려면 `MultiPartParser`를 사용해야합니다.
- 파서의 `media_type`이 어떤 컨텐츠 타입과도 매치되므로, `FileUploadParser`는 일반적으로 API View에 설정되는 유일한 파서입니다.
- `FileUploadParser`는 장고의 기본 `FILE_UPLOAD_HANDLERS` 세팅과 `request.upload_handlers` 인자를 따라갑니다.

```python
# views.py
class FileUploadView(views.APIView):
    parser_classes = [FileUploadParser]

    def put(self, request, filename, format=None):
        file_obj = request.data['file']
        # ...
        # do some stuff with uploaded file
        # ...
        return Response(status=204)

# urls.py
urlpatterns = [
    # ...
    re_path(r'^upload/(?P<filename>[^/]+)$', FileUploadView.as_view())
]
```



### Custom Parsers

커스텀 파서를 적용하기 위해 `BaseParser`를 오버라이딩하고, `.media_type` 프로퍼티를 설정한 뒤, `.parse(self, stream, media_type, parser_context)` 메서드를 적용합니다.

메서드의 반환값은 `request.data` 프로퍼티로부터 생성된 데이터가 되어야합니다.

`.parse()`의 인자는 다음과 같습니다.

- `stream` : 요청 바디에 있는 stream 형태의 객체
- `media_type` : 옵션으로, 설정되었다면 요청으로 들어온 컨텐츠의 미디어 타입을 결정합니다. 요청의 `Content-Type` 헤더에 따라, 렌더러에 의해 정해지는 `media_type` 인자보다 더 특정지어지는 미디어 타입 파라미터를 포함합니다.
- `parser_context` : 옵션으로, 설정되었다면 요청 컨텐츠를 파싱하는 과정에서 추가적으로 필요한 컨텍스트들이 딕셔너리 형태로 제공됩니다. 이 프로퍼티는 기본적으로 다음 기본 키값을 가집니다. - `view`, `request`, `args`, `kwargs`

```python
class PlainTextParser(BaseParser):
    """
    평범한 텍스트 파서
    """
    media_type = 'text/plain' # 미디어 타입 설정

    def parse(self, stream, media_type=None, parser_context=None):
        """
        단순히 요청 바디에 있는 문자열을 표시
        """
        return stream.read()
```

