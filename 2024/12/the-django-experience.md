---
author: "Marco Pessotto"
date: 2024-11-26
title: "The Django Experience"
tags:
 - perl
 - web
 - mojolicious
 - catalyst
 - mvc
 - django
 - orm
 - dbic
 - dancer
---

### Django and Mojolicious

Recently I've been working on a project with a
[Vue](https://vuejs.org/) front-end and two back-ends, one in Python
using the [Django](https://www.djangoproject.com/) framework and one
in Perl using the [Mojolicious](https://www.mojolicious.org/)
framework, so it's probably a good time to spend some words to share
the experience and do a quick comparison.

[Previously](/blog/2022/04/perl-web-frameworks/) I wrote a post about
Perl web frameworks, now I'm expanding the subject crossing into
another language.

Django was chosen because it's been around for almost 20 years now and
provides the needed maturity and stability for a project which aims to
be long-running and low-budget. On this regard, so far it proved a
good choice. Recently it saw a major version upgrade without any
problem at all. It could be argued that I should have used the [Django
REST Framework](https://github.com/encode/django-rest-framework)
instead of plain Django. However, at the time the decision was made,
it seemed that adding a framework on the top of another was a bit
excessive. I don't have many regrets about this though.

Mojolicious is an old acquaintance, it used to have a fast-paced
development but seems very mature now, and it's even ported to
[JavaScript](https://mojojs.org/).

Both frameworks have just a few dependencies (which could be something
normal in the Python world, but it's not in the Perl one) and
excellent documentation.

They both follow the Model-Controller-View pattern, so let's examine
the components.

### Views

Both frameworks come with a built-in template system (which can be
swapped out with something else), but in this case we can skip the
topic altogether as both are used only as back-end talking JSON,
without any HTML rendering involved. 

https://docs.djangoproject.com/en/5.1/topics/templates/
https://docs.mojolicious.org/Mojo/Template

However, given that we're writing API, let's see how the rendering looks.

```perl
use Mojo::Base 'Mojolicious::Controller', -signatures;
sub check ($self) {
    $self->render(json => { status => 'OK' });
}
```

```python
from django.http import JsonResponse
def status(request):
    return JsonResponse({ "status":  "OK" })
```

Nothing complicate here, just provide the right call.

### Models



### Controllers

### The languages
