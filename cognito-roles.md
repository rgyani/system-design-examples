# Extend Cognito to validate user-permissions on resources

Say you have a cognito user pool and some services, and you want to allow read/write permissons on these services  
eg.


User 1 can read+write Service B, but can only read from Service A
User 2 can read+write Service C, but can only read from Service B and only write to Service D

|User | Service A | Service B | Service C | Service D |
|---|---|---|---|--|
| User 1 | R | R/W | N   | N |
| User 2 | N |   R | R/W | W |


Lets design a system where the administrator can assign permissions to these users


## Using Cognito User Pool Attributes

Attributes are pieces of information that help you identify individual users, such as name, email address, and phone number. Cognito user pool has a set of default standard attributes like address, email, name etc

However, You can also add up to 50 custom attributes to your user pool. You can specify a minimum and/or maximum length for custom attributes. However, the maximum length for any custom attribute can be no more than 2048 characters.

### Assigning Service permissions using Attributes

In the user-pool, we can create a custom attribute called service_permissions, and from the Administration screen, we can assign permissions to the services to the individual users e.g.

User 1: service_permissions would be "service_a_read, service_b_read_write"  
User 2: service_permissions would be "service_b_read, service_c_read_write"

However to get around problems like 2048 character limit, instead we can use a bit mask like

User 1: service_permissions = RANN  
User 2: service_permissions = NRAW

where A = R+W    
and N = no permission

and **ALSO** we maintain a global enum of services which we can only append to   
so service A, will always be at index 0 in the string  
   service B, will always be at index 1 in the string  
and so on

### how will admin assign permissions

Say, from the admin page, the Administrator wants to create a new User 3, who has only read permission to Service D,   
we can issue a request like so

```javascript
import { Auth } from 'aws-amplify';

 // This should ideally be via a backend, I am just illustrating the point
Auth.signUp({
  username,
  password,
  attributes: {
    email: 'user3@example.com',              
    'custom:service_permissions': 'NNNR'   # the permission matrix
  }
})
.then(data => console.log(data))
.catch(err => console.error(err));
```

### how will user authentication work

Say the user signs in using
```javascript
import { Auth } from 'aws-amplify';

Auth.signIn(username, password)
  .then(user => {
    console.log(user);
  })
  .catch(err => {
    console.error(err);
  });
```

The user object returned on successful authentication typically looks something like this:
```json
{
  "username": "user1",
  "pool": {
    "userPoolId": "us-east-1_XXXXXXX",
    "clientId": "XXXXXXXXXXXXXXXXXXXX",
    "client": {}
  },
  "Session": null,
  "authenticationFlowType": "USER_SRP_AUTH",
  "client": {},
  "signInUserSession": {
    "idToken": {
      "jwtToken": "eyJraWQiOiJ...FzQ",
      "payload": {
        "sub": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
        "aud": "XXXXXXXXXXXXXXXXXXXX",
        "email": "user@example.com",
        "email_verified": true,
        "auth_time": 1609459200,
        "iss": "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXX",
        "cognito:username": "user1",
        "exp": 1609462800,
        "iat": 1609459200,
        "jti": "e07270b1-1234-5678-9012-34567890abcd",
        "custom:service_permissions": "NNNR"
      }
    },
    "refreshToken": {
      "token": "eyJjdHkiOiJKV1QiLCJlbmMiOiJBMTI5N...WJoaEdFdnZ3cFNRZ3g"
    },
    "accessToken": {
      "jwtToken": "eyJraWQiOiJ...FzQ",
      "payload": {
        "sub": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
        "event_id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
        "token_use": "access",
        "scope": "aws.cognito.signin.user.admin",
        "auth_time": 1609459200,
        "iss": "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXX",
        "exp": 1609462800,
        "iat": 1609459200,
        "jti": "12345678-1234-5678-9012-34567890abcd",
        "client_id": "XXXXXXXXXXXXXXXXXXXX",
        "username": "user1"
      }
    },
    "clockDrift": 0
  },
  "attributes": {
    "sub": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "email_verified": true,
    "email": "user@example.com",
    "custom:service_permissions": "NNNR"
  },
  "preferredMFA": "NOMFA"
}
```

**Note the field "custom:service_permissions": "NNNR"**

This can be used by the frontend to enable disable links for services in the UI

### how will services validate this

say you have api gateway endpoints for the services A,B,C and so on  
in the lambda implementations we can now validate the service_permissions like so

```javascript
const jwt = require('jsonwebtoken');

// say this lambda is the Write-handler for Service A
const service_index = 1
const required_permissions = ["W", "A"]

exports.handler = async (event) => {
    const token = event.headers.Authorization;
    const decoded = jwt.decode(token, { complete: true });

    // Extract custom:service_permissions from the decoded token payload
    const permissions = decoded.payload['custom:service_permissions'];

    if (required_permissions.indexOf(permissions[service_index]) == -1) {
        return {
            statusCode: 403,
            body: JSON.stringify({ message: 'Forbidden: invalid service permission' }),
        };
    }

    // Proceed with normal processing
    return {
        statusCode: 200,
        body: JSON.stringify({ message: 'Access granted' }),
    };
};
```

