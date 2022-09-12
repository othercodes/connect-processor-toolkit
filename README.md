# Connect Processor Toolkit

Bunch of tools to ease the development of Connect Processors.

## Builders

Builders allow us to easily create Request, Assets and TierConfiguration objects using a fluent interface.

```python
from connect.processors_toolkit.requests import RequestBuilder
from connect.processors_toolkit.requests.assets import AssetBuilder

# create a new request object.
request = RequestBuilder()
request.with_id('PR-1234-5678-9000-001')
request.with_type('purchase')
request.with_status('pending')

# create a new asset with some params in it.
asset = AssetBuilder()
asset.with_asset_id('AS-1234-5678-9000')
asset.with_asset_status('processing')
asset.with_asset_param('PARAM_ID_001', 'some-value')

# attach the asset into the request.
request.with_asset(asset)
```

The builders can also be used to read data from the requests, assets or tier configurations.

```python
import os
import json
from connect.processors_toolkit.requests import RequestBuilder

with open(os.path.dirname(__file__) + '/purchase_request.json') as file:
    request_file = json.load(file)

# create a new request object from a request dict.
request = RequestBuilder(request_file)

# access to any request value.
request_id = request.id()
request_note = request.note()

# access to any asset value.
asset = request.asset()
asset_id = asset.asset_id()
subscription_id = asset.asset_param('PARAM_SUBSCRIPTION_ID', 'value')
```
## Configuration Mixins

We can also use the WithConfigurationHelper to easily access to the configuration.

```python
from typing import Dict
from connect.processors_toolkit.configuration.mixins import WithConfigurationHelper
from connect.processors_toolkit.requests import RequestBuilder


class PurchaseFlow(WithConfigurationHelper):
    def __init__(self, config: Dict[str, str]):
        self.config = config

    def process(self, request: RequestBuilder):
        # access to configuration by key.
        api_key = self.configuration('API_KEY')

        # this will raise a MissingConfigurationParameterError exception as the key is missing.
        missing = self.configuration('MISSING_KEY')

```

## Logger Mixins

Sometimes you need to log all the time the request id in each log line, to avoid repeating all the time the id string
you can use the logger mixing `WithBoundedLogger` that will automatically bind the logger instance to a request.

```python
from logging import LoggerAdapter
from connect.processors_toolkit.logger.mixins import WithBoundedLogger
from connect.processors_toolkit.requests import RequestBuilder


class PurchaseFlow(WithBoundedLogger):
    def __init__(self, logger: LoggerAdapter):
        self.logger = logger

    def process(self, request: RequestBuilder):
        self.logger.info('Hello world')  # output: Hello world

        # bind the logger to the request. 
        self.bind_logger(request)

        self.logger.info('Hello world')  # output: PR-1000-2000-3000-4000-001 Hello world

```

## Application

Sometimes you need to create many objects and dependencies during the life cycle of the processor. A dependency
container will help you to decouple dependencies from the use cases. This decoupling allows an easy way to change
implementations and even facilitates the testing cases.

To implement the dependency container within our processor you just need to replace the `Extension` class by
the `Application` in the extension class definition:

```python
from connect.eaas.extension import ProcessingResponse
from connect.processors_toolkit.application import Application


class HTTPServiceClient:
    def __init__(self, api_key: str, api_url: str):
        self.api_key = api_key
        self.api_url = api_url


# use 'Application' instead of 'Extension'
# class MyAwesomeExtension(Extension):
class MyAwesomeExtension(Application):
    def process_asset_purchase_request(self, request):
        self.logger.info(f"Purchase request with id {request['id']}")

        # this will make a new instance of HTTPServiceClient 
        # using the configuration to load api_key and api_url 
        api = self.container.get(HTTPServiceClient)

        return ProcessingResponse.done()

    def process_asset_change_request(self, request):
        self.logger.info(f"Change request with id {request['id']}")
        return ProcessingResponse.done()
```

If the complexity increases you may want to define custom mapping of dependencies, in this case you can configure a
`Dependency` schema in the `dependency()` method.

```python
from connect.processors_toolkit.application import Application, Dependencies
from connect.eaas.extension import ProcessingResponse


class HTTPServiceClient:
    def __init__(self, api_key: str, api_url: str):
        self.api_key = api_key
        self.api_url = api_url


class PurchaseFlow:
    # just declare what dependencies your class need by keyword argument.
    # by default the container injects:
    # - the ConnectClient instance for 'client' keyword argument
    # - the LoggerAdapter instance for 'logger' keyword argument
    # - the configuration dictionary for 'config' keyword argument
    def __init__(self, client, logger, config, api_client):
        self.client = client
        self.logger = logger
        self.config = config
        self.api_client = api_client

    def process(self, request) -> ProcessingResponse:
        self.logger.info(f"Purchase request with id {request['id']}")

        return ProcessingResponse.done()


class MyAwesomeExtension(Application):
    def dependencies(self) -> Dependencies:
        dependencies = Dependencies()
        dependencies.to_class('api_client', HTTPServiceClient)
        dependencies.to_instance('app_code_name', 'cool-app-code-name')

        return dependencies

    def process_asset_purchase_request(self, request):
        # the container will make the PurchaseFlow injecting all required dependencies
        # even our custom HTTPServiceClient that will be injected each time some class 
        # required the keyword argument 'api_client'.
        return self.container.get(PurchaseFlow).process(request)

```

