---
layout: post
title:  "Django 문서의 'Making quries' 정리"
categories: jekyll update
---

## Creating objects

모델 클래스는 데이터베이스 테이블을 나타낸다
클래스의 인스턴스는 데이터베이스 테이블의 특정 레코드를 나타낸다

객체를 생성하려면 키워드 인자를 사용하여 모델 클래스에 인스턴스를 생성 후, save ()를 호출하여 객체를 데이터베이스에 저장한다

```python
>>> from blog.models import Blog
>>> b = Blog(name='Beatles Blog', tagline='All the latest Beatles news.')
>>> b.save()
```

---

## Saving changes to objects

`save()` 사용해서 이미 데이터베이스에 있는 객체의 변경사항을 저장할 수 있다
django는 save()를 호출 할 때 까지 데이터베이스에 접근하지 않는다

### Saving ForeignKey and ManyToManyField fields

**<ForeignKey>**

**ForeignKey** 필드를 업데이트하는 것은 보통의 필드를 저장하는 것과 똑같은 방식으로 작동한다. 필드에 올바른 유형의 객체를 할당하기만 하면 된다.

```python
>>> from blog.models import Entry
>>> entry = Entry.objects.get(pk=1)
>>> cheese_blog = Blog.objects.get(name="Cheddar Talk")
>>> entry.blog = cheese_blog
>>> entry.save()
```

**<ManyToManyField>**

**ManyToManyField** 를 업데이트 할 때는 조금 다르다. 필드에 **add()** 메소드를 사용하여 레코드를 추가한다.

```python
>>> from blog.models import Author
>>> joe = Author.objects.create(name="Joe")
>>> entry.authors.add(joe)
```

한번에 여러가지 레코드를 추가하려면 **add()** 에 여러가지 인자를 넣으면 된다

```python
>>> john = Author.objects.create(name="John")
>>> paul = Author.objects.create(name="Paul")
>>> george = Author.objects.create(name="George")
>>> ringo = Author.objects.create(name="Ringo")
>>> entry.authors.add(john, paul, george, ringo)
```

---

## Retrieving objects

데이터베이스에서 객체를 검색하기위해 모델 클래스의 **Manager** 를 통해 **QuerySet** 을 생성한다
QuerySet는 데이터베이스의 객체 컬렉션을 나타낸다. 0 개, 하나 또는 여러 개의 **filter** 를 가질 수 있다. **filter** 는 주어진 매개 변수를 기반으로 쿼리 결과의 범위를 좁 힙니다. SQL 용어에서 **QuerySet** 은 SELECT 문과 같으며 **filter** 는 WHERE 또는 LIMIT와 같은 제한 절이다.

모델의 **Manager** 를 사용해 **QuerySet** 을 가져올 수 있다.
>Manager : django 모델에 데이터베이스 쿼리 작업을 제공하는 인터페이스이다. django 어플리케이션의 모든 모델들은 모두 하나 이상의 manager가 존재한다.

```python
>>> Blog.objects
<django.db.models.manager.Manager object at ...>
>>> b = Blog(name='Foo', tagline='Bar')
>>> b.objects
Traceback:
    ...
AttributeError: "Manager isn't accessible via Blog instances."
```

>Manager는 모델 인스턴스가 아닌 모델 클래스로만 접근할 수 있다

### Retrieving all objects

**all()** 메소드를 사용하면 테이블의 모든 객체를 QuerySet으로 가져올 수 있다

```python
>>> all_entries = Entry.objects.all()
```

### Retrieving specific objects with filters

필요한 객체만을 선별해서 QuerySet으로 가져오는 두가지 방법이 있다

**filter(**kwargs)**
주어진 검색 매개변수와 **일치하는** 객체들을 포함하는 QuerySet을 리턴한다

**exclude(**kwargs)**
주어진 검색 매개변수와 **일치하지 않는** 객체들을 포함하는 QuerySet을 리턴한다

#### Chaining filter

필터를 연결하여 쓸 수 있다.

```python
>>> Entry.objects.filter(
...     headline__startswith='What'
... ).exclude(
...     pub_date__gte=datetime.date.today()
... ).filter(
...     pub_date__gte=datetime(2005, 1, 30)
... )
```


#### QuerySets are lazy

쿼리셋을 생성하는 것은 데이터베이스에 영향이 없다. 장고는 쿼리셋이 평가될 때 까지 쿼리를 실행하지 않는다.

```python
>>> q = Entry.objects.filter(headline__startswith="What")
>>> q = q.filter(pub_date__lte=datetime.date.today())
>>> q = q.exclude(body_text__icontains="food")
>>> print(q)
```
>위의 4개의 코드 라인이 있는데 오로지 마지막인 `print(q)` 에서만 데이터베이스에 도달한다.

