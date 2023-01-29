[back](sqlite.md) | [index](index.md)

### Service preparation

The service layer will have access to the repositories and is the part of the application that will handle the business logic, the flow of the calls, and the validation for the requests.

Besides users and secrets and shares, the service will also handle the authorization and authentication features, which haven't yet been discussed.

For the moment, this is the current state of the service structure (knowing that there isn't yet a service here, but the modules that it will have access to).

```
─ service
   ├─ user.Repository
   ├─ secret.Repository
   ├─ shared.Repository
   └─ keys.Repository
```

To handle authentication and authorization, two specific packages will be added:

- `crypt`: handles general cryptography-related actions for this application
- `authz`: handles authorization, issuing and parsing JWT

Let's prepare those first:

[next](cryptography.md)