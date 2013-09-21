---
layout: post
title: "Mockito: Структура проекта"
date: 2012-06-23 22:24
comments: true
categories: Mockito
published: true
---

<p><a href="https://code.google.com/p/mockito/" rel="nofollow">Mockito</a> - библиотека для динамического создания mock-объектов и манипуляции ими. Разработка начата в 2007'ом году польским разработчиком Стивеном Фабером (Szczepan Faber). Первый релиз выпущен в 2008'ом. Последний в начале июня 2012'го. Официальный сайт находится на <em>Google Code</em>’e. Issue Tracker и Wiki там же, но в репозитории сохранились текстовые файлы с заметками и планами на будущее.</p>

<!--more-->

<p><em>VCS</em> - <em>Mercurial</em>.</p>

<p>В качестве CI используется <em>Jenkins</em>.</p>

<p>Основной build tool - <em>Ant</em>. Осуществляется переход на <em>Gradle</em>. Это логично, т.к. основной разработчик <em>Mockito</em> работает главным инженером в Gradleware. Из комментариев gradle-скрипта:</p>

<blockquote># В определенный момент мы полностью перейдем на Gradle. Сейчас же Mockito находится в состоянии миграции на новый build tool и тем самым показывает как Ant и Gradle могут сосуществовать (для сборки из Gradle-скрипта вызывается соответствующий target Ant-скрипта).</blockquote>

<p>В составе проекта идет утилита <em>gradle-wrapper (gradlew)</em>, которая позволяет собирать проект без предустановленного <em>Gradle</em>.</p>

<p>Для распространения через <em>Maven</em> репозиторий на версионный контроль добавлены заготовки pom скриптов, но использовать <em>Maven</em> в качестве инструмента для сборки проекта разработчики скорее всего не будут(о чем свидетельствуют комментарии разработчиков).</p>

<p>На версионный контроль поставлены файлы проектов <em>Eclipse</em> и <em>Intelij IDEA</em>, что очень хорошо для слабо задокументированного в области разработки Open Source проекта. В проекте используется Checkstyle, его конфигурация также есть на версионном контроле.</p>

<p>В скрипте сборке используются Ant-расширения:</p>
<ul class="enum">
	<li><a href="http://pmd.sourceforge.net/pmd-5.0.0/" title="PMD scans Java source code and looks for potential problems">PMD</a> - это аналог Findbug’a, выполненный в виде библиотеки. Используется для анализа кода на наличие потенциальных проблем.</li>
    
	<li><a href="http://code.google.com/p/jarjar/" title="Jar Jar Links is a utility that makes it easy to repackage Java libraries and embed them into your own distribution.">JarJar</a> - продвинутый аналог стандартной таски Jar. Основное отличие - перепаковка внешних jar’ок. Используется для сборки jar’ок mockito.</li>
	
    <li><a href="http://ant4hg.free.fr/" title="ANT4HG is an ANT task for HG (mercurial)">Ant4HG</a> - набор Task для работы с mercurial. Используется для деплоя изменений на версионный контроль и CI.</li>
	
    <li><a href="http://maven.apache.org/ant-tasks/index.html" title="The Mavent Ant Tasks allow several of Maven's artifact handling features to be used from within an Ant build">Maven Ant Tasks</a> - набор Task для выполнения pom-скриптов и работы с Maven репозиториями. Используется для деплоя релизов проекта в локальный и удаленный репозитории.</li>
</ul>
	
<p>Исходный код обильно покрыт тестами. Есть пакет с примером использования <em>Mockito</em>: <code>org.mockitousage.examples.use</code>. Это неплохая точка отсчета в изучении архитектуры проекта. К тому же во многих примерах JavaDoc документации <em>Mockito</em> используются объекты из этого пакета.</p>

<p>Непосредственно исходный код проекта разделен на две части. Внешнее API проекта (пакет <code>org.mockito</code>) и внутренняя реализация (пакет <code>org.mockito.internal</code>). Достаточно странный подход в разработке, т.к. разделение, в основном, условное и ничто не мешает им пренебречь ни разработчику <em>Mockito</em>, ни пользователю этой библиотеки.</p>

<p>На данном этапе стоит отметить продвинутый build-скрипт, хорошее покрытие unit-тестами и немного захламленную структуру проекта.</p>