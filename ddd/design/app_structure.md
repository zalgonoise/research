[back](design.md) | [index](index.md)

### App structure

The entities in this model are simple, there are users who create secrets, and these secrets can be accessed by either the owner or other users if the secret is shared. The diagram below showcases this model:

![Diagram](../media/Secr_Diagram.jpg)

Of course, the app will have a few other services to support authentication, authorization and encryption, for example. But these services are not the core-purpose of the app.

[next](error_handling.md)