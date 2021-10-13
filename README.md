Short Keycloak Glossary
Below there are terms that I used in this article and their meaning within the Keycloak:

Admin Console
Web-based GUI where you can “click out” all configurations required by your instance to work as you desire. 
Users
Entities that are able to log into the protected system. They have a set of editable attributes and can be a part of a group and/or have specific roles assigned to them.
Role
A type or category of user that exists within an organization. Applications often base on roles to restrict access to resources.  
Groups
Entities that are used to manage a set of users. Similarly to users, groups have editable attributes and you can also assign roles to a group. Users that become members of a group inherit their attributes and roles.
Realm
A realm manages a set of users, credentials, roles, and groups. A user belongs to and logs into a realm. Realms are isolated from one another and can only manage and authenticate the users that they control.
Client
Entities that can request Keycloak to authenticate a user. Most often, clients are applications and services which want to use Keycloak to secure themselves. Clients may also be entities wanting to request identity information or an access token so that they can securely invoke other services secured by Keycloak.
Identity Token
A token providing identity information about the user. Part of the OpenID Connect specification.
Access Token
A token that can be provided as part of an HTTP request. 
Local Keycloak Instance
Before we start doing any integration and configurations we need to run our local Keycloak instance. I recommend using the Keycloak Docker image but you can use the standalone version as well.

In the case of Docker Image, the following command should do the job.

Shell
1
docker run -p 8090:8080 -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin quay.io/keycloak/keycloak:15.0.2


It will run Keycloak image if you have one locally or pull it and then run if you do not. Additionally, it will forward docker image port 8080 to 8090 on your local device (it will be important later on). It will also set Keycloak Admin Console credentials to admin/admin.

For standalone distribution, first you need to download the .zip file from the official page. After downloading the .zip file, you need to run the commands below in the download directory.

Shell
1
unzip keycloak-15.0.2.zip 
2
cd keycloak-15.0.2/bin
3
./standalone.sh -Djboss.socket.binding.port-offset=10


Unfortunately, to be able to log into Admin Console you will have to create the admin user yourself. Go to http://localhost:8090/auth/ and fill the form in the Administrator Console part. For the purpose of this exercise, admin/admin will be enough.





Then you should be able to log in to Keycloak Admin Console.



Configuration
In this paragraph, I will describe all configurations needed by Spring and Keycloak to work together. We will start by configuring the Keycloak instance and then we will move on to Spring Boot configuration.

Keycloak
1. Log in to Admin Console on localhost:8090

2. Create a new realm named demo







3. Create a new client named demo-app with public access type, set its Valid Redirect URIs to * (I do not recommend this for any production services), and save your changes.









4. Create a new user with credentials test/test











For the purpose of this article, the above configuration of Keycloak will be sufficient. Now we move on to configuring your Spring Boot application. I assume that you already have the Spring Boot project and you are using Maven. For Gradle based project the only difference will be the style of adding project dependencies.

Spring Boot
As I mentioned in the previous article, integrating Spring Boot and Keycloak will require a library named: spring-boot-keycloak-starter but it is not all, you will have to do a few more things.

1. Add the library mentioned above in your Maven or Gradle build file as dependency.

2. Add spring-boot-starter-security in your Maven or Gradle build file as dependency.

3. Add org.keycloak.bom.keycloak-adapter-bom as dependency in dependecnymanager tag.



XML
1
<dependencies>
2
        <dependency>
3
            <groupId>org.keycloak</groupId>
4
            <artifactId>keycloak-spring-boot-starter</artifactId>
5
            <version>15.0.2</version>
6
        </dependency>
7
        <dependency>
8
            <groupId>org.springframework.boot</groupId>
9
            <artifactId>spring-boot-starter-security</artifactId>
10
        </dependency>
11
    </dependencies>
12
​
13
    <dependencyManagement>
14
        <dependencies>
15
            <dependency>
