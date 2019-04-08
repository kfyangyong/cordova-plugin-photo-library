[![That's how it looks in real app](https://img.youtube.com/vi/qHnnRsZ7klE/0.jpg)](https://www.youtube.com/watch?v=qHnnRsZ7klE)

We needed a code to displays photo library on HTML page. That gets thumbnail of arbitrary sizes, works on multiple platforms, and is fast.

So here it is.

- Displays photo gallery as web page, and not as boring native screen which you cannot modify. This brings back control over your app to you.
For example, you can use [PhotoSwipe](https://github.com/dimsemenov/photoswipe) library to present photos.
- Works on Android, iOS and browser (cordova serve).
- Fast - does not do base64 and uses browser cache.
- On device, provides custom schema to access thumbnails: cdvphotolibrary://thumbnail?fileid=xxx&width=128&height=128&quality=0.5 .
- Can save photos (jpg, png, animated gifs) and videos to specified album on device.
- Handles permissions.
- Handles images [EXIF rotation hell](http://www.daveperrett.com/articles/2012/07/28/exif-orientation-handling-is-a-ghetto/).
- On iOS, written in Swift and not Objective-C.

**Work in progress**

Contributions are welcome.

# Install

    cordova plugin add cordova-plugin-photo-library --variable PHOTO_LIBRARY_USAGE_DESCRIPTION="To choose photos" --save

# Usage

Add cdvphotolibrary protocol to Content-Security-Policy, like this:

```
<meta http-equiv="Content-Security-Policy" content="default-src 'self' data: gap: https://ssl.gstatic.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob: cdvphotolibrary:">
```

For remarks about angular/ionic usage, see below.

## Displaying photos

```js
cordova.plugins.photoLibrary.getLibrary(
  function (library) {

    // Here we have the library as array

    library.forEach(function(libraryItem) {
      console.log(libraryItem.id);          // Id of the photo
      console.log(libraryItem.photoURL);    // Cross-platform access to photo
      console.log(libraryItem.thumbnailURL);// Cross-platform access to thumbnail
      console.log(libraryItem.filename);
      console.log(libraryItem.width);
      console.log(libraryItem.height);
      console.log(libraryItem.creationDate);
      console.log(libraryItem.mimetype);    // Only for Android platform
      console.log(libraryItem.size);        // Only for Android platform
      console.log(libraryItem.nativeURL);   // Do not use, as it is not accessible on all platforms
    });

  },
  function (err) {
    console.log('Error occured');
  },
  { // optional options
    thumbnailWidth: 512,
    thumbnailHeight: 384,
    quality: 0.8
  },
  function partialCallback(partialLibrary) { // optional
    // If this callback provided and loading library takes time, it will be called each 0.5 seconds with
    // library that filled up to this time. You can start displaying photos to user right then.  
  }
);
```

This method is fast, as thumbails will be generated on demand.

## Saving photos and videos

``` js
var url = 'file:///...'; // url can also be dataURL, but giving it a file path is much faster
var album = 'MyAppName';
cordova.plugins.photoLibrary.saveImage(url, album, function () {}, function (err) {});
```

```js
// iOS quirks: video provided cannot be .webm . Use .mov or .mp4 .
cordova.plugins.photoLibrary.saveVideo(url, album, function () {}, function (err) {});
```


## Permissions

The library handles tricky parts of aquiring permissions to photo library.

If any of methods fail because lack of permissions, error string will be returned that begins with 'Permission'. So, to process on aquiring permissions, do the following:
```js
cordova.plugins.photoLibrary.getLibrary(
  function (library) { },
  function (err) {
    if (err.startsWith('Permission')) {
      // call requestAuthorization, and retry
    }
    // Handle error - it's not permission-related
  }
);
```

requestAuthorization is cross-platform method, that works in following way:

- On android, will ask user to allow access to storage
- On ios, on first call will open permission prompt. If user denies it subsequent calls will open setting page of your app, where user should enable access to Photos.

```js
cordova.plugins.photoLibrary.requestAuthorization(
  function () {
    // User gave us permission to his library, retry reading it!
  },
  function (err) {
    // User denied the access
  }, // if options not provided, defaults to {read: true}.
  {
    read: true,
    write: true
  }
);
```

## In addition you can ask thumbnail or full image for each photo separately, as cross-platform url or as blob

```js
// Use this method to get url. It's better to use it and not directly access cdvphotolibrary://, as it will also work on browser.
cordova.plugins.photoLibrary.getThumbnailURL(
  libraryItem, // or libraryItem.id
  function (thumbnailURL) {

    image.src = thumbnailURL;

  },
  function (err) {
    console.log('Error occured');
  },
  { // optional options
    thumbnailWidth: 512,
    thumbnailHeight: 384,
    quality: 0.8
  });
```

```js
cordova.plugins.photoLibrary.getPhotoURL(
  libraryItem, // or libraryItem.id
  function (photoURL) {

    image.src = photoURL;

  },
  function (err) {
    console.log('Error occured');
  });
```

```js
// This method is slower as it does base64
cordova.plugins.photoLibrary.getThumbnail(
  libraryItem, // or libraryItem.id
  function (thumbnailBlob) {

  },
  function (err) {
    console.log('Error occured');
  },
  { // optional options
    thumbnailWidth: 512,
    thumbnailHeight: 384,
    quality: 0.8
  });
```

```js
// This method is slower as it does base64
cordova.plugins.photoLibrary.getPhoto(
  libraryItem, // or libraryItem.id
  function (fullPhotoBlob) {

  },
  function (err) {
    console.log('Error occured');
  });
```

# ionic / angular

As mentioned [here](https://github.com/terikon/cordova-plugin-photo-library/issues/15) by dnmd, cdvphotolibrary urls should bypass sanitization to work.

In angular2, do following:

Define Pipe that will tell to bypass trusted urls. cdvphotolibrary urls should be trusted:

```js
@Pipe({name: 'safeUrl'})
export class SafeUrl {
  constructor(private sanitizer:DomSanitizer){}
  transform(url) {
    return this.sanitizer.bypassSecurityTrustUrl(url);
  }
}
```

Then in your component, use cdvphotolibrary urls applying the safeUrl pipe:

```js
@Component({
   selector: 'app',
   template: '<img [src]="url | safeUrl"></div>',
   pipes: [ SafeUrl ]
})

export class AppComponent {
    public url: string = 'placeholder.jpg';
    constructor() {
      // fetch thumbnail URL's
      this.url = libraryItem.thumbnailURL;
    }
}
```

If you use angular1, you need to add cdvphotolibrary to whitelist:

```js
var app = angular
  .module('myApp', [])
  .config([
    '$compileProvider',
    function ($compileProvider) {
      $compileProvider.aHrefSanitizationWhitelist(/^\s*(https?|cdvphotolibrary):/);
      // Angular before v1.2 uses $compileProvider.urlSanitizationWhitelist(...)
    }
  ]);
```

# TypeScript

TypeScript definitions are provided in [PhotoLibrary.d.ts](https://github.com/terikon/cordova-plugin-photo-library/blob/master/PhotoLibrary.d.ts)

# TODO

- [#7](https://github.com/terikon/cordova-plugin-photo-library/issues/7) saveImage and saveVideo should return saved libraryItem.
- [#14](https://github.com/terikon/cordova-plugin-photo-library/issues/14) partialCallback currently implemented only for iOS platform.
Android and browser platform implementations needed.
- [#16](https://github.com/terikon/cordova-plugin-photo-library/issues/16) Include album name in libraryItem.
- [#17](https://github.com/terikon/cordova-plugin-photo-library/issues/17) Enable filter option for getLibrary, that will select specific album.
- Improve documentation.
- iOS: PHImageContentMode.AspectFill returns images that larger than requested. If it really so, perform manual resizing.
- Provide cancellation mechanism for long-running operations, like getLibrary.
- Add unit tests.

# Optional enchancements

- iOS: it seems regex causes slowdown with dataURL, and (possibly) uses too much memory - check how to do regex on iOS in better way.
- Browser platform: Separate to multiple files.
- Browser platform: Compile plugin with webpack.
- Android: caching mechanism like [this one](https://developer.android.com/training/displaying-bitmaps/cache-bitmap.html) can be helpful.
- Implement save protocol with HTTP POST, so no base64 transformation will be needed for saving.
- Browser - implement saving to folder.
- EXIF rotation hell is not handled on browser platform.
- Pre-fetching data to file-based cache on app start can improve responsiveness. Just this caching should occur as low-priority thread. Cache can be updated
by system photo libraries events.


# References

Parts are based on

- https://github.com/subitolabs/cordova-gallery-api
- https://github.com/SuryaL/cordova-gallery-api
- https://github.com/ryouaki/Cordova-Plugin-Photos
- https://github.com/devgeeks/Canvas2ImagePlugin

## Relevant platform documentation

https://developer.android.com/reference/org/json/JSONObject.html
https://developer.android.com/reference/android/provider/MediaStore.Images.Media.html
https://developer.android.com/reference/android/provider/MediaStore.Images.Thumbnails.html
https://developer.android.com/reference/android/graphics/BitmapFactory.Options.html
https://developer.android.com/reference/android/media/ThumbnailUtils.html

https://www.raywenderlich.com/76735/using-nsurlprotocol-swift
