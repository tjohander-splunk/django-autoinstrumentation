# Problem
This is cloned from [opentelemetry-python](https://github.com/open-telemetry/opentelemetry-python)

The Splunk Opentelemetry Python auto-insturmentation does not send traces in to O

# Otel Collector Setup
A Collector instance is running as a Docker container, bootstrapped with `docker-compose`.  This is the config.  Enviroment variables provided in an `.env` file, config provided in a mounted local volume. The setup has been validated to work with other local, instrumented applications.
```yaml
  splunk-otel-collector:
    image: quay.io/signalfx/splunk-otel-collector:latest
    ports:
      - "13133:13133"
      - "14250:14250"
      - "14268:14268"
      - "4317:4317"
      - "6060:6060"
      - "8888:8888"
      - "9080:9080"
      - "9411:9411"
      - "9943:9943"
    volumes:
      - ./docker-volumes/otel:/etc/otel/collector
    env_file:
      - ./docker-volumes/otel/.env
```

# Django Server
Environment Variables in Terminal session 1:
```bash
python -m venv venv-server
source venv-client/bin/activate
pip install django
pip install 'splunk-opentelemetry[all]' "protobuf == 3.20.1" <----This seems to help getting the sqllite spans into O11y
splunk-py-trace-bootstrap

export OTEL_SERVICE_NAME='django-example-server'
export OTEL_EXPORTER_OTLP_ENDPOINT='http://localhost:4317'
export OTEL_RESOURCE_ATTRIBUTES='deployment.environment=tj-devlab'
export DJANGO_SETTINGS_MODULE='instrumentation_example.settings'

splunk-py-trace python3 ./manage.py runserver --noreload
```

This is the Django project booting up:
```bash
(venv)  tjohander@C02FV37YMD6P  ~/splunk/projects/django-autoinstrumentation  splunk-py-trace python3 ./manage.py runserver --noreload
Performing system checks...

System check identified no issues (0 silenced).
July 14, 2022 - 13:28:31
Django version 3.2.14, using settings 'instrumentation_example.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.

```

# "Vanilla" Python Client
Commands execute to configure and bootstrap a pure-vanilla Python client in a separate window:
```bash
python -m venv venv-client
source venv-client/bin/activate
pip install opentelemetry-sdk
pip install opentelemetry-instrumentation-django
pip install requests

export OTEL_SERVICE_NAME='django-example-client'
export OTEL_EXPORTER_OTLP_ENDPOINT='http://localhost:4317'
export OTEL_RESOURCE_ATTRIBUTES='deployment.environment=tj-devlab'

python client.py hello
```
The Client outputs spans to the console:
```json
{
    "name": "client-server",
    "context": {
        "trace_id": "0x7c58c69c789d08fca93a1aaa2a8f4122",
        "span_id": "0x9b60edf18db15457",
        "trace_state": "[]"
    },
    "kind": "SpanKind.INTERNAL",
    "parent_id": "0x7c20cc1b074a34c0",
    "start_time": "2022-07-14T13:48:33.651197Z",
    "end_time": "2022-07-14T13:48:34.662826Z",
    "status": {
        "status_code": "UNSET"
    },
    "attributes": {},
    "events": [],
    "links": [],
    "resource": {
        "telemetry.sdk.language": "python",
        "telemetry.sdk.name": "opentelemetry",
        "telemetry.sdk.version": "1.11.1",
        "deployment.environment": "tj-devlab",
        "service.name": "django-example-client"
    }
}

```

# Troublehsooting Steps
* Tried downgrading to Django LTS 
* Tried manual instrumentations with:
```python
from opentelemetry.instrumentation.django import DjangoInstrumentor <-----------


def main():
    os.environ.setdefault(
        "DJANGO_SETTINGS_MODULE", "instrumentation_example.settings"
    )

    DjangoInstrumentor().instrument()    <----------------- 

    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)


if __name__ == "__main__":
    main()
```