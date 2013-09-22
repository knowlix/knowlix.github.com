---
layout: post
title: "Прицельный обзор: OAuth вне браузера через Scribe"
date: 2013-03-03 22:53
comments: true
categories: [Scribe, Sighting review]
---
<p><a href="https://github.com/fernandezpablo85/scribe-java">Scribe</a> - простая java библиотека, позволяющая проводить авторизацию на сервисах по OAuth протоколу. Библиотека интересна тем, что рекомендуется <a href="https://dev.twitter.com/docs/twitter-libraries#java">twitter&#8217;ом</a> и <a href="https://developer.linkedin.com/documents/libraries-and-tools">linkedin&#8217;ом</a> для работы с их реализациями OAuth аутентификации.</p>

<!--more-->


<p>Библиотека хостится на Github’е, а в качестве CI используется <em>Travis-CI</em>. Кто не знает, <a href="https://github.com/travis-ci/travis-ci">Travis-CI</a> - распределенная система сборки проекта, тесно интегрирующаяся с Github репозиторием.</p>

<p>Build Tool - Maven. Проект реализован в одном модуле.</p>

<p>Проект ведет рядовой разработчик аутсорсинговой компании в южной америке, которая занимается некоторыми задачами разработки LinkedIn’а.</p>

<p>Библиотека удобно оборачивает OAuth протокол для авторизации из java приложения, предлагая весомый набор предустановленных OAuth провайдеров, таких как Twitter, Google, Yahoo. В комплекте идет пакет examples, где показано как работать с каждым из провайдеров.</p>

<p>Все начинается с создания OAuthService объекта:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>OAuthService service = new ServiceBuilder()
</span><span class='line'>       .provider(TwitterApi.class)
</span><span class='line'>             .apiKey("6icbcAXyZx67r8uTAUM5Qw")
</span><span class='line'>             .apiSecret("SCCAdUUc6LXxiazxH3N0QfpNUvlUy84mZ2XZKiv39s")
</span><span class='line'>             .build();</span></code></pre></td></tr></table></div></figure>


<p>Классический Builder-подход, где задается <code>OAuthProvider</code> и данные для аутентификации. Отдельный провайдер отвечает за отдельный сервис. В данном случае - это Twitter. Провайдер ответственен за создание <code>OAuthService’а</code>, который несет в себе специфичную для конкретного сервиса информацию - url, версия OAuth и т.д.</p>

<p>Разработчик грамотно организовал дерево классов. Все провайдеры унаследованы от интерфейса <code>Api</code>, в котором описана сигнатура единственного метода <code>createService</code>. На следующем уровне этот интерфейс расширяют абстрактные классы, специфичные для разных версий OAuth (реализованы обе версии). Конкретные провайдеры наследуют класс, реализующий интерфейс <code>Api</code> с привязкой к версии OAuth протокола  и определяют внутри себя URL для доступа к сервису:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
<span class='line-number'>11</span>
<span class='line-number'>12</span>
<span class='line-number'>13</span>
<span class='line-number'>14</span>
<span class='line-number'>15</span>
<span class='line-number'>16</span>
<span class='line-number'>17</span>
<span class='line-number'>18</span>
<span class='line-number'>19</span>
<span class='line-number'>20</span>
<span class='line-number'>21</span>
<span class='line-number'>22</span>
<span class='line-number'>23</span>
<span class='line-number'>24</span>
<span class='line-number'>25</span>
<span class='line-number'>26</span>
<span class='line-number'>27</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>public class FacebookApi extends DefaultApi20
</span><span class='line'>{
</span><span class='line'>  private static final String AUTHORIZE_URL = "https://www.facebook.com/dialog/oauth?client_id=%s&redirect_uri=%s";
</span><span class='line'>  private static final String SCOPED_AUTHORIZE_URL = AUTHORIZE_URL + "&scope=%s";
</span><span class='line'>
</span><span class='line'>  @Override
</span><span class='line'>  public String getAccessTokenEndpoint()
</span><span class='line'>  {
</span><span class='line'>  return "https://graph.facebook.com/oauth/access_token";
</span><span class='line'>  }
</span><span class='line'>
</span><span class='line'>  @Override
</span><span class='line'>  public String getAuthorizationUrl(OAuthConfig config)
</span><span class='line'>  {
</span><span class='line'>  Preconditions.checkValidUrl(config.getCallback(), "Must provide a valid url as callback. Facebook does not support OOB");
</span><span class='line'>
</span><span class='line'>  // Append scope if present
</span><span class='line'>  if(config.hasScope())
</span><span class='line'>  {
</span><span class='line'>  return String.format(SCOPED_AUTHORIZE_URL, config.getApiKey(), OAuthEncoder.encode(config.getCallback()), OAuthEncoder.encode(config.getScope()));
</span><span class='line'>  }
</span><span class='line'>  else
</span><span class='line'>  {
</span><span class='line'>      return String.format(AUTHORIZE_URL, config.getApiKey(), OAuthEncoder.encode(config.getCallback()));
</span><span class='line'>  }
</span><span class='line'>  }
</span><span class='line'>}</span></code></pre></td></tr></table></div></figure>


