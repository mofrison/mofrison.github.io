---
title:  "Unity3d: Загрузка ресурсов из Сети"
repository: Unity3d-Network
preview: /assets/images/posts/2021-04-04-unity3d-download-resources/preview.jpg
date:   2021-04-04 10:00:00 +0300
categories: ru cases
tags: [C#, Unity3d]
lang: Ru
layout: post
---

Поговорим о загрузке удалённых ресурсов в **Unity3d** приложение. В сети много материалов на эту тему, но во многих из них информация либо уже устарела, либо имеет существенные недостатки, речь о которых пойдёт в этой статье.

## Предыстория
Работая над VR-проектом **PanoramicView**, я поставил себе задачу сделать загрузку контента для этого приложения максимально гибким и удобным. Основной концепцией была возможность добавления новых локаций с панорамными видами без обновления всего приложения. В **Unity3d** есть класс `UnityWebRequest` который помогает загружать разлиные типы ресурсов из сети интернет.

Помимо изображений, звуков и видео мне требовалось загружать и позиционировать интерактивные элементы для осуществления переходов между локациями. Это можно было сделать с помощью процедурной генерации, загружая json-файл с координатами переходов и идентефикатором новой локации. Но зачем делать новую структуру данных, когда всё это есть в сценах в максимально наглядном виде. Поэтому я стал искать возможность загрузки сцен, и как оказалась такая возможность была. Для этого служат **[AssetBundles](https://docs.unity3d.com/ru/current/Manual/AssetBundlesIntro.html)**, с их помощью можно загружать множество различных сущностей в **Unity3d** в том числе и целые сцены.

Разработчиками предусмотрена возможность загружать все ресурсы через AssetBundles, но это оказалось не эффективно так как на распаковку даже небольших изображений требовалось значительное время. Поэтому для загрузки медиаресурсов опртимальным оказалось пользоваться специализированными методами, предоставляемыми классом `UnityWebRequest`. Организовать загрузку ресурсов из внешних источников мне очень помогла статья _[Работа с внешними ресурсами в Unity3D](https://habr.com/ru/post/433366/)_, автор которой [Ichimitsu](https://habr.com/ru/users/Ichimitsu/). Он же подсказал мне, как стделать этот процесс максимально быстрым.

## Что не так с Coroutine
В стандартной документации предлагают вынести процесс загрузки в Coroutine:
```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;

public class MyBehaviour : MonoBehaviour
{
    void Start()
    {
        StartCoroutine(GetText());
    }

    IEnumerator GetText()
    {
        using (UnityWebRequest uwr = UnityWebRequestAssetBundle.GetAssetBundle("http://www.my-server.com/mybundle"))
        {
            yield return uwr.SendWebRequest();

            if (uwr.result != UnityWebRequest.Result.Success)
            {
                Debug.Log(uwr.error);
            }
            else
            {
                // Get downloaded asset bundle
                AssetBundle bundle = DownloadHandlerAssetBundle.GetContent(uwr);
            }
        }
    }
}
```
Это позволяет параллельно с осуществлением загрузки другие выполнять процессы, но ни для кого не секрет, что Coroutine исполняется в основном потоке. Соответственно, такая загрузка ресурсов нагружает основной поток, а чем больше нагружается поток тем медленнее становятся загрузки. Я заметил это, когда решил анимировать визуализацию прогресса скачивания изображений. Вторым существенным минусом Coroutine является то, что в них не работает конструкция `try`-`catch`-`finally`. Это усложняет обработку ошибок, которые при работе с сетью не редкость.

Для решения этих проблем на помощь придут _**асинхронные методы**_. В них я и вынес загрузку ресурсов из основного потока.

## Асинхронные методы C#
Асинхронный метод обладает следующими признаками:
* В заголовке метода используется модификатор `async`
* Метод содержит одно или несколько выражений `await`
* В качестве возвращаемого типа используется один из следующих: `void`, `Task`, `Task<T>`, `ValueTask<T>`

Асинхронный метод, как и обычный, может использовать любое количество параметров или не использовать их вообще. Однако асинхронный метод не может определять параметры с модификаторами `out` и `ref`.

Также стоит отметить, что слово `async`, которое указывается в определении метода, не делает автоматически метод асинхронным. Оно лишь указывает, что данный метод может содержать одно или несколько выражений `await`.

## Реализация
Для взаимодействия с удалённым сервером я решил создать статический класс `Network` в котором определил основные методы для загрузки различных типов ресурсов.

Название метода                                 | Реализуемая функция
------------------------------------------------|-------------------------
[SendWebRequest](#sendwebrequest)               | Используется для взаимодействия с сетью: отправки запросов и получения данных.
[GetSize](#getsize)                             | Применяется для того чтобы узнать размер файла во внешнем хранилище.
[GetText](#gettext)                             | Предназначен для скачивания файлов в текстовом формате.
[GetData](#getdata)                             | Позволяет получать из сети данные в виде массива `byte`.
[GetTexture](#gettexture)                       | Реализует скачивание изображений с возможностью сохранения их в кэш.
[GetAudioClip](#getaudioclip)                   | Обеспечивает скачивание аудиозаписи с возможностью сохранения её в кэш.
[GetVideoStream](#getvideostream)               | Предоставляет адрес видеофайла для его воспроизведения в виде потока.
[GetAssetBundle](#getassetbundle)               | Загружает заранее подготовленных файлов **AssetBundle**.
[GetAssetBundleVersion](#getassetbundleversion) | Определяет актуальную версию **AssetBundle**. 
[GetHash128](#gethash128)                       | Служит для того чтобы извлечь hash из манифеста **AssetBundle**.
[GetCachedPath](#getcachedpath)                 | Используется для получения пути к закэшированному файлу, если таковой имеется.

В методах для скачивания ресурсов с возможностью крэширования(кроме GetAssetBundle), испльзуются методы предоставляемые статическим классом **ResourceCache**.
**ResourceCache** помогает cохранять данные в памяти устройства и взаимодействовать с ними. Он подробно рассмотрен в статье [Unity3d: Хранение данных на устройстве]({{site.url}}/ru/cases/unity3d-caching-resources).

### SendWebRequest
Основным для класса `Network` является приватный асинхронный метод `WebRequest`, с помощью которого загружаются данные из сети. В качестве аргументов он принимает заранее подготовленный `UnityWebRequest`, `CancellationTokenSource`, `Action<float>`.
```csharp
private static async Task<UnityWebRequest> WebRequest(UnityWebRequest request, CancellationTokenSource cancelationToken = null, System.Action<float> progress = null)
{
    while (!Caching.ready)
    {
        if (cancelationToken != null && cancelationToken.IsCancellationRequested)
        {
            return null;
        }
        await Task.Yield();
    }

#pragma warning disable CS4014
    request.SendWebRequest();
#pragma warning restore CS4014

    while (!request.isDone)
    {
        if (cancelationToken != null && cancelationToken.IsCancellationRequested)
        {
            request.Abort();
            request.Dispose();
            return null;
        }
        else
        {
            progress?.Invoke(request.downloadProgress);
            await Task.Yield();
        }
    }

    progress?.Invoke(1f);
    return request;
}
```
Как можно заметить тело метода начинается с цикла ожидания готовности кэша. Далее идёт отправка самого запроса: `request.SendWebRequest()`. После чего выполняется цикл ожидающий выполнения этого самого запроса. В течении этого цикла, через `Action<float>` в вызывающий метод передаётся прогресс выполнения этой операции. И в завершении метода запрос возвращается в вызывавший метод.

Для того чтобы процесс ожидания кэша и выполнения запроса происходили асинхронно в циклы был добавлен вызов `await Task.Yield()`, к названию метода был добавлен модификатор `async`, а возвращаемый `UnityWebRequest` был обёрнут в `Task<>`. О `await Task.Yield()` я узнал из статьи [Async/await в Unity](https://habr.com/ru/company/otus/blog/506962/). Ранее вместо него я пользовался вызовом `await new WaitForEndOfFrame()` из плагина  **AsyncAwaitUtil**, но решил отказаться в целях сокращения внешних зависимостей.

Кроме того в обоих циклах проверяется флаг `cancelationToken.IsCancellationRequested`, через которого методу можно сообщить о необходимости прервать выполнение операций и вернуть `null`.

### GetSize
Метод используется для того чтобы узнать размер файла во внешнем хранилище. В качестве единственного аргумента принимает строку с url-адресом файла.
```csharp
public static async Task<long> GetSize(string url)
{
    UnityWebRequest request = await SendWebRequest(UnityWebRequest.Head(url));
    var contentLength = request.GetResponseHeader("Content-Length");
    if (long.TryParse(contentLength, out long returnValue))
    {
        return returnValue;
    }
    else
    {
        throw new Exception("[Netowrk] error: " + request.error + " " + url);
    }
}
```
Метод асинхронно отправляет запрос `UnityWebRequest` на получение заголовка и ожидает его выполнения. Как только запрос выполнен, из него с помощью метода `GetResponseHeader("Content-Length")` извлекается текст, который конвертируется в чило методом `long.TryParse`. Число возвращается вызвавшему методу в виде `Task<long>`.

### GetText
Метод позволяет скачивать файлы в текстовом формате. В качестве единственного аргумента принимает строку с url-адресом файла.
```csharp
public static async Task<string> GetText(string url)
{
    var uwr = await SendWebRequest(UnityWebRequest.Get(url));
    if (uwr != null && !uwr.isHttpError && !uwr.isNetworkError)
    {
        return uwr.downloadHandler.text;
    }
    else
    {
        throw new Exception("[Netowrk] error: " + uwr.error + " " + uwr.url);
    }
}
```
При вызове этого метода асинхронно отправляется стандартный запрос `UnityWebRequest` и ожидается его выполнение. После того как запрос выполнен из него извлекается текст и возвращается в виде строки.

### GetData
Метод `GetData` позволяет получать из сети данные в виде байтового массива. На вход он принимает url-адрес файла в виде строки, токен для прерывания загрузки и `Action`, для отображения прогресса загрузки.
```csharp
public static async Task<byte[]> GetData(string url, CancellationTokenSource cancelationToken, System.Action<float> progress = null)
{
    UnityWebRequest uwr = await SendWebRequest(UnityWebRequest.Get(url), cancelationToken, progress);
    if (uwr != null && !uwr.isHttpError && !uwr.isNetworkError)
    {
        return uwr.downloadHandler.data;
    }
    else
    {
        throw new Exception("[Netowrk] error: " + uwr.error + " " + uwr.uri);
    }
}
```
В методе осуществляется отправка запроса через метод `SendWebRequest` описанный ранее. Метод также является асинхронным и на выходе мы получаем данные в виде `Task` c массивом `byte`.

### GetTexture
Метод для получения изображения в виде `Texture2D`, принимает в качестве аргументов строку с url-адресом, `cancelationToken`, `Action` для отбражения прогресса и флаг опрделяющий, кэшировать загруженное изображение или нет.
```csharp
public static async Task<Texture2D> GetTexture(string url, CancellationTokenSource cancelationToken, System.Action<float> progress = null, bool caching = true)
{
    string path = await url.GetCachedPath();
    bool isCached = path.Contains("file://");
    UnityWebRequest request = UnityWebRequestTexture.GetTexture(path);

    UnityWebRequest uwr = await SendWebRequest(request, cancelationToken, isCached? null : progress);
    if (uwr != null && !uwr.isHttpError && !uwr.isNetworkError)
    {
        Texture2D texture = DownloadHandlerTexture.GetContent(uwr);
        texture.name = Path.GetFileName(uwr.url);
        if (caching && !isCached) 
        {
            try
            {
                ResourceCache.Caching(uwr.url, uwr.downloadHandler.data);
            }
            catch (System.Exception e)
            {
                Debug.LogWarning("[Netowrk] error: " + e.Message);
            }
        }
        return texture;
    }
    else
    {
        throw new Exception("[Netowrk] error: " + uwr.error + " " + uwr.uri);
    }
}
```
В этом методе для скачивания изображения используется специализированный запрос `UnityWebRequestTexture.GetTexture(url)`. Кроме того перед отправкой запроса, проверяется наличие этого файла в памяти устройства. Для этого применяется метод расширения `GetCachedPath(this string paht)`. Если файл с тем же именем и того же размера лежит по аналогичному локальному пути, url подменяется на путь к этому файлу. Метод возвращает объект типа `Texture2D`, обёрнутый в `Task`.

### GetAudioClip
Метод для получения аудиофайла в виде `AudioClip`, принимает в качестве аргументов строку с url-адресом, `cancelationToken`, `Action` для отбражения прогресса, флаг определяющий, кэшировать загруженный аудиофайл или нет, и `AudioType`, указывающий на формат аудиозаписи. Предпочтительный формат: **OGG**
```csharp
public static async Task<AudioClip> GetAudioClip(string url, CancellationTokenSource cancelationToken, System.Action<float> progress = null, bool caching = true, AudioType audioType = AudioType.OGGVORBIS)
{
    string path = await url.GetCachedPath();
    bool isCached = path.Contains("file://");
    UnityWebRequest request = UnityWebRequestMultimedia.GetAudioClip(path, audioType);
        
    UnityWebRequest uwr = await SendWebRequest(request, cancelationToken, isCached ? null : progress);
    if (uwr != null && !uwr.isHttpError && !uwr.isNetworkError)
    {
        AudioClip audioClip = DownloadHandlerAudioClip.GetContent(uwr);
        audioClip.name = Path.GetFileName(uwr.url);
        if (caching && !isCached)
        {
            try
            {
                ResourceCache.Caching(uwr.url, uwr.downloadHandler.data);
            }
            catch (System.Exception e)
            {
                Debug.LogWarning("[Netowrk] error: " + e.Message);
            }
        }
        return audioClip;
    }
    else
    {
        throw new Exception("[Netowrk] error: " + uwr.error + " " + uwr.uri);
    }
}
```
В этом методе для скачивания аудиофайла используется специализированный запрос `UnityWebRequestMultimedia.GetAudioClip(url, audioType)`. Кроме того перед отправкой запроса, проверяется наличие этого файла в памяти устройства. Для этого применяется метод расширения: `GetCachedPath(this string paht)`. Если файл с тем же именем и того же размера лежит по аналогичному локальному пути, url подменяется на путь к этому файлу.  Метод возвращает объект типа `AudioClip`, обёрнутый в `Task`.

### GetVideoStream
Метод предоставляет путь к видеофайлу на устройстве если он был ранее закэширован. Принимает в качестве аргументов строку с url-адресом, `cancelationToken`, `Action` для отображения прогресса, и  флаг опрделяющий, кэшировать загруженный аудиофайл или нет.
```csharp
private delegate void AsyncOperation();

public static async Task<string> GetVideoStream(string url, CancellationTokenSource cancelationToken, System.Action<float> progress = null, bool caching = true)
{
    string path = await url.GetCachedPath();
    if (!path.Contains("file://"))
    {
        AsyncOperation cachingVideo = async delegate {
            try
            {
                if (caching && ResourceCache.CheckFreeSpace(await GetSize(url)))
                {
                    ResourceCache.Caching(url, await GetData(url, cancelationToken, progress));
                }
            }
            catch (Exception e)
            {
                Debug.LogWarning("[Netowrk] error: " + e.Message);
            }
        };
        cachingVideo();
        return url;
    }
    else { return path; }
}
```
В методе сначала проверяется наличие этого файла в памяти устройства. Если файл с тем же именем и того же размера лежит по аналогичному локальному пути, возвращается путь к файлу на устройстве. Если такого файла нет - возвращается url-адрес файла и параллельно запускается процесс загрузки файла на устройство. Загрузка происходит с помощью метода `GetData` в заранее определённом делегате `AsyncOperation`, чтобы не ждать пока видео загрузится на устройство. Я пробовал запустить скачивание данных в отдельном `Task`, с помощью `Task.Run()`, но это не сработало. В целом данный подход загрузки и воспроизведения видео не претендует на самый эффективный, но придумать лучше мне пока не удалось. Адрес файла возвращается в виде строки, он применяется для воспроизведения потоковых видеозаписей с помощью `VideoPlayer`.
 
> **Примечание:** Чтобы видео файл успешно воспроизводился, он должен соответствовать формату потокового видео, а веб-сервер, к которому вы подключаетесь, должен соответствовать протоколу [**HLS**(HTTP Live Streaming)](https://ru.wikipedia.org/wiki/HLS)


### GetAssetBundle
Метод служит для загрузки заранее подготовленных файлов **AssetBundle**. Принимает в качестве аргументов строку с url-адресом, `cancelationToken`, `Action` для отображения прогресса, и  флаг опрделяющий, кэшировать загруженный аудиофайл или нет.
```csharp
public static async Task<AssetBundle> GetAssetBundle(string url, CancellationTokenSource cancelationToken, System.Action<float> progress = null, bool caching = true)
{
    UnityWebRequest request;
    CachedAssetBundle cachedAssetBundle = await GetCachedAssetBundle(url);
    if (Caching.IsVersionCached(cachedAssetBundle) || (caching && ResourceCache.CheckFreeSpace(await GetSize(url))))
    {
        request = UnityWebRequestAssetBundle.GetAssetBundle(url, cachedAssetBundle, 0);
    }
    else
    {
        request = UnityWebRequestAssetBundle.GetAssetBundle(url)
    }

    UnityWebRequest uwr = await SendWebRequest(request, cancelationToken, Caching.IsVersionCached(cachedAssetBundle)? null : progress);
    if (uwr != null && !uwr.isHttpError && !uwr.isNetworkError)
    {
        AssetBundle assetBundle = DownloadHandlerAssetBundle.GetContent(uwr);
            if (caching) 
            {
                // Deleting old versions from the cache
                Caching.ClearOtherCachedVersions(assetBundle.name, cachedAssetBundle.hash);
            }
        return assetBundle;
    }
    else
    {
        throw new Exception("[Netowrk] error: " + uwr.error + " " + uwr.uri);
    }
}
```
Сначала в методе проверяется наличие ранее закешированной версии **AssetBundle** с помощью вспомогательного метода `GetAssetBundleVersion(Uri uri)` предсатвленного ниже. Если находится закэшированная версия совпадающая с той, что хранится по указанной ссылке, то загружается она. Если нет совпадений, то загружается версия **AssetBundle** из сети. Кэшируется новая версия или нет, зависит от состояния флага `caching`. Так же от этого флага зависит, будут ли удалены ранее загруженные версии. Метод возвращает `AssetBundle` обёрнутый в `Task`.

### GetAssetBundleVersion
Вспомогательный метод, помогает определить актуальную версию **AssetBundle**. В качестве единственного аргумента принимает строку с url-адресом.
```csharp
private static async Task<CachedAssetBundle> GetAssetBundleVersion(string url)
{
    Hash128 hash = default;
    string localPath = new System.Uri(url).LocalPath;
    try
    {
        string manifest = await GetText(url + ".manifest");
        hash = GetHashFromManifest(manifest);
        return new CachedAssetBundle(localPath, hash);
    }
    catch (Exception e)
    {
        Debug.LogWarning("[Netowrk] error: " + e.Message);
        DirectoryInfo dir = new DirectoryInfo(url.ConvertToCachedPath());
        if (dir.Exists)
        {
            System.DateTime lastWriteTime = default;
            foreach (var item in dir.GetDirectories())
            {
                if (lastWriteTime < item.LastWriteTime)
                {
                    if (hash.isValid && hash != default) 
                    { 
                        Directory.Delete(Path.Combine(dir.FullName, hash.ToString()), true);
                    }
                    lastWriteTime = item.LastWriteTime;
                    hash = Hash128.Parse(item.Name);
                }
                else { Directory.Delete(Path.Combine(dir.FullName, item.Name), true); }
            }
            return new CachedAssetBundle(localPath, hash);
        }
        else
        {
            throw new Exception("[Netowrk] error: Nothing was found in the cache for " + url);
        }
    }
}
```
Прежде чем искать необходимую актуальную версию **AssetBundle** на устройстве, метод обращается к файлу манифеста **AssetBundle**, чтобы извлечь из него hash. Если hash успешно загружен то метод упаковывает его в структуру `CachedAssetBundle` и возвращает её. Если загрузить манифест не удалось, то метод ищет закэшированные версии на устройстве и возвращает hash запакованный в структуру `CachedAssetBundle` последней из них, если таковые имеются. 

Я выбрал структуру `CachedAssetBundle` так, как кроме hash она содержит путь к файлу **AssetBundle**, что уменьшает риск коллизий в случае совпадения имён файлов.

> **Внимание:** При кэшировании файлов с одинаковым именем и локальным адресом возникнет коллизия и будет закэширован только последний загруженный файл. Такая ситуация может возникнуть если в адресах загружаемых файлов отличаются только имена доменов.

### GetHash128
Вспомогательный метод, который служит для того чтобы извлечь hash из манифеста  **AssetBundle**.
В качестве единственного аргумента принимает строку с url-адресом файла манифеста.
```csharp
private static Hash128 GetHash128(this string str)
{
    var hashRow = str.Split("\n".ToCharArray())[5];
    var hash = Hash128.Parse(hashRow.Split(':')[1].Trim());
    if (hash.isValid && hash != default) { return hash; }
    else { throw new Exception("[Netowrk] error: couldn't extract hash from manifest."); }
}
```

### GetCachedPath
Метод для получения пути к закэшированному файлу, если таковой имеется.
В качестве единственного аргумента принимает строку с url-адресом файла.
```csharp
private static async Task<string> GetCachedPath(this string url)
{
    string path = url.ConvertToCachedPath();
    if (File.Exists(path))
    {
        try
        {
            if (new FileInfo(path).Length != await GetSize(url)) { return url; }
        }
        catch (Exception e)
        {
            Debug.LogWarning("[Netowrk] error: " + e.Message); 
        }
        return "file://" + path;
    }
    else return url;
}
```
При вызове метода url-адрес конвертируется в путь к файлу и если он там находится, то его размер сравнивается с размером файла во внешнем хранилище. Если размеры совпадают то метод возвращает путь к файлу в виде строки, если нет - возвращает utl-адрес.

Я создал отдельный репозиторий с [примерами](https://github.com/{{ site.github.owner_name }}/Unity3d-download-resources), чтобы вы могли их протестировать.
Вы также можете использовать Кэширование данных на устройстве_ for **Unity3d** в своем проекте, добавив [этот репозиторий](https://github.com/{{ site.github.owner_name }}/{{ page.repository }}) как [подмодуль](https://git-scm.com/book/en/v2/Git-Tools-Submodules):

    git submodule add https://github.com/mofrison/Unity3d-Network

Спасибо что дочитали до конца, надеюсь это Вам пригодится :)
