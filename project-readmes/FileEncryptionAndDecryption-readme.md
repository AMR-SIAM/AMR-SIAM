# Crypto Service Documentation

## Process Diagrams

### Encryption Process
```mermaid
graph TD
    subgraph "Encryption Process"
        A1["encryptFile(QString inputPath, QString password)"] --> B1["processEncryption(string inputPath, string outputPath, string password)"]
        B1 --> C1["writeFileHeader(outputPath)"]
        B1 --> D1["processKeys(outputPath, key, iv, salt, password)"]
        B1 --> E1["writeFileSize(outputPath, inputPath)"]
        
        D1 --> D1_1["keyManager_->deriveSalt(salt)"]
        D1 --> D1_2["keyManager_->generateKeyWithSalt(password, salt, key)"]
        D1 --> D1_3["keyManager_->generateIV(iv)"]
        D1 --> D1_4["Store key and IV in class members"]
        D1 --> D1_5["keyManager_->generateMasterKeyAndIV(password, masterKey, masterIV)"]
        D1 --> D1_6["encryptChunk(key, masterKey, masterIV)"]
        D1 --> D1_7["fileHandler_->appendToFile(outputPath, salt)"]
        D1 --> D1_8["fileHandler_->appendToFile(outputPath, keyResult.encryptedData)"]
        D1 --> D1_9["fileHandler_->appendToFile(outputPath, ivResult.encryptedData)"]
        
        B1 --> F1["Process File Content"]
        F1 --> F1_1["getFileChunk(inputPath, offset, chunkSize)"]
        F1 --> F1_2["encryptChunk(chunk, stored_key_, stored_iv_, isLastChunk)"]
        F1 --> F1_3["fileHandler_->appendToFile(outputPath, sizeBytes)"]
        F1 --> F1_4["fileHandler_->appendToFile(outputPath, encryptedData)"]
        F1 --> F1_5["progressUpdated signal"]
        
        F1_2 --> F1_2_1["Validate key and IV sizes"]
        F1_2 --> F1_2_2["EVP_EncryptInit_ex with stored key/IV"]
        F1_2 --> F1_2_3["EVP_EncryptUpdate"]
        F1_2 --> F1_2_4["EVP_EncryptFinal_ex"]
        F1_2 --> F1_2_5["Store last block as next IV"]
    end
```

### Decryption Process
```mermaid
graph TD
    subgraph "Decryption Process"
        A2["decryptFile(QString inputPath, QString password)"] --> B2["processDecryption(string inputPath, string outputPath, string password)"]
        B2 --> C2["validateInputFile(inputPath)"]
        B2 --> D2["initializeDecryption(inputPath, password, params)"]
        B2 --> E2["processContentToDecrypt(inputPath, outputPath, params)"]
        
        C2 --> C2_1["formatHandler_->isValidFormat(inputPath)"]
        
        D2 --> D2_1["readMetadata(inputPath, salt, encryptedKey, encryptedIV, originalFileSize)"]
        D2 --> D2_2["decryptKeys(encryptedKey, encryptedIV, salt, password, key, iv)"]
        
        D2_2 --> D2_2_1["keyManager_->generateMasterKeyAndIV(password, masterKey, masterIV)"]
        D2_2 --> D2_2_2["keyManager_->decryptKeyAndIV(encryptedKey, encryptedIV, masterKey, masterIV, key, iv)"]
        D2_2 --> D2_2_3["keyManager_->generateKeyWithSalt(password, salt, derivedKey)"]
        D2_2 --> D2_2_4["Verify decrypted key matches derived key"]
        
        E2 --> E2_1["processNextDecryptedChunk(inputFile, outputPath, params, totalBytesWritten)"]
        E2_1 --> E2_1_1["readChunkSize(inputFile, encryptedSize)"]
        E2_1 --> E2_1_2["readAndDecryptChunk(inputFile, encryptedSize, params, encryptedChunk, decryptedChunk)"]
        E2_1 --> E2_1_3["writeDecryptedChunk(outputPath, decryptedChunk, originalFileSize, totalBytesWritten)"]
        E2_1 --> E2_1_4["updateProgress(totalBytesWritten, originalFileSize)"]
        
        E2_1_2 --> E2_1_2_1["decryptor_->decrypt(encryptedChunk, params.key, params.currentIV, decryptedChunk)"]
        E2_1_2 --> E2_1_2_2["Update IV for next chunk"]
    end
```

