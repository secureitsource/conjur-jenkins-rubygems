# Orchestrating Trust in your DevOps Pipeline

## Questions and Answers from Live Webinar

### Can we use CyberArk-Conjur to Manage SSL Certificates across an organization?

Yes. Managing certificates is something that Conjur excels at. It's not yet a full blown CA, but certificates can be stored, retrieved, updated and rolled out to machines and groups of machines very easily.

### Can the variable be SSL cert instead of a password?

Yes. The 'password' or 'secret' is a variable in Conjur... it could be a cert, API key, or any other piece of sensitive data.

### What is the big ampersand theory?

The ampersand is an anchor tag... means that this node might be repeated. The policy syntax follows standard YAML syntax.

### Can you show us a Jenkins file using these concepts?

Going through it now. If this isn't enough you can check out the sample code in the GitHub repo. If that doesn't cover it, let us know.

### Is it possible to fetch data from external sources like AD, LDAP for secret validations?

Yes.  Conjur Enterprise Edition has the ability to synchronize with LDAP (this includes Microsoft Active Directory).

### What does 'RBAC' mean in point 2a on page 14?

Role Based Access Control -- it's a model for governing access. More maintainable than something like ABAC -- Attribute Based Access Control or PBAC -- Path Based Access Control.

### In addition to the screenshot of the policy's role graph that can be given to auditors, can we provide an evidence that demonstrates that the policy is actually working as expected?

The audit events that are tracked perform `pass`/`fail` checks against that policy that are reported in those logs.  So you’ll be able to make sure a policy is effective by monitoring those results.

Remember, policy as code should be tested in Development just as you'd test your applications in Development before promoting the code.

### Is CyberArk FIPS compliant? If yes can we get some document? 

CyberArk has some products that are FIPS compliant, but Conjur is not. It is on our roadmap and something we are working towards.

The [Enterprise Password Vault](http://lp.cyberark.com/rs/316-CZP-275/images/sb-CyberArk%20Digital%20Vault%20-%20Built%20for%20Security%209.19.2016-web.pdf) that holds the audit data currently and will hold the secrets in the near future is and documentation can be provided for it.

### When will the Conjur database integrate with the CyberArk EPV?  Currently today, Cyberark EPV is separate/different from the Conjur database/repository.

We are currently building this integration. We are looking to release it before the end of the year.

### We are using CloudBees Jenkins (Ent) and are very much interested in CyberArk if we can a get Secret management solution.

More information on CyberArk Conjur Enterprise can be found here: https://www.cyberark.com/products/privileged-account-security-solution/cyberark-conjur/ or you can email info@cyberark.com and request a demo or meeting.

Also, there's a great Reddit community that can provide help and share insights, too, at [/r/CyberArk](https://reddit.com/r/CyberArk).
