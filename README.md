# FHIR Access Proxy

This is a simple access-control proxy that sits in front of a FHIR store (e.g.,
a HAPI FHIR server, GCP FHIR store, etc.) and controls access to the FHIR data.

The seed for this repository was taken from the
[hapi-fhirstarters-simple-server](https://github.com/FirelyTeam/fhirstarters/tree/master/java/hapi-fhirstarters-simple-server)
part of
the [FirelyTeam/fhirstarters](https://github.com/FirelyTeam/fhirstarters)
GitHub repository.

Currently this is mostly a proof-of-concept and a work in progress, implementing
ideas in [go/fhir-access-proxy](go/fhir-access-proxy).

# How to run this proxy

First we need to set the followings as the proxy configuration:

- The location of the FHIR store; this comes from `${PROXY_TO}` environment
  variable, e.g.,:

```shell
$ export PROXY_TO=https://healthcare.googleapis.com/v1/projects/fhir-sdk/locations/us/datasets/synthea-sample-data/fhirStores/gcs-data/fhir
```

The above example FHIR store is based on data generated by the experimental
[Synthea-HIV](https://github.com/GoogleCloudPlatform/openmrs-fhir-analytics/tree/master/synthea-hiv)
module.

- If the server needs to be run in `DEV` mode, set the `RUN_MODE` env var:

```shell
$ export RUN_MODE=DEV
```

Setting to `DEV` relaxes some checks done by the server, for example, verifying
the JWT token if the proxy and the IDP are run locally.

- The access token issuer; this comes from `${TOKEN_ISSUER}` variable, e.g.,:

```shell
$ export TOKEN_ISSUER=http://35-224-71-179.nip.io/auth/realms/test
```

The above example is a test authentication server based on
[Keycloak](https://github.com/Alvearie/keycloak-extensions-for-fhir) and has a
single test user in its `test` realm. Note this is not a secure server (hence
no `https`) and is purely set up for test purposes. The Keycloak image's
configuration can be found
[here](docker/keycloak/keycloak_setup.sh). The image itself is maintained at
[gcr.io/fhir-sdk/keycloak-with-setup](`https://gcr.io/fhir-sdk/keycloak-with-setup`)
.

To run:

```
docker run -p 80:8080 -p 443:8443 -e KEYCLOAK_USER=admin \
-e KEYCLOAK_PASSWORD=SOME_PASS -e FHIR_USER=test-fhir \
-e FHIR_PASS=SOME_TEST_PASS --name keycloak-h2-setup \
gcr.io/fhir-sdk/keycloak-with-setup
```

- GCP credentials if the FHIR server is a GCP FHIR store. Here are some options:

* If you have access to the FHIR store, you can use your own credentials by
  doing [application-default login](https://cloud.google.com/sdk/gcloud/reference/auth/application-default/login)
* Use a service account with required access. In the above example `fhir-sdk`
  project, the `smart-proxy-account` service account is created for this
  purpose. So you can run the proxy docker image on a VM with this service
  account.
* [not-recommended] You can create and download a key file for the above service
  account, then use it with"
  ```shell
  $ export GOOGLE_APPLICATION_CREDENTIALS="PATH_TO_THE_JSON_KEY_FILE"
  ```

Once you have set all of the above, then you can run the proxy server by:

```shell
mvn jetty:run -Djetty.http.port=8081
```

There is also an option for running the proxy through docker:

```shell
$ docker run -p 8080:8080 \
-e TOKEN_ISSUER=http://35-224-71-179.nip.io/auth/realms/test \
-e PROXY_TO=https://healthcare.googleapis.com/v1/projects/fhir-sdk/locations/us/datasets/synthea-sample-data/fhirStores/gcs-data/fhir \
gcr.io/fhir-sdk/fhir-proxy:latest
```

Note if this is not on a VM with proper service account (e.g., on a local host),
you need to pass GCP credentials to it, for example by mapping the
`.config/gcloud` volume (i.e., add `-v ~/.config/gcloud:/root/.config/gcloud` to
the above command).

# How to use this proxy

Once the proxy is running, we first need to fetch an access token from the
`${TOKEN_ISSUER}`; you should know the test username and password plus the
`client_id` (please disregard `scope` for now; it can be skipped):

```shell
$ curl -X POST -d 'client_id=CLIENT_ID' -d 'username=USER' -d 'password=PASS' \
-d 'grant_type=password' -d 'scope=email' \
"http://35-224-71-179.nip.io/auth/realms/test/protocol/openid-connect/token"
```

we need the `access_token` of the returned JSON to be able to convince the proxy
to authorize our FHIR requests (note there is also a `refresh_token` in the
above response). Here is a simple way to automate this:

```shell
$ export ACCESS_TOKEN=$(curl -X POST -d 'client_id=CLIENT_ID' -d 'username=USER' \
-d 'password=PASS' -d 'grant_type=password' -d 'scope=patient/*.read' \
"http://35-224-71-179.nip.io/auth/realms/test/protocol/openid-connect/token" | \
python3 -m json.tool | awk '/access_token/{gsub(/"|,/, "", $2); print $2;}')
```

with this done, you can access the FHIR store with a valid `access_token` as you
wish:

```shell
$ curl -X GET -H "Authorization: Bearer ${ACCESS_TOKEN}" \
-H "Content-Type: application/json; charset=utf-8" \
'http://localhost:8081/Patient/f16b5191-af47-4c5a-b9ca-71e0a4365824'
```

```shell
$ curl -X PUT -H "Authorization: Bearer ${ACCESS_TOKEN}" \
-H "Content-Type: application/json; charset=utf-8" \
'http://localhost:8081/Patient/f16b5191-af47-4c5a-b9ca-71e0a4365824' \
-d @Patient_f16b5191-af47-4c5a-b9ca-71e0a4365824_modified.json
```

You can impose patient level restrictions by setting `${ACCESS_CHECKER}`:

```shell
$ export ACCESS_CHECKER=list
```

With this setup, the user should have a valid user attribute `patient_list`
which is the ID of a `List` FHIR resource with all the patients that this user
has access to.
