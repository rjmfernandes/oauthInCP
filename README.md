# OAuth 2.0 support for Confluent Platform 

Reference:
- https://docs.confluent.io/platform/current/kafka/authentication_sasl/authentication_sasl_oauth.html


# Keycloak

Add following to /etc/hosts:

```
127.0.0.1	keycloak
```

Start:

```shell
docker compose up -d keycloak
```

Access Administration Console in http://keycloak:8080/ with user/password admin/admin and create a Realm `myrealm`.

Create a Client named `my-resource-center`. Click Next. 

Toggle everything to ON except "Implicit flow". Click Save.

Type the Root URL for your application. For example:

```
http://keycloak:8080/my-resource-server
```

You should have something like this:

![Capability Config](kc-1.jpg)

Under Credentials copy the Client Secret for Client Authenticator "Client Id and Secret". Update the file [client.properties](./client.properties) with the corresponding copied secret.     

Under Client scopes go to `my-resource-center-dedicated` and add a Mapper (name it "test" for example) from configuration Audience and set Included Client Audience `my-resource-center`.

# Start CP

```shell
docker compose up -d broker
```

Execute:

```shell
kafka-topics --bootstrap-server localhost:9095 --topic test --create --partitions 3 --replication-factor 1 --command-config ./client.properties
```

And after:

```shell
kafka-topics --bootstrap-server localhost:9095 --list --command-config ./client.properties
```