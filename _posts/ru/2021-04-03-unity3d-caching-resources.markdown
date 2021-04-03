---
title:  "Unity3d: Хранение данных на устройстве"
repository: Unity3d-download-resources
preview: /assets/images/posts/2021-04-03-unity3d-caching-resources/preview.jpg
date:   2021-04-03 10:00:00 +0300
categories: ru cases
tags: [C#, unity3d]
lang: Ru
layout: post
---

В этой статье речь идёт о сохранении данных в постоянную память устройства и организация доступа к ним. Она призвана подготовить Вас, дорогой читатель, к более сложной статье про загрузку ресурсов из сети. По сути это её вводная часть, которая была выделена в отдельный пост, потому что тема оказалась очень объёмной.

## Предыстория
Работая над VR-проектом **PanoramicView**, я поставил себе задачу сделать загрузку контента для этого приложения максимально гибким и удобным. Основной концепцией была возможность добавления новых локаций с панорамными видами без обновления всего приложения. В **Unity3d** есть класс `UnityWebRequest` который помогает загружать различные типы ресурсов из сети интернет.

Помимо изображений, звуков и видео мне требовалось загружать и позиционировать интерактивные элементы для осуществления переходов между локациями. Это можно было сделать с помощью процедурной генерации, загружая json-файл с координатами переходов и идентефикатором новой локации. Но зачем делать новую структуру данных, когда всё это есть в сценах в максимально наглядном виде. Поэтому я стал искать возможность загрузки сцен, и как оказалось такая возможность была. Для этого служат [**AssetBundles**](https://docs.unity3d.com/ru/current/Manual/AssetBundlesIntro.html), с их помощью можно загружать множество различных сущностей в **Unity3d** в том числе и целые сцены.

Разработчиками предусмотрена возможность загружать все ресурсы через **AssetBundle**, но это оказалось неэффективно так как на распаковку даже небольших изображений требовалось значительное время. Поэтому для загрузки медиаресурсов оптимальным оказалось пользоваться специализированными методами, предоставляемыми классом `UnityWebRequest`. Организовать загрузку ресурсов из внешних источников мне очень помогла статья [_Работа с внешними ресурсами в Unity3D_](https://habr.com/ru/post/433366/). Автор которой [Ichimitsu](https://habr.com/ru/users/Ichimitsu/) подсказал мне, как сделать этот процесс максимально быстрым, за что ему огромное спасибо!

## Зачем хранить ресурсы в памяти устройства
Думаю многие разработчики прежде всего мобильных приложений сталкивались с тем, что ресурсы не помещаются в установочные файлы приложения. Особенно актуально это для тех кто использует фотореалистичный контент(панорамы 360). Небольшим подспорьем в этом выглядит использование файлов расширения, но они тоже ограничены в размере. Кроме того загрузка дополнительного контента требует обновления приложения через магазин.

Можно загружать из сети все необходимые ресурсы при каждом запуске, но несмотря на стремительное развитие интернета, это всё ещё довольно длительный, ресурсозатратны и ненадёжный процесс. Поэтому рационально будет ресурсы загружаемые из сети кэшировать в память устройства и при необходимости обращаться к ним.

## Настройка кэширования
Прежде чем что то сохранять надо настроить путь для хранения данных. Для этого я определил метод `ConfiguringCaching`:
```csharp
private static string cachingDirectory = "data";

public static void ConfiguringCaching(string directoryName)
{
    cachingDirectory = directoryName;
    var path = Path.Combine(Application.persistentDataPath, cachingDirectory);
    if (!Directory.Exists(path))
    {
        Directory.CreateDirectory(path);
    }
    UnityEngine.Caching.currentCacheForWriting = UnityEngine.Caching.AddCache(path);
}
```
Метод на вход получает имя директории в которую мы будем сохранять загруженные данные. По умолчанию я назвал эту папку _data_. Полный путь к этой директории формируется с помощью [Application.persistentDataPath](https://docs.unity3d.com/ScriptReference/Application-persistentDataPath.html). Это значение представляет собой путь к каталогу, в котором можно сохранять данные, между запусками. При публикации на iOS и Android persistentDataPath указывает на общедоступный каталог на устройстве. Файлы в этом месте не удаляются обновлениями приложений. Файлы по-прежнему могут быть удалены непосредственно пользователями.

Так же в этом методе путь к папке добавляется в `Caching.currentCacheForWriting`, для того чтобы настроить кэширование **AssetBundles**. **AssetBundles** обладает своими механизмами кэширования и нужно лишь правильно их использовать.

### Наличие свободного места в памяти устройства
Прежде чем сохранять данные в памяти устройства, необходимо удостовериться в том, что данные на него поместятся. Для этого предварительно в проект был добавлен плагин **SimpleDiskUtils**. В нём есть библиотеки для разных ОС, с помощью которых можно выяснить сколько свободного места в памяти есть на устройстве. В следующем методе размер загружаемого файла сравнивается со свободным местом оставшимся на устройстве. В качестве аргумента метод принимает размер файла в байтах.
```csharp
private const float MIB = 1048576f;

public static bool CheckFreeSpace(float sizeInBytes)
        {
#if UNITY_EDITOR_WIN
            var logicalDrive = Path.GetPathRoot(Application.persistentDataPath);
            var availableSpace = SimpleDiskUtils.DiskUtils.CheckAvailableSpace(logicalDrive);
#elif UNITY_EDITOR_OSX
        var availableSpace = SimpleDiskUtils.DiskUtils.CheckAvailableSpace();
#elif UNITY_IOS
        var availableSpace = SimpleDiskUtils.DiskUtils.CheckAvailableSpace();
#elif UNITY_ANDROID
        var availableSpace = SimpleDiskUtils.DiskUtils.CheckAvailableSpace(true);
#endif
            return availableSpace > sizeInBytes / MIB;
        }
```
При генерации **AssetBundle** могут возникнуть проблемы с обращением к разным методам несмотря на наличие директивы `#if-#else`, чтобы этого избежать лишние вызовы можно закомментировать или удалить.

### Сохранение данных
Сохранение данных в постоянную память устройства реализует метод `Caching`. На вход он принимает url-адрес файла в виде строки и массив типа `byte` с данными.
```csharp
public static void Caching(string url, byte[] data)
{
    if (CheckFreeSpace(data.Length))
    {
        string path = url.ConvertToCachedPath();

        DirectoryInfo dirInfo = new DirectoryInfo(Application.persistentDataPath);
        if (!dirInfo.Exists)
        {
            dirInfo.Create();
        }
        dirInfo.CreateSubdirectory(Directory.GetParent(path).FullName);
        File.WriteAllBytes(path, data);
    }
    else { throw new Exception("[Caching] error: Not available space to download " + data.Length / MIB + "Mb"); }
}
```
Прежде всего метод проверяет, есть ли место для представленного массива с данными на устройстве. Если место есть, url-адрес конвертируется в путь файлу на устройстве с помощью метода `ConvertToCachedPath(string url)`, который мы рассмотрим далее. После чего создаются необходимые репозитории и файл сохраняется по предоставленному пути, с помощью стандартного метода `File.WriteAllBytes(string path, byte[] data)`. Если места нет - генерируется исключение.

### Получение пути к сохранённому файлу
Для того чтобы сохранить файл и потом к нему обращаться необходимо получить путь к нему в памяти устройства. Для этого служит метод `ConvertToCachedPath` В качестве единственного аргумента принимает строку с url-адресом файла.
```csharp
public static string ConvertToCachedPath(this string url)
{
    try
    {
        if (!string.IsNullOrEmpty(url))
        {
            var path = Path.Combine(Application.persistentDataPath, cachingDirectory + new System.Uri(url).LocalPath);
            return path.Replace("\\", "/");
        }
        else
        {
            throw new Exception("[Caching] error: Url address was entered incorrectly " + url); ;
        }
    }
    catch (System.UriFormatException e)
    {
        throw new Exception("[Caching] error: " + url + " " + e.Message);
    }
}
```
Путь к файлу генерируется из `Application.persistentDataPath`, имени папки для кэширования и локального пути полученного из url-адреса. 
Важно отметить, что метод не гарантирует нахождения файла по указанному пути. Чтобы это проверить используйте: `File.Exists(path)`.

## Вместо заключения
Как можно заметить предоставленный код не отличается какой либо сложностью и при этом выполняет важную функцию сохранения данных на устройстве.
В следующем посте мы воспользуемся им для кэширования ресурсов загружаемых из сети интернет.
