Grendel
=======

What Grendel Is
---------------

Grendel is a RESTful web service which allows for the secure storage of users'
documents. When a Grendel user is created, an OpenPGP keyset (a master key for
signing/verifying and a sub key for encrypting/decrypting) is generated. When
the user stores a document, the document is signed with the user's master key
and encrypted with their sub key.


How To Build It
---------------

Run `mvn clean package` and look for the JAR file in the `target` directory.


How To Configure It
-------------------

Grendel requires a single configuration file, usually named
`grendel.properties`. It should look something like this:

    hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
    hibernate.connection.username=grendel
    hibernate.connection.password=magicsecretsauce
    hibernate.connection.url=jdbc:mysql://grendel.db:3306/grendel?zeroDateTimeBehavior=convertToNull
    hibernate.c3p0.min_size=10
    hibernate.c3p0.max_size=50
    hibernate.generate_statistics=true

The `zeroDateTimeBehavior=convertToNull` option may be required to compensate
for MySQL's storage of null `DATETIME` values as all-zero strings.

It's recommended that your properties file be stored so that only the system
user running the Grendel server can access it.


How To Run It
-------------

For help, simply run:
    
    java -jar target/grendel-${VERSION}.jar

(Replace `${VERSION}` with whatever version you're running.)

To generate a full database schema script, run this:
    
    java -jar target/grendel-${VERSION}.jar schema -c grendel.properties

To generate a migration database schema script, run this:

    java -jar target/grendel-${VERSION}.jar schema --migration -c grendel.properties

To run Grendel as a web service, run this:
    
    java -jar target/grendel-${VERSION}.jar server -c grendel.properties -p 8080

This will run Grendel on port 8080.


The Users Resource (`/users/`)
------------------------------

The **Users Resource** provides access to the collection of Grendel users.

No authentication is required for this resource.

`GET`
-----

Sending a `GET` request to `/users/` will return an `application/json` object
with a list of all users' ids and URIs:

    > GET /users/ HTTP/1.1
    > Accept: application/json
    >
    
    < HTTP/1.1 200 OK
    < Content-Type: application/json
    <
    < {
    <   "users":[
    <       {
    <           "id":"user1",
    <           "uri":"http://grendel.server/users/user1"
    <       }
    <   ]
    < }

The `uri` property is the absolute URI of the user's **User Resource**.


`POST`
------

Sending a `POST` request to `/users` with an `application/json` object will
create a new user and return the URI of the new user's **User Resource**.

    > POST /users/ HTTP/1.1
    > Content-Type: application/json
    >
    > {
    >   "id": "newuser",
    >   "password": "woowoo"
    > }
    
    < HTTP/1.1 201 Created
    < Location: http://grendel.server/users/newuser

Both the `id` and the `password` properties are **required**. This action may
take some time (~1s), as Grendel will generate an OpenPGP keyset for the user.

**N.B.:** The user's id is immutable; use an immutable piece of data (e.g., a
primary key) instead of a modifiable piece of data (e.g., a username or email
address).


The User Resource (`/users/{id}`)
---------------------------------

The **User Resource** provides access an individual user's metadata.

Basic authentication, using the user's id and password, is required for all
methods of this resource.

`GET`
-----

Sending a `GET` request to `/users/codahale` will return an `application/json`
object with information about the user `codahale`:

    > GET /users/codahale HTTP/1.1
    > Accept: application/json
    > Authorization: Basic Y29kYWhhbGU6d29vd29v
    >
    
    < HTTP/1.1 200 OK
    < Content-Type: application/json
    < 
    < {
    <   "id":"codahale",
    <   "modified-at":"20091227T211120Z",
    <   "created-at":"20091227T211120Z",
    <   "documents":[
    <       {
    <           "uri":"http://grendel.server/users/codahale/document1.txt",
    <           "name":"document1.txt"
    <       }
    <   ],
    <   "keys":"[2048-RSA/0A895A19, 2048-RSA/39D1621B]"
    < }

The `documents` property is a list of all the user's documents, including
absolute URIs.


`PUT`
-----

Sending a `PUT` request to `/users/codahale` with an `application/json` object
will change the user `codahale`'s password:

    > PUT /users/codahale HTTP/1.1
    > Content-Type: application/json
    > Authorization: Basic Y29kYWhhbGU6d29vd29v
    >
    > {
    >   "password": "secretstuff"
    > }
    
    < HTTP/1.1 204 No Content

The `password` property is **required**. The Basic authentication should use the
old password; the next request will require Basic authentication credentials
with the new password.


`DELETE`
--------

Sending a `DELETE` request to `/users/codahale` will delete user `codahale`
**and** all their documents:

    > DELETE /users/codahale HTTP/1.1
    > Authorization: Basic Y29kYWhhbGU6d29vd29v
    >
    
    < HTTP/1.1 204 No Content


The Document Resource (`/users/{id}/{name}`)
--------------------------------------------

The **Document Resource** provides access a user's documents.

Basic authentication, using the user's id and password, is required for all
methods of this resource.


`GET`
-----

Sending a `GET` request to `/users/codahale/document1.txt` will return the
document named `document1.txt` belonging to user `codahale` in whatever content
type the document was stored with:

    > GET /users/codahale/document1.txt HTTP/1.1
    > Authorization: Basic Y29kYWhhbGU6d29vd29v
    >
    
    < HTTP/1.1 200 OK
    < Content-Length: 10
    < Cache-Control: private, no-cache, no-store, no-transform
    < Content-Type: text/plain
    <
    < yay for me


`PUT`
-----

Sending a `PUT` request to `/users/codahale/document1.txt` will store the
request entity as `document1.txt` with the specified content type:

    > PUT /users/codahale/document1.txt HTTP/1.1
    > Content-Type: text/plain
    > Authorization: Basic Y29kYWhhbGU6d29vd29v
    >
    > i am super secret

    < HTTP/1.1 204 No Content

Sending `PUT` request to an existing document will overwrite its contents;
doing so to a non-existent document will create it.


`DELETE`
--------

Sending a `DELETE` request to `/users/codahale/document1.txt` will delete the
document named `document1.txt` belonging to user `codahale`:

    > DELETE /users/codahale/document1.txt HTTP/1.1
    > Authorization: Basic Y29kYWhhbGU6d29vd29v
    >

    < HTTP/1.1 204 No Content