### Retrieving a single object with get()

**filter()** 도 하나의 객체를 가져올 수 있지만, 하나의 객체를 가져올 때는 **get()** 을 사용한다.

```python
>>> one_entry = Entry.objects.get(pk=1)
```

### Limiting QuerySets

파이썬은 배열 슬라이스 구문을 가지고 쿼리셋을 특정 갯수의 결과로 제한할 수 있다.



```python
#처음 다섯개의 객체를 리턴
>>> Entry.objects.all()[:5]

#6번쨰~10번째 객체를 리턴
>>> Entry.objects.all()[5:10]

# 10개중에 짝수번쨰 객체를 리턴
>>> Entry.objects.all()[:10:2]

# 제목을 알파벳 순으로 정렬, 두 개가 같은 결과를 가진다
# 첫번째는 IndexError를 발생시키고, 두번째는 일치하는 객체가 없으면 DoesNotExist를 발생시킨다
>>> Entry.objects.order_by('headline')[0]
Entry.objects.order_by('headline')[0:1].get()
```

### Field lookups

기본 검색 키워드 인자는 `field__lookuptype=value` 형식이다

```python
>>> Entry.objects.filter(pub_date__lte='2006-01-01')
# SQL 문으로 표현하면
SELECT * FROM blog_entry WHERE pub_date <= '2006-01-01';

# id로 검색
>>> Entry.objects.filter(blog_id=4)

# __exact
>>> Entry.objects.get(headline__exact="Cat bites dog")

# __exact를 생략해도 똑같다
>>> Blog.objects.get(id__exact=14)  
>>> Blog.objects.get(id=14)      

# __iexaxt 는 대소문자를 구별하지 않는 검색이다
>>> Blog.objects.get(name__iexact="beatles blog")

# __contains 대소문자를 구별하여 검색한다
Entry.objects.get(headline__contains='Lennon')
```

`__startswith` ,`__endswith` 는 각각 시작하는 단어, 끝나는 단어를 검색할때 사용한다


### Lookups that span relationships

원하는 필드를 찾기위해 `__` 를 사용해서 필드이름을 연결한다

```python
>>> Blog.objects.filter(entry__headline__contains='Lennon')

# __isnull=True, 해당 필드가 빈값인 항목을 리턴한다
Blog.objects.filter(entry__authors__name__isnull=True)

```

#### Spanning multi-valued relationships

```python
# 제목에 Lennon이 들어가고 2008년에 작성된 글을 리턴한다
Blog.objects.filter(entry__headline__contains='Lennon', entry__pub_date__year=2008)

# 제목이 Lennon이 들어가거나 2008년에 작성된 글을 모두 리턴한다
Blog.objects.filter(entry__headline__contains='Lennon').filter(entry__pub_date__year=2008)
```

### Filters can reference fields on the model

**F()** 를 사용해서 동일한 모델 인스턴스의 두 개의 다른 필드 값을 비교할 수 있다.

```python
>>> from django.db.models import F
>>> Entry.objects.filter(n_comments__gt=F('n_pingbacks'))

# 2배 더 많은 코멘트가 있는 블로그를 리턴
>>> Entry.objects.filter(n_comments__gt=F('n_pingbacks') * 2)

# 엔트리 등급이 핑백 카운트와 코멘트 카운트의 합보다 작은 모든 엔트리를 리턴
>>> Entry.objects.filter(rating__lt=F('n_comments') + F('n_pingbacks'))

# 작성자의 이름과 블로그의 이름이 같은 검색결과 리턴
>>> Entry.objects.filter(authors__name=F('blog__name'))

# 작성되고나서 3일 이후에 변경된 것들을 리턴
>>> from datetime import timedelta
>>> Entry.objects.filter(mod_date__gt=F('pub_date') + timedelta(days=3))


```

### The pk lookup shortcut

다음 세 문장은 모두 같다
```python
>>> Blog.objects.get(id__exact=14) # 기본 양식
>>> Blog.objects.get(id=14) # __exact 생략
>>> Blog.objects.get(pk=14) # pk는 id__exact와 같다
```

```python
# id가 1, 4, 7 인 블로그들을 리턴한다
>>> Blog.objects.filter(pk__in=[1,4,7])

# id가 14보다 큰 블로그들을 리턴한다
>>> Blog.objects.filter(pk__gt=14)
```

### Escaping percent signs and underscores in LIKE statements

```python
>>> Entry.objects.filter(headline__contains='%')
```

```SQL
SELECT ... WHERE headline LIKE '%\%%';
```

### Caching and QuerySets

