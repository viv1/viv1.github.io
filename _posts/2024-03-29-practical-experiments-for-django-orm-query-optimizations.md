---
layout: post
title: Practical Experiments for Optimizing Django query with the power of SQL joins
date: 2024-03-29 23:13
category: Performance
tags: ["django", "performance", "optimization", "sql", "practical", "hands-on", "prefetch_related", "select_related", "join", "python", "n+1", "n+1 queries", "orm"]
summary: Here we experiment with a couple of query optimization techniques for Django.
---
<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->
- [Introduction](#introduction)
- [Concepts](#concepts)
- [Practical](#practical)
   * [Project Setup](#project-setup)
   * [Experiment with a small number of records](#experiment-with-a-small-number-of-records)
   * [Experiment with a larger number of records](#experiment-with-a-larger-number-of-records)

<!-- TOC end -->
<!-- TOC --><a href="#" name="introduction"></a>
## Introduction

`Python` is a remarkable programming language - it's simple, powerful, and abstracts away several complicated internal concepts, which makes it easy to code with. `Django` framework extends this simple powerful nature of python to the web.

However, the very abstraction and simplicity that make `Python` and `Django` so appealing can sometimes lead developers, especially those without formal computer science training, to overlook (or maybe just forget about) the intricacies of how things work under the hood. 

One concept that I have often seen several developers overlook in their professional lives is the concept of `joins` in database queries. This might not be apparent on a small scale, but when handling vast amounts of data, the programs can become unweildy.

We will walk through how a simple optimization concepts can make a huge difference as the scale increases.

<!-- TOC --><a href="#" name="concepts"></a>
## Concepts

`Django's ORM` is tightly coupled with the `SQL` (relational) tables. Each Model defined in `Django` has a correponding table defined in `SQL`. When we query in `Django Model`, we are actually querying the corresponnding `SQL` table. So, it is obvious that any optimization techniques used in SQL would also be beneficial when querying `Django models`.

Another important point to note is that making a `SQL` call (i.e. a Database call) is expensive. So, we want to minimize the number of calls to the Database.

We assume you already know what SQL joins are. Informally, `SQL joins` allows you to query 2 or more related tables faster. To read more, you can checkout [SQL JOIN](https://en.wikipedia.org/wiki/Join_(SQL){:target="_blank"}) .

As we would expect, `Django` has corresponding methods to optimize queries:

**`select_related`**
- Performs a `SQL JOIN` to fetch related data in the same query.
- Suitable for ***one-to-one*** and ***one-to-many*** relationships.
- More efficient than separate queries when you know you'll need the related data.

**`prefetch_related`**:
- Uses *separate queries* to fetch related data, but *batches the queries to minimize database hits*.
- Suitable for ***many-to-many*** and ***many-to-one*** relationships.
- More efficient than separate queries when you'll need the related data.

In summary, `select_related` is optimized for `one-to-one` and `one-to-many` relationships, while `prefetch_related` is optimized for `many-to-many` and `many-to-one` relationships. # The choice depends on the type of relationship and the likelihood of needing the related data.

<!-- TOC --><a href="#" name="practical"></a>
## Practical

<!-- TOC --><a href="#" name="project-setup"></a>
### Project Setup

Let's set up a very simple Django project so that we can run our experiments. 

1.Assuming you have `python3` installed, run the following to start `virtualenv` and install `Django` :

```bash
python3 -m venv venv
source venv/bin/activate
pip install Django
pip install Faker
```

> If you don't want to go through the setup, you can clone the
[repository](https://github.com/viv1/blog-code-examples){:target="_blank"} and go to the example directory: `cd django_join` after the above step. And then continue reading from [Experiment 1](#experiment-with-a-small-number-of-records)
{: .prompt-tip }

2.Run the following to start a new project named `django-join` and create an app/module inside it called `myapp` :

```bash
django-admin startproject django_join
django-admin startapp myapp
```

3.Now, enable the `myapp` by including it under `INSTALLED_APPS` in `settings.py` :

```bash
vim django_join/settings.py
```

Here:
```python

...
INSTALLED_APPS = [
    'myapp', # ADD YOUR APP HERE

    # EXISTING MODELS
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
...

```

4.Create New Django models in `myapp/models.py`:

```python
from django.db import models
from django.core.validators import MaxValueValidator

# Create your models here.
class Author(models.Model):
    name = models.CharField(max_length=100)
    age = models.IntegerField(validators=[MaxValueValidator(100)])

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
```


5.Run migration commands to create the model tables. 

```bash
python manage.py makemigrations myapp
python manage.py migrate
```

6.Let's also create a command to populate our tables with random data. 

- Install `Faker` library:
```bash
pip install Faker
```

- Create the file in this location (Create subfolders as needed):

```bash
vim myapp/management/commands/populate_data.py
```
- Paste the following. This basically generates random data given author count and max number of books for an author.

```python
from django.core.management.base import BaseCommand
from faker import Faker
from myapp.models import Author, Book
import random

class Command(BaseCommand):
    help = 'Populates the database with random authors and books'

    def add_arguments(self, parser):
        parser.add_argument('authors', type=int, help='The number of authors to create')
        parser.add_argument('books', type=int, help='The number of books to create per author')

    def handle(self, *args, **kwargs):
        faker = Faker()

        authors_count = kwargs['authors']
        max_books_per_author = kwargs['books']

        for _ in range(authors_count):
            author = Author.objects.create(name=faker.name())
            # Generate random number of books upto max_count
            book_count_for_this_author = random.randint(1, books_per_author)
            for _ in range(book_count_for_this_author):
                Book.objects.create(title=faker.sentence(), author=author)

        self.stdout.write(self.style.SUCCESS(f'Successfully added {authors_count} authors and {authors_count * books_per_author} books'))
```

7.Create another file to run the 1st experiment. This experiment will check the difference in number of queries. 

```bash
vim myapp/management/commands/get_db_query_count.py
```
Paste the following content:
```python
from django.db import connection, reset_queries
from myapp.models import Author, Book

# Measuring for select_related
reset_queries()
books = Book.objects.select_related('author').all()
for book in books:
    temp = book.author.name
print(f"Queries with select_related: {len(connection.queries)}")

reset_queries()
books = Book.objects.all()
for book in books:
    temp = book.author.name
print(f"Queries without select_related: {len(connection.queries)}")

# Measuring for prefetch_related
reset_queries()
authors = Author.objects.prefetch_related('book_set').all()
for author in authors:
    temp = author.book_set.count()
print(f"Queries with prefetch_related: {len(connection.queries)}")

reset_queries()
authors = Author.objects.all()
for author in authors:
    temp = author.book_set.count()
print(f"Queries without prefetch_related: {len(connection.queries)}")

```

8.Create another file to run the 2nd experiment. This experiment will measure the time difference while running the commands.

```bash
vim myapp/management/commands/get_time_difference.py
```
Paste the following content:


```python
from django.core.management.base import BaseCommand
from django.db import connection, reset_queries
from django.db.models import Count
from myapp.models import Author, Book

import timeit

class Command(BaseCommand):
    help = 'Measure average time taken to run commands. It gives average run time of 10 commands.'

    def handle(self, *args, **kwargs):
        # With prefetch_related
        execution_time1 = timeit.timeit('get_total_book_count_with_prefetch_related()', globals=globals(), number=10) / 10
        print(f"Avg time with prefetch_related: {execution_time1}")

        # Without prefetch_related
        execution_time2 = timeit.timeit('get_total_book_count_without_prefetch_related()', globals=globals(), number=10) / 10
        print(f"Avg time without prefetch_related: {execution_time2}")

        print("prefetch_related performance improvement RATIO:", execution_time2 / execution_time1)

        print()
        # With select_related
        execution_time3 = timeit.timeit('get_total_book_count_with_select_related()', globals=globals(), number=10) / 10
        print(f"Avg time with select_related: {execution_time3}")

        # Without select_related
        execution_time4 = timeit.timeit('get_total_book_count_without_select_related()', globals=globals(), number=10) / 10
        print(f"Avg time without select_related: {execution_time4}")

        print("select_related performance improvement RATIO:", execution_time4 / execution_time3)



# Without prefetch_related
def get_total_book_count_without_prefetch_related():
    authors = Author.objects.all()
    count = 0
    for author in authors:
        count += author.book_set.count()
    # print(count)

# With prefetch_related
def get_total_book_count_with_prefetch_related():
    authors = Author.objects.prefetch_related('book_set').all()
    count = 0
    for author in authors:
        count += author.book_set.count()
    # print(count)

# Without select_related
def get_total_book_count_without_select_related():
    books = Book.objects.all()
    total_age = 0 
    for book in books:
        total_age += book.author.age
    # print(total_age // books.count())

# With select_related
def get_total_book_count_with_select_related():
    books = Book.objects.select_related('author').all()
    total_age = 0
    for book in books:
        total_age += book.author.age
    # print(total_age // books.count())

```

<!-- TOC --><a href="#" name="experiment-with-a-small-number-of-records"></a>
### Experiment with a small number of records

We will first populate a small dataset, and then run the commands to get db query count and time.

```bash
(venv) viv1@Viveks-MacBook-Pro django_join % python manage.py populate_data 5 5
Successfully added 5 authors and 18 books

(venv) viv1@Viveks-MacBook-Pro django_join % python manage.py get_db_query_count
Queries with prefetch_related: 2
Queries without prefetch_related: 6
Additional Queries: 4

Queries with select_related: 1
Queries without select_related: 19
Additional Queries: 18
(venv) viv1@Viveks-MacBook-Pro django_join % python manage.py get_time_difference
Avg time with prefetch_related: 0.0008449875000000023
Avg time without prefetch_related: 0.0017883291999999996
prefetch_related performance improvement RATIO: 2.1163972248110117

Avg time with select_related: 0.00023677920000000075
Avg time without select_related: 0.0030041750000000004
select_related performance improvement RATIO: 12.68766428807932

```

Even with the small amount of records, we can see the performance improvement in terms of time and the number of db calls. In fact, what we are observing in the query part is the ***famous N+1 query problem*** , where there is N number of DB calls for each of the N records. With this optimization, we can reduce it to a single DB call.


<!-- TOC --><a href="#" name="experiment-with-a-larger-number-of-records"></a>
### Experiment with a larger number of records

```bash
(venv) viv1@Viveks-MacBook-Pro django_join % python manage.py populate_data 100 10
Successfully added 100 authors and 549 books
(venv) viv1@Viveks-MacBook-Pro django_join % python manage.py get_db_query_count
Queries with prefetch_related: 2
Queries without prefetch_related: 106
Additional Queries: 104

Queries with select_related: 1
Queries without select_related: 568
Additional Queries: 567
(venv) viv1@Viveks-MacBook-Pro django_join % python manage.py get_time_difference
Avg time with prefetch_related: 0.0058029208
Avg time without prefetch_related: 0.029672083399999993
prefetch_related performance improvement RATIO: 5.113301460188806

Avg time with select_related: 0.003976683299999994
Avg time without select_related: 0.099798425
select_related performance improvement RATIO: 25.09589461147186
```

With just a bit higher number of records, we can see the performance improvement is even higher.

## Conclusion

Optimizing database queries is crucial for achieving high performance and scalability in Django applications. Being aware of very simple concepts like joins can make our application faster, more responsive and efficient, especially when dealing with large datasets and complex relationships between models.

## References

[prefetch_related](https://docs.djangoproject.com/en/5.0/ref/models/querysets/#django.db.models.query.QuerySet.prefetch_related){:target="_blank"}

[select_related](https://docs.djangoproject.com/en/5.0/ref/models/querysets/#django.db.models.query.QuerySet.select_related){:target="_blank"}
