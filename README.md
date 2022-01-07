# Gitlab with URL token based authentication
### A self hosted Gitlab server protected behind a token based authentication proxy

I ran into this issue recently, I wanted a portfolio of projects that was not compleatly public but that could be accessed by just sharing a link.  
I couldn't find an easy way to do it on Github and also wanted to self host it anyway.

### Overview
The system is made up of three services:
* A dockerized Gitlab instance
* An NginX reverse proxy using the auth_request directive
* [A custom wrote lightweight authentication server](https://github.com/nss0xvu1yqufia4p/authserver)

The Gitlab instance is set up to run within the Docker-Compose network behind the NginX server.  
The route to the Gitlab instance is protected using the auth_request directive. This instructs the server to first make a request to the auth server before forwarding the request through to the Gitlab instance or returning a 403 error. The auth server in turn returns either 2XX response to indicate the user is authenticated, any other response indicates the user is not authenticated

### The auth server
The auth server uses a 128 bit secret, which is passed as an environment variable along with the domain name of the gitlab server.  
#### To generate a token:
* A random 64 bit value is generated
* This is concatenated with the 128 bit secret
* The concatenated value is then hashed using SHA-256
* The hash is truncated to 128 bits and concattenated with the 64 bit value
* This 192 bit value is then encoded using base 64.
* As 192 is divisible by 3 the resulting token does not need padding

#### To validate a token:
* Decode the base 64 encoded token
* Split the result into the 8 byte random value and the truncated hash.
* Use the auth server secret key compute the hash from the 8 byte value
* If the two hashes are equal then the token is valid

The auth server first stores the token from the url as a session cookie and then redirects to the Gitlab server.  
Upon attempting to access the Gitlab server the NginX proxy then calls the auth server again, the auth server then authenticates the request by checking for and validating the session cookie.

This enables a basic level of access control to the Gitlab server keeping all tokens ephemeral and removing the need for a database.
The Gitlab server can then be made fully public and will seem seamless when accessesed through a valid link.  
The server itself is written in rust and built using musl and multi stage builds to produce a container image just 9.9MB in size.
