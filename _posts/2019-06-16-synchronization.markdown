---
layout: post
title:  "Синхронизация в потокобезопасных контейнерах"
date:   2019-06-16 11:33:30 +0300
categories: technologies
tags: [technologies, concurrency, java]
image:
  feature: 2019-06-16/sync.jpg
  teaser: 2019-06-16/sync-teaser.jpg
---

Сегодня мы поговорим о многопоточности. Не о том многоядерном параллелизме, который должен спасти нас от замедления закона Мура, а о ситуации, когда есть общая память и множество потоков, которые принимают запросы и взаимодействуют с этой памятью. В качестве абстракции общей памяти, как правило, используют потокобезопасные контейнеры. Внутри они используют синхронизацию потоков, чтобы их несогласованные действия не испортили данные. Мы рассмотрим различные способы синхронизации, написав потокобезопасный сет на языке Java.

Статья основана на одной из лекций курса Параллельное программирование в [Computer Science Center](https://compscicenter.ru/). Видеозаписи лекций 2016 года есть на [youtube](https://www.youtube.com/playlist?list=PLlb7e2G7aSpQCPeKTcVBHJns_JOxrc_fT).

# Сет на основе односвязного списка

Мы будем разбираться в теме, используя достаточно простую реализацию абстрактного контейнера **сета** (множества). Напомню, сет хранит только уникальные элементы и соответствует математическому понятию *множества*. Он включает три основных метода: добавление и удаление элементов и проверка на наличие элемента в контейнере. Есть много способов реализовать сет (поэтому эта структура и относится к абстрактным). Например, одной из самых популярных реализаций в языке Java является *HashSet*. Внутри он использует хеш-таблицу, ключами которой являются элементы сета, а значениями — фейковый элемент.

{% highlight java %}
public interface MySet<T extends Comparable<T>> {
    boolean add(T value);
    boolean remove(T value);
    boolean contains(T value);
}
{% endhighlight %}

Для того, чтобы разобраться в самых разных способах синхронизации сета, мы будем использовать менее производительную реализацию — на основе односвязного списка. Для поддержки уникальности значений, в сете используется следующий инвариант: каждый элемент списка больше предыдущего. Для простоты наш сет будет работать только для сравниваемых элементов (реализующих интерфейс *Comparable*). Реализация каждого из методов сета основывается на поиске места в списке, где должен находиться элемент. Ниже представлен код первоначальной версии контейнера, не поддерживающей синхронизацию.

{% highlight java %}
class Node<T extends Comparable<T>> {
    private T value;
    private Node<T> next;

    Node(T value, Node<T> next) {
        this.value = value;
        this.next = next;
    }

    T value() {
        return value;
    }

    Node<T> getNext() {
        return next;
    }

    void setNext(Node<T> next) {
        this.next = next;
    }
}

public class SimpleSet<T extends Comparable<T>> implements MySet<T> {
    private Node<T> head;

    public SimpleSet() {
        head = new Node<>(null, null);
    }

    @Override
    public boolean contains(T value) {
        Node<T> curr = head.getNext();
        while (curr != null && curr.value().compareTo(value) < 0) {
            curr = curr.getNext();
        }
        return curr != null && value.compareTo(curr.value()) == 0;
    }

    @Override
    public boolean add(T value) {
        Node<T> prev = head;
        Node<T> curr = prev.getNext();

        while (curr != null && curr.value().compareTo(value) < 0) {
            prev = curr;
            curr = curr.getNext();
        }
        if (curr != null && value.compareTo(curr.value()) == 0)
            return false;

        Node<T> node = new Node<>(value, curr);
        prev.setNext(node);
        return true;
    }

    @Override
    public boolean remove(T value) {
        Node<T> prev = head;
        Node<T> curr = prev.getNext();

        while (curr != null && curr.value().compareTo(value) < 0) {
            prev = curr;
            curr = curr.getNext();
        }
        if (curr != null && value.compareTo(curr.value()) == 0) {
            prev.setNext(curr.getNext());
            return true;
        }
        return false;
    }
}
{% endhighlight %}

# Линеаризуемость

Прежде чем переходить к синхронизации, нужно определиться со способом проверки правильности работы нашего контейнера. Мы не будем писать модульные тесты на проверку того, что сет действительно вставляет и удаляет элементы — в этом вы можете поупражняться самостоятельно. Нам нужен способ для выявления более сложных ситуаций — возникающих в многопоточной среде. Давайте для начала рассмотрим пример, который покажет, что текущая реализация сета некорректна в случае более чем одного потока.

1. В сете находятся элементы 1, 2 и 4.
2. Вызов метода добавления элемента 3.
3. Метод отработал до момента создания нового узла, и управление было передано другому потоку.
4. Вызов метода удаления элемента 2.
5. Метод полностью отработал, перенеся указатель элемента 1 на элемент 4.
6. Управление вернулось первому потоку, перенесен указатель элемента 2 на элемент 3.
7. Логически метод завершился успешно, но физически элемент 3 не был добавлен.

Такие ситуации сложно промоделировать обычными тестами. Тут нам поможет понятие **линеаризуемости**. Это свойство программы, при котором результат любого параллельного выполнения операций эквивалентен некоторому последовательному выполнению. В реальности получить все возможные перестановки операций в параллельном выполнении кажется слишком затратным и трудно выполнимым, но это можно сделать с последовательным выполнением. Системы тестирования линеаризуемости именно это и проделывают, вычисляя результаты выполения после таких перестановок. Далее запускаются те же операции в параллельной среде и производится проверка, что результат выполнения соответствует одному из полученных в последовательном запуске. Производится значительное число параллельных запусков, чтобы можно было говорить об отсутствии ошибок синхронизации с определенной вероятностью.

Для проверки линеаризуемости нашего сета мы будем использовать библиотеку [Lin-chek](https://github.com/Devexperts/lin-check). Пример подключения ее в Maven:

{% highlight xml %}
<repositories>
    <repository>
        <id>devexperts</id>
        <url>https://dl.bintray.com/devexperts/Maven/</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>com.devexperts.lincheck</groupId>
        <artifactId>lincheck</artifactId>
        <version>2.0</version>
    </dependency>
</dependencies>
{% endhighlight %}

Напишем тест для нашего исходного сета. Для этого необходимо определить операции, которые будут задействованы в выполнении, как будут генерироваться аргументы для операций (в данном случае это целые числа из промежутка от 1 до 5) и опции запуска. Установим 50 итераций, 3 потока и 5 действий на каждый поток.

{% highlight java %}
@Param(name = "value", gen = IntGen.class, conf = "1:5")
@StressCTest
public class SimpleSetTest {
    private MySet<Integer> set = new SimpleSet<>();

    @Operation
    public Boolean add(@Param(name = "value") int value) {
        return set.add(value);
    }

    @Operation
    public Boolean remove(@Param(name = "value") int value) {
        return set.remove(value);
    }

    @Operation
    public Boolean contains(@Param(name = "value") int value) {
        return set.contains(value);
    }

    @Test
    public void test() {
        Options opts = new StressOptions()
                .iterations(50)
                .threads(3)
                .actorsPerThread(5)
                .logLevel(LoggingLevel.INFO);
        LinChecker.check(SimpleSetTest.class, opts);
    }
}
{% endhighlight %}

Вывод теста выглядит следующим образом:

{% highlight bash %}
= Iteration 1 / 50 =
Execution scenario (init part):
[add(1), remove(4), remove(5), contains(3), contains(1)]
Execution scenario (parallel part):
| contains(4) | contains(2) | add(2)      |
| add(5)      | contains(5) | contains(3) |
| contains(3) | contains(4) | remove(3)   |
| contains(1) | contains(5) | remove(5)   |
| add(1)      | contains(2) | contains(1) |
Execution scenario (post part):
[remove(4), add(4), contains(5), contains(3), contains(3)]

= Iteration 2 / 50 =
Execution scenario (init part):
[remove(1), add(4), remove(3), add(3), contains(4)]
Execution scenario (parallel part):
| contains(1) | contains(1) | contains(3) |
| contains(3) | contains(3) | contains(3) |
| add(1)      | contains(4) | remove(4)   |
| add(1)      | contains(1) | remove(1)   |
| contains(1) | contains(1) | remove(4)   |
Execution scenario (post part):
[add(2), remove(5), remove(2), remove(3), add(3)]

= Iteration 3 / 50 =
Execution scenario (init part):
[remove(5), add(3), contains(2), add(2), add(3)]
Execution scenario (parallel part):
| remove(2)   | remove(4) | contains(3) |
| add(4)      | remove(3) | contains(3) |
| remove(4)   | remove(1) | add(4)      |
| add(4)      | add(3)    | remove(2)   |
| contains(4) | remove(4) | add(4)      |
Execution scenario (post part):
[contains(4), remove(3), contains(3), contains(2), contains(2)]
= Invalid execution results: =
Execution results (init part):
[false, true, false, true, false]
Execution results (parallel part):
| false | false | true  |
| true  | true  | true  |
| true  | false | true  |
| true  | true  | true  |
| true  | true  | false |
Execution results (post part):
[true, false, false, false, false]

java.lang.AssertionError: Invalid interleaving found
{% endhighlight %}

Как и ожидалось, тест выявил ошибку линеаризации. Это связано с тем, что наша исходная реализация сета не содержит никакой синхронизации потоков. Уже на третьей итерации теста проявилась проблема доступа к одному элементу из разных потоков. Что ж, приступим к изучению различных методов синхронизации и окрасим вывод теста в зеленый цвет.

# Грубая синхронизация

Начнем с простейшего способа заставить наш сет работать на сколь угодно большом числе потоков. Для этого достаточно добавить в контейнер глобальный мьютекс, который будем захватывать при каждой операции. Такая синхронизация называется **грубой**.

В качестве мьютекса в Java можно использовать класс *ReentrantLock*. Приставка *reentrant* показывает, что данный примитив синхронизации можно захватывать повторно без предварительного освобождения. В нашем случае такой ситуации возникнуть не может, но в большом проекте можно за этим не уследить и при использовании обычного замка получить дедлок даже на одном потоке.

Воспользуемся первоначальной версией сета в реализации структуры с грубой синхронизацией.

{% highlight java %}
public class HardSyncSet<T extends Comparable<T>> implements MySet<T> {
    private MySet<T> set = new SimpleSet<>();
    private Lock mutex = new ReentrantLock();

    @Override
    public boolean add(T value) {
        mutex.lock();
        try {
            return set.add(value);
        } finally {
            mutex.unlock();
        }
    }

    @Override
    public boolean remove(T value) {
        mutex.lock();
        try {
            return set.remove(value);
        } finally {
            mutex.unlock();
        }
    }

    @Override
    public boolean contains(T value) {
        mutex.lock();
        try {
            return set.contains(value);
        } finally {
            mutex.unlock();
        }
    }
}
{% endhighlight %}

Запуск тестов покажет вам, что такая синхронизация работает верно. Действительно, в любой момент времени с контейнером работает только один поток, поэтому линеаризуемость нашего сета очевидна. Тем не менее, в этом то и кроется главная проблема данной реализации. При большом размере структуры и большом числе потоков с запросами, просадка по производительности будет колоссальной. Наша цель: иметь возможность физического параллелизма в разных частях списка. То есть, если один поток хочет добавить элемент в одно место, а другой поток в другое, то они могли бы это делать одновременно.

Прежде чем мы перейдем к другим методам синхронизации, отметим, что и грубую синхронизацию можно несколько оптимизировать. Для этого достаточно использовать не *ReentrantLock*, а *ReentrantReadWriteLock*. Такая оптимизация позволит разделить блокировки на запись и чтение. Благодаря этому, можно делать проверки на наличие элемента параллельно, если в текущий момент не происходит изменений в структуре. Заметим также, что в первом случае вместо явного мьютекса можно было использовать блок *synchronized*.

# Тонкая синхронизация

В грубой синхронизации использовался один мьютекс на весь контейнер. Альтернатива такому способу синхронизации — использование мьютекса в каждом из элементов сета и блокировка константного числа элементов в процессе операции в каждый момент времени. Этот метод называется **тонкой** синхронизацией. Добавим мьютекс в узел списка:

{% highlight java %}
class Node<T extends Comparable<T>> {
    private Lock mutex = new ReentrantLock();
    /* ... */

    void lock() {
        mutex.lock();
    }

    void unlock() {
        mutex.unlock();
    }
}
{% endhighlight %}

Приведу пример реализации только одного из методов сета — остальные операции аналогичны. Для нашего контейнера число элементов, которые необходимо держать одновременно заблокированными, равно двум. Это следует из того, что нам достаточно поддерживать инвариант *prev.next == curr*, чтобы не потерять связь между элементами списка. 

{% highlight java %}
@Override
public boolean add(T value) {
    Node<T> prev = head;
    lock(prev);
    Node<T> curr = prev.getNext();
    lock(curr);
    try {
        while (curr != null && curr.value().compareTo(value) < 0) {
            unlock(prev);
            prev = curr;
            curr = curr.getNext();
            lock(curr);
        }
        if (curr != null && value.compareTo(curr.value()) == 0)
            return false;

        Node<T> node = new Node<>(value, curr);
        prev.setNext(node);
        return true;
    } finally {
        unlock(curr);
        unlock(prev);
    }
}

private void lock(Node<T> node) {
    if (node != null) {
        node.lock();
    }
}

private void unlock(Node<T> node) {
    if (node != null) {
        node.unlock();
    }
}
{% endhighlight %}

В этом методе синхронизации мы платим памятью, добавляя мьютекс в каждый из элементов сета. Но в то же время мы можем значительно выиграть в физическом параллелизме. Действительно, если один поток производит операцию в конце списка, ничего не мешает другим потокам перемещаться по предшествующим позициям. Однако, возможна и ситуация, когда этот поток остановится на одном из первых элементов списка, тем самым не давая другим потокам продвинуться дальше.

# Оптимистичная синхронизация

В данном методе синхронизации мы хотим избавиться от промежуточных блокировок на пути к искомому элементу. Для этого мы бежим по списку вообще без блокировок, пока не доберемся до нужного места, и только тогда блокируем пару элементов. Однако, до того мгновения, как мы повесили блокировки на найденные элементы, они могли быть физически удалены из сета. В этом то и заключается **оптимистичность** этого метода. Мы заново пробегаемся по списку (снова без блокировок) и, если достигаем нашей пары элементов, то она по-прежнему в сете и с ней уже ничего не произойдет. Тогда то мы и можем выполнить необходимую операцию. Иначе, повторяем весь процесс заново.

{% highlight java %}
private boolean validate(Node<T> prev, Node<T> curr) {
    Node<T> node = head;
    while (node != null && (prev.value() == null || node.value() == null || node.value().compareTo(prev.value()) <= 0)) {
        if (node == prev)
            return prev.getNext() == curr;
        node = node.getNext();
    }
    return false;
}

@Override
public boolean contains(T value) {
    while (true) {
        Node<T> prev = head;
        Node<T> curr = head.getNext();
        while (curr != null && curr.value().compareTo(value) < 0) {
            prev = curr;
            curr = curr.getNext();
        }
        lock(prev);
        lock(curr);
        try {
            if (!validate(prev,curr))
                continue;
            return curr != null && value.compareTo(curr.value()) == 0;
        } finally {
            unlock(curr);
            unlock(prev);
        }
    }
}

@Override
public boolean add(T value) {
    while (true) {
        Node<T> prev = head;
        Node<T> curr = prev.getNext();
        while (curr != null && curr.value().compareTo(value) < 0) {
            prev = curr;
            curr = curr.getNext();
        }
        lock(prev);
        lock(curr);
        try {
            if (!validate(prev, curr))
                continue;
            if (curr != null && value.compareTo(curr.value()) == 0)
                return false;
            Node<T> node = new Node<>(value, curr);
            prev.setNext(node);
            return true;
        } finally {
            unlock(curr);
            unlock(prev);
        }
    }
}

@Override
public boolean remove(T value) {
    while (true) {
        Node<T> prev = head;
        Node<T> curr = prev.getNext();
        while (curr != null && curr.value().compareTo(value) < 0) {
            prev = curr;
            curr = curr.getNext();
        }
        lock(prev);
        lock(curr);
        try {
            if (!validate(prev, curr))
                continue;
            if (curr != null && value.compareTo(curr.value()) == 0) {
                prev.setNext(curr.getNext());
                return true;
            }
            return false;
        } finally {
            unlock(curr);
            unlock(prev);
        }
    }
}
{% endhighlight %}

Благодаря такой оптимистичности мы больше не блокируем элементы на нашем пути. Поэтому если первый поток застрянет где-то в начале списка, это не помешает другим потокам пройти дальше и выполнить свою работу. Однако, существуют вероятность того, что перед тем, как блокировки будут установлены, другие потоки удалят найденные элементы из сета, и придется повторять всю работу заново. Этот подход является предпочтительным, только если обход контейнера достаточно производителен, а обход с блокировками работает слишком медленно.

# Ленивая синхронизация

Заметим, что второго прохода по списку можно избежать. Для этого достаточно не удалять физически элементы сета, а помечать их как удаленные. Тогда после того, как блокировки будут установлены, останется проверить соответствующие флаги. Эта идея лежит в основе **ленивой** синхронизации. Добавим новое поле в узел списка.

{% highlight java %}
class Node<T extends Comparable<T>> {
    private boolean marked = false;
    /* ... */

    void mark() {
        marked = true;
    }

    boolean isMarked() {
        return marked;
    }
}
{% endhighlight %}

В операцию удаления нужно добавить одну строчку с вызовом *mark()* для соответствующего узла, а добавление не изменилось вовсе. Валидация же значительно упростилась:

{% highlight java %}
private boolean validate(Node<T> prev, Node<T> curr) {
    return !prev.isMarked() && (curr == null || !curr.isMarked()) && prev.getNext() == curr;
}
{% endhighlight %}

Благодаря этой оптимизации, мы подошли к важному классу неблокирующей синхронизации. Алгоритм называется **неблокирующим**, если в нем не используются традиционные примитивы синхронизации, например, мьютексы. Метод проверки наличия элементов теперь не только не использует блокировки, но и завершает свое выполнение за число шагов, не зависящее от действий других потоков. Это определение класса **Wait-free** алгоритмов (без ожидания). Этот класс является самым сильным среди алгоритмов неблокирующей синхронизации.

{% highlight java %}
@Override
public boolean contains(T value) {
    Node<T> curr = head.getNext();
    while (curr != null && curr.value().compareTo(value) < 0) {
        curr = curr.getNext();
    }
    return curr != null && value.compareTo(curr.value()) == 0 && !curr.isMarked();
}
{% endhighlight %}

Мы получили выигрыш в производительности, но снова заплатили памятью, так как элементы не удаляются физически из сета, а хранятся вечно. Уменьшить воздействие этой проблемы, можно периодически пробегаясь по списку и удаляя помеченные узлы. Это можно делать, когда нагрузка на контейнер незначительна (например, ночью). Тем не менее, это серьезная оптимизация, которая напрямую подвела нас к ключевому методу синхронизации в контейнерах.


# Неблокирующая синхронизация

Последним шагом будет избавление от блокировок вовсе. Мы сделаем полностью неблокирующий контейнер, но для этого нам нужно познакомиться с еще одним понятием. **Compare And Set** (CAS) — атомарная ассемблерная интструкция, которая сравнивает значение в памяти с одним из аргументов и в случае совпадения записывает другой аргумент в память. Важно, что эти две логические операции производятся физически атомарно, то есть между ними не может выполниться никакая другая операция ни одного процесса. Это основной механизм неблокирующей синхронизации в многопоточных контейнерах.

В Java операция CAS представлена в классах *Atomic*. Они присутствуют для всех примитивных типов (например, *AtomicBoolean*), для ссылок (*AtomicReference&lt;T&gt;*), а так же для одновременного хранения ссылки и флага (*AtomicMarkableReference&lt;T&gt;*) и ссылки и числа (*AtomicStampedReference&lt;T&gt;*). В нашей задаче мы объединим ссылку на следующий элемент списка и метку о том, что узел удален, в одно *Atomic* поле. Замена ссылки будет производиться следующим образом: сравнивается первый аргумент с действительной ссылкой и то, что флаг по-прежнему равен *false*, и в случае успеха ссылка заменяется вторым аргументом. Все это производится атомарно. Код обновленного узла списка:

{% highlight java %}
class Node<T extends Comparable<T>> {
    private T value;
    private final AtomicMarkableReference<Node<T>> next;

    Node(T value, Node<T> next) {
        this.value = value;
        this.next = new AtomicMarkableReference<>(next, false);
    }

    T value() {
        return value;
    }

    Node<T> getNext() {
        return next.getReference();
    }

    boolean casNext(Node<T> old, Node<T> next) {
        return this.next.compareAndSet(old, next, false, false);
    }

    boolean mark(Node<T> next) {
        return this.next.compareAndSet(next, next, false, true);
    }

    boolean isMarked() {
        return next.isMarked();
    }
}
{% endhighlight %}

Теперь перейдем к самому сложному: напишем сет без блокировок, только с использованием CAS-операций. Операции мутации сета используют метод *search*, который находит нужную пару элементов, как и в прошлых реализациях. Он использует CAS, если встречает помеченный узел, чтобы физически удалить его. В остальном реализация стандартная. Также CAS используется в ситуациях, когда сет изменяется. С его помощью мы пытаемся подменить ссылку или флаг у узла списка. Если это не выходит сделать, значит другой поток нас опередил, и нужно повторить операцию заново.

{% highlight java %}
public class LockFreeSet<T extends Comparable<T>> implements MySet<T> {
    private Node<T> head;

    public LockFreeSet() {
        this.head = new Node<>(null, null);
    }

    @Override
    public boolean add(T value) {
        while (true) {
            Pair<T> pair = search(head, value);
            Node<T> prev = pair.prev;
            Node<T> curr = pair.curr;
            
            if (curr != null && curr.value().compareTo(value) == 0) {
                return false;
            } else {
                Node<T> node = new Node<>(value, curr);
                if (prev.casNext(curr, node)) {
                    return true;
                }
            }
        }
    }

    @Override
    public boolean remove(T value) {
        while (true) {
            Pair<T> pair = search(head, value);
            Node<T> prev = pair.prev;
            Node<T> curr = pair.curr;

            if (curr == null || curr.value().compareTo(value) != 0) {
                return false;
            } else {
                Node<T> next = curr.getNext();
                if (!curr.mark(next))
                    continue;
                prev.casNext(curr, next);
                return true;
            }
        }
    }

    @Override
    public boolean contains(T value) {
        Node<T> curr = head.getNext();
        while (curr != null && curr.value().compareTo(value) < 0) {
            curr = curr.getNext();
        }
        return curr != null && curr.value().compareTo(value) == 0 && !curr.isMarked();
    }

    private Pair<T> search(T value) {
        retry:
        while (true) {
            Node<T> prev = head;
            Node<T> cur = head.getNext();
            Node<T> next;
            while (cur != null) {
                next = cur.getNext();
                if (cur.isMarked()) {
                    if (!prev.casNext(cur, next))
                        continue retry;
                    cur = next;
                } else {
                    if (cur.value().compareTo(value) >= 0)
                        return new Pair<>(prev, cur);
                    prev = cur;
                    cur = next;
                }
            }
            return new Pair<>(prev, null);
        }
    }

    class Pair<T extends Comparable<T>> {
        final Node<T> prev, curr;

        Pair(Node<T> prev, Node<T> cur) {
            this.prev = prev;
            this.curr = cur;
        }
    }
}
{% endhighlight %}

Контейнер, который мы написали принадлежит классу **Lock-free** алгоритмов (без блокировок). В этом классе мы не можем гарантировать число шагов выполнения, не зависящее от других потоков (как в Wait-free). Но мы можем быть уверены, что если нам приходится повторять все заново, то какому-то другому потоку удалось выполнить свою задачу. Данный класс алгоритмов гарантирует постоянный прогресс системы.

Мы рассмотрели разные подходы к синхронизации потоков в контейнерах. Каждый из них имеет право на жизнь, у каждого есть свои недостатки. Где-то приходится платить памятью, где-то процессорным временем, а где-то сложностью разработки и трудноуловимыми ошибками.
