---
layout: post
title: Как поднять блог на Github Pages
permalink: setup-blog-on-github/
tags: github github-pages jekyll
---

Первый пост блога, инструкция.

*Обновлено в ноябре 2026: актуальные команды (Bundler, Docker), версии, работа с плагинами и живые ссылки.*

---

[comment]: <> (This is a comment, it will not be included)
[comment]: <> (in  the output file unless you use it in)
[comment]: <> (a reference style link.)
[comment]:<--# Как поднять сайт на Github Pages. блог распрацоўніка-->

## Содержание

- [Лирика](#lirics)
- [Установка и настройка окружения](#setup)
- [Кастомизация](#customizations)
- [Итоги](#summaries)
- [Ссылки](#refs)

## Лирика   {#lirics}

Давно зрела идея завести личный сайт для того, что бы туда скидывать разные решения, которые нашел в процессе программирования и хотелось бы запомнить. Да и просто иметь место, где мыслью по древу иногда можно растекаться.

Останавливало два фактора: лень и незнание веб-технологий(и отсутсвие большого желания изучать их). Использовать же современные готовые движки типа [Wordpress](https://wordpress.org/), [Blogger](https://www.blogger.com/) или [LiveJournal](https://www.livejournal.com) и иже с ними не хотелось из-за их перегруженности, "вебдванольности", не подконтрольности мне. Да и большинство функционала, который предлагают современные блоги/дневники, мне не нужно.

И вот тут в очередной раз гуляя по [Github](https://github.com) я наталкиваюсь на [Github Pages](https://pages.github.com). Полчаса чтения и я понимаю &mdash; это то что нужно. Суть в чем? Сервис предлагает пользователям Github бесплатный хостинг статических html-страниц для специально созданного репозитория. Простые страницы все же не блог — так какой же от сервиса толк? Там есть генератор удобный, который из пачки шаблонов и простого текста сгенерирует сайт пользователю на радость командой одной — git push — прелесть какая:smile:.

#### Если коротко, то вот что меня привлекло завести блог на Github Pages:

- отсутствие желания заниматься самому настройкой сервера, следить за обновлениями, безопасностью и все такое. На Github Pages это происходит без моего участия;
- набор статических html-страниц, а значит быстро будет отображаться в браузере;
- есть генератор этих самых статических страниц &mdash; [Jekyll](https://jekyllrb.com/), а значит не надо будет самому заниматься версткой постов;
- раз статические страницы, то не надо никаких баз данных, php и тому подобных вещей;
- посты пишутся в простом [markdown](https://en.wikipedia.org/wiki/Markdown)-формате, а значит подойдет тем кто не хочет вообще заморачиваться с html;
- git:smile:;  это значит многое: версионность, локальное хранение, безопасность(шифрованные данные же пересылаем :smile: ) и еще очень много хорошего;
- возможность привязать к своему домену блог, бесплатный HTTPS в том числе;
- возможность кастомизации по самое не хочу ... для тех кто хочет:wink:.

## Установка и настройка окружения  {#setup}

Вместо чтения кучи документации по Jekyll, я нашел крутую репу [poole](https://github.com/poole/poole). Как пишет ее автор:

> "a clear and concise foundational setup for any Jekyll site. And it has a super minimal look... It does so by furnishing a full vanilla Jekyll install with example templates, pages, posts, and styles."

Т.е. минимально необходимый набор для запуска блога: форкнул, поменял настройки под себя и пользуешься на здоровье. Правило "Все уже написано до нас!" работает безотказно :) .

[demo.getpoole.com](https://demo.getpoole.com/) &mdash; рабочее демо блога:

![The demo pool website](https://f.cloud.github.com/assets/98681/1834359/71ae4048-73db-11e3-9a3c-df38eb170537.png)

#### Приступим к созданию своего блога:

- клонируем репозиторий [poole](https://github.com/poole/poole) (или любую другую Jekyll-тему);
- кладем в корень `Gemfile`:

{% highlight ruby %}
source "https://rubygems.org"

gem "github-pages", group: :jekyll_plugins
gem "webrick"   # с Ruby 3.0 не входит в стандартную библиотеку, без него jekyll serve не стартует
{% endhighlight %}

  Гем `github-pages` &mdash; это ровно то окружение, которым Github собирает сайт: та же версия Jekyll и тот же набор плагинов. Важно не ставить просто последний Jekyll: Github до сих пор собирает на Jekyll 3.x (на ноябрь 2026 &mdash; 3.10), а последний &mdash; 4.x. Если случайно задействовать новую фичу, локально все будет работать, а на Github &mdash; нет. Текущие версии [здесь](https://pages.github.com/versions/);

- запускаем сайт локально. Проще всего через Docker, тогда Ruby вообще ставить не надо:

{% highlight bash %}
docker run -ti --rm -v "$PWD":/srv/jekyll -v blog-gems:/usr/local/bundle \
  -w /srv/jekyll -p 4000:4000 ruby:3.3 \
  bash -c 'bundle install && bundle exec jekyll serve --host 0.0.0.0 --force_polling'
{% endhighlight %}

  Volume `blog-gems` кэширует установленные гемы между запусками, `--force_polling` нужен, чтобы Jekyll замечал изменения файлов в примонтированной папке (на macOS без него сайт не перегенерируется). Раньше я пользовался образом `starefossen/github-pages`, но он давно заброшен.

  Без Docker: ставим Ruby (на Windows &mdash; [RubyInstaller](https://rubyinstaller.org/) в варианте *Ruby+Devkit*), затем `bundle install` и `bundle exec jekyll serve`;

- после запуска сайт доступен по адресу <http://localhost:4000>. Jekyll создает папку `_site`, в которую записывает сгенерированный блог (ее стоит добавить в `.gitignore`). Каждый файл в репозитории будет скопирован внутрь папки, за исключением тех файлов/папок, которые начинаются с подчеркивания. Markdown-файлы будут автоматически конвертированы в соотв. страницы. В папке `_posts` должны находится посты в markdown-формате. При чем имя файла должно быть [вида](https://jekyllrb.com/docs/posts/) `год-месяц-день-название_поста.md`. Иначе для него не сгенерируется соответствующая html-страница. Черновики можно держать в папке `_drafts` и смотреть локально с флагом `--drafts`. `_config.yml` &mdash; конфигурационный файл Jekyll, меняем его под свои надобности, подробно почитать о настройке [здесь](https://jekyllrb.com/docs/configuration/). **Важное примечание:** по умолчанию Jekyll при любом изменении файлов перезапишет сайт, но если вы поменяли что-то в `_config.yml`, то надо перезапустить Jekyll;
- добавляем посты, меняем шаблоны и вообще делаем что желаем;
- создаем на Github новый репозиторий с именем вида `github_username.github.io`;
- заливаем/пушим в него наш блог;
- в *Settings → Pages* проверяем, что источник &mdash; *Deploy from a branch* и выбрана нужная ветка. Сборка идет минуту-две, ее ход и ошибки видны во вкладке *Actions* (`pages-build-deployment`). После этого блог доступен по адресу <https://github_username.github.io>;
- если есть свой домен, то Github легко позволяет привязать его к блогу: [руководство](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site) от Github. Там же включается *Enforce HTTPS* &mdash; сертификат Github выпускает сам.

## Кастомизация     {#customizations}

В принципе дальнейшие шаги уже не обязательны, если вы дошли сюда, то скорее всего у вас уже успешно работает блог. В этой секции я описываю изменения, которые я захотел сделать.

Дефолтная тема poole мне подошла практически идеально. Но захотелось изменить еще некоторые вещи.

Добавил три ссылки вверху:

- [О блоге](/about/)
- [Архив записей](/archive/) - динамически формирующийся список записей, для быстрой навигации;
- [Лента](/atom.xml) - для желающих подписаться на обновления блога.

Для этого я добавил в `_config.yml`:

{% highlight yaml %}
pages_list:
  О блоге: '/about'
  Архив записей: '/archive'
  Лента: '/atom.xml'
{% endhighlight %}

И модифицировал шаблон [_layouts/default.html](https://github.com/alexprivalov/alexprivalov.github.io/blob/master/_layouts/default.html):

{% highlight html %}
{% raw %}
<h3 class="masthead-title">
    <a href="/" title="Home">{{ site.title }}</a>

    {% for page in site.pages_list %}
      &nbsp;&nbsp;&nbsp;
      <small><a href="{{ page[1]  }}">{{ page[0] }}</a></small>
    {% endfor %}

    <p> <small>{{ site.tagline }}</small> </p>
</h3>
{% endraw %}
{% endhighlight %}

Еще оказалось что с нуля нет поддержки тегов, что есть не классно:open_mouth:. Для Jekyll есть отдельный плагин, который реализует теги, но в нашем случае это не подходит: Github собирает сайт только с плагинами из [белого списка](https://pages.github.com/versions/) (`jemoji`, `jekyll-paginate`, `jekyll-feed` и т.п.), а плагина для тегов там нет. Подключаются они в `_config.yml` в секции `plugins:` (раньше она называлась `gems:`). Памятуя о правиле "Все написано до нас", я полез в Google и спустя 5-ть минут [решение](http://web.archive.org/web/20160130131036/http://cdlaap.github.io/2014/06/29/add-tags-page-to-jekyll-blog.html) найдено. Расписывать много не хочется, решение полностью скопировано, для наглядности будет удобно посмотреть [diff](https://github.com/alexprivalov/alexprivalov.github.io/commit/89dae334c91831def6577744d1cb38599e9a174b). А именно файлы [poole.css](https://github.com/alexprivalov/alexprivalov.github.io/blob/master/public/css/poole.css), [index.html](https://github.com/alexprivalov/alexprivalov.github.io/blob/master/index.html), [post.html](https://github.com/alexprivalov/alexprivalov.github.io/blob/master/_layouts/post.html) и [tags.html](https://github.com/alexprivalov/alexprivalov.github.io/blob/master/tags.html).

Честно говоря я ни грамма не понял, что я за магию сделал, для того что бы теги заработали(что взять с С++ разработчика?:stuck_out_tongue_winking_eye:), но все получилось и работает так как мне нужно. Вот, что <s>крест животворящий</s> свободный софт делает!:smile:

Если белого списка мало, сегодня есть выход: собирать сайт своим [GitHub Actions workflow](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages) (в *Settings → Pages* источник *GitHub Actions*). Тогда доступны любые плагины и Jekyll 4, но и за версиями следить придется самому.

## Итоги    {#summaries}

Я получил личный блог/сайт, не особо окунаясь в мир веба, могу писать разной степени полезности посты и продолжать программировать на своих любимых С/С++.

## Ссылки   {#refs}

- официальный сайт [Jekyll](https://jekyllrb.com/) с очень хорошей [документацией](https://jekyllrb.com/docs/);
- документация [Github Pages](https://docs.github.com/en/pages) и [про Jekyll на Github Pages](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll);
- [poole](https://github.com/poole/poole) репозиторий;
- [руководство](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax) по markdown;
- [пост](http://web.archive.org/web/20140918184211/http://joshualande.com/jekyll-github-pages-poole/) чувака, на который я ориентировался при настройке блога (копия в Web Archive);
- [пост](https://sylvaindurand.org/making-jekyll-multilingual/) чувака для настройки мультилингвальности блога;
- [настройка](http://davidensinger.com/2013/03/setting-the-dns-for-github-pages-on-namecheap/) DNS для Namecheap домена для  блога на Github;
- [пост](http://web.archive.org/web/20160130131036/http://cdlaap.github.io/2014/06/29/add-tags-page-to-jekyll-blog.html) чувака, который рассказывает как добавить поддержку тегов без плагина Jekyll (копия в Web Archive);
- [репозиторий](https://github.com/alexprivalov/alexprivalov.github.io) моего блога.
