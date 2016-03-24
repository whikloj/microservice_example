## Using CLAW PHP microservices

All of these examples are using the PHP microservices. This is **not** using the Camel services.

1. Setup the Virtual Machine

  This assumes you have a working [vagrant](https://www.vagrantup.com/) environment.

 1. Clone this repository https://github.com/whikloj/islandora.git to a new location that does not overlap with your old Islandora code.
   `git clone https://github.com/whikloj/islandora.git CLAW`
 2. Go to the `install` sub-directory and start the vagrant VM `vagrant up`.
 3. Once it has completed successfully you should see something like this
    ```
    ==> default: paragonie/random_compat suggests installing ext-libsodium (Provides a modern crypto API that can be used to generate random bytes.)
    ==> default: ramsey/uuid suggests installing ramsey/uuid-console (A console application for generating UUIDs with ramsey/uuid)
    ==> default: ramsey/uuid suggests installing ircmaxell/random-lib (Provides RandomLib for use with the RandomLibAdapter)
    ==> default: ramsey/uuid suggests installing ext-uuid (Provides the PECL UUID extension for use with the PeclUuidTimeGenerator and PeclUuidRandomGenerator)
    ==> default: ramsey/uuid suggests installing moontoast/math (Provides support for converting UUID to 128-bit integer (in string form).)
    ==> default: ramsey/uuid suggests installing ramsey/uuid-doctrine (Allows the use of Ramsey\Uuid\Uuid as Doctrine field type.)
    ==> default: ramsey/uuid suggests installing ext-libsodium (Provides the PECL libsodium extension for use with the SodiumRandomGenerator)
    ==> default: Writing lock file
    ==> default: Generating autoload files
    ```
 4. You now have a virtual machine with:
   * Drupal at http://localhost:8000
   * [Fedora 4](https://wiki.duraspace.org/display/FF/Fedora+Repository+Home) at http://localhost:8080/fcrepo/rest
   * [Blazegraph](https://wiki.blazegraph.com/wiki/index.php/Main_Page) at http://localhost:8080/bigdata
   * Apache Karaf at http://localhost:8181
   * CLAW PHP microservices at http://localhost:8282/

 5. Have fun 


1. Create a resource in Fedora.

With your VM up and running you can browse to <a href="http://localhost:8080/fcrepo/rest" target="_blank">your Fedora instance</a>.
You should see something like this. 
[[https://github.com/whikloj/microservice_example/blob/master/images/blank_fcrepo.jpg|alt=Fedora4]]

Now you need curl, if you don't have it installed you can log into the virtual machine by typing `vagrant ssh` from the `/install` directoy (same as when you did `vagrant up`).

This makes use of the [ResourceService](https://github.com/whikloj/islandora/tree/sprint-002-thinCollection/services/ResourceService) and is meant as a straight push of the data you give it to Fedora.

To make things easier we are going to create a PCDM Object for now. Create a file in your current directory called `object.ttl`. This will be a text file and the format we are going to use is [Turtle](https://www.w3.org/TR/turtle/) (here is a [helpful tutorial](http://haystack.csail.mit.edu/blog/2008/11/06/a-quick-tutorial-on-the-tutrle-rdf-serialization/) on the syntax). But you could also use [N-Triples](https://www.w3.org/TR/n-triples/), [N3](https://www.w3.org/TR/n-triples/), [RDF XML](https://www.w3.org/TR/rdf-syntax-grammar/) or [JSON-LD](https://www.w3.org/TR/json-ld/).

Most of the time in the code we will be using JSON-LD, but for this exercise turtle is easier.

The `object.ttl` should look like this:
```
@prefix pcdm: <http://pcdm.org/models#> .
@prefix nfo: <http://www.semanticdesktop.org/ontologies/2007/03/22/nfo/v1.2/> .
@prefix dc: <http://purl.org/dc/elements/1.1/> .

<> a pcdm:Object ;
  dc:title "Object Number 1" ;
  nfo:uuid "4a51d7bc-83ed-4468-803a-17c411520bc0" .
```

The UUID is one I generated [here](https://www.uuidgenerator.net/), each object should have a unique UUID. So if you want to create multiple objects with this file, make sure to change the UUID before you post it again.

Now to create an object we are going to use the PUT method, which means we can tell Fedora where to put it.

We will put a resource right at the root of the repository and call it *object1*.

`curl -i -XPUT -H"Content-type: text/turtle" --data-binary "@object.ttl" http://localhost:8282/islandora/resource//object1`

**Important** to use the **PUT** HTTP method you have to have 2 backslahes between resource and the name of the object. Why? [Read more]

If all goes well you should see something like:
```
HTTP/1.1 201 Created
Date: Thu, 24 Mar 2016 02:17:49 GMT
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.5.9-1ubuntu4.14
ETag: "18112aa1d85a3448236ec82164ab15bc2d9e2ec8"
Cache-Control: private, must-revalidate
Last-Modified: Thu, 24 Mar 2016 02:17:49 GMT
Location: http://localhost:8080/fcrepo/rest/object1
Content-Length: 41
Connection: close
Content-Type: text/plain; charset=UTF-8

http://localhost:8080/fcrepo/rest/object1
```

This is the response direct from Fedora, so you see the Fedora URL in the `Location:` header and the body.

If you browse to http://localhost:8080/fcrepo/rest/object1 you should see your object in the Fedora interface.
[[https://github.com/whikloj/microservice_example/blob/master/images/fcrepo_object1.jpg|alt="Object 1 in Fedora"]]

Congrats!

You can also perform the above steps but use a **POST** instead of a **PUT**, the difference is that you provide no name and Fedora creates one for you.

Let's put another object as a child of object1.

Open your `object.ttl` file and edit it to be:
```
@prefix pcdm: <http://pcdm.org/models#> .
@prefix nfo: <http://www.semanticdesktop.org/ontologies/2007/03/22/nfo/v1.2/> .
@prefix dc: <http://purl.org/dc/elements/1.1/> .

<> a pcdm:Object ;
  dc:title "Object Number 2" ;
  nfo:uuid "5fa71ed6-f5f3-4831-8662-b1b132815a52" .
```

I've modified the *dc:title* and the *nfo:uuid* to be different from object1.

Now if we look back at [the routes](https://github.com/whikloj/islandora/blob/sprint-002-thinCollection/services/ResourceService/src/Provider/ResourceServiceProvider.php#L142-L143) you'll see that the POST route only accepts one parameter. This is the parent Uuid or nothing to place it at the root level.

To place our resource as a child of Object 1 we need to get the UUID of Object 1. You can check Fedora and copy the UUID, it appears in the properties like this.
[fcrepo_uuid.jpg]

`curl -i -XPOST -H"Content-type: text/turtle" --data-binary "@object.ttl" http://localhost:8282/islandora/resource/4a51d7bc-83ed-4468-803a-17c411520bc0`

We use the UUID as Drupal will give us one for new resources and we can match them up to link Drupal to Fedora.

If your above command works you'll see something like:
```
HTTP/1.1 201 Created
Date: Thu, 24 Mar 2016 02:32:04 GMT
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.5.9-1ubuntu4.14
ETag: "6605471b8cd6f18a1d9b1170511f26f8bc156785"
Cache-Control: private, must-revalidate
Last-Modified: Thu, 24 Mar 2016 02:32:05 GMT
Location: http://localhost:8080/fcrepo/rest/object1/4d/7c/07/83/4d7c0783-7efd-46de-adf6-ac0bda9784a7
Content-Length: 90
Connection: close
Content-Type: text/plain; charset=UTF-8

http://localhost:8080/fcrepo/rest/object1/4d/7c/07/83/4d7c0783-7efd-46de-adf6-ac0bda9784a7
```
**Note**: Your *Location* will not match the above, but should otherwise the results should be the same. If you browse to the Fedora URL you should see your new **Object Number 2**. 

You'll also notice that it has a property `fedora:hasParent http://localhost:8080/fcrepo/rest/object1`, if you click on that link and goto **Object Number 1** you'll that it has a complementary property 
`ldp: contains http://localhost:8080/fcrepo/rest/object1/4d/7c/07/83/4d7c0783-7efd-46de-adf6-ac0bda9784a7`

Awesome.

2. Delete a resource

Deleting a resource is much the same. Let's delete **Object Number 2**. To delete we use the **DELETE** HTTP method, and the UUID for the resource. Object Number 2 has a UUID of `5fa71ed6-f5f3-4831-8662-b1b132815a52`.

**Important**: The theory in linked data is that rarely would you want to reuse a URL, once that URL has been associated with a object/resource/whatever. You might delete it, but rarely will you use it again. **Tombstones** are left behind by Fedora 4 ([API](https://wiki.duraspace.org/display/FEDORA4x/RESTful+HTTP+API#RESTfulHTTPAPI-RedDELETEDeletearesource)) when you delete a resource. They are located at http://localhost:8080/fcrepo/rest/path/to/resource/fcr:tombstone.

You can delete a tombstone and the microservices deal with that by accepting a `?force=true` querystring on the URL. 

So....

 1. To delete only Object Number 2 and leave the tombstone, you do

    `curl -i -XDELETE http://localhost:8282/islandora/resource/5fa71ed6-f5f3-4831-8662-b1b132815a52`
    
 2. To delete Object Number 2 and remove it's tombstone, you do
 
    `curl -i -XDELETE http://localhost:8282/islandora/resource/5fa71ed6-f5f3-4831-8662-b1b132815a52?force=true`
    
Whichever you choose to do you should see:
```
HTTP/1.1 204 No Content
Date: Thu, 24 Mar 2016 02:46:10 GMT
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.5.9-1ubuntu4.14
Connection: close
Cache-Control: no-cache
Content-Length: 0
Content-Type: text/html
```

The object is gone, you can check with your browser at the old URL and notice it is no longer `ldp:contained` by **object1**.

3. Create a PCDM Collection

Everything before this has been creating straight Fedora 4 objects. Now we will use a more complex microservice to create a PCDM Collection object.

The CollectionService has [3 routes](https://github.com/whikloj/islandora/blob/sprint-002-thinCollection/services/CollectionService/src/Provider/CollectionServiceProvider.php#L86-L92)

* The first creates a Collection by POSTing to `/islandora/collection`. 
* The second adds an object as a member of the collection by POSTing to `/islandora/collection/<collection uuid>/member/<object uuid>`
* The third removes an object from a collection by DELETE'ing to `/islandora/collection/<collection uuid>/member/<object uuid>`
  
This microservice is slightly different from the ResourceService in that we know that a Collection needs a UUID to be used by CLAW. So we set one if there isn't one previously. 

What this means is even if we don't provide a UUID, one will be assigned automatically.

So let's create a pcdm:Collection at the root level

`curl -i -XPOST http://localhost:8282/islandora/collection`

There is currently no PUT route, so you can't yet specify the path of the resource.

The above will result is something like
```
HTTP/1.1 201 Created
Date: Thu, 24 Mar 2016 03:09:36 GMT
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.5.9-1ubuntu4.14
ETag: "7253f1619e0954a11c5f1138ea55ba9cbe6f3cd2"
Cache-Control: private, must-revalidate
Last-Modified: Thu, 24 Mar 2016 03:09:37 GMT
Location: http://localhost:8282/islandora/resource/f07c0d38-8290-4c2d-877c-233dfce374ea
Content-Length: 77
Connection: close
Link: <http://localhost:8282/islandora/resource/f07c0d38-8290-4c2d-877c-233dfce374ea/members>; rel="hub"
Content-Type: text/plain; charset=UTF-8

http://localhost:8282/islandora/resource/f07c0d38-8290-4c2d-877c-233dfce374ea
```

You'll notice that the **Location** and **Link** headers as well as the body of the response all refer to our microservices. In fact if you go in your browser to the URL in your Location header. You will see the RDF for that object returned by the microservices.

Or we can use the microservices with curl...

`curl -i http://localhost:8282/islandora/resource/f07c0d38-8290-4c2d-877c-233dfce374ea`

Which should display alot of RDF for your Collection.

You should actually go to your Fedora repository and look at the Collection, because we actually created 2 objects.

We created the pcdm:Collection object, and then we created a child called **members** that is an [Indirect Container](https://www.w3.org/TR/ldp/#ldpic)

[[https://github.com/whikloj/microservice_example/blob/master/images/pcdm_collection.jpg|alt="PCDM Collection Object"]]

Here you can see the UUID, the rdf:type of pcdm:Collection and a child called **members**. 

Click that link and lets look at it.

[[https://github.com/whikloj/microservice_example/blob/master/images/indirect_container.jpg|alt="Indirect container in Fedora 4" ]]

The 3 important properties on this indirect container are 
* ldp:hasMemberRelation - http://pcdm.org/models#hasMember
* ldp:insertedContentRelation - http://www.openarchives.org/ore/terms/proxyFor
* ldp:membershipResource - http://localhost:8080/fcrepo/rest/68/6f/28/07/686f2807-f0cf-4459-8b4a-ea06ebcbd5e6

Those 3 properties define how we will add a property **somewhere else** in the repository whenever an object is added to this indirect container.

When we add an object to this indirect container, it will check to see if that object has a property matching the *ldp:insertedContentRelation*. It will then add a new property to the resource defined by *ldp:membershipResource* with a relationship of *ldp:hasMemberRelation*.

So the triple would be

`<http://localhost:8080/fcrepo/rest/68/6f/28/07/686f2807-f0cf-4459-8b4a-ea06ebcbd5e6> <http://pcdm.org/models#hasMember> <whatever is in proxyFor>`

So if we add an object with RDF like:
```
@prefix pcdm: <http://pcdm.org/models#> .
@prefix ore: <http://www.openarchives.org/ore/terms/> .
 
<> a pcdm:Object ;
  ore:proxyFor <http://localhost:8080/fcrepo/rest/object1> .
```

Then our triple becomes
`<http://localhost:8080/fcrepo/rest/68/6f/28/07/686f2807-f0cf-4459-8b4a-ea06ebcbd5e6> <http://pcdm.org/models#hasMember> <http://localhost:8080/fcrepo/rest/object1>`

We have added a pcdm:hasMember property to our Collection pointing to our Object Number 1, without touching either of them.

Let's do it.

4. Adding a member to a pcdm:Collection

To do this we will need the UUID for the collection and the resource.
* Collection - f07c0d38-8290-4c2d-877c-233dfce374ea
* Object - 4a51d7bc-83ed-4468-803a-17c411520bc0

Remember the pattern for this is 
`http://localhost:8282/islandora/collection/<collection uuid>/member/<object uuid>`

or

`curl -i -XPOST http://localhost:8282/islandora/collection/f07c0d38-8290-4c2d-877c-233dfce374ea/member/4a51d7bc-83ed-4468-803a-17c411520bc0`

The response will be about the creation of the proxy object and be a child of the **members** indirect container.

```
HTTP/1.1 201 Created
Date: Thu, 24 Mar 2016 03:47:50 GMT
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.5.9-1ubuntu4.14
ETag: "023bf18a79c3645040e4a06f7db0f8d62d701b51"
Cache-Control: private, must-revalidate
Last-Modified: Thu, 24 Mar 2016 03:47:51 GMT
Location: http://localhost:8282/fcrepo/rest/68/6f/28/07/686f2807-f0cf-4459-8b4a-ea06ebcbd5e6/members/1c/f2/93/58/1cf29358-fc4a-47c1-8b4c-9f0e199fc87f
Content-Length: 139
Connection: close
Content-Type: text/plain; charset=UTF-8

http://localhost:8282/fcrepo/rest/68/6f/28/07/686f2807-f0cf-4459-8b4a-ea06ebcbd5e6/members/1c/f2/93/58/1cf29358-fc4a-47c1-8b4c-9f0e199fc87f
```

Go back to your web browser and check out your **members** indirect container and its parent the pcdm:Collection.

The indirect container **members** now has a child object.
[[https://github.com/whikloj/microservice_example/blob/master/images/indirect_container2.jpg|alt="Indirect Container with membership property"]]

And the pcdm:Collection now has a pcdm:hasMember property to **Object Number 1**.
[[https://github.com/whikloj/microservice_example/blob/master/images/pcdm_collection2.jpg|alt="Collection with pcdm:hasMember property"]]


5. Deleting a member from a pcdm:Collection

If you remember the routes, you'll remember that this uses the same URL as adding a member. But we use the **DELETE** HTTP method.

So taking the URL from above and replacing POST with DELETE, we get.

`curl -i -XDELETE http://localhost:8282/islandora/collection/f07c0d38-8290-4c2d-877c-233dfce374ea/member/4a51d7bc-83ed-4468-803a-17c411520bc0`

and the result is

```
HTTP/1.1 204 No Content
Date: Thu, 24 Mar 2016 03:56:39 GMT
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.5.9-1ubuntu4.14
Connection: close
Cache-Control: no-cache
Content-Length: 0
Content-Type: text/html
```