## Implementation Details

### Encryption Process Flow

1. **Entry Point (`encryptFile`)**
   - Takes input file path and password
   - Creates output file path through user selection
   - Initiates encryption process in a worker thread

2. **Main Process (`processEncryption`)**
   - Writes file header using `writeFileHeader`
   - Processes encryption keys using `processKeys`
   - Writes original file size
   - Processes file content in chunks

3. **Key Processing (`processKeys`)**
   - Generates salt using `deriveSalt`
   - Generates key from password and salt
   - Generates initialization vector (IV)
   - Stores key and IV in class members for chunk encryption
   - Creates master key and IV for key encryption
   - Encrypts and saves the key and IV
   - Stores all metadata in the output file

4. **Content Processing**
   - Reads input file in chunks
   - Encrypts each chunk using AES-256-CBC encryption
   - Uses stored key and IV from key processing phase
   - Validates key and IV sizes before encryption
   - Uses CBC mode with IV chaining
   - Writes encrypted chunk size and data
   - Updates progress through signals
   - Comprehensive error handling with detailed logging

### Decryption Process Flow

1. **Entry Point (`decryptFile`)**
   - Validates input file format
   - Gets output file path from user
   - Initiates decryption process in a worker thread

2. **Validation (`validateInputFile`)**
   - Checks file format signature
   - Verifies file integrity

3. **Initialization (`initializeDecryption`)**
   - Reads file metadata (salt, encrypted key/IV)
   - Decrypts the encryption key and IV
   - Verifies password correctness through key derivation
   - Stores decrypted key and IV in params structure

4. **Content Processing (`processContentToDecrypt`)**
   - Processes file in chunks through `processNextDecryptedChunk`
   - Reads chunk sizes and encrypted data
   - Decrypts chunks using AES decryption
   - Maintains IV chaining for CBC mode
   - Writes decrypted data
   - Updates progress
   - Verifies final file size

### Error Handling and Progress Reporting

Both processes include:
- Comprehensive exception handling
- Detailed error logging at each step
- Progress updates through signals
- Operation completion notification
- Resource cleanup
- Key and IV size validation
- File integrity checks

### File Format Structure

The encrypted file format includes:
1. Signature (3 bytes)
2. Salt (16 bytes)
3. Encrypted Key (48 bytes)
4. Encrypted IV (32 bytes)
5. Original File Size (8 bytes)
6. Encrypted Content (variable size)
   - Each chunk prefixed with its size
   - Chunk data encrypted with AES-256-CBC
   - IV chaining between chunks

### Recent Improvements

1. **Key Management**
   - Added proper storage of encryption key and IV
   - Improved key derivation and verification
   - Enhanced key and IV size validation

2. **Error Handling**
   - Added detailed error logging
   - Improved exception handling
   - Added validation checks for key and IV sizes
   - Better error messages for debugging

3. **Code Organization**
   - Streamlined encryption and decryption flows
   - Improved separation of concerns
   - Enhanced code maintainability

## Requirements
- C++17
- Qt Framework 6
- OpenSSL Library
  
## Demo

https://github.com/user-attachments/assets/07635df2-9e47-4aac-a976-693d0dc554bb

## ScreenShots

![Selectfile](https://github.com/user-attachments/assets/470322f7-b007-4231-b152-2e5f66cad863)

![enc1](https://github.com/user-attachments/assets/15c69f32-5435-4de5-842c-85db1da40bb5)

![enc2](https://github.com/user-attachments/assets/fb0ac608-c6f6-4178-bc0f-7c7999a5ff3c)

![enc3](https://github.com/user-attachments/assets/83a9fc87-1f1e-488b-8028-80228ea79632)

![dec1](https://github.com/user-attachments/assets/1c0304a7-8e9a-499f-b7db-318d12b928c5)





