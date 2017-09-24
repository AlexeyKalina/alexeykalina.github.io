---
layout: post
title:  "C# 7.1 - новое в привычном"
date:   2017-09-24 10:43:30 +0300
categories: technologies
tags: [technologies, с#]
image:
  feature: 2017-09-24/new-language-version.png
  teaser: 2017-09-24/new-language-version-teaser.jpg
credit: Death to Stock Photo
creditlink: ""
---

Весной этого года был выпущен релиз языка C# версии 7.0. В нем было много достаточно интереснных нововведений. За прошедшее время был собран определенный фидбек, и вот в августе появился на свет C# 7.1. В этом релизе есть как и новые конструкции языка, так и доработки предыдущей версии. В этом посте мы разберемся с новыми фичами и, там где нужно, обратимся к тому, что добавили в C# этой весной.

### Асинхронный вызов метода Main (Async Main)

**Проблема**: 
У вас есть приложение, и его код должен запускаться асинхронно. Как правило это небольшие тестовые приложения.

**Решение в предыдущих версиях языка**: 
Необходимо создать метод, помеченный ключевым словом async, и вызвать его из метода *Main*.

**Решение в C# 7.1**: 
В новой версии языка ключевое слово async можно приписывать к методу *Main*. При этом возвращающее значение должно быть либо *Task* либо *Task\<int\>*.

**Пример**: 
{% highlight C# %}
public static async Task Main(string[] args)
{
    try
    {
        var client = new System.Net.Http.HttpClient();
        var response = await client.GetAsync("https://www.google.com");
        Console.WriteLine(response.StatusCode);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Ошибка: {ex}");
    }
}
{% endhighlight %}

### Типизированные по умолчанию литералы (Target-typed default literals)

**Проблема**: 
Исользование значений по умолчанию для конкретных типов. Такая проблема появляется регулярно, например, при использовании *Generic*. Также часто возникает желание вернуть дефолтное значение из метода или проинициализировать переменную значением по умолчанию. 

**Решение в предыдущих версиях языка**: 
Использование ключевого слово *default(T)*. Оно возвращает значение по умолчанию для типа *T*.

**Решение в C# 7.1**: 
Теперь у программистов C# есть более элегантное решение этой проблемы. В языке появился синтаксический сахар для *default(T)* в виде ключевого слово **default**, которое автоматически определяет тип значения по умолчанию. Оно поддерживается при инициализации локальных переменных, в возвращаемых значениях, в тернарных операторах, в инициализаторах массивов и объектов, с оператором *is*, с *generics*, а также как опциональный параметр. Все возможные способы использования представлены в примере.

**Пример**: 
Примеры надуманные, но демонстрируют все возможности новой фичи.
{% highlight C# %}
public Student Create(string firstName, string lastName, long id = default)
{
    DateTime birthDate = default;
    if (id is default)
        id = GenerateId();
    if (StudentExists(id))
        return default;
    return new Student
    {
        Id = id,
        FirstName = firstName,
        LastName = lastName,
        BirthDate = birthDate,
        AvgScore = default
    };
}
{% endhighlight %}
{% highlight C# %}
public class PriceContainer<T>
{
    // ...
    public void AddTestValue()
    {
        T product = default;
        int price = IsSaleTime() ? default : 100;
        container.Add(product, price);
    }
}
{% endhighlight %}

### Предположительные имена кортежей (Tuple name inference)
**Проблема**: 
Возвращение нескольких значений из метода.

**Решение в предыдущих версиях языка**: 
Есть несколько путей для решения этой распространенной проблемы:
1. Ключевое слово *out*. Думаю, все согласны, что это не самый удобный способ.
2. Класс *Tuple<...>* как возвращаемое значение. Требует подробного описания и выделения памяти для объекта кортежа.
3. Кастомные классы разработчика. Нужно писать много лишнего кода, когда хочется просто вернуть группу значений.
4. Анонимные типы. Большие накладные расходы по производительности и отстутствие статической проверки типов.

**Решение в C# 7.1**: 
В версии языка 7.0 был введен новый способ для множественного возвращения данных из метода. Это новый вид кортежей - типы (**tuple types**) и литералы (**tuple literals**).
{% highlight C# %}
(type1, type2, ... , typeN) Method() // tuple type
{
    // ...
    return (var1, var2, ..., varM); // tuple literal
}
{% endhighlight %}
{% highlight C# %}
var result = Method(); 
Console.WriteLine($"{result.Item1} {result.Item3}");
{% endhighlight %} 
В коде использующем кортежи можно обращаться к конкретным значениям по именам: 
{% highlight C# %}
(type1, type2, ... , typeN) Method() // tuple type
{
    // ...
    return (name1: var1, name2: var2, ..., nameM: varM); // tuple literal
}
{% endhighlight %}
{% highlight C# %}
var result = Method(); 
Console.WriteLine($"{result.name1} {result.name3}");
{% endhighlight %}
В последнем релизе была добавлена возможность в некоторых случаях не прописывать имена в кортежах, но тем не менее иметь к ним доступ в коде не через зарезервированные имена *ItemN*. Предположительные имена срабатывают для локальных переменных, свойств объектов и условных членов (*x?.y*). 

Преимущество таких кортежей перед тем же классом *System.Tuple<...>* в том, что они являются типами значений и не требуют выделения памяти в куче. Кроме того, два кортежа-литерала эквивалентны, если эквивалентны все их значения, то есть их можно использовать, например, в качестве ключей для словаря.

**Пример**:
{% highlight C# %}
DateTime today = DateTime.Today;
books.Select(book => (book.Name, book.Exists, book.Publishers?.Count, today))
     .Where(info => info.Exists && info.Count > 0 || info.today.DayOfWeek == DayOfWeek.Sunday);
{% endhighlight %}

### Сопоставление шаблонов для Generic (Generic Pattern mathing)

**Проблема**: 
Беопасное приведение типов.

**Решение в предыдущих версиях языка**: 
Использование *is* в условии и последующее приведение типа. Другой вариант: использование ключего слова *as*. Проблема в том, что оно не различает несоответствие типов и сравнение с *null* и кроме того не работает с *Non-Nullable* типами.

**Решение в C# 7.1**: 
В C# 7.0 добавили новое понятие **pattern matching**, пришедшее из функциональных языков. Оно имеет три выражения: **const patterns**, **type patterns** и **var patterns**. В *const patterns* производится сопоставление с конкретными значениями (включая *null*, что очень удобно), в *type patterns* - с типами, а *var pattern* сопоставится со всем, в том числе и *null*. Удобство шаблонов типов в том, что при сопоставление происходит автоматический cast и новую переменную можно использовать в последующем коде. Проблема C# 7.0 состояла в недостаточной поддержке generics для этой фичи (сопоставление производилось только при наличие явных ограничений типа - *where T : ConcreteType*). После крайнего релиза pattern matching производится с любыми открытыми типами. 

**Пример**: 
Pattern matching на данный момент поддерживается для двух конструкций языка - *is* и *switch-case*. 
{% highlight C# %}
public int GetNumber(object input)
{
    if(input is null)
        return -1;
    if(input is 5)
        return 5;
    if(input is "Seven")
        return 7;
    if(input is int number)
        return number;
    if(input is string text)
        return text.Length;
    if(input is var another)
        return 0;
    throw new ArgumentException("wrong input");
}
{% endhighlight %}
{% highlight C# %}
public int GetNumber<T>(T input)
{
    switch (input)
    {
        case int number:
            return number;
        case string text:
            return text.Length;
        default:
            return 0;
    }
}
{% endhighlight %}

### Заключение

Релиз языка C# 7.1 не принес кардинально новых изменений, что и подразумевает под собой минорное изменение версии. Однако, появившиеся нововведения могут еще немного облегчить жизнь разработчикам. Наслаждайтесь программированием!