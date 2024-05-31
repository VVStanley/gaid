# Настройка opentelemetry в Авто. Часть 1

Всем привет! Этот гайд будет основан на подключении телеметрии в проекте Авто, у нас используется Django. 
Opentelemetry - это набор библиотек и инструментов предназначенные для сбора метрик, логов и трассировки, подробнее можно почитать на [ВИКИ](https://wiki.skbkontur.ru/pages/viewpage.action?pageId=649627602).


## Установка необходимых зависимостей
**Пакеты Контура:**

- `httptoolkitopentelemetry`
Для сбора данных с нашей библиотеки kontur_http_toolkit в нее интегрирована [opentelemetry-instrumentation](https://git.skbkontur.ru/py-libs/kontur_http_toolkit/-/tree/master/kontur/httptoolkitopentelemetry?ref_type=heads) В readme указанно как ее установить.

- `kontur_opentelemetry`
Эта библиотека добавляет propagators: kontur и vostok. propagators - это такие механизмы, которые позволяют переносить трассировки и метрики через границы сервисов с помощью заголовков

**Сторонние пакеты:**

- `opentelemetry-instrumentation-django`
Пакет для автоматического инструментирования Django-приложений нужен для интеграции Opentelemetry в проекты на Django

- `opentelemetry-sdk`
Пакет для сбора, обработки и экспорта данных телеметрии для Python, включает в себя TracerProvider и SpanProcessor

- `opentelemetry-exporter-otlp-proto-grpc`
Используется для экспорта собранных телеметрических данных в коллектор или систему мониторинга

- `opentelemetry-instrumentation-httpx` ??????


## Подключение

Подключение происходит в файле manage.py

```python
from kontur.httptoolkitopentelemetry import HttpxTransportInstrumentor
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.django import DjangoInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

if __name__ == "__main__":
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "auto.settings")

    # Создание и настройка TracerProvider
    provider = TracerProvider()

    # Регистрируем спан процессор
    provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))

    # Для отладки можно вывести данные трассировки в консоль
    # provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
    
    # Установка созданного TracerProvider в качестве глобального провайдера
    trace.set_tracer_provider(provider)
    
    # Инструментирование Django
    DjangoInstrumentor().instrument(tracer_provider=provider)
    
    # Инструментирование http toolkit
    HttpxTransportInstrumentor.instrument(tracer_provider=provider)

    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)
```

Настройки енв
```python

# Прописываем пропогаторы которые будут использоваться для контекстной передачи данных
OTEL_PROPAGATORS: "tracecontext,kontur,vostok"
# Настройка сжатия данных при отправке телеметрических данных через OTLP
OTEL_EXPORTER_OTLP_COMPRESSION: "gzip"
# Точка к которой экспортёр должен отправлять телеметрические данные
OTEL_EXPORTER_OTLP_ENDPOINT: "https://opentm-cloud.kontur.host"
# Сервис, который генерирует телеметрические данные
OTEL_SERVICE_NAME: "django-auto"
```



Теперь в заголовках запроса можно получить Trace-Id
```
{
  'X-Kontur-Trace-Id': '15faec191ff5763544c32caa7c8aab4c'
}
```

И просмотреть результат трассировки в сервисе https://mon.kontur.ru/contrails/

Например: https://mon.kontur.ru/contrails/23d72662837132063bb2cf7df4e2a01d

![img.png](img.png)

Надеюсь эта статья поможет с настройкой телеметрии. Всем добра!