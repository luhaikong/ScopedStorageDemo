# ScopedStorageDemo
A demo to show how scoped storage worked on Android 10 and backward compatible with previous versions.

相关链接：https://mp.weixin.qq.com/s/31esIqMudRRDBY8JDs8D4A

/   前言   /

距离Android 10系统正式发布已经过去大半年左右的时间了，你的应用程序已经对它进行适配了吗？

在Android 10众多的行为变更当中，有一点是非常值得引起我们重视的，那就是作用域存储。这个新功能直接颠覆了长久以来我们一直惯用的外置存储空间的使用方式，因此大量App都将面临着较多代码模块的升级。

然而，对于作用域存储这个新功能，官方的资料并不多，很多人也没有搞明白它的用法。另外它也不属于《第一行代码》现有的知识架构体系，虽然我有想过在第3版中加入这部分内容的讲解，但几经思考之后还是决定以一讲单独文章的方式来讲解这部分内容，也算是作为《第一行代码 第3版》的内容扩展吧。

本篇文章对作用域存储进行了比较全面的解析，相信看完之后你将能够轻松地完成Android 10作用域存储的适配升级。

/   理解作用域存储   /

Android长久以来都支持外置存储空间这个功能，也就是我们常说的SD卡存储。这个功能使用得极其广泛，几乎所有的App都喜欢在SD卡的根目录下建立一个自己专属的目录，用来存放各类文件和数据。

那么这么做有什么好处吗？我想了一下，大概有两点吧。第一，存储在SD卡的文件不会计入到应用程序的占用空间当中，也就是说即使你在SD卡存放了1G的文件，你的应用程序在设置中显示的占用空间仍然可能只有几十K。第二，存储在SD卡的文件，即使应用程序被卸载了，这些文件仍然会被保留下来，这有助于实现一些需要数据被永久保留的功能。

然而，这些“好处”真的是好处吗？或许对于开发者而言这算是好处吧，但对于用户而言，上述好处无异于一些流氓行为。因为这会将用户的SD卡空间搞得乱糟糟的，而且即使我卸载了一个完全不再使用的程序，它所产生的垃圾文件却可能会一直保留在我的手机上。

另外，存储在SD卡上的文件属于公有文件，所有的应用程序都有权随意访问，这也对数据的安全性带来了很大的挑战。

为了解决上述问题，Google在Android 10当中加入了作用域存储功能。

那么到底什么是作用域存储呢？简单来讲，就是Android系统对SD卡的使用做了很大的限制。从Android 10开始，每个应用程序只能有权在自己的外置存储空间关联目录下读取和创建文件，获取该关联目录的代码是：context.getExternalFilesDir()。关联目录对应的路径大致如下：

/storage/emulated/0/Android/data/<包名>/files

将数据存放到这个目录下，你将可以完全使用之前的写法来对文件进行读写，不需要做任何变更和适配。但同时，刚才提到的那两个“好处”也就不存在了。这个目录中的文件会被计入到应用程序的占用空间当中，同时也会随着应用程序的卸载而被删除。

那么有些朋友可能会问了，我就是需要访问其他目录该怎么办呢？比如读取手机相册中的图片，或者向手机相册中添加一张图片。为此，Android系统针对文件类型进行了分类，图片、音频、视频这三类文件将可以通过MediaStore API来进行访问，而其他类型的文件则需要使用系统的文件选择器来进行访问。

另外，我们的应用程序向媒体库贡献的图片、音频或视频，将会自动拥有其读写权限，不需要额外申请READ_EXTERNAL_STORAGE和WRITE_EXTERNAL_STORAGE权限。而如果你要读取其他应用程序向媒体库贡献的图片、音频或视频，则必须要申请READ_EXTERNAL_STORAGE权限才行。WRITE_EXTERNAL_STORAGE权限将会在未来的Android版本中废弃。

好了，关于作用域存储的理论知识就先讲到这里，相信你已经对它有了一个基本的了解了，那么接下来我们就开始上手操作吧。

/   我一定要升级吗？   /

一定会有很多朋友关心这个问题，因为每当适配升级面临着需要更改大量代码的时候，大多数人的第一想法都是能不升就不升，或者能晚升就晚升。而在作用域存储这个功能上面，恭喜大家，暂时确实是可以不用升级的。

目前Android 10系统对于作用域存储适配的要求还不是那么严格，毕竟之前传统外置存储空间的用法实在是太广泛了。如果你的项目指定的targetSdkVersion低于29，那么即使不做任何作用域存储方面的适配，你的项目也可以成功运行到Android 10手机上。

