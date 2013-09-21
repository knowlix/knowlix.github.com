---
layout: post
title: "Netty: Управление соединениями"
date: 2012-08-07 21:35
comments: true
categories: Netty
---
В прошлых статьях упоминалось, что Netty использует два разных пула потоков для организации соединений и чтения данных. Первый пул - Boss Pool, второй - Worker Pool. Как вы могли уже заметить в предыдущей статье, запуск boss потока выполняется в Sink объекте, внутри метода привязки открытого socket канала к конкретному адресу.

<!--more-->

Класс boss потока описан внутри Sink объекта, и новый поток запускается с помощью утилитарных классов и wrapper'ов:

{% codeblock %}
Executor bossExecutor =
	((NioServerSocketChannelFactory) channel.getFactory()).bossExecutor;
DeadLockProofWorker.start(bossExecutor,
		new ThreadRenamingRunnable(new Boss(channel),
				"New I/O server boss #" + id + " (" + channel + ')'));
{% endcodeblock %}

Netty именует потоки для ведения более человеко-понятных логов. Так обертка `ThreadRenamingRunnable` применяет заданное имя, когда поток запустится.

`Boss` - класс boss потока, который будет отслеживать новые соединения.

{% codeblock %}
Boss(NioServerSocketChannel channel) throws IOException {
	this.channel = channel;

	selector = Selector.open();

	boolean registered = false;
	try {
		channel.socket.register(selector, SelectionKey.OP_ACCEPT);
		registered = true;
	} finally {
		if (!registered) {
			closeSelector();
		}
	}

	channel.selector = selector;
}
{% endcodeblock %}

Во первых, Boss-поток связывается с *Netty* каналом. Также в конструкторе создается новый экземпляр Nio селектора. 

>Есть прекрасный класс в Nio `java.nio.channels.Selector`, который позволяет избежать создания большого числа потоков, следя за активностью каналов. Подписавшись на некое событие, можно получить ключи каналов, которые это действие совершили, и дальше уже работать с ними. Отслеживание активности каналов и возвращение только нужных берет на себя `Selector`. Таким образом, можно работать с несколькими каналами в одном потоке. Но в данном случае, `Seletor` используется немного для других целей и чуть позже мы увидим для каких именно. 

Но для начала, необходимо зарегистрировать Nio каналы в селекторе и указать тип активности, за которым необходимо следить. В нашем случае используется `SelectionKey.OP_ACCEPT` для наблюдения за соединениями. Mетод `socket.register` выбрасывает `IOException`, если канал уже закрыт. В данном случае, селектор нам не нужен и его нужно закрыть, но т.к. само исключение пробрасывается выше, используется отдельная переменная для проверки успешности регистрации.

{% codeblock %}
public void run() {
	final Thread currentThread = Thread.currentThread();

	channel.shutdownLock.lock();
	try {
		for (;;) {
			try {
				if (selector.select(1000) > 0) {
					selector.selectedKeys().clear();
				}

				for (;;) {
					SocketChannel acceptedSocket = channel.socket.accept();
					if (acceptedSocket == null) {
						break;
					}
					registerAcceptedChannel(acceptedSocket, currentThread);

				}

			} catch (SocketTimeoutException e) {
			} catch (CancelledKeyException e) {
			} catch (ClosedSelectorException e) {
			} catch (ClosedChannelException e) {
				break;
			} catch (Throwable e) {
				if (logger.isWarnEnabled()) {
					logger.warn(
							"Failed to accept a connection.", e);
				}

				try {
					Thread.sleep(1000);
				} catch (InterruptedException e1) {
				}
			}
		}
	} finally {
		channel.shutdownLock.unlock();
		closeSelector();
	}
}
{% endcodeblock %}

Основной метод потока вызывает блокировку, используя `ReentrantLock` канала. Таким образом, один канал будет иметь только один работающий Boss поток в одно и тоже время. Кроме того, `ReentrantLock` считается более эффективным решением в условиях жесткой конуренции и обладает некоторыми функциональными преимуществами по сравнению с synchronized блоками.

Как это нередко бывает в потоке используются бесконечные циклы для постоянного выполнения. Далее, с помощью селектора прослушиваются каналы на наличие новых соединений. Во время выполнения метода `selector.select` поток блокируется до тех пор пока не будет получено соединение, либо не будет превышен таймаут. В целях освобождения памяти, ключи, по которым можно было бы получить доступ к Nio каналу, очищаются.

После истечения секунды на ожидание соединения, либо в случае появления этих самых соединений начинает свое выполнение внутренний бесконечный цикл, который принимает все появившиеся соединения без каких-либо задержек. В случае, если селектор был прерван по таймауту и никаких соединений не поступило, то при первом вызове `channel.socket.accept()` будет возвращен null. Если же соединения присутствуют, то возвращается новый сокет канал, который отправляется в метод `registerAcceptedChannel` для создания нового worker потока, который будет следить за событиями чтения, записи внутри канала.

Стоит отметить, что в данном `Selector'е` всегда будет зарегистрирован только один boss канал, поэтому ключи, которые он возвращает Netty не интересуют. `Selector` в данном контексте - это оптимальный `sleep` метод, который может прерываться, когда появляется необходимость. Еще стоит обратить внимание, что исключения в основном игнорируются. Работа потока прекращается в случае закрытия канала. Если было выбрашено IOException или, возможно, какое-то Runtime исключение, то поток приостанавливается на секунду и далее продолжает свою работу в нормальном режиме.

В конце выполнения потока, блокировка снимается.

В следующей статье мы рассмотрим как обрабатываются события worker каналов.
