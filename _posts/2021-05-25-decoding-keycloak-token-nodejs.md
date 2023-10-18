---
layout: post
title:  "How to decode a Keycloak JWT in NodeJS v16"
date:   2021-05-26 15:45:32 +0530
categories: keycloak, jwt, typescript
---
Level: `advanced`

Pre-requisites: NodeJS (TypeScript), Keycloak, JWT Concepts

### Overview
[Keycloak][keycloak_link] is a Free & Open Source Identity Management Platform that is rising in popularity and comes with out-of-the-box support for OpenIDConnect & SAML Protocols which enable you to implement Single Sign-On (SSO). [`keycloak-connect`][npm_kc_connect] is Keycloak's official NodeJS Adapter for backend apps and enables authorization for APIs and backend services by providing session and middleware methods.

Few things to be noted (of utmost importance):
  - We'll be using NodeJS v16 as class [**`X509Certificate`**][x5c_documentation] is not available in the built-in `crypto` module of earlier versions.
  - It is assumed that a user and keycloak client are already created and the credentials (username and password) are handy. Any setup in Keycloak is out of the scope of this guide.
  - The Code snippets are written in typescript and we'll be using the `request-promise` library for the HTTP calls as it returns a promise. Neat!
  - Initialize a fresh npm project and initialize a typescript project inside the project directory using `tsc --init`.

### Problem Statement
A lot of times, it is possible that only the decoded token is required for some easy validation checks. So instead of using the full-blown keycloak adapter and its middlewares for the API Routes, we can decode the token contents and use the basic `if else` statements.


### The Concept
A JWT has 3 parts: **header, payload** and **signature** and 2 signing methods:
  - **Using a Secret**: Tokens are signed using a Secret Key which is also used for verifying the token
  - **Public-Private Key Cryptography**: The token issuing (`iss` in the token payload) server signs the token using their Private key and the client applications or services can verify the token using the Public Key.

We will skip the secret usage method as it is easy to set up and use. Let us be concerned about the second method only for now, *i.e.* using the Public Key.

Let's begin!!

### Finding the Public Key
Keycloak (or any other OpenIDConnect compatible IdP) supports JWT and is required to have a `.well-known/openid-configuration` URL to publish its all configurations in a standard JSON as per OIDC specs.

You can get the URL by clicking the **OpenID Endpoint Configuration** link in your realm settings or use this pattern: `$KEYCLOAK_SERVER_URL/auth/realms/$REALM/.well-known/openid-configuration`.

![Keycloak Realm settings](/assets/kc_realm_settings.png)

To get the Public Key, we need the JSON Web Key Sets (JWKS) Endpoint from the well-known configuration response which looks like this:
```jsonc
{
  "issuer": "http://localhost:8080/auth/realms/demo",
  "authorization_endpoint": "http://localhost:8080/auth/realms/demo/protocol/openid-connect/auth",
  "token_endpoint": "http://localhost:8080/auth/realms/demo/protocol/openid-connect/token",
  "introspection_endpoint": "http://localhost:8080/auth/realms/demo/protocol/openid-connect/token/introspect",
  "userinfo_endpoint": "http://localhost:8080/auth/realms/demo/protocol/openid-connect/userinfo",
  "end_session_endpoint": "http://localhost:8080/auth/realms/demo/protocol/openid-connect/logout",
  // The JWKS Endpoint needed
  "jwks_uri": "http://localhost:8080/auth/realms/demo/protocol/openid-connect/certs",
  "check_session_iframe": "http://localhost:8080/auth/realms/demo/protocol/openid-connect/login-status-iframe.html",
  "grant_types_supported": [
    "authorization_code",
    "implicit",
    "refresh_token",
    "password",
    "client_credentials",
    "urn:ietf:params:oauth:grant-type:device_code",
    "urn:openid:params:grant-type:ciba"
  ]
  // ... Rest
}
```

The response from the JWKS Endpoint describes the JSON Web Keys (JWKs) of the issuer. Let us have a look at the JWKS Endpoint response for a single key as of now and understand its contents:

```jsonc
{
  "keys": [
    {
      "kid": "6rAfUH4i1C-0toyAYuL_gbUf38ZXYK9DId24mUtYupE",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "irXXDkUyoqnQVNMye3gJCYddob2hmf1IE4sPRDW29VDUb0zHaKObFO8HyuCbi7GKfqVnaxw69tTD0A8C5OSYhPvd4Xv-pINbzRVSKL_goDc6vHCs4t7X5WCEZXyOhNbTU0q-tlAJOsQpnTZ0JMhJrV00Voh5UJX48-StquULww8YCui6ZvVcSank1ZZUHLLnB1FpDQuIJJ_n2MbTnV4vWR3_yTDaRi3CEfBERZ9umrvVH7seU01gvF9B1J8oFoKvJY-v2yUnVOzJdm73RdbCAFhi9brYY5LNeRtOkAj_luuug_OU2hT6OfQRSCUIbLvw9NyDyDOZ2GpVJW_oUyus1w",
      "e": "AQAB",
      "x5c": [
        "MIIClzCCAX8CBgF5oXNWUTANBgkqhkiG9w0BAQsFADAPMQ0wCwYDVQQDDARkZW1vMB4XDTIxMDUyNTAyNTQyMVoXDTMxMDUyNTAyNTYwMVowDzENMAsGA1UEAwwEZGVtbzCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAIq11w5FMqKp0FTTMnt4CQmHXaG9oZn9SBOLD0Q1tvVQ1G9Mx2ijmxTvB8rgm4uxin6lZ2scOvbUw9APAuTkmIT73eF7/qSDW80VUii/4KA3OrxwrOLe1+VghGV8joTW01NKvrZQCTrEKZ02dCTISa1dNFaIeVCV+PPkrarlC8MPGAroumb1XEmp5NWWVByy5wdRaQ0LiCSf59jG051eL1kd/8kw2kYtwhHwREWfbpq71R+7HlNNYLxfQdSfKBaCryWPr9slJ1TsyXZu90XWwgBYYvW62GOSzXkbTpAI/5brroPzlNoU+jn0EUglCGy78PTcg8gzmdhqVSVv6FMrrNcCAwEAATANBgkqhkiG9w0BAQsFAAOCAQEAJCjmeOAK/GDlpSVgX80hHhfIKEB1+rQjTDg+rhlCUKLZKrmrKSc2X6FTuYxZWDXw9WO2dyWpdSpI8LibGVwXtwcnl8Wdf2xg00JuTMoO1rAO9FqMOidG4ioS7pte499vBJMjlzO26ApY2o6XrEHE2fOcRLk/6D8J9oXUQi2P3/DMc+BOq6ULxnS136y4wVlqUJM7cree1cn4YUr0Lq0QSZ0wx5JUAupo3XHyXpz0ji8kqI7ekI7O9JPT96LsLnhaSBIYvH7FINC3geLuGtBSnn48xihb/wjyE0Ept1hqFGc9QdaFHEzLvRhPZ7FOKfZU9uNRyY+ptQksIYuQayJa5A=="
      ],
      "x5t": "nLGBNRmlc6Y9jOQ6Xuoo-1BihbY",
      "x5t#S256": "ddjbPoE_yFCvVzdELsYTcbCP9Or1RA1zE-0L3NsnrrU"
    }
  ]
}
```

Each property in the key is defined by the JWK specification [RFC 7517 Section 4][rfc_7517] or, for algorithm-specific properties, in [RFC 7518][rfc-7518].

| Property Name | Description  |
| :------------ | :----------: |
|  `alg`          | The specific cryptographic algorithm used with the key. |
|  `kty`          | The family of cryptographic algorithms used with the key. |
|  `use`          | How the key was meant to be used; `sig` represents the signature. |
|  `x5c`          | The x.509 certificate chain. The first entry in the array is the certificate to use for token verification; the other certificates can be used to verify this first certificate.  |
|  `n`            | The modulus for the RSA public key. |
|  `e`            | The exponent for the RSA public key.  |
|  `kid`          | The unique identifier for the key.  |
|  `x5t`          | The thumbprint of the x.509 cert (SHA-1 thumbprint).  |


To the code.

### The Code

Install the following required npm dependencies after you have init a fresh npm project:
```shell 
npm i --save request-promise request jsonwebtoken

## Get the latest Node Type definitions
## for autocomplete & reduce compilation errors
npm i --save-dev @types/node@latest

npm i --save-dev @types/request-promise @types/jsonwebtoken
```