而如果你的targetSdkVersion已经指定成了29，也没有关系，假如你还不想进行作用域存储的适配，只需要在AndroidManifest.xml中加入如下配置即可：

<manifest ... >
  <application android:requestLegacyExternalStorage="true" ...>
    ...
  </application>
</manifest>

这段配置表示，即使在Android 10系统上，仍然允许使用之前遗留的外置存储空间的用法来运行程序，这样就不用对代码进行任何修改了。当然，这只是一种权宜之计，在未来的Android系统版本中，这段配置随时都可能会失效（目前Android 11预览版已经确认，这段配置至少在Android 11上不会失效）。因此，我们还是非常有必要现在就来学习一下，到底该如何对作用域存储进行适配。

另外，本篇文章中演示的所有示例，都可以到ScopedStorageDemo这个开源库中找到其对应的源码。

开源库地址是：

https://github.com/guolindev/ScopedStorageDemo

/   获取相册中的图片   /

首先来学习一下如何在作用域存储当中获取手机相册里的图片。注意，虽然本篇文章中我是以图片来举例的，但是获取音频、视频的用法也是基本相同的。

不同于过去可以直接获取到相册中图片的绝对路径，在作用域存储当中，我们只能借助MediaStore API获取到图片的Uri，示例代码如下：

val cursor = contentResolver.query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, null, null, null, "${MediaStore.MediaColumns.DATE_ADDED} desc")
if (cursor != null) {
    while (cursor.moveToNext()) {
        val id = cursor.getLong(cursor.getColumnIndexOrThrow(MediaStore.MediaColumns._ID))
        val uri = ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id)
        println("image uri is $uri")
    }
    cursor.close()
}

上述代码中，我们先是通过ContentResolver获取到了相册中所有图片的id，然后再借助ContentUris将id拼装成一个完整的Uri对象。一张图片的Uri格式大致如下所示：

content://media/external/images/media/321

那么有些朋友可能会问了，获取到了Uri之后，我又该怎样将这张图片显示出来呢？这就有很多种办法了，比如使用Glide来加载图片，它本身就支持传入Uri对象来作为图片路径：

Glide.with(context).load(uri).into(imageView)

而如果你没有使用Glide或其他图片加载框架，想在不借助第三方库的情况下直接将一个Uri对象解析成图片，可以使用如下代码：

val fd = contentResolver.openFileDescriptor(uri, "r")
if (fd != null) {
    val bitmap = BitmapFactory.decodeFileDescriptor(fd.fileDescriptor)
    fd.close()
    imageView.setImageBitmap(bitmap)
}

上述代码中，我们调用了ContentResolver的openFileDescriptor()方法，并传入Uri对象来打开文件句柄，然后再调用BitmapFactory的decodeFileDescriptor()方法将文件句柄解析成Bitmap对象即可。

Demo效果：



这样我们就将获取相册中图片的方式掌握了，并且这种方式在所有的Android系统版本中都适用。

那么接下来，我们开始学习如何将一张图片添加到相册。

/   将图片添加到相册   /

将一张图片添加到手机相册要相对稍微复杂一点，因为不同系统版本之间的处理方式是不太一样的。

我们还是通过一段代码示例来直观地学习一下，代码如下所示：

fun addBitmapToAlbum(bitmap: Bitmap, displayName: String, mimeType: String, compressFormat: Bitmap.CompressFormat) {
    val values = ContentValues()
    values.put(MediaStore.MediaColumns.DISPLAY_NAME, displayName)
    values.put(MediaStore.MediaColumns.MIME_TYPE, mimeType)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        values.put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DCIM)
    } else {
        values.put(MediaStore.MediaColumns.DATA, "${Environment.getExternalStorageDirectory().path}/${Environment.DIRECTORY_DCIM}/$displayName")
    }
    val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)
    if (uri != null) {
        val outputStream = contentResolver.openOutputStream(uri)
        if (outputStream != null) {
            bitmap.compress(compressFormat, 100, outputStream)
            outputStream.close()
        }
    }
}

这段代码演示了如何将一个Bitmap对象添加到手机相册当中，我来简单解释一下。

想要将一张图片添加到手机相册，我们需要构建一个ContentValues对象，然后向这个对象中添加三个重要的数据。一个是DISPLAY_NAME，也就是图片显示的名称，一个是MIME_TYPE，也就是图片的mime类型。还有一个是图片存储的路径，不过这个值在Android 10和之前的系统版本中的处理方式不一样。Android 10中新增了一个RELATIVE_PATH常量，表示文件存储的相对路径，可选值有DIRECTORY_DCIM、DIRECTORY_PICTURES、DIRECTORY_MOVIES、DIRECTORY_MUSIC等，分别表示相册、图片、电影、音乐等目录。而在之前的系统版本中并没有RELATIVE_PATH，所以我们要使用DATA常量（已在Android 10中废弃），并拼装出一个文件存储的绝对路径才行。

