## Getting started

For this tutorial you will need

* JDK (I advise 8, but you can take 7 as well).

* Git.

* Grails 3.2.11 (you can install it with http://sdkman.io on most Unix based systems.)

After all is installed clone the repo:

```
git clone https://github.com/9ci/angle-grinder
```

and switch to branch `rest_tutorial` branch, and go to `angle-grinder/grails/restTutorial`, the final result is in
the `snapshot` folder for each step

So first let's create new grails app:

```
$ grails create-app -profile rest-api -features hibernate4 resttutorail
Application created at angle-grinder/grails/restTutorial
```

Grails 3 provides several different profiles you can read about them in the [grails docs]

## 1. Creating an API with Grails Web Services

As described in the [grails ws docs]
we will use the default out of the box functionality as a starting point.

### 1.1 Creating a GORM domain

```
grails create-domain-class Contact
```

Then set it up like so:

**Contact.groovy**
```groovy
package resttutorial

class Contact {
  String firstName
  String lastName
  String email
  Boolean inactive

  static constraints = {
    firstName nullable: false
    inactive bindable: false
  }
}
```

To avoid writing `nullable: true` we will set the default to allow nulls for fields
Add the following to `grails-app/conf/application.groovy`

**application.groovy**
```groovy
grails.gorm.default.constraints = {
  '*' (nullable: true, blank: true)
}
```

We will load 100 rows of mock test data from a file `Contacts.json` in resources.
The mock data was generated from a great tool https://www.mockaroo.com

Add the following code to `grails-app/init/BootStrap.groovy`

**BootStrap.groovy**
```groovy
package resttutorial

import groovy.json.JsonSlurper

class BootStrap {
    def grailsApplication
    def init = { servletContext ->
        def data = new JsonSlurper().parse(new File("../resources/Contacts.json"))
        data.each{
          Contact contact = new Contact(it)
          contact.save(failOnError:true,flush: true)
        }
    }
    def destroy = {
    }
}
```

#### 1.2 Adding the `@Resource` annotation to our domain
:url-dr: {docs-grails}#domainResources

So now we can start working on creating REST Api for our app.
The easiest way is to use {url-dr}[domain resources].
So as we see from {url-dr}[docs] we just need to update our domain a bit (just add {docs-grails-api}/grails/rest/Resource.html[@Resource] anotation) in such a way:

**Contact.groovy**
```groovy
import grails.rest.Resource

@Resource(uri = '/contact', formats = ["json"])
class Contact {
  ...
}
```

> :memo: **On plural resource names**
As you will notice we did not pluralize it to contacts above as many will do.
We are aware of the debate on this in the rest world. We feel this will cause confusion down the line to do it.
>1. English plural rules like "cherry/cherries" or "goose/geese/moose/meese" are not the nicest thing to think of while developing API, particularly when english is not your mother tongue.
>2. Many times, as in Grails, we want to generate endpoint from the model, which is usually singular. It does not play nicely with the above pluralization exceptions and creates more work maintaining UrlMappings.
>3. When the model is singular, which is normally is for us, keeping the rest endpoint singular will have the rest developers and the grails developers speaking the same language
>4. The argument "usually you start querying by a Get to display a list" does not refer to any real use case. And we will end up querying single items as much as and even more than a list of items.

##### The `RestfullController`

`@Resource` creates a RestfullController for the domain

> :bulb: **The `@Resource` annotation**  
> is used in an ASTTransformation that creates a controller that extends RestfullController. See [ResourceTransform](https://github.com/grails/grails-core/blob/master/grails-plugin-rest/src/main/groovy/org/grails/plugins/web/rest/transform/ResourceTransform.groovy) for details on how it does this. Later we will show how to specify the controller to user with superClass property.

### 1.3 Default Endpoints and Status Codes

#### Url Mappings

The [Extending Restful Controllers](http://docs.grails.org/3.2.11/guide/webServices.html#extendingRestfulController) section of the [grails docs] outlines the action names and the URIs they map to:

Table 1. URI, Controller Action and Response Defaults

| URI | Method | Action | Response Data|
|-----|--------|--------|--------------|
| /contact              | GET | index | Paged List
| /contact/create       | GET | create | Contact.newInstance() unsaved
| /contact              | POST | save | The successfully saved contact (same as show's get)
| /contact/${id}        | GET | show | The contact for the id
| /contact/${id}/edit   | GET | edit | The contact for the id. same as show
| /contact/${id}        | PUT | update | The successfully updated contact
| /contact/${id}        | DELETE | delete | Empty response with HTTP status code 204


#### Status Code Defaults

Piecing together the {docs-HttpStatus}[HttpStatus codes] and results from RestfullController, RestResponder and _errors.gson,
these are what looks like the out of the box status codes as of Grails 3.2.2

Table 2. Status Codes Out Of Box

| Status Code               | Description
|------|-------------------|
| 200 - OK                  | Everything worked as expected. default
| 201 - CREATED             | Resource/instance was created. returned from `save` action
| 204 - NO_CONTENT          | response code on successful DELETE request
| 404 - NOT_FOUND           | The requested resource doesn't exist.
| 405 - METHOD_NOT_ALLOWED  | If method (GET,POST,etc..) is not setup in `static allowedMethods` for action or resource is read only
| 406 - NOT_ACCEPTABLE      | Accept header requests a response in an unsupported format. not configed in mime-types. RestResponder uses this
| 422 - UNPROCESSABLE_ENTITY | Validation errors.



### 1.4 API Namespace

A Namespace is a mechanism to partition resources into a logically named group.

So the controllers that response for the REST endpoints we will move to separate namespace to avoid cases when we need to
have Controllers for GSP rendering or some other not related to REST stuff.

As a our preferred namespace design we will use the "api" namespace prefix for the rest of the tutorial.
So we will add `namespace = 'api'` on the contact @Resource. @Resource has also property `uri` but it will override namespace property,
for example if @Resource(namespace = 'api', uri='contacts', formats = ["json"]) url for resource will be `localhost:8080/contacts`, not

**Contact.groovy**
```groovy
@Resource(namespace = 'api', formats = ["json"])
class Contact
```

Also we need to update UrlMappings.groovy, there are two ways:

1. Add `/api` prefix to each mapping for example  `get "/api/$controller(.$format)?"(action:"index")`
2. Use `group` property

We will use the second case:

**UrlMappings.groovy**
```groovy
package resttutorial

class UrlMappings {

    static mappings = {
      group("/api") {
        delete "/$controller/$id(.$format)?"(action:"delete")
        get "/$controller(.$format)?"(action:"index")
        get "/$controller/$id(.$format)?"(action:"show")
        post "/$controller(.$format)?"(action:"save")
        put "/$controller/$id(.$format)?"(action:"update")
        patch "/$controller/$id(.$format)?"(action:"patch")
      }
        ...
    }
}
```

You can see all available endpoints that Grails create for us with url-mappings-report:


```

$ grails url-mappings-report

Dynamic Mappings
 |    *     | ERROR: 500                                | View:   /error           |
 |    *     | ERROR: 404                                | View:   /notFound        |
 |   GET    | /api/${controller}(.${format)?            | Action: index            |
 |   POST   | /api/${controller}(.${format)?            | Action: save             |
 |  DELETE  | /api/${controller}/${id}(.${format)?      | Action: delete           |
 |   GET    | /api/${controller}/${id}(.${format)?      | Action: show             |
 |   PUT    | /api/${controller}/${id}(.${format)?      | Action: update           |
 |  PATCH   | /api/${controller}/${id}(.${format)?      | Action: patch            |

Controller: application
 |    *     | /                                                  | Action: index            |

Controller: contact
 |   GET    | /api/contact/create                                | Action: create           |
 |   GET    | /api/contact/${id}/edit                            | Action: edit             |
 |   POST   | /api/contact                                       | Action: save             |
 |   GET    | /api/contact                                       | Action: index            |
 |  DELETE  | /api/contact/${id}                                 | Action: delete           |
 |  PATCH   | /api/contact/${id}                                 | Action: patch            |
 |   PUT    | /api/contact/${id}                                 | Action: update           |
 |   GET    | /api/contact/${id}                                 | Action: show             |

```


### 1.5 Using CURL to test CRUD and List

Fire up the app with `run-app`

##### GET (list):
```
curl -i -X GET -H "Content-Type: application/json"  localhost:8080/api/contact
HTTP/1.1 200
X-Application-Context: application:development
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 31 Jul 2017 12:30:31 GMT

[{"id":1,"email":"mscott0@ameblo.jp","firstName":"Marie","lastName":"Scott"},{"id":2,"email":"jrodriguez1@scribd.com","firstName":"Joseph","lastName":"Rodriguez"}, ...
```

##### POST:
```
curl -i -X POST -H "Content-Type: application/json" -d '{"firstName":"Joe", "lastName": "Cool"}' localhost:8080/api/contact
HTTP/1.1 201
X-Application-Context: application:development
Location: http://localhost:8080/api/contact/101
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 31 Jul 2017 12:30:44 GMT

{"id":101,"firstName":"Joe","lastName":"Cool"}
```
##### GET (by id):
```
curl -i -X GET -H "Content-Type: application/json"  localhost:8080/api/contact/101
HTTP/1.1 200
X-Application-Context: application:development
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 31 Jul 2017 12:31:00 GMT

{"id":101,"firstName":"Joe","lastName":"Cool"}
```

##### PUT:
```
curl -i -X PUT -H "Content-Type: application/json" -d '{"firstName": "New Name", "lastName": "New Last name"}' localhost:8080/api/contact/101
HTTP/1.1 200
X-Application-Context: application:development
Location: http://localhost:8080/api/contact/101
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 31 Jul 2017 12:32:01 GMT

{"id":101,"firstName":"New Name","lastName":"New Last name"}
```

##### DELETE:
```
curl -i -X DELETE -H "Content-Type: application/json"  localhost:8080/api/contact/50
HTTP/1.1 204
X-Application-Context: application:development
Date: Mon, 31 Jul 2017 12:32:24 GMT
```

##### 422 - Post Validation Error:
```
curl -i -X POST -H "Content-Type: application/json" -d '{"lastName": "Cool"}' localhost:8080/api/contact
HTTP/1.1 422
X-Application-Context: application:development
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 31 Jul 2017 12:32:41 GMT

{"message":"Property [firstName] of class [class resttutorial.Contact] cannot be null","path":"/contact/index","_links":{"self":{"href":"http://localhost:8080/contact/index"}}}
```

##### 404 - Get Error:
```
curl -i -X GET -H "Content-Type: application/json"  localhost:8080/api/contact/105
HTTP/1.1 404
X-Application-Context: application:development
Content-Type: application/json;charset=UTF-8
Content-Language: en-US
Transfer-Encoding: chunked
Date: Mon, 31 Jul 2017 12:32:55 GMT

{"message":"Not Found","error":404}
```

##### 406 - NOT_ACCEPTABLE:

We did not setup XML support so we will get a 406. You may try adding XML to formats to see if this.
```
curl -i -X GET -H "Accept: application/xml"  http://localhost:8080/api/contact/8
HTTP/1.1 406
X-Application-Context: application:development
Content-Length: 0
Date: Mon, 31 Jul 2017 12:33:13 GMT
```

### 1.6 Functional Tests for the API

The next step is to add functional tests for our app. One option is to use Grails functional tests and RestBuilder.
We will cover another javscript option later the angle-grinder section
The line in the buidl.gradle that allows us to use RestBuilder is
```
testCompile "org.grails:grails-datastore-rest-client"
```

it is added by default when you create a grails app with `-profile rest-api`

#### POST testing example

Here is an example of `POST` request (creating of a new contact).
RestBuilder we use to emulate request from external source. Note, in Grails3 integration tests run on the random port,
so you cant call `http://localhost:8080/api/contact` , but we can use `serverPort` variable instead. And to make it more
intelligent lets use baseUrl. See example:

**ContactSpec.groovy**
```groovy
package resttutorial

import grails.plugins.rest.client.RestBuilder
import grails.plugins.rest.client.RestResponse
import grails.test.mixin.integration.Integration
import org.grails.web.json.JSONElement
import spock.lang.Shared
import spock.lang.Specification

@Integration
class ContactSpec extends Specification {

    @Shared
    RestBuilder rest = new RestBuilder()

    def getBaseUrl(){"http://localhost:${serverPort}/api"}

    void "check POST request"() {
        when:
        RestResponse response = rest.post("${baseUrl}/contact"){
          json([
            firstName: "Test contact",
            email:"foo@bar.com",
            inactive:true //is bindable: false - see domain, so it wont be set to contact
          ])
        }

        then:
        response.status == 201
        JSONElement json = response.json
        json.id == 101
        json.firstName == "Test contact"
        json.lastName == null
        json.email == "foo@bar.com"
        json.inactive == null
    }
}
```

More tests examples are in the snapshot1 project's
{url-snapshot1}/src/integration-test/groovy/resttutorial/ContactSpec.groovy[ContactSpec.groovy]

### 1.7 GSON and Grails Views Defaults

As you can see by inspecting the views directory, by default Grails creates a number of gson files. Support for them is
provided with http://views.grails.org/latest/#_introduction[Grails Views Plugin]

The obvious question how does it work. If you look at sources of the RestfullController it doesn't "call" this templates
explicitly. So under the hood plugin just looks on request, if url ends on `.json`(localhost:8080/api/contact/1.json) or if
`Accept` header containing `application/json` the .gson view will be rendered.

If you delete default generated templates, then it will show default Grails page. Go ahead and try to delete `notFound.gson`
and try

```
curl -i -X GET -H "Content-Type: application/json"  localhost:8080/api/contact/105
HTTP/1.1 404
X-Application-Context: application:development
Content-Type: text/html;charset=utf-8
Content-Language: en-US
Content-Length: 990
Date: Mon, 31 Jul 2017 12:34:06 GMT

<!DOCTYPE html><html><head><title>Apache Tomcat/8.5.5 - Error report</title><style type="text/css">H1 ...
```

##### error.gson
{url-snapshot1}/grails-app/views/error.gson[See source]

This is for internal server errors. As you can see this is where the 500 status code gets set, and error message is specified.

It is called when we get `500` error, the same as for `gsp` look at UrlMapping: `"500"(view: '/error')`

##### notFound.gson
{url-snapshot1}/grails-app/views/notFound.gson[See source]
This is for case when resource isn't found. As you can see this is where the 404 status code gets set, and error message is specified.

It is called when we get `404` error, the same as for `gsp` look at UrlMapping: `"404"(view: '/notFound')`

##### errors/_errors.gson
{url-snapshot1}/grails-app/views/errors/_errors.gson[See source]
This is for validation errors. As you can see this is where the `UNPROCESSABLE_ENTITY`(422) status code gets set, and
error messages for entity specified.

It is rendered on {src-grails-rest}/src/main/groovy/grails/rest/RestfulController.groovy#L99[see src]
so if entity has errors it will look for `views/contact/_errors.gson` and if it doesn't exist then `views/errors/_errors.gson`

You can read more about defaults http://views.grails.org/latest/#_content_negotiation[here]

##### object/_object.gson
{url-snapshot1}/grails-app/views/object/_object.gson[See source]
This is for transforming entity to JSON object.

The rendering of this template is called for example here: {src-grails-rest}/src/main/groovy/grails/rest/RestfulController.groovy#L114[Save method]
So by convention if you have  `views/contact/_contact.gson` it will render it, in other case `views/object/_object.gson`,
which just render object as Json, so if we delete it it will still work in the same way because `respond instance` make
the same.


So all this files are default tempaltes for rendering in JSON all types of the responses and before delete them we need
to implement our own gson templates.

### Snapshot 1 of this tutorial is at this point

##  2. The DAO plugin with REST support

### 2.1 Introduction

The DAO plugin adds a new Service artifact to sit in between the controller interface and the restful logic.
At it core its just a specialized transactional service to deal with CRUD, searching and other functionality relating to a domain.
The mains goals are to reduce boiler plate in the controller, centralizing transactional domain logic out of the controller,
make it easier to reuse the crud across the application without the controller and simplify testing.

Add in the dependency for the plugin. Currently the Snapshot of the new version is published, so you need to add repository and dependency:

``` groovy
...
repositories {
...
    maven { url "http://dl.bintray.com/9ci/grails-plugins" }
}
...
dependencies {
...
    compile "org.grails.plugin:dao:3.0.3.SNAPSHOT"
```

### 2.2 RestDaoController

Dao plugin will setup a default DAO for every domain and it has RestDaoController that overrides the methods of the
default Grails `RestfullController` and simplifies the logic by pushing most of it down to the DAOs.

The `@Resource` has a property `superClass` that allows us to use another controller as basic for building rest endpoints,
and we will set `RestDaoController` as super class for our Contact:

**Contact.groovy**

```groovy
import grails.rest.Resource
import grails.plugin.dao.RestDaoController

@Resource(namespace = 'api', superClass = RestDaoController)
class Contact {
  ...
}
```

Now run the tests to make sure our functional tests still pass with the defaults.

### 2.3 Implementing a DAO service

Lets say we want to customize the insert to allow a user to pass in a name and have it be split into first and last names.

The test for this case will look like:

**ContactSpec.groovy**

```groovy
  given:
  RestBuilder rest = new RestBuilder()

  when: "name is passed"
  def response = rest.post("${baseUrl}/contact"){
    json([
      name: "Joe Cool",
      email: "foo@bar.com"
    ])
  }

  then:
  response.status == 201
  JSONElement json = response.json
  json.firstName == "Joe"
  json.lastName == "Cool"
  }
}
```

More tests examples are in the snapshot2 project's {url-snapshot2}/src/integration-test/groovy/resttutorial/ContactSpec.groovy[ContactSpec.groovy]

We will setup a concrete implementation of a dao for the contact as opposed to clogging up the business logic in the controller.
The plugin will recognize that we want to use this base on the naming convention SomeDomainNameDao
In either the grails-app/services/resttutorial or grails-app/dao/resttutorial directory add the ContactDao.groovy

We need to add `@Transactional` because services, and thus our DAO, are not transactional by default starting from Grails 3.

**ContactDao.groovy**
```groovy
package resttutorial

import grails.plugin.dao.GormDaoSupport
import grails.transaction.Transactional

@Transactional
class ContactDao extends GormDaoSupport {
	Class domainClass = Contact

    @Override
    Map insert(Map params) {
      String name = params.remove("name")
      if(name){
        def (fname, lname) = name.split()
        params.firstName = fname
        params.lastName = lname
      }
      super.insert(params)
    }
}
```

Now we can run tests again to be sure that new functionality works along with out new test.

### 2.4 Snapshot 2 of the tutorial is at this point

#### Implementing a RestDaoController

Use Case: A user can not update the inactive field since its bindable false.

To implement this use case we have two ways to go:

1. Override `delete` method for the controller, so it will set `inactive` field to true, instead of deleting from DB
2. Add separate endpoint for this action, so we keep ability to delete Contact

For both cases we can't use `@Resource` on our domain because it doesn't allow us to change the controller actions that are used for our resource.
So we need to create our own controller and extend it from RestDaoController which gives us ability to customize actions
using DAOs.

We will remove the `@Resource` annotation from the contact domain and add the ContactController.groovy, but
`@Resource`, not only creates controller based on resource, but also updates urlMappings, so now we need to add our url by hands.
It will look somethings like this: `"/api/contact"(resources: "contact")` it will add url mappings for our newly created controller.

**UrlMappings.groovy**
```groovy
  static mappings = {
    .....
     "/api/contact"(resources: "contact")
  }
```

And controller:

**ContactController.groovy**
```groovy
package resttutorial.api

import resttutorial.Contact
import grails.plugin.dao.RestDaoController

class ContactController extends RestDaoController {
    static responseFormats = ['json']
    static namespace = "api"

    ContactController() {
      super(Contact)
    }
}
```

You can run tests - it will work in the same way as it does with annotation.

So lets return to our use case. And take a look closer for both ways that we have.

The first way to override the delete method. I do not realy like this approach because `DELETE` should really delete entity.
And the second reason is that how should we activate our contact, the only way is to use `PUT` action and pass `inactive = false`,
but due to the fact that it is unbindable, we need to add handling exactly for this situation which make the code messy.

The other way is to add separate endpoint.

> :memo: **REST Standarts**
We should keep in mind some principals when we build REST API
>1. REST is resource-oriented, not service-oriented. Resources are nouns, not verbs we should delegate verbs using HTTP verbs.
>2. The next standard is based on the Keep it Simple, Stupid (KISS) principle. We really need two base URLs per resource:
one for multiple values and one for the specific value.
>3. Associations. An APIs should be very intuitive when you're developing them for associations. The following URL
is self-explained: we request user with id 3 and contact with id 8: `GET /user/3/contact/8`
We have traversed two levels in this URL. One level is the user, and the second level is the contact that the user is has.


According to the first standard we shouldn't use something like 'contact/inactivate', instead we can use a nested "resource"
`active`, and due to 3rd point of the note it should look like something like `contact/2/active`, when we need to inactivate
the contact it will send `DELETE` request, for activation - `POST`.

For now lets implement just making contact inactive.
To add custom end point we need to add nested url for resource and result will be look like:

**UrlMappings.groovy**
```groovy
    static mappings = {
      .....

       "/api/contact"(resources: "contact"){
            delete "/active"(action: "inactivate")
            // For future execise add `activate` action that will activate a contact
            // post "/active"(action: "activate")
        }
    }
```

**ContactController.groovy**
```groovy
package resttutorial.api

import resttutorial.Contact
import grails.plugin.dao.RestDaoController

class ContactController extends RestDaoController {
    static responseFormats = ['json']
    static namespace = "api"

    ContactController() {
      super(Contact)
    }

    def inactivate() {
       Contact contact = dao.inactivate(params.contactId as Long)
       respond contact
     }
}
```

So it will show default `404` error, we can customize `notFound.gson` file to make it show not only default `'Not found'`,
but our message from exception:

**notFound.gson**
```json
import groovy.transform.Field

response.status 404
@Field String text

json {
	message text ?: "Not found"
	error 404
}
```

and in controller

**ContactController.groovy**
```groovy
...
  contact = dao.inactivate(params.contactId as Long)

...
```

RestDaoController has `handleDomainNotFoundException` which handles the exception, one can override it if needed.


> :memo:
If you want to be able to call the action by "contact/inactivate/3" the only reason why you can't do this is UrlMapping,
but it is easy to change by adding `"/$controller/$action?/$contactId?"{}` , I've used `$contactId` because `params.contactId`
is used in the controller.
>

Add logic to the dao:

**ContactDao.groovy**
```groovy
package resttutorial

import grails.plugin.dao.DaoUtil
import grails.plugin.dao.GormDaoSupport
import grails.transaction.Transactional

@Transactional
class ContactDao extends GormDaoSupport {
    ...
  Contact inactivate(Long id) {
      Contact contact = Contact.get(id)

      DaoUtil.checkFound(contact, [id: id] ,domainClass.name) // Throws DomainNotFoundException
      DaoUtil.checkVersion(contact , [id: id].version)

      contact.inactive = true
      contact.persist()
      contact
    }
    ....
}
```

Update our rest sanity tests

**ContactSpec.groovy**
```groovy
    void "check inactivate endpoint"() {
        when:
        RestResponse response = rest.delete("${baseUrl}/contact/2/active")

        then:
        response.status == 200
        response.json != null
        JSONElement json = response.json
        json.inactive == true
    }
```

Update our dao tests

**ContactDaoSpec.groovy**
```groovy
    void "check inactivate"() {
        when:
        def result = contactDao.inactivate(5)

        then:
        result.inactive == true
    }
```

More tests examples are in the snapshot3 project's
{url-snapshot3}/src/integration-test/groovy/resttutorial/ContactSpec.groovy[ContactSpec.groovy] and {url-snapshot3}/src/integration-test/groovy/resttutorial/ContactDaoSpec.groovy[ContactDaoSpec.groovy]

### 2.5 Paging data
When returning a list, it will be necessary to support paging.
There is no single rest standard for paging so we will settle on the following.

Paging will leverage query parameters as shown in the following example:

```
https://localhost:8080/api/contact/?max=10&page=1
```

and will result in a wrapped response

```
page: 1,
total: 10,
records: 100,
rows:[
  {"id":1,"email":"mscott0@ameblo.jp","firstName":"Marie","lastName":"Scott"},
  {"id":2,"email":"jrodriguez1@scribd.com" ...
]
```

Few words about what this parameters means:

- __page__ is the page we are on
- __total__ is the total number of pages based on max per page setting
- __records__ is the total number of records we have
- __rows__ is the list of data


 Currently you will get next response on index endpoint:

```json
{
    "page": 1,
    "total": 10,
    "records": 100,
    "rows":
    [
        {
            "id": 1,
            "email": "mscott0@ameblo.jp",
            "firstName": "Marie",
            "lastName": "Scott"
        },
        {
...
```
but if you create a file

>**views/contact/_contact.gson**

```
import groovy.transform.*

@Field Contact contact

json {firstName contact.firstName}
```

The response will be changed to

  ```json
     "page": 1,
     "total": 10,
      "records": 100,
      "rows":
      [
          {
              "firstName": "Marie"
          },
          {
              "firstName": "Joseph"
          },
          {
              "firstName": "Julie"
          },{
  ....
```


This happens because index endpoint looks for template for rendering entity.

### Snapshot 3 of tutorial app is at this point

## 3. Creating the Angular UI

### 3.1 Introduction

For adding UI we will use a handy too called the https://github.com/9ci/angle-grinder[Angle-Grinder] plugin that helps to integrate Angular
with Grails.

Angle-Grinder uses assets-pipeline plugin, so we should include both to our `build.gradle`. Also we should add `compile "org.grails:grails-dependencies"`,
that Angle-Grinder requires. So finally we should add:

```groovy
compile "org.grails:grails-dependencies"
compile "com.bertramlabs.plugins:asset-pipeline-grails:2.11.2"
compile "nine:angle-grinder:3.0.3.SNAPSHOT"
compile 'net.errbuddy.plugins:babel-asset-pipeline:2.1.0'
```

To make it easier to understand the next steps lets dive into how Angle-Grinder plugin works. It renders Grails gsp pages
with all assets(so you do not need to worry about it), and with Angular code, after it is rendered browser executes  JS
code from the page. As a result we need to have actions for gsp rendering, and good decision is to isolate our REST Api
controllers from controllers that will render pages.

### 3.2 Gsp rendering
We have REST controller in the separate folder lets create one for page rendering.

**ContactController.groovy**
```groovy
package resttutorial

class ContactController {

    def index() {}
}
```

and then create folder `/views/contact` and `index.gsp` in it:

**index.gsp**
```html
<!doctype html>
<html>
  <head>
    <meta name="layout" content="main"/>
    <title>Welcome to Tutorial</title>
  </head>
  <body >
  </body>
</html>
```

To apply styling and javascript we need to include Angle-Grinder assets to our app. It is really easy with
assets-pipeline plugin. First we need to create the `assets` folder in `grails-app` and add `javascript` and
`stylesheets` directories. These would be added automatically if we did a create-app without limiting it to a rest-profile above

Then we create `application.css` file in `stylesheets` folder and `application.js` in `javascript` where we put
"links" on Angular sources:

**application.js**
```
//= require angleGrinder/vendor.js
//= require angleGrinder/angleGrinder.js
```


**application.css**
```
/*
*= require angleGrinder/bootstrapAll.css
*= require angleGrinder/angleGrinder.css
*= require_self
*/
```

And now we need to include them in our gsp:

**index.gsp**
```html
<head>
	<meta name="layout" content="main"/>
	<title>Welcome to Tutorial</title>
	<asset:stylesheet href="application.css"/>
	<asset:javascript src="application.js"/>
</head>
```

To see how it works lets add a header for our page and add some content:

**index.gsp**
```html
<!doctype html>
<html>
  <head>
    <meta name="layout" content="main"/>
    <title>Welcome to Tutorial</title>
    <asset:stylesheet href="application.css"/>
    <asset:javascript src="application.js"/>
  </head>
  <body >
    <nav class="navbar navbar-default navbar-static-top">
      <div class="container">
        Rest Tutorial
      </div>
    </nav>
    <div class="container">
      Content goes here
    </div>
  </body>
</html>
```

I've added a styling for header for our page see `views/contact/index.gsp`

### 4. UI
#### 4.1 List
Now when we have a html template lets create an angular app, and we will start from displaying a list. Following new trands
lets use `es6` we've already included dependency for `babel-asset-pipeline` above.

First we need to create module and add routes for it:

**grails-app/assets/javascript/contact/contactApp.es6**
```js
angular.module("contactApp", ["angleGrinder"])
  .constant('RestContext', 'api')
  .controller('ListCtrl', ListCtrl)
  .config([
    "RoutesServProvider", function (RoutesServ) {
      RoutesServ.setRoutes({contact: {"/": {page: "list"}}});
    }
  ]);
```

`ResourceTemplateServ` - service provided by Ag-Grinder that creates path for template.

`app.constant('RestContext', 'api');` - currently AG-Grinder supports 2 ways of building requests for resources (REST and
with actions) and to make it use REST approach we need to specify the namespace for it.

`RoutesServ.setRoutes({contact: {"/": {page: "list"}}})` is service that will create routes for us

We need to update our `contact/index.gsp` to make it "see" our angular app:

**index.gsp**
```html
...
<body ng-app="contactApp"> %{--The ngApp directive designates the root element of the application--}%
...
  <div class="container">
    <ng-view></ng-view> %{-- ngView is a directive that complements the $route service by including the rendered template of the current route into the layout--}%
  </div>
</body
```

The next step is to create an angular controller for list:

**assets/javascript/contact/ListCtrl.es6**
```js
class ListCtrl {
  constructor($scope, Resource, DialogCrudCtrlMixin, pathWithContext, RoutesServ) {
    var colModel = [
      {
        name: "id",
        label: "ID"
      }, ...
    ];

    $scope.gridOptions = {
      path: "/api/contact",
      colModel: colModel,
      multiselect: true,
      shrinkToFit: true,
      sortname: "id",
      sortorder: "asc",
      rowNum: 5,
      rowList: [5, 10, 20, 100]
    };

    DialogCrudCtrlMixin($scope, {
      Resource: Resource,
      gridName: "contactGrid",
      templateUrl: pathWithContext("contact/form")
    });

    $scope.save = (contact) => {
      contact.save().then(function (resp) {
        console.log(resp);
      })
    };
  }
}
ListCtrl.$inject = ['$scope', 'Resource', 'DialogCrudCtrlMixin', 'pathWithContext', 'RoutesServ'];
```

Then add "links" on angular controller in `application.js`:

**application.js**
```
//= require angleGrinder/vendor.js
//= require angleGrinder/angleGrinder.js
//= require contact/ListCtrl.es6
//= require contact/contactApp.es6
```

And add action in controller:

**ContactController.groovy**
```groovy
package resttutorial

class ContactController {

    def index() {}

    def list() {
      render template: "list"
    }
}
```

Then lets create a list template:

**views/contact/_list.gsp**
```html
<h3 class="page-header">Contact list</h3>

<div ag-grid="gridOptions" ag-grid-name="contactGrid"></div>
```

Where `ag-grid` - directive that takes parameters from scope and renders grid, and  `ag-grid-name` - set the name to grid
to make available from scope.

That's all what we need to display a grid to user.

![](../assets/list.png)

#### 4.2 DELETE

As I mentioned above Ag-Grinder has a lot of handy tools, one of the is `DialogCrudCtrlMixin` which adds CRUD actions for
the grid:

**assets/javascript/contact/ListCtrl.es6**
```js
var ListCtrl = (function() {
  ListCtrl.$inject = ["$scope", "Resource", "DialogCrudCtrlMixin"];

  function ListCtrl($scope, Resource, DialogCrudCtrlMixin, pathWithContext) {
  ...
    DialogCrudCtrlMixin($scope, {
      Resource: Resource,
      gridName: "contactGrid"
    });

  }

...

angular.module("contactApp").controller("ListCtrl", ListCtrl);
```

For each grid row we have action column with gear, when you click on it a menu with delete button will appear.

We do not even need to specify what resource should it use, just to add resource name in `index.gsp`:

**index.gsp**
```html
<body ng-app="contactApp" data-resource-name="contact"
	  data-resource-path="/contact">
```

And it will create the path by its self.

So run the application to try.

#### 4.3 CREATE

To add create functionality we need to prepare create form:

**_form.gsp**
```html
<div class="modal-header">
  <button type="button" class="close" ng-click="closeDialog()">&times;</button>

    <span>Create</span>
</div>

<form name="editForm" class="form-horizontal no-margin" ag-submit="save(contact)">
  <div class="modal-body">
    <div>
      <label class="control-label">First Name</label>
      <div class="row">
        <div class="col-md-4">
          <input type="text" name="firstName" ng-model="contact.firstName" ng-required="true" class="form-control"/>
        </div>
      </div>
    </div>
  </div>
  <div class="modal-footer">
    <ag-cancel-button ng-click="closeDialog()"></ag-cancel-button>
    <ag-submit-button></ag-submit-button>
  </div>
</form>
```

You probably noticed several new directives, I'll provide a brief description for them:

. __ag-submit__ - runs the method when form is submited and handles validation for nested forms if they are

. __ag-cancel-button__ - just provide styling for cancel button

. __ag-submit-button__ - styling and shows "..." during form submit

Now we just need to specify template so `DialogCrudCtrlMixin` now where is form template:

**ListCtrl.js**
```js
...
 DialogCrudCtrlMixin($scope, {
      Resource: Resource,
      gridName: "contactGrid",
      templateUrl: pathWithContext("contact/form")
    });
...
```

And the last step we need to add button that will trigger contact creating:

**_list.gsp**
```html
<div class="ag-panels-row">
  <div class="ag-panel">
    <div class="navbar navbar-toolbar navbar-grid navbar-default">
      <div class="navbar-inner with-selected-pointer with-grid-options">
        <ul class="nav navbar-nav">
          <li>
            <a ng-click="createRecord()">
              <i class="fa fa-plus"></i> Create Contact
            </a>
          </li>
        </ul>
      </div>
    </div>
    <div ag-grid="gridOptions" ag-grid-name="contactGrid"></div>
  </div>
</div>
```

`createRecord()` methos is already in `$scope`, thanks again to `DialogCrudCtrlMixin`.

Then add action in controller:

**ContactController.groovy**
```groovy
package resttutorial

class ContactController {

...

    def form() {
        render template: "form"
    }
}
```

So you can try to create a new contact.

#### 4.4 EDIT

You will be suprised, but edit is already works, try Edit button in grid dropdown. The only thing
that we need to change is to change labels for form modal window:

._form.gsp
```html
<div class="modal-header">
	<button type="button" class="close" ng-click="closeDialog()">&times;</button>
	<span ng-show="contact.persisted()" > Update</span>
	<span ng-hide="contact.persisted()" > Create</span>
</div>
...
```

### Snapshot 4 of tutorial app is at this point
#### 4.5 Add more field types

So now lets add some more fields for our domain to take a look on some other widgets of Angle-Grinder

.Contact.groovy
```groovy
import java.time.*

class Contact {
  Salutations salutation
  String firstName
  String lastName
  String email

  LocalDate dateOfBirth
  TimeZone timeZone
  LocalDateTime activateOnDate

  Date dateCreated
  Date lastUpdated
  Boolean inactive

  static constraints = {
    firstName nullable: false
    dateOfBirth nullable: true
    inactive bindable:false
  }

  enum Salutations {
    Ninja,
    Mr,
    Mrs,
    Ms,
    Dr,
    Rev
  }
}
```

As you can see we have java 8 date types here. Due to the fact that Hibernate5 supports the new date types lets update
our dependencies, also see section about java8 in http://docs.grails.org/latest/guide/single.html#otherNovelties[docs]:

**build.gradle**
```groovy
...
    classpath "com.bertramlabs.plugins:asset-pipeline-gradle:2.11.2"
    classpath "org.grails.plugins:grails-java8:1.1.0"
    classpath "org.grails:grails-gradle-plugin:3.2.11"
    classpath "org.grails.plugins:hibernate4:6.0.12"
    classpath "org.grails.plugins:views-gradle:1.2.0.M2"

...
    compile "org.grails.plugins:hibernate4"
    compile "org.hibernate:hibernate-core:4.3.11.Final"
    compile "org.hibernate:hibernate-ehcache:4.3.11.Final"
    compile "org.grails.plugins:grails-java8:1.1.0.BUILD-SNAPSHOT"
    compile "org.hibernate:hibernate-java8:5.1.1.Final"
    compile "org.grails.plugins:views-json:1.2.0.M2"
    compile "org.grails.plugins:views-json-templates:1.2.0.M2"
```

To make it parse string date we need to add list of the available date formats:

**application.groovy**
```groovy
  grails.databinding.dateFormats = ["yyyy-MM-dd'T'HH:mm:ss'Z'", "yyyy-MM-dd'T'HH:mm:ss.S'Z'","yyyy-MM-dd'T'HH:mm:ss","yyyy-MM-dd"]
```

Dao plugin contains several converters for java 8 dates for GSON templates, https://github.com/9ci/grails-dao/tree/grails3/dao-plugin/src/main/groovy/grails/plugin/dao/converters[see sources]

And now we need to update some fields to our form: {url-snapshot5}/grails-app/views/contact/_form.gsp[_form.gsp]

![](../assets/form.png)

#### 4.5.1 Add Location

Lets add one more domain to our project.

**Address.groovy**
```groovy
package resttutorial

import grails.rest.Resource

@Resource(namespace = "api", formats = ["json"])
class Address {
    static belongsTo = [contact: Contact]
    String street
    String city
    String state
    String postalCode
    String country

    static constraints = {
        street nullable: false
    }
}
```

To Contact we will add:

**Contact.groovy**
```groovy
...
 static hasOne = [address: Address]
...
```

Also, we need to modify UrlMapping to be able to get address as nested resource:

**UrlMapping.groovy**
```groovy
"/api/contact"(resources: "contact", namespace: "api"){
  "/address"(resources: "address")
  delete "/active"(controller: "contact", action: "inactivate")
 }
"/api/address"(resources: "address")
```

lets take a look on `url-mappings-report `:

```
 |    *     | ERROR: 404                                        | View:   /notFound            |
 |    *     | ERROR: 500                                        | View:   /error               |
 |    *     | ERROR: 500                                        | View:   /error               |
 |    *     | /                                                 | View:   /index               |
 |   POST   | /api/${controller}(.${format)?                    | Action: save                 |
 |   GET    | /api/${controller}(.${format)?                    | Action: index                |
 |  PATCH   | /api/${controller}/${id}(.${format)?              | Action: patch                |
 |   PUT    | /api/${controller}/${id}(.${format)?              | Action: update               |
 |   GET    | /api/${controller}/${id}(.${format)?              | Action: show                 |
 |  DELETE  | /api/${controller}/${id}(.${format)?              | Action: delete               |
 |    *     | /${controller}/${action}?/${id}?                  | Action: (default action)     |

Controller: address
 |   GET    | /api/address/create                               | Action: create               |
 |   GET    | /api/contact/${contactId}/address/create          | Action: create               |
 |   GET    | /api/contact/${contactId}/address/${id}/edit      | Action: edit                 |
 |   POST   | /api/contact/${contactId}/address                 | Action: save                 |
 |   GET    | /api/contact/${contactId}/address                 | Action: index                |
 |   GET    | /api/address/${id}/edit                           | Action: edit                 |
 |  DELETE  | /api/contact/${contactId}/address/${id}           | Action: delete               |
 |  PATCH   | /api/contact/${contactId}/address/${id}           | Action: patch                |
 |   PUT    | /api/contact/${contactId}/address/${id}           | Action: update               |
 |   GET    | /api/contact/${contactId}/address/${id}           | Action: show                 |
 |   POST   | /api/address                                      | Action: save                 |
 |   GET    | /api/address                                      | Action: index                |
 |  DELETE  | /api/address/${id}                                | Action: delete               |
 |  PATCH   | /api/address/${id}                                | Action: patch                |
 |   PUT    | /api/address/${id}                                | Action: update               |
 |   GET    | /api/address/${id}                                | Action: show                 |

Controller: contact
 |   GET    | /api/contact/create                               | Action: create               |
 |   GET    | /api/contact/${id}/edit                           | Action: edit                 |
 |  DELETE  | /api/contact/${contactId}/active                  | Action: inactivate           |
 |   POST   | /api/contact                                      | Action: save                 |
 |   GET    | /api/contact                                      | Action: index                |
 |  DELETE  | /api/contact/${id}                                | Action: delete               |
 |  PATCH   | /api/contact/${id}                                | Action: patch                |
 |   PUT    | /api/contact/${id}                                | Action: update               |
 |   GET    | /api/contact/${id}                                | Action: show                 |
```

As you can see we can get address by 2 ways:

 - `/api/adress`
 - `/api/contact/${contactId}/address`

But on `/api/contact/1/address` we won't get the address that is related to contact with id = 1, but a list of all addresses,
to make more smart we need to implement AddressController:

**AddressController.groovy**
```groovy
package resttutorial.api

import grails.plugin.dao.Pager
import grails.plugin.dao.RestDaoController
import resttutorial.Address

class AddressController extends RestDaoController {
  static responseFormats = ['json']
  static namespace = "api"

	AddressController() {
    super(Address)
  }

  @Override
  protected List<Address> listAllResources(Map params) {
    def crit = Address.createCriteria()
    def pager = new Pager(params)
    def datalist = crit.list(max: pager.max, offset: pager.offset) {
      if(params.contactId){
          eq "contact.id", (params.contactId as Long)
      }
      if (params.sort)
        order(params.sort, params.order)
    }
    return datalist
  }
}
```

the `listAllResources` method is called by `index` action (see {dao-plugin}/grails-app/controllers/grails.plugin.dao/RestDaoController.groovy[RestDaoController])
and if `contactId` is in params it will limit list by addresses just for this contact.

Lets add some test. You can see them in {url-snapshot5}[snapshot5] folder.

### 4.6. Add tests(GEB)
After we've added integration test, lets add some function tests to be sure that our app works.
For testing we'll use Geb. Please, take a look at https://github.com/basejump/grails3-geb-example[example] to understand
how you can configure and use ged tests.

Our tests we will place in `geb` folder.

The test should have a `@Rollback` anotation, from Grails docs:

> :memo:
In Grails 3.0 tests rely on grails.transaction.Rollback annotation to bind the session in integration tests.

>**ContactGebSpec.groovy**

>``` groovy
package geb
import geb.spock.GebSpec
import grails.test.mixin.integration.Integration
import grails.transaction.Rollback
import resttutorial.Contact
import spock.lang.*

>@Integration
@Rollback
class ContactGebSpec extends GebSpec {

  >def getBaseUrl(){"http://localhost:${serverPort}"}

>  void "Check contact page"() {
>    when: "The contact page is visited"
>    go "${baseUrl}/contact"
>    sleep(1000)
>
>   then: "The title and label are correct"
>    title == "Welcome to Tutorial"
>    $("h3").text() == 'Contact list'
>  }
>}
```

In this test we open contact page with our grid - `go "http://localhost:${serverPort}/contact"` and check page title and grid label.
Access to the elements is provided in jQuery-like language, that makes accessing the elements very easy. You can see it
in `$("h3").text() == 'Contact list'`

Try to run tests with `grails test-app -Dgeb.env=chrome`

As a next step lets add tests for contact editing:

**ContactGebSpec.groovy**
```groovy
  void "Check edit contact"() {
      when: "The home page is visited"
      go "${baseUrl}/contact"
      def lastRow = $(".jqgrow.ui-row-ltr")[-1]
      lastRow.find(".jqg-row-action").click()
      sleep(1000)
      $(".row_action_edit").click()

      then: "Dialog is opened"
      sleep(5000)
      $(".modal-dialog form") != null
      $(".modal-dialog form").firstName == "Susan"
      $(".modal-dialog form").lastName == "Duncan"
      $(".modal-dialog form").email == "sduncan4@diigo.com"
      $(".modal-dialog form").salutation == "Rev"

      when: "Changed values and save"
      $(".modal-dialog form").firstName = "Dr. Who"
      $(".modal-dialog [type='submit']").click()
      sleep(5000)

      then: "Contact list label"
      Contact contact = Contact.get(5)
      contact.firstName == "Dr. Who"
	}
```

Some explanation for the test:
`.jqgrow.ui-row-ltr` - CSS class that each grid row has, so when we execute $(".jqgrow.ui-row-ltr") we get list of all
grid rows, so `$(".jqgrow.ui-row-ltr")[-1]` will take the last of displayed rows.
`.jqg-row-action` is CSS class for action button(gear), after we click on it we need to wait for a while(`sleep(1000)`),
to give some time for JS to show menu, and then "click" on edit button $(".row_action_edit").click()
For form we can get values just by name property: `$("form").firstName == "Susan"`
And the last step is to check form saving.

You can find some more tests in snapshot5 project's {url-snapshot5}/src/integration-test/groovy/geb/ContactGebSpec.groovy[ContactGebSpec.groovy].

#### 4.7 Add filters for grid

For now all looks good, but what if we have big amount of data in grid then we need to add ability to filter it.
Lets add filter for our grid:

First we need to create a template for our search form:

**views/contact/_searchForm.gsp**
```html
<form ag-search-form="contactGrid" class="form-horizontal form-multi-column no-margin ag-search-form">

  <div class="col-md-4">
    <div class="control-group">
      <label class="control-label">Name</label>

      <div class="controls">
        <input class="input-block-level" type="text" ng-model="filters.firstName">
      </div>
    </div>
  </div>

  <div class="col-md-4">
    <div class="control-group">
      <label class="control-label">Email</label>

      <div class="controls">
        <input class="input-block-level" type="text" ng-model="filters.email">
      </div>
    </div>
  </div>

  <div class="pull-right">
    <ag-search-button></ag-search-button>
    <ag-reset-search-button></ag-reset-search-button>
  </div>
</form>
```

The Ag-Grinder has handy directives for search forms:

 - __ag-search-form__ you need to specify name of the grid that search form is related to and it will add action for search
 - __ag-search-button__ triggers grid filtering
 - __ag-reset-search-button__ clears search fields and reload grid to show it without filtering

Now we need to add action for form rendering:

**controllers/ContaсtController.groovy**
```groovy
...
    def searchForm(){
        render template: "searchForm"
    }
...
```

To make it show we need to modify the list template:

**views/contact/_list.gsp**
```html
<h3 class="page-header">Contact list</h3>

<div class="well">
  <g:include action="searchForm"/>
</div>

...
```

as a result we will get something like this:

![](../assets/searchForm.png)

We don't need to make any changes to our JS code, Ag-Grinder directives will send filter data, we just need to update
`api/ContactController`:

**controllers/api/ContactController.groovy**
```groovy
  def updateDomain(){
    Map result = dao.update(fullParams(params, request))
    DaoUtil.flush()
    result
  }

  @Override
  protected List<Contact> listAllResources(Map params) {
    def crit = Contact.createCriteria()
    def pager = new Pager(params)
    def filters = params.filters ? JSON.parse(params.filters) : null
    def datalist = crit.list(max: pager.max, offset: pager.offset) {
      if (filters) {
        if (filters.firstName){
          ilike "firstName", filters.firstName + "%"
        }
        if (filters.email){
          ilike "email", filters.email + "%"
        }
      }
      if (params.sort)
        order(params.sort, params.order)
    }
    return datalist
  }
```

And now lets add a Geb test for it:

**ContactGebSpec.groovy**
```groovy
...

  void "Check grid filtering"() {
    when: "The home page is visited"
    go "${baseUrl}/contact"
    def searchForm = $("form.ag-search-form")
    searchForm.filtersFirstName = "Jos"
    searchForm.find("[type='submit']").click()
    sleep(1000)

    then: "Should be 1 row after filtering"
    $(".jqgrow.ui-row-ltr").size() == 1 // just one row in grid

    when: "Reset filtering"
    $("[ng-click='resetSearch(filters)']").click()
	  sleep(1000)

    then: "Should be 5 rows"
    $(".jqgrow.ui-row-ltr").size() == 5
  }

  ...
```

### Snapshot 5 of tutorial app is at this point

## 5. Spring Security
### 5.1 Introduction
There is an excellent tutorial that we will take as a starting point for adding Spring Security to our app, see theoretical
http://alvarosanchez.github.io/grails-angularjs-springsecurity-workshop/#_adding_security_with_spring_security_rest_50_minutes[part]
and how we can add spring security to our Grails backend on http://alvarosanchez.github.io/grails-angularjs-springsecurity-workshop/#_securing_the_rest_api_20_minutes[@alvarosanchez].

A user name and a password we will take from the tutorial not to make it confusing.

### Adding simple login screen

After we've passed through the first two steps from the previous tutorial the Grails part is already configured for spring
security. And now we need to add login screen. Deu to the fact that we are using Ag-Grinder we will take a bit another
approach for login implementation.

To be able to login on one page and stay logged in for all others we should have one main angular module that will handle it.
For this we need to move common parts of pages to layout:

**views/layouts/main.gsp**
```html
<!doctype html>
<html>
  <head>
    <meta name="layout" content="main"/>
    <title><g:layoutTitle default="Welcome to Tutorial"/></title>
    <g:layoutHead/>
    <asset:stylesheet href="application.css"/>
    <asset:javascript src="application.js"/>
  </head>
  <body context-path="${request.contextPath}"
        data-resource-name="${pageProperty(name: 'body.data-resource-name')}"
        data-resource-path="${pageProperty(name: 'body.data-resource-path')}"
        ng-app="contactApp">
    <nav class="navbar navbar-default navbar-static-top">
      <div class="container-fluid">
        <div class="navbar-header">
          <button type="button" class="collapsed navbar-toggle"
            data-toggle="collapse"
            data-target="#drop"
            aria-expanded="false">
          </button>
          <a href="#" class="navbar-brand">Rest Tutorial</a>
        </div>

        <div class="collapse navbar-collapse" id="drop">
          <ul class="nav navbar-nav">
            <li class="active"><a href="#">Contacts</a></li>
            <li><a href="#">Link 1</a></li>
            <li><a href="#">Link 2</a></li>
          </ul>
        </div>
      </div>
    </nav>
    <div>
        <g:layoutBody/>
    </div>
  </body>
</html>
```

and

**views/contact/index.gsp**
```html
<html>
  <head>
    <meta name="layout" content="main"/>
    <title>Welcome to Tutorial</title>
  </head>
  <body data-resource-name="contact" data-resource-path="/contact">
    <div class="container">
      <ng-view></ng-view>
    </div>
  </body>
</html>
```

Now we need to implement controller for logining, as you remember from the http://alvarosanchez.github.io/grails-angularjs-springsecurity-workshop/#_securing_the_rest_api_20_minutes[tutorial],
we need to pass our username and password to `/api/login` action and get `access_token` that should be added for each request.

**assets/javascript/auth/LoginCtrl.es6**
```js
angular.module("tutorial", ["angleGrinder"]);
var LoginCtrl;
var auth = angular.module("tutorial");

LoginCtrl = (function() {
  LoginCtrl.$inject = ["$scope", "pathWithContext", "$window", "$http", "$rootScope"];
  function LoginCtrl($scope, pathWithContext, $window, $http, $rootScope) {
    $rootScope.authenticated = $window.sessionStorage.token != null;
    $scope.user = {};
    $scope.login = function() {
      return $http.post(pathWithContext('/api/login'), {
        "username": $scope.user.username,
        "password": $scope.user.password
      }).then(function(response) {
        $rootScope.authenticated = !!response.data.access_token;
        $window.sessionStorage.token = response.data.access_token;
      });
    };
  }

  return LoginCtrl;
})();

auth.controller("LoginCtrl", LoginCtrl);
```

`$rootScope.authenticated` - is a flag that shows should we show login page or not
After user is logged in we will save `access_token` to `$window.sessionStorage.token`.

So lets add the controller to our layout:

**views/layouts/main.gsp**
```html
<body context-path="${request.contextPath}"
      data-resource-name="${pageProperty(name: 'body.data-resource-name')}"
      data-resource-path="${pageProperty(name: 'body.data-resource-path')}">
<div ng-app="tutorial" ng-controller="LoginCtrl">
...

<div id="page" class="container">
  <div ng-if="!authenticated">
    <table>
      <tbody>
      <tr>
        <td>
          Username:
        </td>
        <td>
          <input type="text" name="username" ng-model="user.username"/>
        </td>
      </tr>
      <tr>
        <td>
          Password:
        </td>
        <td>
          <input type="password" name="password" ng-model="user.password"/>
        </td>
      </tr>
      <tr>
        <td colspan="2">
          <button type="button" ng-click="login()">Login</button>
        </td>
      </tr>
      </tbody>
    </table>
  </div>

  <div ng-if="!!authenticated">
    <div>
      <g:layoutBody/>
    </div>
  </div>
```

Add controller in `application.js`:

**application.js**
```
...
//= require auth/LoginCtrl.es6
...
```

The next step is to add received token to each request, for this purpose we need to implement AuthIntersepter:

**assets/javascript/auth/LoginCtrl.es6**
```js
angular.module("tutorial", ["angleGrinder"]);

auth.factory('authInterceptor', ["$rootScope", "$window", "pathWithContext", "$q", function($rootScope, $window, pathWithContext, $q) {
  return {
    request: function(config) {
      config.headers = config.headers || {};
      if ($window.sessionStorage.token != null) {
        config.headers.Authorization = 'Bearer ' + $window.sessionStorage.token;
      }
      return config;
    },
    responseError: function(response) {
      var unauthorized = 401;
      if (response.status ### unauthorized) {
        return $window.location = pathWithContext("/");
        }
    }
  };
}]).config(function($httpProvider) {
  $httpProvider.interceptors.push('authInterceptor');
});

....
```

As you see on each error response we update a location so the page is reloaded and we then show the login page, but you
can show the modal window which will block the page or handle it in the way you want.

Now we have login functionality, so we need logout. Rest spring security doesn't have such endpoint, so we will just clear
session storage:

**assets/javascript/auth/LoginCtrl.es6**
```js
....

 $scope.logout = function() {
      $rootScope.authenticated = false;
      $window.sessionStorage.token = undefined;
    };
....
```

and in layout:

**views/layouts/main.gsp**
```html
....
  <ul class="pull-right nav navbar-nav">
    <li ng-click="logout()">
      <a href="#">Logout</a>
    </li>
  /ul>
```

### 5.2 Our CRUD screens should still work and be secured now

If you run the test now they will fail because we need to login first:

.ContactControllerSpeck.groovy
```groovy
 @Shared
    RestBuilder rest = new RestBuilder()

    def getBaseUrl() { "http://localhost:${serverPort}/api" }

    String token

    def setup() {
        RestResponse response = rest.post(baseUrl + "/login") {
            json([
                    "username": "user",
                    "password": "pass"
            ])
        }
        token = "Bearer " +response.json.access_token
    }

....

    void "check POST request"() {
          when:
          RestResponse response = rest.post("${baseUrl}/contact") {
              headers["Authorization"] = token
              json([
                      firstName: "Test contact",
                      "email"  : "foo@bar.com",
                      inactive : true
              ])
          }
....
```

The same we should make `for AddressControllerSpec`

After you refactor test we have only one integration test failing to fix it we need to modify our UrlMappings:


**UrlMappings.groovy**
```groovy
"/api/contact"(resources: "contact", namespace: "api"){
        "/address"(resources: "address", namespace: "api")
```

To make the nested resource match security url pattern.

Also, we need to fix geb tests. For this we need to add one more step(should be the first) to login:

**ContactGebSpec.groovy**
```groovy

void "Login"() {
    when: "The contact page is visited"
    go "${baseUrl}/contact"
    def inputs = $("input")

    then:
    inputs.size() == 2

    when:
    inputs[0].value("user")
    inputs[1].value("pass")
    sleep(1000)
    $("button")[1].click()

    then:
    sleep(1000)
    title == "Welcome to Tutorial"
  }

```

Geb tests will fail because of known issue https://github.com/alvarosanchez/grails-spring-security-rest/issues/166
to avoid it we need to update dependencies:

**build.gradle**
```
...
    compile("org.grails.plugins:spring-security-rest:2.0.0.M2"){exclude group: 'com.google.guava'}
...
```

Guava has to be excluded because of dependency conflict with Guava used by selenium's ChromeDriver.

## 6. Other Tools

Sometimes it is not enough only integration tests but developer need to play with endpoints by hands, so lets take a look
on two handy tools for testing rest endpoints.

### 6.1 Using postman for testing

First we need to install https://chrome.google.com/webstore/detail/postman/fhbjgbiflinjbdggehcddcbncdddomop/related?hl=en[Postman type in Chrome].
And press `Launch App`

![](../assets/postman.png)

For testing we need to get auth token, in `Request URL` we put `http://localhost:8080/api/login`, then select `POST`
as requst type on the left from `Request URL`. Then click on `body` tab that is under URL and enter type `raw`.
Here we should specify our credentials:

```json
{"username": "user", "password": "pass"}
```

The last step is to press `SEND` button to send the request. At the bottom you will get something like:

```json
{
  "username": "user",
  "roles": [
    "ROLE_ADMIN"
  ],
  "token_type": "Bearer",
  "access_token": "eyJhbGciOiJIUzI1NiJ9.eyJwcmluY2lwYWwiOiJINHNJQUFBQ...",
    "expires_in": 3600,
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9.eyJwcmluY2lwYWwiOiJINHNJQUFBQ..."
 }
```

And to be able to archive endpoints we need to add "Authorization" header to each request.
So open `Headers` tab and paste `Authorization` as a key and `Bearer <access_token from our login request>` as value

Now you can change url and test different endpoints.

### Using Intellij for testing

If you are using Intellij idea you can use build in rest client for testing. To open it got to "Tools" section on top and
select `Test RESTful Web Service`. You will see:

![](../assets/restClient.png)

In the `Host/port` section type `http://localhost:8080` and to the `Path` input `/api/login`
On the right side of the `Request` tab, select `text` as body type and type in

```json
{"username": "user", "password": "pass"}
```

To send a request press green triangle. So now we have our `access_token` in a response.
So now we just need to set header for future requests.

To add header press green plus. And set `Name` to `Authorization` and value to `Bearer <access_token from our login request>`

Now you can change url and test different endpoints.



[grails docs]: http://docs.grails.org/3.2.11/guide
[grails-api]: http://docs.grails.org/3.2.11/api
[grails ws docs]: http://docs.grails.org/3.2.11/guide/webServices.html
[src-grails-rest]: https://github.com/grails/grails-core/blob/master/grails-plugin-rest
