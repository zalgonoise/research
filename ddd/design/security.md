[back](actions_and_operations.md) | [index](index.md)


### Security

Initially, the users will login using a dedicated endpoint, using a username + password combination. The call returns a JWT if their credentials are valid, and the caller can use the JWT as an Authorization HTTP header calling users and secrets endpoints. JWT expire within one hour of being created.

Creating a user does not require authentication.

When a user is created, the service will create an encryption private key in the Bolt database that will be used to encrypt the user's secrets' values. Also, when created, a random 128-byte-long salt value is generated and appended to the user's password; and the resulting value is hashed. The SQLite database will store this salt and hash values, base64-encoded, and not the user's plaintext password.

A user can only access their own resources. A secret shared by user A with user B is perceived as a (new) secret owned by user B in a read operation; which will contain an expiry. However, the actual owner is in control of this secret -- as shared secrets are read-write only for the actual owner. Shared secrets have a different key in the format of `username:secretkey`.

While there is a reserved username (`root`) despite not having a roles / privileges model; the Bolt database will still store user secrets in buckets identified by the user's (internal) ID, and not their username. User secrets are encrypted with AES, using a 32-byte key generated and stored when the user is first created. Like passwords, secrets will not be stored in plaintext (beyond the database's own encryption).

[next](product_format.md)