```python
>>> print([e.headline for e in Entry.objects.all()])
>>> print([e.pub_date for e in Entry.objects.all()])
```
>위의 두 문장들은 데이터베이스 쿼리가 두번 실행되기 때문에 데이터베이스 로드가 두배로 늘어난다

```python
>>> queryset = Entry.objects.all()
>>> print([p.headline for p in queryset]) # 쿼리셋을 평가한다
>>> print([p.pub_date for p in queryset]) # 평가된 쿼리셋의 캐쉬를 재사용한다
```
>두번 실행을 피하기위해 저장을 하고나서 불러온다

#### When QuerySets are not cached

쿼리셋 객체에서 반복적으로 특정 인덱스를 얻으면 매번 데이터베이스를 접근한다
```python
>>> queryset = Entry.objects.all()
>>> print(queryset[5]) # 데이터베이스에 접근
>>> print(queryset[5]) # 다시 데이터베이스에 접근
```

전체 쿼리셋이 평가된 상태라면 대신 캐시를 확인한다
```python
>>> queryset = Entry.objects.all()
>>> [entry for entry in queryset] # 데이터베이스에 접근
>>> print(queryset[5]) # 캐시 사용
>>> print(queryset[5]) # 캐시 사용
```

전체 쿼리셋을 평가하여 캐시를 채우는 몇가지 방법들
```python
>>> [entry for entry in queryset]
>>> bool(queryset)
>>> entry in queryset
>>> list(queryset)
```

## Complex lookups with Q objects

**Q객체** 는 객체를 캡슐화한다. 캡슐화하여 여러 조건들을 사용할 수 있다

```python
from django.db.models import Q
Q(question__startswith='What')
```

Q객체는 &(and)와 |(or)을 사용해 결합될 수 있다
```python
Q(question__startswith='Who') | Q(question__startswith='What')

# ~은 not을 의미함
Q(question__startswith='Who') | ~Q(pub_date__year=2005)
```

filer(), exclude(), get() 등의 검색함수들은 하나 이상의 Q객체를 받을 수 있다
```python
Poll.objects.get(
    Q(question__startswith='Who'),
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
)
```

검색함수는 Q객체와 키워드인자를 같이 받을 수 있다. 이때 Q객체는 키워드 인자보다 앞에 나열되야 한다

```python
Poll.objects.get(
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6)),
    question__startswith='Who',
)
```


## Comparing objects

두 객체를 비교한다

```python
>>> some_entry == other_entry
>>> some_entry.id == other_entry.id

>>> some_obj == other_obj
>>> some_obj.name == other_obj.name
```


## Deleting objects

**delete()** 를 통해 객체를 삭제한다. 삭제된 개체의 개수와 해당 딕셔너리를 리턴한다

```python
>>> e.delete()
(1, {'weblog.Entry': 1})
```

여러 객체를 한번에 삭제할 수 있다.
```python
>>>
# 2005년에 작성된 객체들을 모두 삭제한다
Entry.objects.filter(pub_date__year=2005).delete()
(5, {'webapp.Entry': 5})

# 모든 객체를 불러와 삭제한다
Entry.objects.all().delete()
```

`on_delete` 가 사용된 ForeignKey를 가진 객체라면 모든 객체가 다 지워진다

## Copying model instances

pk값을 None으로 설정하여 쉽게 모델 인스턴스를 복사할수 있다
```python
blog = Blog(name='My blog', tagline='Blogging is easy')
blog.save() # blog.pk == 1

blog.pk = None
blog.save() # blog.pk == 2
```

상속을 받았을땐 좀더 복잡해진다. pk와 id값을 둘다 None으로 설정해야한다
```python
class ThemeBlog(Blog):
    theme = models.CharField(max_length=200)

django_blog = ThemeBlog(name='Django', tagline='Django is easy', theme='python')
django_blog.save() # django_blog.pk == 3

django_blog.pk = None
django_blog.id = None
django_blog.save() # django_blog.pk == 4
```

ManyToManyFieldd의 경우에는 엔트리를 복사한 후에 새로운 엔트리에 대해 Many-to-many 관계를 설정해야한다
```python
entry = Entry.objects.all()[0] # some previous entry
old_authors = entry.authors.all()
entry.pk = None
entry.save()
entry.authors.set(old_authors)
```

OneToOneField의 경우에는 객체를 복사한 후에 새 개체의 필드에 할당해야 한다.
```python
detail = EntryDetail.objects.all()[0]
detail.pk = None
detail.entry = entry
detail.save()
```

## Updating multiple objects at once

