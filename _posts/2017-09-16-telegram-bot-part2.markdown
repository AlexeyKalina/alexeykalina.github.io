---
layout: post
title:  "Телеграм бот. Часть 2. TelegramBotCore"
date:   2017-09-16 20:43:30 +0300
categories: technologies
tags: [technologies,telegram]
image:
  feature: 2017-09-16/telegram1.jpg
  teaser: 2017-09-16/telegram1-teaser.jpg
credit: Death to Stock Photo
creditlink: ""
---

Всем привет! Мы продолжаем заниматься телеграм ботом. Напомню, что суть бота состоит в том, что он должен возвращать подходящие рецепты по списку введенных ингредиентов. В [прошлой части](https://alexeykalina.github.io/technologies/telegram-bot-part1.html) мы занимались поиском необходимых данных, теперь же займемся реализацией приложения. Думаю, после прочтения этого поста Вы поймете, что создать бота проще простого:)

### Бот-батька

Первым шагом в создании любого телеграм-бота является небольшая переписка с отцом всех ботов. Вам нужно написать */start* боту с именем **@BotFather**, ответить на несколько его вопросов и получить API token. Благодаря этому токену у вас будет доступ к своему детищу в вашем коде.

![BotFather]({{ site.baseurl }}/assets/img/2017-09-16/telegram2.jpg "BotFather")

### Создание приложения

Теперь у нас есть бот, который ничего не умеет делать. Для реализации логики я использовал язык C#. Многие не любят его за так называемое "отсутствие кроссплатформенности". Так вот, уже больше года как был выпущен фреймворк **.NET Core**, благодаря которому на C# можно писать под разные платформы. Эта технология стремительно развивается (в августе произошел значимый релиз версии 2.0). Код телеграм бота написан на этом фреймворке, более того разработка целиком проводилась на Linux. Да, пока по удобству это сильно уступает разработке на привычной Visual Studio, однако имеет право на жизнь, и, я уверен, постепенно будет все лучше и лучше.

Бот написан как обычное консольное приложение. Для продакшена это не лучший вариант, но для учебного варианта самое то. Вся логика программы строится на обработке событий, воспроизводимых пользователем. В качестве библиотеки взаимодействующей с Bot API телеграма воспользуемся **TelegramBotCore**.

Создадим клиента, для общения с API. Тут нам и пригодится токен, полученный от BotFather.

{% highlight C# %}
var bot = new TelegramBotClient(token);
{% endhighlight %}

Вся логика приложения строится на обработке событий, инициируемых пользователем.

### Обработка событий

В приложении используются всего два обработчика события, но они вполне решают поставленные задачи. Добавим простейшую обработку сообщений пользователя:

{% highlight C# %}
bot.OnMessage += BotOnMessageReceived;
{% endhighlight %}

{% highlight C# %}
private async void BotOnMessageReceived(object sender, MessageEventArgs messageEventArgs)
{
    keyboard = new InlineKeyboardMarkup(new[]
    {
        new[]
        {
            new InlineKeyboardCallbackButton("Показать еще", "next"),
            new InlineKeyboardCallbackButton("Ингредиенты", "ingredients"),
        }
    });
    // Поиск рецепта в Neo4J...
    await bot.SendTextMessageAsync(message.Chat.Id, recipe.Name, 
                        parseMode: ParseMode.Html, replyMarkup: keyboard);
}
{% endhighlight %}

Полученный результат отправляем сообщением пользователю. Помимо этого показываем пару кнопок для просмотра ингредиентов и вывода следующего рецепта.

Обработка нажатий на кнопки происходит в следующем коде:

{% highlight C#  %}
bot.OnCallbackQuery += BotOnCallbackQueryReceived;
{% endhighlight %}

{% highlight C# %}
private async void BotOnCallbackQueryReceived(object sender, CallbackQueryEventArgs callbackQueryEventArgs)
{
    if (callbackQueryEventArgs.CallbackQuery.Data.Equals("next"))
    {
        // Код определения следующего рецепта...
        await bot.SendTextMessageAsync(callbackQueryEventArgs.CallbackQuery.Message.Chat.Id, 
        nextRecipe, parseMode: ParseMode.Html, replyMarkup: keyboard);
    }
    else if (callbackQueryEventArgs.CallbackQuery.Data.Equals("ingredients"))
    {
        var body = new StringBuilder();
        body.AppendLine(string.Format("<b>{0}</b>", recipe.Name));

        for (var counter = 1; counter <= recipe.Ingredients.Count; counter++)
            body.AppendLine(string.Format("{0}. {1}", counter, ingredient));

        await bot.SendTextMessageAsync(callbackQueryEventArgs.CallbackQuery.Message.Chat.Id, 
        body.ToString(), parseMode: ParseMode.Html, replyMarkup: keyboard);
    }
}
{% endhighlight %}

В зависимости от значения свойства *Data* полученного коллбэка, мы определяем на какую кнопку было произведено нажатие и отправляем соответствующее сообщение в чат. 

### Запуск бота

Я опускаю подробности взаимодействия с Neo4J, а также описание поддержки работы с многими пользователями (необходимо разделять запросы приходящие из разных чатов, чтобы ответы не смешивались). Все это можно посмотреть в [репозитории](https://github.com/AlexeyKalina/RecipesTelegramBot). Когда все готово, мы можем наконец запустить нашего бота:

{% highlight C#  %}
bot.StartReceiving();
{% endhighlight %}

Вот так выглядит окончательный вариант бота в телеграме:

![RecipesBot]({{ site.baseurl }}/assets/img/2017-09-16/telegram3.jpg "RecipesBot")

### Заключение

Здесь приведено минимальное взаимодействие с Bot API телеграма, но его вполне хватит для написание своего простого бота и последующего его расширения. Как видите, код получился совсем простой, и при желании можно создать решение действительно полезное в повседневной жизни, при этом не приложив больших усилий.