有了ContentValues对象之后，接下来调用ContentResolver的insert()方法即可获得插入图片的Uri。但仅仅获得Uri仍然是不够的，我们还需要向该Uri所对应的图片写入数据才行。调用ContentResolver的openOutputStream()方法获得文件的输出流，然后将Bitmap对象写入到该输出流当中即可。

以上代码即可实现将Bitmap对象存储到手机相册当中，那么有些朋友可能会问了，如果我要存储的图片并不是Bitmap对象，而是一张网络上的图片，或者是当前应用关联目录下的图片该怎么办呢？

其实方法都是相似的，因为不管是网络上的图片还是关联目录下的图片，我们都能获取到它的输入流，只要不断读取输入流中的数据，然后写入到相册图片所对应的输出流当中就可以了，示例代码如下：

fun writeInputStreamToAlbum(inputStream: InputStream, displayName: String, mimeType: String) {
    val values = ContentValues()
    values.put(MediaStore.MediaColumns.DISPLAY_NAME, displayName)
    values.put(MediaStore.MediaColumns.MIME_TYPE, mimeType)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        values.put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DCIM)
    } else {
        values.put(MediaStore.MediaColumns.DATA, "${Environment.getExternalStorageDirectory().path}/${Environment.DIRECTORY_DCIM}/$displayName")
    }
    val bis = BufferedInputStream(inputStream)
    val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)
    if (uri != null) {
        val outputStream = contentResolver.openOutputStream(uri)
        if (outputStream != null) {
            val bos = BufferedOutputStream(outputStream)
            val buffer = ByteArray(1024)
            var bytes = bis.read(buffer)
            while (bytes >= 0) {
                bos.write(buffer, 0 , bytes)
                bos.flush()
                bytes = bis.read(buffer)
            }
            bos.close()
        }
    }
    bis.close()
}

这段代码中只是将输入流和输出流的部分重新编写了一下，其他部分和之前存储Bitmap的代码是完全一致的，相信很好理解。

Demo效果：



好了，这样我们就将相册图片的读取和存储问题都解决了，下面我们来探讨另外一个常见的需求，如何将文件下载到Download目录。

/   下载文件到Download目录   /

执行文件下载操作是一个很常见的场景，比如说下载pdf、doc文件，或者下载APK安装包等等。在过去，这些文件我们通常都会下载到Download目录，这是一个专门用于存放下载文件的目录。而从Android 10开始，我们已经不能以绝对路径的方式访问外置存储空间了，所以文件下载功能也会受到影响。

那么该如何解决呢？主要有以下两种方式。

第一种同时也是最简单的一种方式，就是更改文件的下载目录。将文件下载到应用程序的关联目录下，这样不用修改任何代码就可以让程序在Android 10系统上正常工作。但使用这种方式，你需要知道，下载的文件会被计入到应用程序的占用空间当中，同时如果应用程序被卸载了，该文件也会一同被删除。另外，存放在关联目录下的文件只能被当前的应用程序所访问，其他程序是没有读取权限的。

以上几个限制条件如果不能满足你的需求，那么就只能使用第二种方式，对Android 10系统进行代码适配，仍然将文件下载到Download目录下。

其实将文件下载到Download目录，和向相册中添加一张图片的过程是差不多的，Android 10在MediaStore中新增了一种Downloads集合，专门用于执行文件下载操作。但由于每个项目下载功能的实现都各不相同，有些项目的下载实现还十分复杂，因此怎么将以下的示例代码融合到你的项目当中是你自己需要思考的问题。

fun downloadFile(fileUrl: String, fileName: String) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {
        Toast.makeText(this, "You must use device running Android 10 or higher", Toast.LENGTH_SHORT).show()
        return
    }
    thread {
        try {
            val url = URL(fileUrl)
            val connection = url.openConnection() as HttpURLConnection
            connection.requestMethod = "GET"
            connection.connectTimeout = 8000
            connection.readTimeout = 8000
            val inputStream = connection.inputStream
            val bis = BufferedInputStream(inputStream)
            val values = ContentValues()
            values.put(MediaStore.MediaColumns.DISPLAY_NAME, fileName)
            values.put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DOWNLOADS)
            val uri = contentResolver.insert(MediaStore.Downloads.EXTERNAL_CONTENT_URI, values)
            if (uri != null) {
                val outputStream = contentResolver.openOutputStream(uri)
                if (outputStream != null) {
                    val bos = BufferedOutputStream(outputStream)
                    val buffer = ByteArray(1024)
                    var bytes = bis.read(buffer)
                    while (bytes >= 0) {
                        bos.write(buffer, 0 , bytes)
                        bos.flush()
                        bytes = bis.read(buffer)
                    }
                    bos.close()
                }
            }
            bis.close()
        } catch(e: Exception) {
            e.printStackTrace()
        }
    }
}

