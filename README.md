# LAB06: CRUD RESTful Services (2)

## Guiding Notes
In the firtst part of the REST Services module, we revisited the principles behind the REST architectural style and went through some initial read-only examples of how to implement these services. In this module, we will extend those examples to have a complete CRUD API for our HealthProfile service Design


### Project structure (10 min) 
* Start by creating a Web Dynamic Project (as in the last session) in your local workspace, using the the name **sdelab06-local** 
* When asked, mark the option "Generate Deployment Descriptor Stub" to create an initial version of the **web.xml** file
* Integrate **Ivy** dependencies to your project as we learnt in the last module (copy the ivy for this module from the lab's repository **lab06/ivy.xml**, copy also the **lab06/build.xml**). You will also find a guide in this module's *Additional notes* section. 
* Add the following packages to your project: 
 * **introsde.rest.ehealth.dao** - will contain classes whose purpose will be to provide the underlying connection to the persistence layer (e.g., to the database) 
 * **introsde.rest.ehealth.model** - will include classes that represent our domain data model and map the content in our the persistence layer to objects that can be manipulated in Java
 * **introsde.rest.ehealth.resources** - will include the *resources* that are exposed throught the RESTful API, which can be seen as the controllers that receive requests and respond with a representation of the resources that are requested
* In this module, we will start by creating our modules as simple POJOs with JAXB annotations. Then we will create a DAO to access the persistence layer (in this case, a singleton hashmap). After this we will implement our REST resources with Jersey. And finally, we will stitch altogether in a Jersey Application that will work as a standalone server.

### Models (15 min)

* **Models** classes define our domain data model to which the data in our persistence layer will be mapped. We will reuse the model we created in the module 4, so go ahead and copy both **Person.java** and **HealthProfile.java** from **lab06/Example/src/introsde/rest/ehealth/model**. 
* JAXB annotations will allow Jersey to automatically find the way of marshalling and unmarshalling these objects to xml (or JSON). 

### Data Access Objects (DAO) (10 min) 

* **DAO** stands for *data access objects* and is where our data providers are. Modern REST frameworks include ORM (Object Relational Mapping) libraries to map the model into the persistence layer, passing through the basic primitives provide by the DAO classes (we will see about that in the next module) 
* For this example, we will use a [Singleton design pattern][4] to implement a mock of a in memory database.
* **Exercise 06.01:** in the **dao** package, create a **enum class** named **PersonDAO**, which will contain a **HashMap** where the key contains *personId*s and the value is the corresponding *Person*. For this example, this HashMap is our database. 
* the PersonDAO enum should include: 
 * the **instance** value (following the singleton design pattern) 
 * a **constructor** that creates a new HashMap and put some *Person* objects inside
 * a method to access the hashmap from other classes (e.g., getDataProvider())

### READ Resources (30 min) 

* **Resources** implement our service endpoints. Let's start by defining a resource for the **Person Collection** on the path **/person**. 
* Below is an starting version of the **PersonCollectionResource.java**. Add this to your **resource** package. 

    ```java
    package introsde.rest.ehealth.resources;
    
    import introsde.rest.ehealth.dao.PersonDao; // use it to access the data providers
    import introsde.rest.ehealth.model.Person;  // use it to manage person data
    
    import java.io.IOException;
    import java.util.ArrayList;  // use it to store the list of Person to return 
    import java.util.List;
    import javax.ws.rs.Consumes;
    import javax.ws.rs.GET;
    import javax.ws.rs.POST;
    import javax.ws.rs.Path;
    import javax.ws.rs.PathParam;
    import javax.ws.rs.Produces;
    import javax.ws.rs.core.Context;
    import javax.ws.rs.core.MediaType;
    import javax.ws.rs.core.Request;
    import javax.ws.rs.core.UriInfo;
    
    
    //Will map the resource to the URL /person
    @Path("/person")
    public class PersonCollectionResource {
    
        // Return the list of people to the user in the browser
        @GET
        @Produces({MediaType.TEXT_XML, MediaType.APPLICATION_XML, MediaType.APPLICATION_JSON})
        public List<Person> getPersonsList() {
            List<Person> people = new ArrayList<Person>();
            people.addAll(PersonDao.instance.getDataProvider().values());
            return people;
        }
    }
    ```

* Since **PersonColletionResource** is exactly that: a collection resource, we will need another resource to represent the single, specific **Person**. Create this following the example below. Notice that this is a **Sub-Resource** (it has not path annotation). For this reason, the **uriInfo** and the **request** (available in the parent resource) are injected using the @Context annotation so that the Person resource can also access the body of the request or other data in the path: 

    ```java
    package introsde.rest.ehealth.resources;
    
    import introsde.rest.ehealth.dao.PersonDao;
    import introsde.rest.ehealth.model.Person;
    
    import javax.ws.rs.Consumes;
    import javax.ws.rs.DELETE;
    import javax.ws.rs.GET;
    import javax.ws.rs.PUT;
    import javax.ws.rs.Produces;
    import javax.ws.rs.core.Context;
    import javax.ws.rs.core.MediaType;
    import javax.ws.rs.core.Request;
    import javax.ws.rs.core.Response;
    import javax.ws.rs.core.UriInfo;
    import javax.xml.bind.JAXBElement;
    
    public class PersonResource {
        @Context
        UriInfo uriInfo;
        @Context
        Request request;
        
        Long id;
    
        public PersonResource(UriInfo uriInfo, Request request,Long id) {
            this.uriInfo = uriInfo;
            this.request = request;
            this.id = id;
        }
    
        // Application integration
        @GET
        @Produces({ MediaType.APPLICATION_XML, MediaType.APPLICATION_JSON })
        public Person getPerson() {
            Person person = PersonDao.instance.getDataProvider().get(id);
            if (person == null)
                throw new RuntimeException("Get: Person with " + id + " not found");
            return person;
        }
    ```

* Add both uriInfo and request to the PersonCollectionResource

    ```java
        // Allows to insert contextual objects into the class,
        // e.g. ServletContext, Request, Response, UriInfo
        @Context
        UriInfo uriInfo;
        @Context
        Request request;
    ```
    
* Also in PersonCollectionResource, add now the method to load the sub resource when a request with a path like */person/1* arrives. 

    ```java
        // Defines that the next path parameter after the base url is
        // treated as a parameter and passed to the PersonResources
        // Allows to type http://localhost:599/person/1
        // 1 will be treaded as parameter todo and passed to PersonResource
        @Path("{personId}")
        public PersonResource getPerson(@PathParam("personId") String id) {
            return new PersonResource(uriInfo, request, id);
        }
    ```

### Deploy the Services and test them (15 min) 

* As we learnt in the previous module, we can either deploy our services in a **Servlet Container** (e.g., Tomcat) or through an standalone **HTTP Server**. 
* **The HTTP Server Way: ** different than our first example, we will now use the "Jersey" way for deploying through an HTTP server. 
 * Start by creating a root Jersey application (which is similar to our standalone http server of the previous module).
 * Add the following class to your **introsde.rest.ehealth** package:

    ```java
    package introsde.rest.ehealth;

    import java.io.IOException;
    import java.net.URI;
    import java.net.URISyntaxException;
    import java.net.InetAddress;
    
    import org.glassfish.jersey.jdkhttp.JdkHttpServerFactory;
    import org.glassfish.jersey.server.ResourceConfig;
    
    public class App
    {
        public static void main(String[] args) throws IllegalArgumentException, IOException, URISyntaxException
        {
            String protocol = "http://";
            String port_value = "5700";
            if (String.valueOf(System.getenv("PORT")) != "null"){
                port_value=String.valueOf(System.getenv("PORT"));
            }
            String port = ":"+port_value+"/";
            String hostname = InetAddress.getLocalHost().getHostAddress();
            if (hostname.equals("127.0.0.1"))
            {
                hostname = "localhost";
            }
    
            URI BASE_URI = new URI(protocol + hostname + port+"sdelab/");
    
        	System.out.println("Starting sdelab standalone HTTP server...");
            JdkHttpServerFactory.createHttpServer(BASE_URI, createApp());
            System.out.println("Server started on " + BASE_URI + "\n[kill the process to exit]");
        }
    
        public static ResourceConfig createApp() {
        	System.out.println("Starting sdelab REST services...");
            return new MyApplicationConfig();
        }
    }
    ```
    
 * Now, we need to tell the server where to lookup for resources. For this, you will need to create a configuration class **MyApplicationConfig** in the same package. This class is used by jersey to load our **resource classes** and additional features (or modules) provided by Jersey. In this module, we will use the Jackson feature to produce also JSON, which is actually automatically loaded if found in classpath (not all are). An extended list of available modules can be found [here][5]

    ```java
    package introsde.rest.ehealth;
    
    import javax.ws.rs.ApplicationPath;
    
    import org.glassfish.jersey.server.ResourceConfig;
    
    @ApplicationPath("sdelab/resources")
    public class MyApplicationConfig extends ResourceConfig {
        public MyApplicationConfig () {
            packages("introsde.rest.ehealth"); // Jersey will load all the resources under this package
        }
    }
    ```

* **The Servlet Way:** the **@ApplicationPath** annotation in our ApplicationConfig Class will save us from manually having to register a servlet mapping in the web.xml. This is valid for the Servlet API 3.0 (make sure "version" in your web.xml is "3.0"). Classess in the classpath with this annotation will be automatically mapped as servlet.

### CRUD RESTful Example (4): Deploy the Services (10 min) 

* Now, let's try it!. Run **App.java** as a Java application and then open the Postman app to make the following requests. 

    ```
    GET http://localhost:5900/sdelab/person
    Accept: text/xml
    
    
    GET http://localhost:5900/sdelab/person
    Accept: application/xml
        
    GET http://localhost:5900/sdelab/person
    Accept: application/json

    GET http://localhost:5900/sdelab/person/1
    Accept: application/json
    ```
    
* Now, let's try it on Tomcat!. Run the project on a Server and then open the Postman app to make the same requests as before, only this time they will all be preceded by **sdelab06**. 
 
 
### Create, Update and Delete a Person (25 min) 
 
* Add the following method to the PersonCollectionResource (in the RESTful design, we are operating on the "collection" because we are adding a new item to it via a **POST /person**) 

    ```java
        @POST  
        @Produces({MediaType.APPLICATION_XML, MediaType.APPLICATION_JSON}) 
        @Consumes({MediaType.APPLICATION_XML, MediaType.APPLICATION_JSON})
        public Person newPerson(Person person) throws IOException {
            int count = PersonDao.instance.getDataProvider().size();
            Long newId = new Long(count+1);
            person.setPersonId(newId);
            PersonDao.instance.getDataProvider().put(newId, person);
            return person;
        }
    ```

* The Update and Delete operations belongs to the **PersonResource** because they operate on a specific instance of Person. Add the following to the **PersonResource.java**

    ```java
    @PUT
    @Consumes({ MediaType.APPLICATION_XML, MediaType.APPLICATION_JSON })
    public Response putPerson(Person person) {
        if (PersonDao.instance.getDataProvider().containsKey(this.id)) {
            person.setPersonId(this.id);
            PersonDao.instance.getDataProvider().put(person.getPersonId(), person);
            return Response.ok(uriInfo.getAbsolutePath()).build();
        } else {
            return Response.noContent().build();
        }
    }

    @DELETE
    public void deletePerson() {
        Person c = PersonDao.instance.getDataProvider().remove(id);
        if (c == null)
            throw new RuntimeException("Delete: Person with " + id
                    + " not found");
    }
    ```

* Now try the added resources with the following requests: 

    ```
    POST /sdelab/person HTTP/1.1
    Accept: application/json
    Content-Type: application/json
    {
     "firstname":"Pinco",
     "lastname":"Pallino",
     "birthdate":"1978-9-2"
    }
    
    
    POST /sdelab/person HTTP/1.1
    Accept: application/json
    Content-Type: application/xml
    <person>
     <firstname>Cristhian</firstname>
     <lastname>Doe</lastname>
     <birthdate>1957-1-11</birthdate>
    </person>
    
    POST /sdelab/person HTTP/1.1
    Accept: application/json
    Content-Type: application/xml
    <person>
    <firstname>Cristhian</firstname>
    <lastname>Parra</lastname>
    <birthdate>1984-06-21</birthdate>
     <healthprofile>
      <weight>72</weight>
      <height>1.72</height>
     </healthprofile> 
    </person> 
    
    PUT /sdelab/person/6 HTTP/1.1
    Accept: application/json
    Content-Type: application/json
    {
     "firstname": "Cristhian Daniel",
     "lastname": "Parra",
     "birthdate": "1984-06-21",
     "healthprofile": {
        "weight": 72,
        "height": 1.70,
     }
    }
    ```
    
* As you can see, Jersey automatically handles the mapping from and to both JSON and XML, and we were able to reuse our models with JAXB annotations. 

### Exercise

* **Exercise 06.03:** Let's extend our HealthProfile API to manage the history of measure udpates. 
 * Add a List attribute named "history" to the "HealthProfile", where a new value will be added every time the health profile is updated
 * Add a service endpoint to get the health profile history of a person
 * **Where should these services go?**
 * **What should I do if I don't want the health profile to be included in the person resource?** 

### Deploy on Heroku
```
heroku login
heroku create --stack cedar --buildpack https://github.com/IntroSDE/heroku-buildpack-ant.git
git push heroku master
heroku open
```
Add /sdelab/person in your browser to the application url, e.g.:
https://still-forest-6253.herokuapp.com/sdelab/person

[1]: https://drive.google.com/open?id=0B7ShzcEnCJFNWENNN1RpYU9xeUk&authuser=0
[2]: https://drive.google.com/open?id=0B7ShzcEnCJFNQ2FfR21FUUk1Y1E&authuser=0
[3]: https://github.com/cdparra/introsde/tree/master/lab06/Examples
[4]: http://www.vogella.com/articles/DesignPatternSingleton/article.html
[5]: https://jersey.java.net/documentation/latest/modules-and-dependencies.html
