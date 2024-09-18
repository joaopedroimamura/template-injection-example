# Defense

The best way to prevent Server-Side Template Injection is to not allow any users to modify or submit new templates. However, this is sometimes unavoidable due to business requirements.

One of the simplest ways to avoid introducing Server-Side Template Injection vulnerabilities is to always use a "logic-less" template engine, such as Mustache, unless absolutely necessary. Separating the logic from presentation as much as possible can greatly reduce your exposure to the most dangerous template-based attacks.

Another measure is to only execute users' code in a sandboxed environment where potentially dangerous modules and functions have been removed altogether. Unfortunately, sandboxing untrusted code is inherently difficult and prone to bypasses.

Another complementary approach is to accept that arbitrary code execution is all but inevitable and apply your own sandboxing by deploying your template environmnt in a locked-down Docker container, for example.

It's also possible to ensure that user input is properly sanitized and validated before being inserted into templates. Implementing input validation and using context-aware escaping techniques can help mitigate the risk of this vulnerability.