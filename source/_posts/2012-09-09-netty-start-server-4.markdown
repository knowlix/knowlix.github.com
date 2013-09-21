---
layout: post
title: "Netty 4.0: Запуск сервера"
date: 2012-09-09 21:52
comments: true
categories: "Netty 4.0"
---
Сегодня проведем тонкую грань между разными версиями *Netty* на примере запуска socket сервера. Ранее мы уже рассматривали [запуск сервера на базе Netty 3.5](http://queuepy.com/blog/2012/08/04/netty-start-server/). Сегодня рассмотрим, как этот процесс осуществляется в четвертой ветке *Netty*.

<!--more-->

В качестве отправной точки также, как и ранее будем использовать проект, идущий с исходниками ввиде примера Echo сервера. Стоит сразу сказать, что если в версии 3.5 все исходники делились логически с помощью пакетов, то в новой версии исходники разбиты на модули и все примеры использования Netty находятся в отдельном модуле `example`.

{% codeblock %}
ServerBootstrap b = new ServerBootstrap();
try {
	b.eventLoop(new NioEventLoop(), new NioEventLoop())
	 .channel(new NioServerSocketChannel())
	 .option(ChannelOption.SO_BACKLOG, 100)
	 .localAddress(new InetSocketAddress(port))
	 .childOption(ChannelOption.TCP_NODELAY, true)
	 .handler(new LoggingHandler(LogLevel.INFO))
	 .childHandler(new ChannelInitializer<SocketChannel>() {
		 @Override
		 public void initChannel(SocketChannel ch) throws Exception {
			 ch.pipeline().addLast(
					 new LoggingHandler(LogLevel.INFO),
					 new EchoServerHandler());
		 }
	 });

	ChannelFuture f = b.bind().sync();

	f.channel().closeFuture().sync();
} finally {
	b.shutdown();
}
{% endcodeblock %}

Как и в случае с 3ей версией все начинается с `ServerBootstrap` - объекта-конфигуратора *Netty* сервера. Принцип конфигурирования теперь реализован ввиде некого Builder-объекта, где каждый метод-конфигуратор возвращает ссылку на builder до тех пор пока не вызовится метод-строитель. При этом инстанцирование объекта `ServerBootstrap` отделено от его конфигурирования, т.к. есть только конструктор по-умолчанию (без параметров). Данный подход является более чистым, чем в третьей версии. Там при создании объекта `ServerBootstrap` необходимо задать фабрику каналов, при этом задание `PipelineFactory` вынесена в конфигурационный метод. Напомню, что запуск невозможен и без фабрики каналов и без фабрики `pipeline'ов`. Таким образом есть некая неразбериха. В новом же подходе все объекты задаются на одном уровне, при этом в методе-строителе произведется проверка на корректность сконфигурированных объектов.

