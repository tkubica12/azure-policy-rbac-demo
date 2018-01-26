# Intro to Role based access control and policy demo
In traditional IT operations responsibilites are split in order to provide right level of expertize and operational policies are being defined so our security and business continuity goals are achieved. There are two areas that need to be discussed.

First we need to make sure that complex and risky tasks are handled by experts and hold them accountable. At the same time we need to make sure this does not slow down processes too much and others can leverage predefined resources in self-service way. As an example we might want networking and security team to provide connectivity and network security while systems administrators and developers are not able to bypass this (because that puts business under threat), but let them select from predefined "managed" options immediately. In Azure this can be achieved by Role Based Acess Control.

Second because IT is complex human error is inevitable we might want to define set of common policies that would prevent some unwanted situations. For example we might enforce categorization of deployed resources so it is always clear whether it is dev, test or production, what department is paying for that, what data category (like internal, confidential, personal, ...) are used. For certain systems we might want to prevent applications to be deployed in improper network or prevent running resources with no firewall protection. This is where Azure Policy comes.

Azure RBAC is handling what resources and resource groups **users** get access to. Azure Policy is to define rules for **resource groups and subscriptions** that all users have to follow.

- [Role Based Access Control](docs/rbac.md)
- [Azure Policy](docs/policy.md)

# Author
Tomas Kubica, linkedin.com/in/tkubica, Twittter: @tkubica

Blog in Czech: https://tomaskubica.cz

Looking forward for your feedback and suggestions!