## Controller Dispatcher (Route, Build and Execute)

Along with the DI Container you can use the `WithDispatcher` mixin to add the "route, build and execute the controller"
functionality. This feature is specially useful when implement a big amount of custom events of product actions is
required. The dispatcher will build the instance of the correct class and execute it

```python
from typing import Dict, Type

from connect.eaas.extension import CustomEventResponse
from connect.processors_toolkit.application import Application
from connect.processors_toolkit.application.dispatcher import WithDispatcher
from connect.processors_toolkit.application.contracts import CustomEventTransaction


class HelloWorld(CustomEventTransaction):
    def handle(self, request: dict) -> CustomEventResponse:
        return CustomEventResponse.done(
            http_status=200,
            body=f"Hello World",
        )


class GoodByeWorld(CustomEventTransaction):
    def handle(self, request: dict) -> CustomEventResponse:
        return CustomEventResponse.done(
            http_status=200,
            body=f"GoodBye World",
        )


class MyDummyExtension(Application, WithDispatcher):
    def routes(self) -> Dict[str, Type]:
        # just map the event controllers
        return {
            'product.custom-event.hello-world': HelloWorld,
            'product.custom-event.goodbye-world': GoodByeWorld,
        }

    def process_product_custom_event(self, request):
        # the dispatcher will take care of what custom event flow 
        # controller will be called. If no controller is available
        # a default Not Found Flow Controller will be executed.
        return self.dispatch_custom_event(request)
```

The dispatcher can also handle any other event or process of the Extension:

```python
{
    'asset.process.purchase': PurchaseFlow,
    'asset.process.suspend': SuspendFlow,
    'asset.validate.purchase': PurchaseValidationFlow,
    'tier-config.process.setup': SetUpFlow,
    'tier-config.validate.setup': SetUpValidationFlow,
    'product.custom-event.hello-world': HelloWorldCtrl,
    'product.custom-event.goodbye-world': GoodByeWorldCtrl,
    'product.action.sso': SSOCtrl,
}
```

If your controller class is using the `WithBoundedLogger` mixin the Dispatcher will execute the binding automatically on
creating the instance of the controller.

## Class Contracts

There a set of contracts available to work with, these contracts only define the methods that should be implemented for
each type of controller or transaction.

- ProcessingTransaction
- ValidationTransaction
- ProductActionTransaction
- CustomEventTransaction
- ScheduledTransaction

```python
from connect.eaas.extension import ProcessingResponse
from connect.processors_toolkit.api.mixins import WithAssetHelper, WithProductHelper
from connect.processors_toolkit.application.contracts import ProcessingTransaction
from connect.processors_toolkit.configuration.mixins import WithConfigurationHelper
from connect.processors_toolkit.logger.mixins import WithBoundedLogger
from connect.processors_toolkit.requests import RequestBuilder


class PurchaseFlow(
    ProcessingTransaction,
    WithBoundedLogger,
    WithAssetHelper,
    WithProductHelper,
    WithConfigurationHelper,
):
    def process(self, request: RequestBuilder) -> ProcessingResponse:
        # process the request
        return ProcessingResponse.done()
```

## Asset Helper

### Inquire Request

```python
from connect.eaas.extension import ProcessingResponse
from connect.processors_toolkit.api.mixins import WithAssetHelper
from connect.processors_toolkit.application.contracts import ProcessingTransaction
from connect.processors_toolkit.requests import RequestBuilder


class PurchaseFlow(
    ProcessingTransaction,
    WithAssetHelper,
):
    def process(self, request: RequestBuilder) -> ProcessingResponse:
        self.inquire_asset_request(request, 'FAKE-TEMPLATE-ID')
        return ProcessingResponse.done()
```

### Fail Request

```python
from connect.eaas.extension import ProcessingResponse
from connect.processors_toolkit.api.mixins import WithAssetHelper
from connect.processors_toolkit.application.contracts import ProcessingTransaction
from connect.processors_toolkit.requests import RequestBuilder


class PurchaseFlow(
    ProcessingTransaction,
    WithAssetHelper,
):
    def process(self, request: RequestBuilder) -> ProcessingResponse:
        self.fail_asset_request(request, 'There is a reason')
        return ProcessingResponse.done()
```

### Approve Request

```python
from connect.eaas.extension import ProcessingResponse
from connect.processors_toolkit.api.mixins import WithAssetHelper
from connect.processors_toolkit.application.contracts import ProcessingTransaction
from connect.processors_toolkit.requests import RequestBuilder


class PurchaseFlow(
    ProcessingTransaction,
    WithAssetHelper,
):
    def process(self, request: RequestBuilder) -> ProcessingResponse:
        self.approve_asset_request(request, 'A-VALID-TEMPLATE')
        return ProcessingResponse.done()
```

## License

`Connect Processors Toolkit` is released under
the [Apache License Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).