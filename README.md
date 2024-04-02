# OAuth 2.0 support for Confluent Platform 

Reference:
- https://docs.confluent.io/platform/current/kafka/authentication_sasl/authentication_sasl_oauth.html


This project is basically the same as the one here (more complete than our original one):
- https://github.com/confluentinc/kafka-images/tree/master/examples/confluent-server-oauth 

## Keycloak Setup

Add following to /etc/hosts:

```
127.0.0.1	keycloak
```

## Start

```shell
docker compose up -d
```

The project starts keycloak and a single broker in KRaft hybrid controller/broker/MDS mode.

## Keycloak

Access Administration Console in http://keycloak:8080/ with user/password admin/admin.

- You will see there is already a realm `cp` created.
- There is also a client defined `client_app1` with Client Secret `client_app1_secret`.
- It has also defined some users and groups. 

All that was cofigured by defining a [realm json](./keycloak-realm-export.json) to be imported on the [compose.yml](./compose.yml).

## Broker

Our broker is already configured in alignment with our keycloak realm before:

- Realm endpoints.
- It uses for super users some of the users defined.

It also maps [a script](./create-certificates.sh) to generate keys and certificates required by the MDS server.

## Test

Get and access token from keycloak:

```shell
ACCESS_TOKEN=$(curl -X POST \
   -H "Authorization: Basic c3VwZXJ1c2VyX2NsaWVudF9hcHA6c3VwZXJ1c2VyX2NsaWVudF9hcHBfc2VjcmV0" \
   -H "Content-Type: application/x-www-form-urlencoded" \
   -d "grant_type=client_credentials" \
   http://localhost:8080/realms/cp/protocol/openid-connect/token | jq -r ".access_token")
```

Note that we are using as client credentials on the curl command the client_id:client_secret encoded in Base64 and as defined in the realm json definition.

Let's grant access to client_app1 over topic test:

```shell
curl -v \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  'http://localhost:8091/security/1.0/principals/User:client_app1/roles/ResourceOwner/bindings' \
  -d '{"scope":{"clusters":{"kafka-cluster":"vHCgQyIrRHG8Jv27qI2h3Q"}}, "resourcePatterns":[{"resourceType":"Topic", "name":"test", "patternType":"LITERAL"}]}'
  ```

  Let's also give access to a consumer group:

  ```shell
curl -v \
   -H "Authorization: Bearer $ACCESS_TOKEN" \
   -H "Content-Type: application/json" \
   -H "Accept: application/json" \
   -X POST 'http://localhost:8091/security/1.0/principals/User:client_app1/roles/ResourceOwner/bindings' \
   -d '{"scope":{"clusters":{"kafka-cluster":"vHCgQyIrRHG8Jv27qI2h3Q"}}, "resourcePatterns":[{"resourceType":"Group", "name":"console-consumer-group", "patternType":"LITERAL"}]}'
  ```

If you check our [client configuration file](./client.properties) its already using the correct realm endpoint and client id and secret (also uses the same consumer group we just granted access).

So now we can call the producer on one side:

```shell
kafka-console-producer \
  --bootstrap-server localhost:9095 \
  --topic test \
  --producer.config client.properties
```

And the consumer on the other:

```shell
kafka-console-consumer \
  --bootstrap-server localhost:9095 \
  --topic test \
  --consumer.config client.properties
```

## Cleanup

```shell
docker compose down -v
```