Obviously you will setup cognito authentication with API gateway to validate the token   
When you configure API Gateway to use a Cognito User Pool Authorizer, API Gateway automatically:
1. **Receives the JWT Token**: It expects the JWT token to be sent in the Authorization header of the request.
2. **Decodes the Token**: API Gateway decodes the JWT token.
3. **Validates the Token**:
    * It verifies the token’s signature using the public keys provided by Cognito.
    * It checks the token’s expiration.
    * It ensures the token has not been tampered with.
    * It validates the claims within the token against the configuration of the authorizer.


### My request still reached all the way to the lambda, can I not stop this lambda invocation
While you correct, I dont see any hole in the design in terms of security, the other ways to avoid this is even more complicated, lets discuss it just to round up

We can always create a custom lambda authorizer that validates the JWT token and checks the specific custom claims, like so

```javascript
const jwt = require('jsonwebtoken');
const axios = require('axios');
const jwkToPem = require('jwk-to-pem');

const cognitoPoolId = 'us-east-1_XXXXXXX';
const region = 'us-east-1';
const jwksUrl = `https://cognito-idp.${region}.amazonaws.com/${cognitoPoolId}/.well-known/jwks.json`;

// say this lambda is the Custom Athorizer for Service A
const service_index = 1
const required_permissions = ["W", "A"]

let cachedKeys;

const getPublicKeys = async () => {
  if (!cachedKeys) {
    const response = await axios.get(jwksUrl);
    cachedKeys = response.data.keys.reduce((agg, key) => {
      const pem = jwkToPem(key);
      agg[key.kid] = pem;
      return agg;
    }, {});
  }
  return cachedKeys;
};

const verifyToken = async (token) => {
  const decoded = jwt.decode(token, { complete: true });
  if (!decoded) {
    throw new Error('Invalid token');
  }

  const keys = await getPublicKeys();
  const pem = keys[decoded.header.kid];
  if (!pem) {
    throw new Error('Invalid token');
  }

  return jwt.verify(token, pem);
};

exports.handler = async (event) => {
  try {
    const token = event.authorizationToken.split(' ')[1];
    const verified = await verifyToken(token);

   // Extract custom:service_permissions from the decoded token payload
    const permissions = verified.payload['custom:service_permissions'];

    if (required_permissions.indexOf(permissions[service_index]) == -1) {

    if (required_permissions.indexOf(permissions[service_index]) == -1) {
     return {
        principalId: null,
        policyDocument: {
          Version: '2012-10-17',
          Statement: [
            {
              Action: 'execute-api:Invoke',
              Effect: 'Deny',
              Resource: event.methodArn,
            },
          ],
        },
      };
    }

    return {
      principalId: verified.sub,
      policyDocument: {
        Version: '2012-10-17',
        Statement: [
          {
            Action: 'execute-api:Invoke',
            Effect: 'Allow',
            Resource: event.methodArn,
          },
        ],
      },
      context: {
        favoriteColor: favoriteColor,
      },
    };
  } catch (err) {
    return {
      principalId: null,
      policyDocument: {
        Version: '2012-10-17',
        Statement: [
          {
            Action: 'execute-api:Invoke',
            Effect: 'Deny',
            Resource: event.methodArn,
          },
        ],
      },
    };
  }
};
```


You see the problem here, you lambda needs to interpret the variables
```
// say this lambda is the Custom Athorizer for Service A
const service_index = 1
const required_permissions = ["W", "A"]
```
This can be theoritically be done via api gateway mappings, but I haven't done a POC on this yet

### Using WAF instead to Enforce Custom Claims

While not useful in this case, theoritically we can use a WAF also to do some validations for us

1. Open the AWS WAF console.
2. Create a new web ACL.
4. Add a rule to inspect the JWT token.
5. Use regex or JSON match conditions to check for specific claims within the JWT token.

For example, to enforce that custom:service_permission equals A:
```json

  "FieldToMatch": {
    "SingleHeader": {
      "Name": "Authorization"
    }
  },
  "TextTransformations": [
    {
      "Priority": 0,
      "Type": "NONE"
    }
  ],
  "JsonMatchPattern": {
    "MatchScope": "ALL",
    "JsonMatchType": "VALUE",
    "ValueMatch": {
      "ExactMatch": "A"
    },
    "FieldPath": "$.custom:service_permission"
  }
}
```
6. Attach the web ACL to your API Gateway stage.

**Benefits**
* **Performance**: Using API Gateway authorizers and AWS WAF can reduce the overhead and latency compared to using Lambda for token validation.
* **Security**: Ensures that only requests with valid tokens and specific claims reach your backend services.

**Considerations**
* **Complex Logic**: For very complex validation logic, Lambda might still be necessary.
* **Cost**: Can be more expensive, especially if you have a high volume of requests or complex rule sets. AWS WAF pricing includes charges for the number of web ACLs, rules, and requests processed