Next, we declare all the necessary variables and create a function to fetch the JWKS from the JWKS URI. We will also add a function to fetch a fresh token, so that we don't have to deal with expired tokens and other inconveniences it brings. I have also added the Credentials and TokenResponse interfaces for clarity.

```typescript
import { X509Certificate } from 'crypto';
import request from 'request-promise';
import jwt, { GetPublicKeyOrSecret } from 'jsonwebtoken';

interface ICredentials {
  username: string;
  password: string;
  grant_type: string;
  client_id: string;
};

interface ITokenResponse {
  access_token: string;
  expires_in: number;
  refresh_expires_in: number;
  refresh_token: string;
  token_type: string;
  'not-before-policy': number;
  session_state: string;
  scope: string;
}

const JWKUri = 'http://localhost:8080/auth/realms/demo/protocol/openid-connect/certs';
const tokenEndpoint = 'http://localhost:8080/auth/realms/demo/protocol/openid-connect/token';
const credentials: ICredentials = {
  username: 'test', // Use your user credentials here
  password: 'test',
  grant_type: 'password',
  client_id: 'demo-client', // Also change the client_id
};

let publicKey: any;

const getJWKS = (uri: string) => request.get({ uri, json: true });

const getToken = (tokenEndpoint: string, credentials: ICredentials) => request.post({ uri: tokenEndpoint, json: true, form: credentials });

```

The `json: true` parameter in the request options ensures that the response is parsed to JSON by default. Moving on, as per the JWK Specification, the [x5c parameter][x5c_param_documentation] is supposed to be **base64** encoded. Now we declare a self-executing function and start dealing with it.

```typescript
(async () => {
  const jwks = await getJWKS(JWKUri);

  const certBuffer = Buffer.from(jwks.keys[0].x5c[0], 'base64');

  const x509 = new X509Certificate(certBuffer);
})();
```

We can now get the public key from the certificate using the `.publicKey` attribute of the X509Certificate and verify our token.

```typescript
// ... rest of the code ...
  const publicKey = x509.publicKey;

  const tokenResponse: ITokenResponse = await getToken(tokenEndpoint, credentials);
  const token = tokenResponse.access_token;

  const decoded = jwt.verify(token, publicKey as unknown as GetPublicKeyOrSecret, { algorithms: ['RS256'] });

  console.log(decoded);
})();
```

You might notice the type conversion in the `jwt.verify` call. This is because we are sure about the Public Key object and can cast it into the type required by the `verify` function.

**Output**:
```javascript
{
  exp: 1622008367,
  iat: 1622008067,
  jti: '6d7be9d2-3c1e-4c49-8290-b05c24fc192c',
  iss: 'http://localhost:8080/auth/realms/demo',        
  aud: 'account',
  sub: 'f5bad258-ce92-4f08-a765-4a5755c2ed65',
  typ: 'Bearer',
  azp: 'demo-client',
  session_state: '93d288a6-bc7a-4523-8c77-4fcd0aa3ea2f',
  acr: '1',
  realm_access: {
    roles: [ 'offline_access', 'default-roles-demo', 'uma_authorization' ]
  },
  resource_access: { account: { roles: [Array] } },
  scope: 'profile email',
  email_verified: false,
  preferred_username: 'test'
}
```

That's it! We have successfully verified our token.

### Epilogue

The new **`X509Certificate`** class from the `crypto` module simplifies the Public Key generation code a lot. Sadly, it is available only in Node v16.

The code can be further modified:
  - to accept the JWKS URI and Token Endpoint from some configuration or environment variables.
  - to cache the JWKS to reduce the response time of the method. JWKS is not something that changes every minute.
  - to accept the algorithms dyanmically in the `jwt.verify` options. For now, we only looked at the common `RS256` algorithm.

<!-- Links -->
[keycloak_link]: https://keycloak.org
[npm_kc_connect]: https://www.npmjs.com/package/keycloak-connect
[rfc_7517]: https://tools.ietf.org/html/rfc7517#section-4
[rfc_7518]: https://tools.ietf.org/html/rfc7518
[x5c_documentation]: https://nodejs.org/api/crypto.html#crypto_class_x509certificate
[x5c_param_documentation]: https://datatracker.ietf.org/doc/html/rfc7517#section-4.7
