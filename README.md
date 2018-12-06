[![Build Status](https://travis-ci.com/WillFr/rest-helpers.svg?branch=master)](https://travis-ci.com/WillFr/rest-helpers)

[![Coverage Status](https://coveralls.io/repos/github/WillFr/rest-helpers/badge.svg?branch=master)](https://coveralls.io/github/WillFr/rest-helpers?branch=master)

# rest-helpers

## What is Rest-Helpers
Rest-Helpers is a python 3 library that helps you build REST applications quickly, and in a consistent way.
Overall it provides the following:
- [automated and consistent route creation via decorator](#route-decorator-section)
- [REST API versionner to handle various version your API](#versioner-section)
- [non verbose bindings to parse inputs from query string, body, headers ... including deserialization and validation](#binding-section)
- [Oauth binding: adding auth to a route is as easy as adding the binding](#oauth-binding-section)
- binding [validators](#validation-section) and [deserializers](#deserialization-section)
- [Custom autogenerated swagger UI, fully customizable via method documentation, and integrated with okta](#swagger-section)
- [Server side filtering via json_path and json_filter query string arguments](#serverside-filtering)
- [automatic paging for long response payload](#automatic-paging-section)
- [framework agnostic: it currently supports flask and aiohttp and is easy to extend](#framework-agnostic-section)
- [JsonApi compliant response type](#json-api-section)
- [Meaningful default error messages](#error-messages-section)

eg: the following code snippet will create an appropriate route, document it in swagger and inject your custom modifications, take care of input binding and return meaningful responses if they are not proper, take care of exception, take care of versionning, handle server side filtering and paging, and return a proper json_api response.
```python
@swagger.swagger_object
class PlatformResource(Resource):
    """
    Swagger jsonapi_response:
        name:
            type: "string"
            example: "val_300"
        parent:
            type: "string"
            example: "candidate_ready"
    """

    resource_type = "/worlds/platforms"

    def __init__(self, **kwargs):
        self.__dict__.update(kwargs)

@routes.get_all_resources_route(platform_blueprint, PlatformResource, versionner=platform_versionner.AccretePlatformVersionner, exception_handler=exception_handler.common_exception_handler)
def accrete_platform(
        world_name,
        asof:(as_of_date.AsOfDate,from_query_string)=None,
        uag:from_header(header_field="User-Agent")=None,
        include_platform_source:(bool, from_query_string)=False,
        add_hoc_mixins:(list, from_query_string(as_list=True))= []
    ):
    """
    Swagger doc:
        description: "Display the platform taking in account the inheritance tree."
    end swagger

    Swagger parameters:
        User-Agent: null
    end swagger

    Swagger augment_default:
    parameters>-: ["[2]"]

    Arguments:
        world_name {str} -- The name of the world to be accreted.
    """
    return responses.ok(Platform("my_platform"))

```
<a name="route-decorator-section"></a>

## Automated and consistent route creation via decorator
Rest-Helper follow simple routing principles:
- GET gets you a resource or an array of resource, and you can reuse the response payload for a PUT request in order to modify the resource
- Everything else is an 'operation' and therefore uses the POST verb.

Several routing decorators are provided

### route decorator
Eg:
```python
from rest_helper.flask.routes import routes


@route("/abc", options={"methods": ["GET", "POST"], doc=True, versionner=None, exception_handler=None)
def my_function():
    return response.ok()
```

This is the base decorator. It does not impose any REST related constraint and therefore should be used sparsely. It allows you to define any route.
It takes several *optional* arguments, that are going to be similar to other routes:
- doc : {bool} this indicates whether or not this route should be documented in swagger.
- versionner : {versionnerType} this versionner is going to modify the route as defined in the versionner, more on this in the versionner section.
- exception_handler : {exceptionHandlerType} this is going to define how exceptions should be handled.

### resource based route decorator
Eg:
```python
from rest_helper.flask.routes import routes
from rest_helper.jsonapi_objects import Resource

class HostResource(Resource):
    resource_type = "/clusters/hosts"

@routes.get_resource_route(HostResource, doc=True, versionner=None, exception_handler=None)
def my_function(cluster_name, host_name):
    return response.ok()
```

Resource decorator takes a resource class as a first argument and create routes based on that resources. They will autogenerate
the routing scheme based on the resource_type field that must be present in every class inheriting from Resource.

The route scheme is generated based on the verb and can be modified by a versionner. Eg: a get_resource route:
`/resource_type/resource_name/sub_resource_type/sub_resource_name`

In the example above it will lead to:
`/clusters/cluster_abc/hosts/host_a`

With a url_root_versioner it would become:
`/api_version/resource_type/resource_name/sub_resource_type/sub_resource_name`

The resource type and sub resource type are extracted from the resource_type field that must be present in every class inheriting from Resource.

Recomandation: resource_type should be plural.

This decorator will automatically bind resource names to argument named after de-pluralized resource type:
type: clusters => cluster_name
type: hosts => host_name
type: libraries => library_name

If an argument named resource_id is present, Rest-helpers will bind the resource_id to it.
In our example above:
resource_id => /clusters/cluster_abc/hosts/host_a

A resource id uniquely identifies a resource in the entire API: there must be only one host named host_a in a cluster named "cluster_abc".
A resource name uniquely identifies a resource within the scope of their parent: there can be a host named host_a in another cluster not named "cluster_abc".

This is true for all resource based routing decorators.

### get_resource_route decorator
Eg:
```python
from rest_helper.flask.routes import routes
from rest_helper.jsonapi_objects import Resource

class HostResource(Resource):
    resource_type = "/clusters/hosts"

@routes.get_resource_route(HostResource, doc=True, versionner=None, exception_handler=None)
def my_function(cluster_name, host_name):
    return response.ok()
```
This generates a route to get a single resource:
`GET /resource_type/resource_name/sub_resource_type/sub_resource_name`

In the example above it will lead to:
`GET /clusters/cluster_abc/hosts/host_a`

With a url_root_versioner it would become:
`GET /api_version/resource_type/resource_name/sub_resource_type/sub_resource_name`

### get_all_resources_route decorator
Eg:
```python
from rest_helper.flask.routes import routes
from rest_helper.jsonapi_objects import Resource

class HostResource(Resource):
    resource_type = "/clusters/hosts"

@routes.get_all_resources_route(HostResource, doc=True, versionner=None, page_size=10, exception_handler=None)
def my_function(cluster_name):
    return response.ok()
```
This generates a route to get a list of all resource under a parent (or at the root):
`GET /resource_type/resource_name/sub_resource_type/`

In the example above it will lead to:
`GET /clusters/cluster_abc/hosts/`

With a url_root_versioner it would become:
`GET /api_version/resource_type/resource_name/sub_resource_type/`

This route automatically handle the page_size *optional* argument (infinite by default) if used
in combination with the Rest-helpers response types. The page_size argument of the route indicate
the default value of the page size and can be overriden by a query string argument `page_size`.

In addition, the resouce_id argument will be bound to the parent resource:
in the example above: resource_id => /clusters/cluster_abc

### put_resource_route decorator
Eg:
```python
from rest_helper.flask.routes import routes
from rest_helper.jsonapi_objects import Resource

class HostResource(Resource):
    resource_type = "/clusters/hosts"

@routes.put_resource_route(HostResource, doc=True, versionner=None, exception_handler=None)
def my_function(cluster_name, host_name):
    return response.ok()
```
This generates a route to PUT a resource in order to modify it:
`PUT /resource_type/resource_name/sub_resource_type/sub_resource_name`

In the example above it will lead to:
`PUT /clusters/cluster_abc/hosts/sub_resource_name`

With a url_root_versioner it would become:
`PUT /api_version/resource_type/resource_name/sub_resource_type/sub_resource_name`

put_resource_route is intended to create or modify a resource in an idempotent way.

resource_id is bound similarly to get_resource_route.


### patch_resource_route decorator
Eg:
```python
from rest_helper.flask.routes import routes
from rest_helper.jsonapi_objects import Resource

class HostResource(Resource):
    resource_type = "/clusters/hosts"

@routes.patch_resource_route(HostResource, doc=True, versionner=None, exception_handler=None)
def my_function(cluster_name, host_name):
    return response.ok()
```
This generates a route to PATCH a resource in order to modify it:
`PATCH /resource_type/resource_name/sub_resource_type/sub_resource_name`

In the example above it will lead to:
`PATCH /clusters/cluster_abc/hosts/sub_resource_name`

With a url_root_versioner it would become:
`PATCH /api_version/resource_type/resource_name/sub_resource_type/sub_resource_name`

patch_resource_route is intended to partially update resource in an idempotent way.

resource_id is bound similarly to get_resource_route.



### operation_resource_route
Eg:
```python
from rest_helper.flask.routes import routes
from rest_helper.jsonapi_objects import Resource

class HostResource(Resource):
    resource_type = "/clusters/hosts"

@routes.operation_resource_route(HostResource, operation_name="transform", doc=True, versionner=None, exception_handler=None)
def my_function(cluster_name, host_name):
    return response.ok()
```
This generates a route to PUT a resource in order to modify it:
`POST /resource_type/resource_name/sub_resource_type/sub_resource_name/transform`

In the example above it will lead to:
`POST /clusters/cluster_abc/hosts/sub_resource_name/transform`

With a url_root_versioner it would become:
`POST /api_version/resource_type/resource_name/sub_resource_type/sub_resource_name/transform`

Operation routes transform a resource. It can be done in a readonly way, for example a special view of the resource, or
in a permanent write way where the transformation is applied permanently. Operations can also be used for long running operation, starting a VM for instance.

resource_id is bound similarly to get_resource_route.

### group_operation_resource_route
Eg:
```python
from rest_helper.flask.routes import routes
from rest_helper.jsonapi_objects import Resource

class HostResource(Resource):
    resource_type = "/clusters/hosts"

@routes.group_operation_resource_route(HostResource, operation_name="transform", doc=True, versionner=None, exception_handler=None)
def my_function(cluster_name):
    return response.ok()
```
This generates a route to PUT a resource in order to modify it:
`POST /resource_type/resource_name/sub_resource_type/transform`

In the example above it will lead to:
`POST /clusters/cluster_abc/hosts/transform`

With a url_root_versioner it would become:
`POST /api_version/resource_type/resource_name/sub_resource_type/transform`

Group operations are similar to operations, except they are applied to the list of resources returned by the get_all_resources_route.
resource_id is bound similarly to get_all_resources_route.

### delete_resource_route decorator
Eg:
```python
from rest_helper.flask.routes import routes
from rest_helper.jsonapi_objects import Resource

class HostResource(Resource):
    resource_type = "/clusters/hosts"

@routes.put_resource_route(HostResource, doc=True, versionner=None, exception_handler=None)
def my_function(cluster_name, host_name):
    return response.ok()
```
This generates a route to PUT a resource in order to modify it:
`DELETE /resource_type/resource_name/sub_resource_type/sub_resource_name`

In the example above it will lead to:
`DELETE /clusters/cluster_abc/hosts/sub_resource_name`

With a url_root_versioner it would become:
`DELETE /api_version/resource_type/resource_name/sub_resource_type/sub_resource_name`

Delete operation are to be used to delete a resource permanently.
resource_id is bound similarly to get_resource_route.

<a name="versioner-section"></a>

## Versioner

The versionner allow you to support former versions of your api. It provides several hooks to transform a request and a
response to bring it to a certain version.

Eg: My service support v1 and v2, but internally, all the code supports only latest, which is v2.
A request entering via a route with a versionner will follow this path :
request at v1 -> route -> versionner: transforms a v1 request into a v2 request -> route handler|
response at v1 <- versionner: transforms a v2 response into a v1 response <- route <-|

The service author defines the logic to transform a v1 request into a v2 request.

The following hooks are provided:
- body hook: will take the current request body and return a transformed version
- body dict hook: similar to body, but act on the JSON deserialized dict body
- headers hook: will take the request headers and return a transformed version
- query string args: will take the request query string and return a transformed version
- response: will take the entire response object and return a transformed version
- response body dict: will take the response body dictionary and return a transformed version

### How to define a versioner:
Versioners should inherit from the BaseVersioner class.

They should:
- define a static methos `version_route` that takes a route objet in parameter and inject the version into it
- define inner class named after supported versions and containing hooks definition.
Eg:
```python
class MyVersionner(versioning.BaseVersionner):
    @staticmethod
    def version_route(route):
        route.rule="/<rest_helper_version>"+route.rule

    class v2:
        pass

    class v1:
        def response_body_dict(self, response):
            v2_response["Myfield"] = "hardcoded back compatible field"

            return v2_response
```

Note that you only have to add the hooks that are useful to you.
Complete list of hook:
```python
def body(self, body):
def body_dict(self, body):
def response(self, response)
def headers(self, headers)
def query_string_args(self, query_string_args)
def response_body_dict(self, response_body_dict)
```

### Url versioner
The url versionner is a commonly used versioner injecting the version at the root of the route:
```python
class UrlRootVersionner(BaseVersionner):
    @staticmethod
    def version_route(route):
        route.rule="/<rest_helper_version>"+route.rule
```

By inheriting from it, you only have to define classes representing each version.


<a name="bindings-section"></a>

## Exception handling
Routes all provide an exception_handler field that catches exceptions and respond appropriately.

An exeption handler can be any callable taking exactly one parameter, the exception, and return a response.
A basic exception handler is provided and will be used by default.

eg:
```python
def common_exception_handler(ex):
    if isinstance(ex, rest_exceptions.NotFoundException):
        return responses.not_found()
    elif isinstance(ex, storage.PlatformNotFound):
        return responses.not_found("Platform not found.")
    elif isinstance(ex, exception.ExternalExceptionBase) or isinstance(ex, rest_exceptions.InvalidDataException):
        message = "".join(ex.args).replace("\\n", "\n")
        return responses.bad_request(message)
    elif isinstance(ex, exception.TransactionCheckFailedException):
        return responses.error(409, "Conflict", """
        The client expected the resource ({}) version to be {} but is actually is {}: the latest version
        of the resource must be used to modify it. Get the resource again before attempting to modify it.""".format(ex.resource_id, ex.sha_was_expected_to_be, ex.actual_sha))
    else:
        return responses.base_exception_handler(ex)
```
## Not so verbose bindings

Rest-helper bindings leverage python3 syntax to be as light and close to their target as possible.
Eg:
```python
def get_platform(
        platform_name,
        asof:(as_of_date.AsOfDate,from_query_string),
        uag:from_header(header_field="User-Agent")=None
    ):
```

In the above example, platform_name would come from the route, asof would come from the query string and is to be deserialized into an `as_of_date.AsOfDate`
object, uag come from the request header named "User-Agent". Both asof and uag are optional, uag is optional because it has a default value but asof is not:
not passing the asof query string argument will result into a properly formatted 400 response.

Argument decorator can be either a binding or a tuple containing (type, binding). When a type is specified as the first tuple element, it will be used to
deserialize the value properly and return an appropriate 400 errors if deserialization fails. Custom types are supported:

```python
from rest_helper import type_deserializers
type_deserializers.type_to_deserializer_mapping[as_of_date.AsOfDate] = bindings.as_of_deserializer
```

Each binding detailed further can be customized via several optional arguments, but default value aim to improve readability, in all case, the foreign field (header_field for instance) will be defaulted to the variable name :
```python
asof:(as_of_date.AsOfDate,from_query_string),
```
is equivalent to
```python
asof:(as_of_date.AsOfDate,from_query_string(query_field="asof")),
```

You get bindings for free if you use a route decorator on top of your function. If you want to use only bindings without a route decorator,
you need to decorate your function with `bind_hints`.

Note: all bindings are available as individual decorator as well. If that is your usage, do not forget to specify the field value.

### route bindings
As mentionned in the route section some arguments are automatically bound from the url path to predictable argument_name. This is based on the resource type: for a resource typed as `/worlds/platforms`:
- a get_resource_route expect a world_name argument and a platform_name argument parsed from the uri (notice world was unpluralized)
- a get_all_resource_route expects only a world_name argument because it serves *all* platforms under a specific world.
- operation_resource_route, put_resource_route, delete_resource_route, patch_resource_route will behave like get_resource_route
- group_operation_resource_route will behave like get_all_resources_route

### base_binder
This is not to be used directly but can be extended to create your own binding.

```python
base_binder(field=None, validator=None, deserializer=None, type=None)
```

`field` is name of the variable to be bound. When the value is extracted it is going to be assigned to a function argument named after `field`
`validator` offers a simple way to validate the data. If left to `None`, and if a type is specified, a default type validator will be used. When the
validation fails, the user receives a properly formatted 400 error.
Deserializer will be used to deserialize the data.
Type is a way to infer deserialization logic from the type, more on that on the deserializer section. You cannot specify both type and deserializer.

Important note: type is automatically infered from the decorator. Since validator and deserializer are generally infered from the type, you typically do not need to specify them. Field is also infered from the variable name. As a result, using the binding looks like :
```python
def my_function(my_arg: (bool, from_query_string))
```
This would leverage provided deserializer and validator to properly deserializing boolean from the query string. All of the following would result in `my_arg` being `True`:
- /myroute/?my_arg
- /myroute/?my_arg=True
- /myroute/?my_arg=TRUE
- /myroute/?my_arg=true

### from_json_body
This binding parses the request body as JSON and assign the result to the decorated argument.

eg:
```python
def my_function(
        data:from_json_body
)
```

### field_from_json_body
This binding parses the request body as JSON, look for a specific field, and assign the result to the decorated argument.

Use the argument `json_field` to specify the path to be extracted.
eg:
```python
def my_function(
        data:field_from_json_body(json_field="/data/attributes/my_field")
)
```

### from_header
This binding parses the specified header and assign the result to the decorated argument.

Use the argument `header_field` to parse from a field not named after the argument.
eg:
```python
def my_function(
        uag:from_header(header_field="User-Agent")=None
)
```

### from_query_string
This binding parses the from the query string and assign the result to the decorated argument.

Use the argument `query_field` to parse from a field not named after the argument.
Use the argument `as_list` (boolean) to specify weather you want the result as a list.
eg:
```python
def my_function(
        dryrun:(bool,from_query_string)=False
)
```
<a name="oauth-binding-section" ></a>

### from_Oauth
This binding parses Oauth headers and verify the token based on the open id protocol: it will decode the JWT token, contact the issuer and verify it was signed properly. It will assign the decoded token to the decorated argument.
eg:
```python
okta = {
    "allowed_domains":config["okta_allowed_domains"],
    "client_id":config["okta_client_id"],
    "valid_tokens": config["okta_valid_tokens"],
    "validate_options": {"verify_at_hash": False}
}

def my_function(
        user_auth: from_Oauth(**okta),
)
```

Note: this binding can take a deserializer that will transform the oauth value into another object.

<a name="deserialization-section"></a>

## Deserialization, in detail
Deserialization of inputs can be done in two ways :
- either specified directly in the binding, via the `deserializer` argument
  eg: `dryrun:(bool,from_query_string(deserializer=MyDeserializer))`
- or by linking a deserializer to a specific type: the deserializer will then be used every time this type is deserialized.
  eg:
```python
from rest_helper import type_deserializers
type_deserializers.type_to_deserializer_tuple_list.append((as_of_date.AsOfDate, as_of_deserializer))
```

### What can be used as a deserializer
A deserializer can be any callable that takes exactly one parameter, the raw input, and returns a corresponding object.

### How does the type_deserializers.type_to_deserializer_tuple_list work ?
This list is pretty flexible. It contains tuples where the first element should be a type, and the second element either a deserializer callable (see above), or
a tuple where the first element can generate a deserializer based on the decorator itself:

eg:
```python
type_to_deserializer_tuple_list = [(bool, lambda x: x is not None and (x == "" or x.lower() == "true" ))]
dryrun:(bool,from_query_string)
```

eg:
```python
type_to_deserializer_tuple_list = [(MyDeserializer, lambda x: x is not None and (x == "" or x.lower() == "true" ))]
dryrun:(bool,MyDeserializer())
```

eg: we want to deserialize differently based on the decorator's `a` field value. If a is equal to 1 we want to deserialize the int to its value
time 2, otherwise we want to deserialize the int to its value plus 1
```python
type_to_deserializer_tuple_list = [(MyDeserializer, (lambda decorator: lambda x:x*2 if decorator.a == 1 else lambda x:x+1,))]
dryrun:(int,MyDeserializer(a=1))
```

It is a bit convoluted, but it allows for integration with various framework such as schematics.

Note: type_to_deserializer_tuple_list is a list in order to take advantage of inheritance. The order matters!
Both type based deserializer and instance based deserializer will leverage inheritance, meaning that if you have a deserializer for bool,
it should be *before* the deserializer for int since bool is a subtype of int.

<a name="validation-section"></a>

## Validation, in detail
Validation works very much alike deserialization.

Validation of inputs can be done in two ways :
- either specified directly in the binding, via the `validator` argument
  eg: `dryrun:(bool,from_query_string(validator=MyValidator))`
- or by linking a validator to a specific type: the validator will then be used every time this type is validated.
  eg:
```python
from rest_helper import validators
validators.type_to_validator_tuple_list.append((as_of_date.AsOfDate, as_of_validator))
```

### How is an input validated
An input will be validated twice : a first time before deserialization, and a second time after deserialization.

### What can be used as a validator
A validator can be any callable taking one argument, the object to be validated, and one optional argument `post`. `post` indicates whether the validator
is dealing with pre or post deserialization validation.
A validator must return a tuple where the first element is a boolean indicating whether or not validation was successful, and the second argument should be the
reason why the object is invalid, in case validation was not successful. The reason will be used to display a meaningful message to the user.

### How does the validators.type_to_validator_tuple_list work ?
Please refer to the "How does the type_deserializers.type_to_deserializer_tuple_list work ?" section above work as the validators.type_to_validator_tuple_list
work exactly the same.

### Provided deserializers and validators
By default, the following deserializers are provided:
- bool
- int
- float
- Decimal
- str
- datetime.datetime
- Model: for schematics
- types.BaseType: for schematics

By default, the following validators are provided:
- Model: for schematics
- types.BaseType: for schematics

<a name="swagger-section"></a>

## Automated custom swagger page documentation

Using rest-helpers, you get automated documentation based on routes and bindings. If you use routes and bindings on your entry
point, they will be documented in a swagger document and try-able from a swagger UI page.

Because a lot of APIs will eventually need to integrate with okta, we have added okta integration directly in the swagger UI:
people can authenticate with their username and password from the swagger ui without having to generate a token on their own.

Note: you can "hide" a route by setting `doc=False` in the route decorator arguments.

### What is a "custom swagger ui"? why ?
There are two reasons for going with a custom swagger UI:
- the classic one looks very old and is not the best UX
- it gives us more control which is useful when adding functionalities such as okta integration, or various other UX improvements.

### How to enable swagger and the swagger UI ?

The following code snippet will add a swagger ui at `{host}/{basepath}/` and a swagger document at `{host}/{basepath}/swagger.json`

```python
swagger_service_doc = {
    "info":{
        "description": "This is the service description.",
        "version": "3.0.0",
        "title": "My Service",
        "contact":{
            "email": "cicdteam@abc.com"
        }
    },
    "host": app.config["current_host"],
    "schemes": [app.config["current_scheme"]],
    "basePath": "/api",
    "tags":[{"name": "my_service"}]
}
okta_config= {
    "baseUrl":app.config["okta_base_url"],
    "clientId": app.config["okta_client_id"],
    "redirectUri": app.config["okta_redirect_url"]
}
flask.add_default_swagger_routes(app, swagger_service_doc, okta=okta_config)
```
### response type
Response types are inferred from the route type : it is assumed that a get_resource route will return the associated resource,
and that a get_all_resource_route will return an array of associated resource (following the json_api spec). Rest-helper *does not8 (yet) automatically detect the response schema, so you *must* document the object type that you are returning. To do so, use the `@swagger.swagger_object` decorator and document the object using yaml syntax.

eg:
```python
@swagger.swagger_object
class LibraryResource(Resource):
    """
    Swagger jsonapi_response:
        name:
            type: "string"
            example: "service_name"
        version_pins:
            type: dictionary
            example: {"shared-version://my_shared_service": 1.master.1}
    end swagger
    """
```

You do not need to document the json_api part of the response, only the object itself.

*Why not automate this process ?*
Outside of very specific case, we believe it is next to impossible to automate *good* documentation. Good documentation
implies proper examples, explanations etc. We might however provide a *basic* documentation in the future, just like we do
with parameters


### How to customize a route

The automated swagger UI is a "best effort". There will be some case where you will want to modify what has been
automatically created. To do so, rest-helper relies on method documentation, just like we did with Resource object.

5 keywords are used to achieve different goals:
- doc: gives you a way to update (see the python update function) the entire swagger associated swagger path dictionary. Can be used on route method documentation.
- parameters: gives you a way to update (see the python update function) a specific parameter dictionary, or to not document it by passing null. eg:
```
This will ensure the user-agent parameter is not documented
Swagger parameters:
    User-Agent: null
```
Can be used on route method documentation
- extra_definition: this is to be used to add extra object definitions (not just json_api object, any kind), it is typically useful when you need subobject to defin complex objects. This can be used either in route method documentation or response object documentation.
- jsonapi_response: this is to be used only to document response object (see previous section)
- augment_default: this is the most complex and powerful one : its intent is to provide a way to augment the default dictionary *and lists*. It can be applied to either route method or response object and work as follow :
- - with regular syntax it works like the python update method but also updates nested dictionaries
- - you can also update a speific index in a list
- - you can also append to a list with
- - you can also delete from a list by index or by elem
To understand how augment default works, it is best to look at the corresponding tests in the test_swagger.py file

In the documentation of a route method, use the following to customize its swagger doc:
```
Swagger <keyword>:
<yaml>
end swagger
```

eg:
```python
@routes.operation_resource_route(platform_blueprint, PlatformResource, operation_name="accrete",versionner=platform_versionner.AccretePlatformVersionner, exception_handler=exception_handler.common_exception_handler)
def accrete_platform(
        platform_name,
        asof:(as_of_date.AsOfDate,from_query_string)=None,
        uag:from_header(header_field="User-Agent")=None,
        include_platform_source:(bool, from_query_string)=False,
        add_hoc_mixins:(list, from_query_string(as_list=True))= []
    ):
    """
    Swagger doc:
        description: "Display the platform taking in account the inheritance tree."
    end swagger

    Swagger parameters:
        User-Agent: null
    end swagger

    Swagger augment_default:
    parameters>-: ["[2]"]

    Arguments:
        platform_name {str} -- The name of the platform to be accreted.
    """

```

<a name="serverside-filtering"></a>

## Server side filtering

When a client is interested only in a very specific part of the response, sending back an entire response is a waste of resource: serializing it, putting it on the network and deserializing it are all significant costs that can be avoided. Specialized libraries like GraphQL do that extremly well but can be heavy to implement. Rest-helper implement a poor man's server side filtering via the json_path query string argument supported on all route methods that return a Response json_api object. While simplistic in nature, it has proven to fit most basic needs.

It supports key name, list index, `*` (foreach) segments, and filtered foreach segments .

eg:
```
GET /resources/name
>>>
{
    data:[
        {
            "name"="name_1"
            "value"={"a":1,"b":2}
        },
        {
            "name"="name_2"
            "value"={"a":3,"b":4}
        },
    ]
}

GET /resources/name?json_path=/0/value/a
>>>
{
    data:1
}

GET /resources/name?json_path=/*/value/b
>>>
{
    data:[2,4]
}

GET /resources/name?json_path=/*:value>a~=(3|4)/value/b
>>>
{
    data:[4]
}
```

### How does array filtering work
In the last example, we used array filtering to select only some elements of the array.
When traversing an array with `*` you can specify a filter:
`*:path>to>elements>inside>the>array{operator}{value}`

currently the follwing operators are supported:
- `==` for equality
- `!=` for different
- `~=` for regexes

This is very basic and does not handle cases where the path does not exist on some elements.

<a name="automatic-paging-section"></a>

## Automatic paging

Similarly to resource filtering, paging is a common use case: returning arrays with thousands of elements is usually a waste of resource. Rest-helpers supports automatic paging for route methods returning a Response object. Paging is based on a page number and a page size. The page number comes from the `page` query string argument. The page size comes from :
1. the `page_size` query string argument if present
2. the `page_size` field used in the Response constructor if the query string argument is not present
3. the `page_size` field used in the route decorator if none of the above is present.

Note: paging will not happen if page_size is not provided somehow.

<a name="json-api-section"></a>

## JSON API responses

Rest-helpers offers support for JSON API compliant responses. More details about JSON API here: http://jsonapi.org. This will help to provide standard response schema making service interoperability easier.

### Resource base class
To take advantage of the JSON API spec implementation, your resource model object should simply inherit from the `Resource` class and use the super constructor.

The resource constructor accepts optional arguments to fully support the JSON API spec.
```python
class Resource(object):
    def __init__(self, resource_type, resource_name, relationships=None, links=None, meta=None, parent=None)
```

- `relationships` should be a dictionary of related resources, keyed by name
- `links` should be a dictionaty of related links, keyed by name
- `meta` can be any object loosely related to the resource
- `parent` is used in the case of a nested resource

A resource is identified uniquely by the field id constructed as follows:
```
id = /grand_parent_type_plural/grand_parent_resource_name/parent_type_plural/parent_name/type_plural/name
type = /grand_parent_type_plural/parent_type_plural/type_plural
```
A resource can be nested under a parent resource where it makes sense.
Eg:
```
id = /authors/mtwain/books/tomsawyer
type = /authors/books
```
call: `Resource("books","tomsawyer",parent=MTwainResourceObject)`

### Standard response
Standard responses are built on top of the JSON API resource and are defined in the `responses` module.
```python
responses.ok(my_resource)
response.created(just_created_resource)
response.bad_request(Exception("The body of the request is incorrect"))
```

<a name="framework-agnostic-section"></a>

## Framework agnostic

Rest-helper attempts to be framework agnostic: we currently support aiohttp and flask and could support any framework that support the ~100 lines adapter created for aiohttp and flask.

### How does it work
Rest helpers implement a combinaition or adapter pattern and proxy pattern. For both flask and aiohttp, we created an adapter
implementing the interface defined in `framework_adapter.py`. The rest of the code does not use any framework specific logic, but takes a framework adapter as the first argument of most methods. proxies are used to make this transparent to the end user: when importing the `response` module from `rest_helpers.flask` you actually proxy `rest_helpers.responses` but inject a framework adapter in every call.

This approach makes the pattern easily extensible, roughly a 100 lines are likely needed to onboard a new framework.

<a name="error-messages-section"></a>

## Meaningful error message

One of the philosophy behind rest-helper is to automate error messages as much as possible in order to provide meaningful error messages. As much as possible, error case lead to error message that tell the user how to correct it. We try to not respond with a blank 400 or 500 but explain in detail what failed.