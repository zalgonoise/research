[back](wrapping_up.md) | [index](index.md)


## Final thoughts

It was very fun writing this document but it's important to point out that this is not a guide, a template or advice on how to design apps in Go. This serves to illustrate how I design and build web apps after 3 years of Go. I've learned a lot (especially in this past year) and there is no way I can transfer all of that knowledge in one document, in one application, no matter how big it is.

With this in mind, the app is ready on a minimum viable product (MVP) release standpoint. From here, there are a ton of new features that could be embed, be it to enhance security (SAML support for online accounts like Google and GitHub), better logging and auditing, actual observability (and not just writing log entries and traces to a file), fine-tuning the databases, workers to automatically clean up expired data, refactoring JWT storage, so many ideas just while writing this paragraph!

However, that does it for this document. I am happy if it helps you in any way, and any corrections / observations are more than welcome.

Later, the `zalgonoise/x/secr` app was published as `zalgonoise/cloaki`.

[index](index.md)