16
                <groupId>org.keycloak.bom</groupId>
17
                <artifactId>keycloak-adapter-bom</artifactId>
18
                <version>15.0.2</version>
19
                <type>pom</type>
20
                <scope>import</scope>
21
            </dependency>
22
        </dependencies>
23
    </dependencyManagement>


4. Configure Spring Boot properties required by Keycloak.

keycloak.auth-server-url=http://localhost:8090/auth — URL to your Keycloak instance

keycloak.realm=demo — name of the realm used to hold users’ data of our application

keycloak.resource=demo-app — name of the client which our Spring Boot application will use to integrate with





keycloak.public-client=true — Spring Boot adapter will not send credentials to authenticate within Keycloak, it will use only client_id, demo-app in our case.



Beware

If your Spring Boot application runs without these properties being set, an exception will be thrown in runtime (welcome to the Spring Magic). What is even more disturbing, it will not happen straight away after the application starts but only when you try to authenticate your user.

Additionally, I will add keycloak.principal-attribute property and set it to preferred_username. This parameter is not required by Keycloak or Spring Boot but will make examples much more readable. With such configuration, we are able to easily extract Keycloak user name from a token available on Spring Boot side. By default - without this configuration, after calling .getName() on Authentication object we will get Keycloak user id which is standard UUID.

5. Create a class that will extend KeycloakWebSecurityConfigurerAdapter and add @EnableWebSecurity annotation. We will use this class to configure security for your application.



Java


6. Now we will fill our security class and create all beans required by Spring.

Firstly, we will create two Spring beans: SessionAuthenticationStrategy, KeycloakConfigResolver.

The first one is responsible for our HTTP sessions behavior when authentication occurs, while the second one is used to resolve Keycloak configuration properties used by Spring Boot. This bean will resolve the properties mentioned in the third point.

For SessionAuthenctionStartegy we should use RegisterSessionAuthenticationStrategy because we are using public Keycloak client for authentication.

7. Next you need to override two configuration methods which will add Keycloak Auth provider and configure our REST API security.

8. The last thing to do is create of a REST endpoint that we will use to test our API security. Here I have prepared a very simple hello-world style controller and endpoint.



Java


That is all the configuration you will need to secure your REST API with Keycloak and Spring Boot. Now let’s test it.

Test
For testing our integration I recommend Postman. In my opinion, Keycloak requests can be somewhat complex, and creating such a request in curl can take some time. If everything is done correctly, sending such a request http://localhost:8080/hello/ from the browser should result in being redirected to the Keycloak login page.  









In Postman result should be similar but with no CSS.







Now we can see that our API is protected by Keycloak and accessing an endpoint will require authentication within the Keycloak. The next thing we have to do is obtaining the access token from Keycloak. Fortunately, Keycloak exposes REST API that can be used to request and refresh access tokens. The following request should be sufficient to get our access token.







Here is a list of request parameters used with a short description:

client_id: the name of the client which you want to use to authenticate your user

username: your user’s name

password: your user’s password

grant_type: a way to exchange a user’s credentials for an access token; here we use the password as Grant Type (OAuth specific mechanism) so we use the username and password to authenticate ourselves within Keycloak and get the correct token

Now we need to go back to the previous request. Then you have to change the authorization type to Bearer Token and copy value form response’s access_token field and paste it as value to Token filed on the right side of the Postman screen.







The response from our backend should be similar to the one on the screen.

Et voilà — your API is secured with Keycloak and you have learned something new (at least I hope so).

Summing up
I prepared step-by-step instructions on how to integrate Keycloak with Spring Boot and secure your REST API. I explained Spring Boot required parameters for Keycloak integration and full code configuration actually responsible for securing an API. I also described, with little help from Postman, how to test if our backend is really secured. Additionally, I explained why we send those exact parameters in token request and what we get as a response from Keycloak. I hope my article deepened your Keycloak or Spring Boot knowledge. Thank you for your time.