한번에 여러 객체들의 필드를 특정값으로 설정할 수 있다
```python
# 2007년에 작성된 글들의 제목을 바꾼다
Entry.objects.filter(pub_date__year=2007).update(headline='Everything is the same')



>>> b = Blog.objects.get(pk=1)

# 모든 엔트리를 변경해서 이 블로그에 포함되게 한다
>>> Entry.objects.all().update(blog=b)



>>> b = Blog.objects.get(pk=1)

# 이 블로그에 속한 모든 헤드라인을 업데이트한다
>>> Entry.objects.select_related().filter(blog=b).update(headline='Everything is the same')



for item in my_queryset:
    item.save()



>>> Entry.objects.all().update(n_pingbacks=F('n_pingbacks') + 1)



# 이것은 에러를 FieldError를 일으킨다
>>> Entry.objects.update(headline=F('blog__name'))    
```

## Related objects

### One-to-many relationships

#### Forward

모델에 ForeignKey가 있는 경우, 해당 모델의 인스턴스는 모델의 단순 속성을 통해 관련 외부 객체에 접근할 수 있다

```python
>>> e = Entry.objects.get(id=2)
>>> e.blog # 관련된 블로그 객체를 리턴한다

# ForeignKey를 통해 가져오고 설정할 수 있다
# 변경사항은 save()를 호출전까지 저장되지 않는다
>>> e = Entry.objects.get(id=2)
>>> e.blog = some_blog
>>> e.save()

# ForeignKey에 null=True가 설정됐다면, None값을 할당후에 관계를 제거할 수 있다
>>> e = Entry.objects.get(id=2)
>>> e.blog = None
>>> e.save() # "UPDATE blog_entry SET blog_id = NULL ...;"

>>> e = Entry.objects.get(id=2)
>>> print(e.blog)  # 데이터베이스에 접근한다
>>> print(e.blog)  # 캐쉬를 사용한다

>>> e = Entry.objects.select_related().get(id=2)
>>> print(e.blog)  # 캐쉬를 사용한다
>>> print(e.blog)  # 캐쉬를 사용한다
```

#### Following relationships “backward”

```python
>>> b = Blog.objects.get(id=1)
>>> b.entry_set.all() # 블로그에 관련된 모든 엔트리 객체를 리턴한다

# b.entry_set은 쿼리셋을 리턴하는 Maneger이다
>>> b.entry_set.filter(headline__contains='Lennon')
>>> b.entry_set.count()


# related_name을 설정해서 겹쳐쓸수있다(override)
>>> b = Blog.objects.get(id=1)
>>> b.entries.all()

# b.entries은 쿼리셋을 리턴하는 Maneger이다
>>> b.entries.filter(headline__contains='Lennon')
>>> b.entries.count()
```

#### Using a custom reverse manager

특정쿼리에 대해서 다른 Manager를 지정할 수 있다
```python
from django.db import models

class Entry(models.Model):
    #...
    objects = models.Manager()  # Default Manager
    entries = EntryManager()    # Custom Manager

b = Blog.objects.get(id=1)
b.entry_set(manager='entries').all()
```

#### Additional methods to handle related objects

`add(obj1, obj2, ...)`
  - 지정된 모델 객체를 관련 객체 세트에 추가한다

`create(**kwargs)`
  - 새 객체를 만들고 저장후, 관련 객체 세트에 배치한다. 새로 생성된 객체를 리턴한다

`remove(obj1, obj2, ...)`
  - 관련 객체 세트에서 지정된 모델 객체를 제거한다

`clear()`
  - 모든 객체를 제거한다
`set(objs)`
  - 관련 객체 세트르를 대체한다

### Many-to-many relationships

```python
e = Entry.objects.get(id=3)
e.authors.all() # 엔트리에서 모든 author객체를 리턴한다
e.authors.count()
e.authors.filter(name__contains='John')

a = Author.objects.get(id=5)
a.entry_set.all() # author에서 모든 엔트리객체를 리턴한다
```

### One-to-one relationships

일대일관계는 다대일관계와 매우 유사하다. 모델에 OneToOneField를 정의하면 해당 모델의 인스턴스는 모델의 단순 속성을 통해 관련 객체에 접근할 수 있다.

```python
class EntryDetail(models.Model):
    entry = models.OneToOneField(Entry, on_delete=models.CASCADE)
    details = models.TextField()

ed = EntryDetail.objects.get(id=2)
ed.entry # 관계된 엔트리 객체를 리턴한다

e = Entry.objects.get(id=2)
e.entrydetail # 관련된 EntryDetail 객체를 리턴한다
```



### Queries over related objects

id=5인 블로그 객체 b가 있을때 다음 세 가지 쿼리들은 모두 같다
```python
Entry.objects.filter(blog=b) # 객체 인스턴스를 사용한다
Entry.objects.filter(blog=b.id) # 인스턴스의 id를 사용한다
Entry.objects.filter(blog=5) # id를 직접적으로 사용한다
```
