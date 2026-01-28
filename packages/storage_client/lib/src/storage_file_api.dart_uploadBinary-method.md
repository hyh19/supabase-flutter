# uploadBinary Method

## Overview

The `uploadBinary` method is a core functionality in the Supabase Storage File API that enables uploading binary file data directly to a storage bucket. This method is particularly useful for web platforms where file handling differs from native platforms, as indicated by the documentation comment "Can be used on the web."

## Method Signature

```dart 81:87:packages/storage_client/lib/src/storage_file_api.dart
Future<String> uploadBinary(
  String path,
  Uint8List data, {
  FileOptions fileOptions = const FileOptions(),
  int? retryAttempts,
  StorageRetryController? retryController,
}) async {
```

## Parameters

### Required Parameters

- **`path`** (`String`): The relative file path within the bucket, excluding the bucket ID itself. Should follow the format `folder/subfolder/filename.png`. The bucket must already exist before attempting upload.

- **`data`** (`Uint8List`): The binary file data to be stored in the bucket. This is the raw binary content of the file as bytes.

### Optional Parameters

- **`fileOptions`** (`FileOptions`, default: `const FileOptions()`): Configuration options for the file upload, including HTTP headers like cache control, content type, and upsert behavior.

- **`retryAttempts`** (`int?`): Overrides the default retry attempts configured across the storage client. Must be greater than or equal to 0 if provided.

- **`retryController`** (`StorageRetryController?`): Allows canceling the retry attempts by calling `cancel()` on the controller during upload.

## Return Value

Returns a `Future<String>` that resolves to the uploaded file's key/path identifier from the storage service response.

## Implementation Details

### Path Processing

```dart 90:90:packages/storage_client/lib/src/storage_file_api.dart
final finalPath = _getFinalPath(path);
```

The method first processes the provided path using the private `_getFinalPath` method, which prefixes the path with the bucket ID:

```dart 23:25:packages/storage_client/lib/src/storage_file_api.dart
String _getFinalPath(String path) {
  return '$bucketId/$path';
}
```

This creates the full storage path in the format `bucketId/folder/subfolder/filename.png`.

### HTTP Request Execution

```dart 91:98:packages/storage_client/lib/src/storage_file_api.dart
final response = await _storageFetch.postBinaryFile(
  '$url/object/$finalPath',
  data,
  fileOptions,
  options: FetchOptions(headers: headers),
  retryAttempts: retryAttempts ?? _retryAttempts,
  retryController: retryController,
);
```

The method makes an HTTP POST request to upload the binary data:

- **URL**: Constructs the endpoint as `$url/object/$finalPath` where `url` is the base storage service URL
- **Data**: Sends the `Uint8List` binary data directly
- **Headers**: Merges custom headers with the client's default headers
- **Retry Logic**: Uses either the provided `retryAttempts` or falls back to the client's default `_retryAttempts`

### Response Handling

```dart 100:100:packages/storage_client/lib/src/storage_file_api.dart
return (response as Map)['Key'] as String;
```

The method extracts the `'Key'` field from the response, which contains the storage key/path identifier for the uploaded file.

## Error Handling

### Input Validation

```dart 88:89:packages/storage_client/lib/src/storage_file_api.dart
assert(retryAttempts == null || retryAttempts >= 0,
    'retryAttempts has to be greater or equal to 0');
```

The method validates that if `retryAttempts` is provided, it must be a non-negative integer.

### Network and Storage Errors

The method relies on the underlying `_storageFetch.postBinaryFile` implementation for error handling, which includes:

- Automatic retry logic with exponential backoff
- Network failure handling
- Storage service error responses

## Usage Context

This method is part of the `StorageFileApi` class, which is instantiated with:

```dart 15:21:packages/storage_client/lib/src/storage_file_api.dart
const StorageFileApi(
  this.url,
  this.headers,
  this.bucketId,
  this._retryAttempts,
  this._storageFetch,
);
```

The class maintains state for the storage service URL, authentication headers, target bucket ID, default retry attempts, and the fetch implementation.

## Platform Considerations

The documentation specifically mentions "Can be used on the web," indicating this method is designed to work across different platforms, including web where `Uint8List` (binary data) is more universally supported than platform-specific `File` objects.

## Comparison with Related Methods

This method is similar to the `upload` method but differs in the data type parameter:

- `upload`: Takes a `File` object (platform-specific)
- `uploadBinary`: Takes `Uint8List` data (cross-platform binary data)

Both methods serve the same core purpose of uploading files to Supabase Storage but cater to different input data formats and platform requirements.

## Thread Safety and Async Behavior

The method is `async` and returns a `Future`, making it suitable for asynchronous file upload operations. The underlying fetch implementation handles retry logic and network operations asynchronously, allowing the application to remain responsive during uploads.
