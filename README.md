# Задание 3

Мобилизация.Гифки – сервис для поиска гифок в перерывах между занятиями.

Сервис написан с использованием [bem-components](https://ru.bem.info/platform/libs/bem-components/5.0.0/).

Работа избранного в оффлайне реализована с помощью технологии [Service Worker](https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API/Using_Service_Workers).

Для поиска изображений используется [API сервиса Giphy](https://github.com/Giphy/GiphyAPI).

В браузерах, не поддерживающих сервис-воркеры, приложение так же должно корректно работать,
за исключением возможности работы в оффлайне.

## Структура проекта

  * `gifs.html` – точка входа
  * `assets` – статические файлы проекта
  * `vendor` –  статические файлы внешних библиотек
  * `service-worker.js` – скрипт сервис-воркера

Открывать `gifs.html` нужно с помощью локального веб-сервера – не как файл.
Это можно сделать с помощью встроенного в WebStorm/Idea веб-сервера, с помощью простого сервера
из состава PHP или Python. Можно воспользоваться и любым другим способом.

## Отладка

1. Сервисворкер не срабатывал потому, что находился в папке assets, а то, что нужно кэшировать - в корне и поэтому не попадало в его скоуп -> перенос файла service-worker.js решил это.

2. Есть промис в

```
self.addEventListener('activate', event => {
  self.addEventListener('activate', event => {
      const promise = deleteObsoleteCaches()
          .then(() => {
              self.clients.claim();

              console.log('[ServiceWorker] Activated!');
          });

      event.waitUntil(promise);
  });
```
заменила на
```
self.addEventListener('activate', event => {
    const promise = deleteObsoleteCaches()
        .then(() => {
            return self.clients.claim()
                .then(() => {
                  console.log('[ServiceWorker] Activated!');
                })
        });

    event.waitUntil(promise);
});
```
Теперь активация дожидается промиса (а waitUntil не дает браузеру убить сервисворкер)

3. Теперь в кэш добавляется избранное, но сама страница в режиме оффлайн все еще не работает.
```
function needStoreForOffline(cacheKey) {
    return cacheKey.includes('vendor/') ||
        cacheKey.includes('assets/') ||
        cacheKey.endsWith('jquery.min.js')
}
```
HTML документ не сохраняется в кэш, добавляю
```
function needStoreForOffline(cacheKey) {
    return cacheKey.includes('vendor/') ||
        cacheKey.includes('assets/') ||
        cacheKey.endsWith('jquery.min.js') ||
        cacheKey.endsWith('gifs.html')
}
```

4. Теперь сервисворкер работает, но не обновляет кэш

```
self.addEventListener('fetch', event => {
    const url = new URL(event.request.url);

    const cacheKey = url.origin + url.pathname;

    let response;
    if (needStoreForOffline(cacheKey)) {
        response = caches.match(cacheKey)
            .then(cacheResponse => cacheResponse || fetchAndPutToCache(cacheKey, event.request));
    } else {
        response = fetchWithFallbackToCache(event.request);
    }

    event.respondWith(response);
});
```

В условии данные сразу забираются из кэша. Убрала `cacheResponse` в начале, теперь они берутся из кэша только после неудачной попытки загрузки из сети.

### Итог
Самым продуктивным оказалось идти по этапам работы сервисворкера и проверять, как отрабатывает каждый. 
