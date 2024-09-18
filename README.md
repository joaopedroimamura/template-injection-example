# Server-Side Template Injection

This application has a template injection vulnerability. It's possible to run code in the input field by formating it with special characters. In this case, Python uses {{}}.

## What is Server-Side Template Injection?

Server-Side Template Injection is when an attacker is able to use native template syntax to inject a malicious payload into a template, which is then executed server-side.

Template engines are designed to generate web pages by combining fixed templates with volatile data. This type of attack can occur when user input is concatenated directly into a template, rather than passed in as data. This allows attackers to inject arbitrary template directives in order to manipulate the tempalte engine, often enabling them to take complete control of the server. As the name suggests, payloads are delivered and evaluated server-side, potentially making them much more dangerous than a typical client-side template-injection.

## Impacts

Server-Side Template Injection vulnerabilities can expose websites to a variety of attacks depending on the template engine in question and how exactly the application uses it. In certain rare circumstances, these vulnerabilities pose no real security risk. However, most of the time, the impact of server-side template injection can be catastrophic.

At the severe end of the scale, an attacker can potentially achieve remote code execution, taking full control of the back-end server and using it to perform other attacks on internal infrastructure.

Even in cases where full remote code execution is not possible, an attacker can often still use it as the basis for numerous other attacks, potentially gaining read access to sensitive data and arbitrary files on the server.

## How it happens

This type of vulnerability arises when user input is concatenated into templates rather than being passed in as data.

Static templates that simply provide placeholders into which dynamic content is rendered are generally not vulnerable to Server-Side Template Injection. The classic example is an email that greets each user by their name, such as the following extract from a Twig template:

```
$output = $twig->render(
    "Dear {first_name},", 
    array("first_name" => $user.first_name)
);
```

This is not vulnerable because the user's first name is merely passed into the template as data.

However, as templates are simply strings, web developers sometimes directly concatenate user input into templates prior to rendering. This time, users are able to customize parts of the email before it is sent.

```
$output = $twig->render("Dear " . $_GET['name']);
```

Instead of a static value being passed into the template, part of the template itself is being dynamically generated using the ```GET``` parameter ```name```. As template syntax is evaluated server-side, this potentially allows an attacker to place a Server-Side Template Injection payload inside the ```name``` parameter:

```http://vulnerable-website.com/?name={{bad-stuff-here}}```

Sometimes, vulnerabilities like this are caused by accident due to poor template design by people unfamiliar with the security implications. However, it's also possible to implement this behavior intentionally. For example, some websites deliberately allow certain privileged users, such as content editors, to edit or submit custom templates by design. This clearly poses a huge security risk if an attacker is able to compromise an account with such privileges.

## Constructing

To craft a successful attack, it would be necessary to detect, identify and exploit it.

### Detect

This type of vulnerability often go unnoticed not because they are complex but because they are only really apparent to auditors who are explicitly looking for them. If you are able to detect that a vulnerability is present, it can be surprisingly easy to exploit it. 

As with any vulnerability, the first step towards exploitation is being able to find it. Perhaps the simplest initial approach is to try fuzzing the template by injecting a sequence of special characters commonly used in template expression, such as ```${{<%[%'"}}%\```. If an exception is raised, this indicates that the injected template syntax is potentially being interpreted by the server in some way. This one sign that a vulnerability may exist.

#### Plaintext content

Most template languages allow you to freely input content either by using HTML tags directly or by using the template's native syntax, which will be rendered to HTML on the back-end before the HTTP response is sent. For example, in Freemarker, the line ```render('Hello ' + username)``` would render to something like ```Hello Carlos```

This can sometimes be exploited for XSS and is in fact often mistaken for a simple XSS vulnerability. However, by setting mathematical operations as the value of the parameter, we can test whether this is also a potential entry point for a Server-Side Template Injection attack.

For example, consider a template that contains the following vulnerable code:

```render('Hello ' + username)```

During auditing, we might test for server-side template injection by requesting a URL such as:

```http://vulnerable-website.com/?username=${7*7}```

If the resulting output contains ```Hello 49```, this shows that the mathematical operation is being evaluated server-side. This is a good proof of concept for a Server-Side Template Injection vulnerability.

#### Code context

In other cases, the vulnerability is exposed by user input being placed within a template expression, as we saw earlier with our email example. This may take the form of a user-controllable variable name being placed inside a parameter, such as:

```
greeting = getQueryParameter('greeting');
engine.render("Hello {{"+greeting+"}}", data);
```

On the websie, the resulting URL would be something like:

```http://vulnerable-website.com/?greeting=data.username```

This would be rendered in the output to ```Hello Carlos```. 

This context is easily missed during assessment because it doesn't result in obvious XSS and is almost indistinguishable from a simple hashmap lookup. One method of testing for Server-Side Template Injection in this context is to first establish that the parameter doesn't contain a direct XSS vulnerability by injecting arbitrary HTML into the value:

```http://vulnerable-website.com/?greeting=data.username<tag>```

In the absence of XSS, this will usually either result in a blank entry in the output (just ```Hello``` with no username), encoded tags, or an error message. The next step is to try and break out of the statement using common templating syntax and attempt to inject arbitrary HTML after it:

```http://vulnerable-website.com/?greeting=data.username}}<tag>```

If this again results in an error or blank output, you have either used syntax from the wrong templating language or, if no template-style syntax appears to be valid, Server-Side Template Injection is not possible. Alternatively, if the output is rendered correctly, along with the arbitrary HTML, this is a key indication that a Server-Side Templat Injection vulnerability is present:

```Hello Carlos<tag>```

### Identify

There are a huge number of templating languages, but many of them use very similar syntax that is specifically chosen not to clash with HTML characters. As a result, it can be relatively simple to create probing payloads to test which template engine is being used.

Simply submitting invalid syntax is often enough because the resulting error mesage will tell you exactly whhich template engine is, and sometimes even which version. For example, the invalid expression ```<%=foobar%>``` triggers the following response from the Ruby-based ERB engine:

```
(erb):1:in `<main>': undefined local variable or method `foobar' for main:Object (NameError)
from /usr/lib/ruby/2.5.0/erb.rb:876:in `eval'
from /usr/lib/ruby/2.5.0/erb.rb:876:in `result'
from -e:4:in `<main>'
```

It's common to use a process of elimination based on which syntax appears to be valid or invalid. A common way of doing this is to inject arbitrary mathematical operations using syntax from different template engines. 

The same payload can sometimes return a successful response in more than one template language. For example, {{7*'7'}} returns ```49``` in Twig and ```7777777``` in Jinja2.

- ${7*7}
  - TRUE: a{\*comment\*}b
    - TRUE: Smarty 
    - FALSE: ${"z".join("ab")}
      - TRUE: Mako
      - FALSE: Unknown
  - FALSE: {{7*7}}
    - TRUE: {{7*'7}}
      - TRUE: Jinja2
      - TRUE: Twig
      - FALSE: Unknown
    - FALSE: Not vulnerable

### Exploit

After detecting that a potential vulnerability exists and successfully identifying the template engine, it's time to begin trying to find ways of exploiting it.

For example, in Python we can use Gadgets to bypass imports or just ```exec(compile())```


