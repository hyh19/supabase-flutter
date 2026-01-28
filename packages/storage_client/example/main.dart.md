# Supabase Storage Client Example Explanation

This document explains the `main.dart` example file from the `storage_client` package, demonstrating how to interact with Supabase Storage using the Dart SDK.

## Overview

The example showcases the core functionality of the Supabase Storage client by demonstrating file operations like uploading, downloading, and managing files in storage buckets. It uses the `SupabaseStorageClient` to perform various operations on a "public" bucket.

## Client Initialization

```dart 8:16:packages/storage_client/example/main.dart
Future<void> main() async {
  const supabaseUrl = '';
  const supabaseKey = '';
  final client = SupabaseStorageClient(
    '$supabaseUrl/storage/v1',
    {
      'Authorization': 'Bearer $supabaseKey',
    },
  );
```

The example begins by creating a `SupabaseStorageClient` instance. The client requires:

- **Storage URL**: Constructed by appending `/storage/v1` to the Supabase project URL
- **Authorization headers**: Uses Bearer token authentication with the Supabase anon key

Note: In a real application, these values would be populated with actual Supabase project credentials and potentially stored securely.

## File Operations

### Binary File Upload

```dart 18:26:packages/storage_client/example/main.dart
  // Upload binary file
  final List<int> listBytes = 'Hello world'.codeUnits;
  final Uint8List fileData = Uint8List.fromList(listBytes);
  final uploadBinaryResponse = await client.from('public').uploadBinary(
        'binaryExample.txt',
        fileData,
        fileOptions: const FileOptions(upsert: true),
      );
  print('upload binary response : $uploadBinaryResponse');
```

This section demonstrates uploading binary data directly:

- Converts a string to bytes using `codeUnits`
- Creates a `Uint8List` from the byte list
- Uses `uploadBinary()` method to upload raw binary data
- Specifies `FileOptions(upsert: true)` to overwrite existing files with the same name
- The response contains metadata about the uploaded file

### File Upload from Disk

```dart 28:33:packages/storage_client/example/main.dart
  // Upload file to bucket "public"
  final file = File('example.txt');
  file.writeAsStringSync('File content');
  final storageResponse =
      await client.from('public').upload('example.txt', file);
  print('upload response : $storageResponse');
```

This demonstrates uploading a physical file:

- Creates a local file and writes content to it
- Uses the `upload()` method to send the file to storage
- Targets the "public" bucket (accessible without authentication)
- The file path in storage matches the local filename

## URL Management

### Creating Signed URLs

```dart 35:38:packages/storage_client/example/main.dart
  // Get download url
  final urlResponse =
      await client.from('public').createSignedUrl('example.txt', 60);
  print('download url : $urlResponse');
```

This creates a temporary, signed download URL:

- Uses `createSignedUrl()` to generate a time-limited access URL
- The URL expires after 60 seconds
- Allows secure, temporary access to private files without exposing permanent URLs

## Download Operations

### Downloading Files

```dart 40:46:packages/storage_client/example/main.dart
  // Download text file
  try {
    final fileResponse = await client.from('public').download('example.txt');
    print('downloaded file : ${String.fromCharCodes(fileResponse)}');
  } catch (error) {
    print('Error while downloading file : $error');
  }
```

This section shows how to download file content:

- Uses the `download()` method to retrieve file data as bytes
- Converts the byte data back to a string for display
- Includes error handling for failed downloads
- The returned data is raw bytes that can be processed as needed

## File Management

### Deleting Files

```dart 48:50:packages/storage_client/example/main.dart
  // Delete file
  final deleteResponse = await client.from('public').remove(['example.txt']);
  print('deleted file id : ${deleteResponse.first.id}');
```

This demonstrates file deletion:

- Uses the `remove()` method with a list of filenames to delete
- Even for single files, the method expects an array of paths
- Returns deletion metadata including file IDs
- The `first.id` accesses the ID of the first (and typically only) deleted file

## Cleanup

```dart 52:54:packages/storage_client/example/main.dart
  // Local file cleanup
  if (file.existsSync()) file.deleteSync();
}
```

The example concludes by cleaning up the local temporary file created for the upload demonstration, ensuring no leftover files remain after execution.

## Key Concepts Demonstrated

1. **Client Configuration**: Proper initialization with URL and authentication
2. **Bucket Targeting**: Using `client.from('bucketName')` to specify storage buckets
3. **Multiple Upload Methods**: Binary data vs file uploads
4. **File Options**: Configuration like upsert behavior
5. **URL Generation**: Creating temporary access links
6. **Error Handling**: Try-catch blocks for download operations
7. **Resource Management**: Proper cleanup of local files

## Usage Patterns

The example follows common patterns for Supabase Storage usage:

- Initialize once, reuse client for multiple operations
- Use descriptive filenames for uploaded content
- Handle errors appropriately for production applications
- Clean up resources when operations complete

This comprehensive example serves as a practical reference for implementing file storage functionality in Dart applications using Supabase.