这段代码总体来讲还是比较好理解的，主要就是添加了一些Http请求的代码，并将MediaStore.Images.Media改成了MediaStore.Downloads，其他部分几乎是没有变化的，我就不再多加解释了。

注意，上述代码只能在Android 10或更高的系统版本上运行，因为MediaStore.Downloads是Android 10中新增的API。至于Android 9及以下的系统版本，请你仍然使用之前的代码来进行文件下载。

Demo效果：



/   使用文件选择器   /

如果我们要读取SD卡上非图片、音频、视频类的文件，比如说打开一个PDF文件，这个时候就不能再使用MediaStore API了，而是要使用文件选择器。

但是，我们不能再像之前的写法那样，自己写一个文件浏览器，然后从中选取文件，而是必须要使用手机系统中内置的文件选择器。示例代码如下：

const val PICK_FILE = 1

private fun pickFile() {
    val intent = Intent(Intent.ACTION_OPEN_DOCUMENT)
    intent.addCategory(Intent.CATEGORY_OPENABLE)
    intent.type = "*/*"
    startActivityForResult(intent, PICK_FILE)
}

override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    when (requestCode) {
        PICK_FILE -> {
            if (resultCode == Activity.RESULT_OK && data != null) {
                val uri = data.data
                if (uri != null) {
                    val inputStream = contentResolver.openInputStream(uri)
                    // 执行文件读取操作
                }
            }
        }
    }
}

这里在pickFile()方法当中通过Intent去启动系统的文件选择器，注意Intent的action和category都是固定不变的。而type属性可以用于对文件类型进行过滤，比如指定成image/*就可以只显示图片类型的文件，这里写成*/*表示显示所有类型的文件。注意type属性必须要指定，否则会产生崩溃。

然后在onActivityResult()方法当中，我们就可以获取到用户选中文件的Uri，之后通过ContentResolver打开文件输入流来进行读取就可以了。

Demo效果：



/   第三方SDK不支持怎么办？   /

阅读完了本篇文章之后，相信你对Android 10作用域存储的用法和适配基本上都已经掌握了。然而我们在实际的开发工作当中还可能会面临一个非常头疼的问题，就是我自己的代码当然可以进行适配，但是项目中使用的第三方SDK还不支持作用域存储该怎么办呢？

这个情况确实是存在的，比如我之前使用的七牛云SDK，它的文件上传功能要求你传入的就是一个文件的绝对路径，而不支持传入Uri对象，大家应该也会碰到类似的问题。

由于我们是没有权限修改第三方SDK的，因此最简单直接的办法就是等待第三方SDK的提供者对这部分功能进行更新，在那之前我们先不要将targetSdkVersion指定到29，或者先在AndroidManifest文件中配置一下requestLegacyExternalStorage属性。

然而如果你不想使用这种权宜之计，其实还有一个非常好的办法来解决此问题，就是我们自己编写一个文件复制功能，将Uri对象所对应的文件复制到应用程序的关联目录下，然后再将关联目录下这个文件的绝对路径传递给第三方SDK，这样就可以完美进行适配了。这个功能的示例代码如下：

fun copyUriToExternalFilesDir(uri: Uri, fileName: String) {
    val inputStream = contentResolver.openInputStream(uri)
    val tempDir = getExternalFilesDir("temp")
    if (inputStream != null && tempDir != null) {
        val file = File("$tempDir/$fileName")
        val fos = FileOutputStream(file)
        val bis = BufferedInputStream(inputStream)
        val bos = BufferedOutputStream(fos)
        val byteArray = ByteArray(1024)
        var bytes = bis.read(byteArray)
        while (bytes > 0) {
            bos.write(byteArray, 0, bytes)
            bos.flush()
            bytes = bis.read(byteArray)
        }
        bos.close()
        fos.close()
    }
}

好的，关于Android 10作用域存储的重要知识点就讲到这里，相信你已经可以完全掌握了。下篇文章中我们会继续学习Android 10适配，讲一讲深色主题的功能，敬请期待。