<p>Версия OAuth определяет реализацию <code>OAuthService</code> класса, тип запроса (GET, POST) и другие специфичные для версии вещи, например формат данных.</p>

<p>Внутри метода <code>build</code> вызывается метод <code>createService</code> провайдера. После получения <code>OAuthService’а</code> действуем согласно алгоритму работы по OAuth протоколу. Рассмотрим первую версию OAuth. Сначала получем <em>Request Token</em>:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
<span class='line-number'>11</span>
<span class='line-number'>12</span>
<span class='line-number'>13</span>
<span class='line-number'>14</span>
<span class='line-number'>15</span>
<span class='line-number'>16</span>
<span class='line-number'>17</span>
<span class='line-number'>18</span>
<span class='line-number'>19</span>
<span class='line-number'>20</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>Token requestToken = service.getRequestToken();
</span><span class='line'>
</span><span class='line'>  public Token getRequestToken()
</span><span class='line'>  {
</span><span class='line'>  config.log("obtaining request token from " + api.getRequestTokenEndpoint());
</span><span class='line'>  OAuthRequest request = new OAuthRequest(api.getRequestTokenVerb(), api.getRequestTokenEndpoint());
</span><span class='line'>
</span><span class='line'>  config.log("setting oauth_callback to " + config.getCallback());
</span><span class='line'>  request.addOAuthParameter(OAuthConstants.CALLBACK, config.getCallback());
</span><span class='line'>  addOAuthParams(request, OAuthConstants.EMPTY_TOKEN);
</span><span class='line'>  appendSignature(request);
</span><span class='line'>
</span><span class='line'>  config.log("sending request...");
</span><span class='line'>  Response response = request.send();
</span><span class='line'>  String body = response.getBody();
</span><span class='line'>
</span><span class='line'>  config.log("response status code: " + response.getCode());
</span><span class='line'>  config.log("response body: " + body);
</span><span class='line'>  return api.getRequestTokenExtractor().extract(body);
</span><span class='line'>  }</span></code></pre></td></tr></table></div></figure>


<p>Внутри метода формируется OAuth запрос, данные для которого берутся из Api объекта  провайдера и этот запрос отправляется на сервер. Вся работа с сетью строится на базе стандартного пакета <code>java.net</code>. Таким образом при отправке запроса(<code>request.send()</code>) сначала открывается соединение:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>private void createConnection() throws IOException
</span><span class='line'>  {
</span><span class='line'>  String completeUrl = getCompleteUrl();
</span><span class='line'>  if (connection == null)
</span><span class='line'>  {
</span><span class='line'>      System.setProperty("http.keepAlive", connectionKeepAlive ? "true" : "false");
</span><span class='line'>      connection = (HttpURLConnection) new URL(completeUrl).openConnection();
</span><span class='line'>  }
</span><span class='line'>  }</span></code></pre></td></tr></table></div></figure>


<p>Далее добавляются <code>header'ы</code> в запрос и пишется тело сообщения:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
<span class='line-number'>11</span>
<span class='line-number'>12</span>
<span class='line-number'>13</span>
<span class='line-number'>14</span>
<span class='line-number'>15</span>
<span class='line-number'>16</span>
<span class='line-number'>17</span>
<span class='line-number'>18</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>void addHeaders(HttpURLConnection conn)
</span><span class='line'>  {
</span><span class='line'>  for (String key : headers.keySet())
</span><span class='line'>      conn.setRequestProperty(key, headers.get(key));
</span><span class='line'>  }
</span><span class='line'>
</span><span class='line'>  void addBody(HttpURLConnection conn, byte[] content) throws IOException
</span><span class='line'>  {
</span><span class='line'>  conn.setRequestProperty(CONTENT_LENGTH, String.valueOf(content.length));
</span><span class='line'>
</span><span class='line'>  // Set default content type if none is set.
</span><span class='line'>  if (conn.getRequestProperty(CONTENT_TYPE) == null)
</span><span class='line'>  {
</span><span class='line'>      conn.setRequestProperty(CONTENT_TYPE, DEFAULT_CONTENT_TYPE);
</span><span class='line'>  }
</span><span class='line'>  conn.setDoOutput(true);
</span><span class='line'>  conn.getOutputStream().write(content);
</span><span class='line'>  }</span></code></pre></td></tr></table></div></figure>


