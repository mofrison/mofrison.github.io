---
title:  "Unity3d: Saving data on the device"
repository: Unity3d-Network
preview: /assets/images/posts/2021-04-03-unity3d-caching-resources/preview.jpg
date:   2021-04-03 10:00:00 +0300
categories: cases
tags: [C#, Unity3d]
lang: En
layout: post
---

In this article, we are talking about saving data to the device's permanent memory and organizing access to it. It is intended to prepare you, dear reader, for a more complex article about downloading resources from the network. In fact, this is its introductory part, which was allocated in a separate post, because the topic turned out to be very voluminous.

## Why store resources in the device memory
I think many developers primarily of mobile applications have encountered the fact that resources do not fit in the application installation files. This is especially true for those who use photorealistic content(360 panoramas). A small help in this is the use of extension files, but they are also limited in size. In addition, downloading additional content requires updating the app through the store.

You can download all the necessary resources from the network at each launch, but despite the rapid development of the Internet, this is still quite a long, resource-consuming and unreliable process. Therefore, it is rational to cache the resources downloaded from the network to the device's memory and, if necessary, access them.

## Configuring caching
Before you save something, you need to configure the path for storing data. To do this, I defined the `ConfiguringCaching` method:
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
The input method gets the name of the directory where we will save the uploaded data. By default, I named this folder _data_. The full path to this directory is formed using [Application.persistentDataPath](https://docs.unity3d.com/ScriptReference/Application-persistentDataPath.html). This value is the path to the directory where you can save data between runs. When publishing on iOS and Android, persistentDataPath points to the public directory on the device. Files in this location are not deleted by app updates. Files can still be deleted directly by users.

Also in this method, the path to the folder is added to `Caching.currentCacheForWriting`, to configure caching **AssetBundles**. **AssetBundles** it has its own caching mechanisms and you only need to use them correctly.

### Availability of free space in the device memory
Before storing data in the device's memory, you need to make sure that the data will fit on it. To do this, the **SimpleDiskUtils** plugin was previously added to the project. It has libraries for different operating systems, with which you can find out how much free memory space there is on the device. In the following method, the size of the downloaded file is compared with the free space remaining on the device. The method takes the file size in bytes as an argument.
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
When generating **AssetBundle**, there may be problems with accessing different methods, despite the presence of the `#if-#else` directive, to avoid this, you can comment out or delete unnecessary calls.

### Saving data
Saving data to the device's permanent memory implements the `Caching` method. As input, it accepts the url of the file as a string and an array of type `byte` with data.
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
First of all, the method checks whether there is a place for the presented array with data on the device. If there is space, the url is converted to the path of the file on the device using the `ConvertToCachedPath(string url)` method, which we will discuss later. After that, the necessary repositories are created and the file is saved to the provided path using the standard `File.WriteAllBytes(string path, byte[] data)` method. If there is no space, an exception is thrown.

### Getting the path to a saved file
In order to save a file and then access it, you need to get the path to it in the device's memory. To do this, use the `ConvertToCachedPath` method Accepts a string with the file url as the only argument.
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
The file path is generated from `Application.persistentDataPath`,  the name of the folder to cache, and the local path obtained from the url.
It is important to note that the method does not guarantee that the file will be found at the specified path. To check this, use: `File.Exists(path)`.

## Instead of a conclusion
As you can see, the provided code does not differ in any complexity and at the same time performs an important function of saving data on the device.
In the next post, we will use it to cache resources downloaded from the Internet.

I have created a separate repository with [examples](https://github.com/{{ site.github.owner_name }}/Unity3d-download-resources) so that you can test them.
You can also use _Saving data on the device_ for **Unity3d** in your project by adding [this repository](https://github.com/{{ site.github.owner_name }}/{{ page.repository }}) as a [submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules):

	git submodule add https://github.com/mofrison/Unity3d-Network

Thank you for reading to the end, I hope this will be useful to you :)