<p>В результате возвращается объект ответа(<code>Response</code>), который в конструкторе принимает данное соединение:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
<span class='line-number'>11</span>
<span class='line-number'>12</span>
<span class='line-number'>13</span>
<span class='line-number'>14</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>Response(HttpURLConnection connection) throws IOException
</span><span class='line'>  {
</span><span class='line'>  try
</span><span class='line'>  {
</span><span class='line'>      connection.connect();
</span><span class='line'>      code = connection.getResponseCode();
</span><span class='line'>      headers = parseHeaders(connection);
</span><span class='line'>      stream = isSuccessful() ? connection.getInputStream() : connection.getErrorStream();
</span><span class='line'>  }
</span><span class='line'>  catch (UnknownHostException e)
</span><span class='line'>  {
</span><span class='line'>      throw new OAuthException("The IP address of a host could not be determined.", e);
</span><span class='line'>  }
</span><span class='line'>  }</span></code></pre></td></tr></table></div></figure>


<p>Как видно, только в момент инстацирования Response объекта произойдет подключение к удаленному ресурсу. Это ничто иное, как размывание логики приложения. Говоря об архитектурных агрехах, можно еще упоминуть следующий факт - объект запроса использует переменные на уровне класса, которые меняются внутри его публичных методов. В итоге, состояние объекта предсказать невозможно.</p>

<p>На выходе из метода создает объект <code>Token</code>, который заполняется данными ответа. Ответ парсится с помощью регулярных выражений. Даже JSON ответ обрабатывается регулярными выражениями. Объект парсера специфичен для конкретной версии OAuth и хранится в провайдере.</p>

<p>Далее создается объект Verifier, который в конструкторе принимает код подтверждения, выдаваемый сервисом:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>Scanner in = new Scanner(System.in);</span></code></pre></td></tr></table></div></figure>


<p>В рамках примера используется командная строка, куда необходимо ввести этот код. Далее вызывается запрос на получение Access Token’а:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>Token accessToken = service.getAccessToken(requestToken, verifier);</span></code></pre></td></tr></table></div></figure>


<p>Принцип работы аналогичен отправки запроса на Request Token. Отличие лишь в данных, которые пересылаются сервису, в  url получателя и в парсере ответа, который строит объект Token’а:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
<span class='line-number'>11</span>
<span class='line-number'>12</span>
<span class='line-number'>13</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>public Token getAccessToken(Token requestToken, Verifier verifier)
</span><span class='line'>  {
</span><span class='line'>  config.log("obtaining access token from " + api.getAccessTokenEndpoint());
</span><span class='line'>  OAuthRequest request = new OAuthRequest(api.getAccessTokenVerb(), api.getAccessTokenEndpoint());
</span><span class='line'>  request.addOAuthParameter(OAuthConstants.TOKEN, requestToken.getToken());
</span><span class='line'>  request.addOAuthParameter(OAuthConstants.VERIFIER, verifier.getValue());
</span><span class='line'>
</span><span class='line'>  config.log("setting token to: " + requestToken + " and verifier to: " + verifier);
</span><span class='line'>  addOAuthParams(request, requestToken);
</span><span class='line'>  appendSignature(request);
</span><span class='line'>  Response response = request.send();
</span><span class='line'>  return api.getAccessTokenExtractor().extract(response.getBody());
</span><span class='line'>  }</span></code></pre></td></tr></table></div></figure>


<p>Далее можно осуществлять запросы к защищенному ресурсу сервиса с Token’ом доступа:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>OAuthRequest request = new OAuthRequest(Verb.GET, PROTECTED_RESOURCE_URL);
</span><span class='line'>service.signRequest(accessToken, request);
</span><span class='line'>request.addHeader("GData-Version", "3.0");
</span><span class='line'>Response response = request.send();</span></code></pre></td></tr></table></div></figure>


<p>Где метод <code>signRequest</code> добавляет в header запроса токен:</p>

<figure class='code'><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
<span class='line-number'>11</span>
<span class='line-number'>12</span>
<span class='line-number'>13</span>
</pre></td><td class='code'><pre><code class=''><span class='line'>public void signRequest(Token token, OAuthRequest request)
</span><span class='line'>  {
</span><span class='line'>  config.log("signing request: " + request.getCompleteUrl());
</span><span class='line'>
</span><span class='line'>  // Do not append the token if empty. This is for two legged OAuth calls.
</span><span class='line'>  if (!token.isEmpty())
</span><span class='line'>  {
</span><span class='line'>      request.addOAuthParameter(OAuthConstants.TOKEN, token.getToken());
</span><span class='line'>  }
</span><span class='line'>  config.log("setting token to: " + token);
</span><span class='line'>  addOAuthParams(request, token);
</span><span class='line'>  appendSignature(request);
</span><span class='line'>  }</span></code></pre></td></tr></table></div></figure>


<p>Ну вот и все. Интересно было читать код Scribe. Конечно пришлось пару раз вспомнить заветы товарища Фаулера, но куда без этого?</p>