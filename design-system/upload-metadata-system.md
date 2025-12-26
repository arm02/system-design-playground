# S3 Presigned URL Upload System

## Overview

This document describes the S3 presigned URL upload system implemented for the Akasia365MC API. The system allows mobile clients to upload files directly to S3 without routing through the backend, reducing server load and improving upload performance.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AKASIA365MC SYSTEM                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────────┐    │
│  │              │         │                  │         │                  │    │
│  │   Mobile     │◄───────►│   API Gateway    │◄───────►│   Core Module    │    │
│  │   Client     │  REST   │  (gatewaypublic) │  HTTP   │   (core)         │    │
│  │              │         │                  │         │                  │    │
│  └──────┬───────┘         └──────────────────┘         └────────┬─────────┘    │
│         │                                                        │              │
│         │                                                        │              │
│         │  Direct Upload                              ┌──────────▼─────────┐   │
│         │  (Presigned PUT)                            │                    │   │
│         │                                             │   Common Module    │   │
│         │                                             │   (Presigned Svc)  │   │
│         │                                             │                    │   │
│         │                                             └──────────┬─────────┘   │
│         │                                                        │              │
│         │                                                        │              │
│  ┌──────▼───────────────────────────────────────────────────────▼───────────┐  │
│  │                                                                           │  │
│  │                              AWS S3 Bucket                                │  │
│  │                                                                           │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────────┐   │
│  │                   │    │                   │    │                       │   │
│  │   PostgreSQL      │    │    Redis          │    │     RabbitMQ          │   │
│  │   (metadata)      │    │    (cache)        │    │   (image processing)  │   │
│  │                   │    │                   │    │                       │   │
│  └───────────────────┘    └───────────────────┘    └───────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         COMMON MODULE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                  PresignedUploadService                         │ │
│  │  ├── initializeUpload(BatchPresignedUploadRequest)              │ │
│  │  ├── initializeSingleUpload(PresignedUploadRequest)             │ │
│  │  ├── confirmUploads(UploadConfirmRequest)                       │ │
│  │  ├── getSessionStatus(sessionId, userId)                        │ │
│  │  ├── cancelUploads(sessionId, userId, uploadIds)                │ │
│  │  ├── cleanupExpiredUploads()                                    │ │
│  │  └── purgeOldRecords(retentionDays)                             │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              PresignedUploadSessionRepository                   │ │
│  │  (JPA Repository - PostgreSQL)                                  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                   ObjectStorageService                          │ │
│  │  ├── presignPut(bucket, key, contentType, expiry)               │ │
│  │  ├── presignGet(bucket, key, expiry)                            │ │
│  │  ├── exists(bucket, key)                                        │ │
│  │  ├── existsWithMinSize(bucket, key, minSize)                    │ │
│  │  └── delete(bucket, key)                                        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                          CORE MODULE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              SuccessStoryUploadService                          │ │
│  │  (Wraps PresignedUploadService for Success Story context)       │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              MobileSuccessStoryController                       │ │
│  │  POST /mobile/success-stories/upload/presigned/init             │ │
│  │  POST /mobile/success-stories/upload/presigned/confirm          │ │
│  │  GET  /mobile/success-stories/upload/presigned/status/{id}      │ │
│  │  DELETE /mobile/success-stories/upload/presigned/cancel/{id}    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              ImageProcessingConsumer (RabbitMQ)                 │ │
│  │  - Resize images (thumbnail, medium)                            │ │
│  │  - Compress original                                            │ │
│  │  - Upload processed images back to S3                           │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Sequence Diagrams

### 1. Initialize Presigned Upload

```
┌────────┐          ┌─────────────┐          ┌──────────────────┐          ┌─────┐
│ Mobile │          │  Controller │          │ PresignedUpload  │          │ S3  │
│ Client │          │             │          │    Service       │          │     │
└───┬────┘          └──────┬──────┘          └────────┬─────────┘          └──┬──┘
    │                      │                          │                       │
    │  POST /upload/       │                          │                       │
    │  presigned/init      │                          │                       │
    │  {files: [...]}      │                          │                       │
    │─────────────────────►│                          │                       │
    │                      │                          │                       │
    │                      │  initializeUpload()      │                       │
    │                      │─────────────────────────►│                       │
    │                      │                          │                       │
    │                      │                          │  Generate S3 Key      │
    │                      │                          │──────────┐            │
    │                      │                          │          │            │
    │                      │                          │◄─────────┘            │
    │                      │                          │                       │
    │                      │                          │  presignPut()         │
    │                      │                          │──────────────────────►│
    │                      │                          │                       │
    │                      │                          │  Presigned PUT URL    │
    │                      │                          │◄──────────────────────│
    │                      │                          │                       │
    │                      │                          │  Save to DB           │
    │                      │                          │  (status: PENDING)    │
    │                      │                          │──────────┐            │
    │                      │                          │          │            │
    │                      │                          │◄─────────┘            │
    │                      │                          │                       │
    │                      │  BatchPresignedUpload    │                       │
    │                      │  Response                │                       │
    │                      │◄─────────────────────────│                       │
    │                      │                          │                       │
    │  {sessionId,         │                          │                       │
    │   uploads: [         │                          │                       │
    │     {uploadId,       │                          │                       │
    │      uploadUrl,      │                          │                       │
    │      s3Key,          │                          │                       │
    │      expiresAt}      │                          │                       │
    │   ]}                 │                          │                       │
    │◄─────────────────────│                          │                       │
    │                      │                          │                       │
```

### 2. Direct Upload to S3

```
┌────────┐                                                              ┌─────┐
│ Mobile │                                                              │ S3  │
│ Client │                                                              │     │
└───┬────┘                                                              └──┬──┘
    │                                                                      │
    │  PUT {presigned_url}                                                 │
    │  Content-Type: image/jpeg                                            │
    │  Body: <binary image data>                                           │
    │─────────────────────────────────────────────────────────────────────►│
    │                                                                      │
    │                                                                      │
    │  HTTP 200 OK                                                         │
    │◄─────────────────────────────────────────────────────────────────────│
    │                                                                      │
    │  (Upload complete - file now in S3)                                  │
    │                                                                      │
```

### 3. Confirm Upload

```
┌────────┐          ┌─────────────┐          ┌──────────────────┐          ┌─────┐
│ Mobile │          │  Controller │          │ PresignedUpload  │          │ S3  │
│ Client │          │             │          │    Service       │          │     │
└───┬────┘          └──────┬──────┘          └────────┬─────────┘          └──┬──┘
    │                      │                          │                       │
    │  POST /upload/       │                          │                       │
    │  presigned/confirm   │                          │                       │
    │  {sessionId,         │                          │                       │
    │   uploadIds: [...]}  │                          │                       │
    │─────────────────────►│                          │                       │
    │                      │                          │                       │
    │                      │  confirmUploads()        │                       │
    │                      │─────────────────────────►│                       │
    │                      │                          │                       │
    │                      │                          │  existsWithMinSize()  │
    │                      │                          │──────────────────────►│
    │                      │                          │                       │
    │                      │                          │  true/false           │
    │                      │                          │◄──────────────────────│
    │                      │                          │                       │
    │                      │                          │  Update DB            │
    │                      │                          │  (status: CONFIRMED)  │
    │                      │                          │──────────┐            │
    │                      │                          │          │            │
    │                      │                          │◄─────────┘            │
    │                      │                          │                       │
    │                      │                          │  Publish Event        │
    │                      │                          │  (UploadConfirmedEvent)│
    │                      │                          │──────────┐            │
    │                      │                          │          │            │
    │                      │                          │◄─────────┘            │
    │                      │                          │                       │
    │                      │  UploadConfirmResponse   │                       │
    │                      │◄─────────────────────────│                       │
    │                      │                          │                       │
    │  {results: [         │                          │                       │
    │    {status,          │                          │                       │
    │     previewUrl}      │                          │                       │
    │  ]}                  │                          │                       │
    │◄─────────────────────│                          │                       │
    │                      │                          │                       │
```

### 4. Image Processing Flow (RabbitMQ)

```
┌──────────────────┐          ┌───────────┐          ┌────────────────┐          ┌─────┐
│ UploadConfirmed  │          │ RabbitMQ  │          │ ImageProcessing│          │ S3  │
│ EventListener    │          │           │          │ Consumer       │          │     │
└────────┬─────────┘          └─────┬─────┘          └───────┬────────┘          └──┬──┘
         │                          │                        │                      │
         │  UploadConfirmedEvent    │                        │                      │
         │  received                │                        │                      │
         │──────────┐               │                        │                      │
         │          │               │                        │                      │
         │◄─────────┘               │                        │                      │
         │                          │                        │                      │
         │  Publish to              │                        │                      │
         │  image.processing.queue  │                        │                      │
         │─────────────────────────►│                        │                      │
         │                          │                        │                      │
         │                          │  Consume message       │                      │
         │                          │───────────────────────►│                      │
         │                          │                        │                      │
         │                          │                        │  Download original   │
         │                          │                        │─────────────────────►│
         │                          │                        │                      │
         │                          │                        │  Image bytes         │
         │                          │                        │◄─────────────────────│
         │                          │                        │                      │
         │                          │                        │  Generate thumbnail  │
         │                          │                        │  Generate medium     │
         │                          │                        │  Compress original   │
         │                          │                        │──────────┐           │
         │                          │                        │          │           │
         │                          │                        │◄─────────┘           │
         │                          │                        │                      │
         │                          │                        │  Upload processed    │
         │                          │                        │  images              │
         │                          │                        │─────────────────────►│
         │                          │                        │                      │
         │                          │                        │  Success             │
         │                          │                        │◄─────────────────────│
         │                          │                        │                      │
         │                          │  ACK                   │                      │
         │                          │◄───────────────────────│                      │
         │                          │                        │                      │
```

### 5. Cleanup Flow (Scheduled Jobs)

```
┌──────────────────┐          ┌──────────────────┐          ┌────────────┐          ┌─────┐
│ ApplicationSchedulers      │ PresignedUpload  │          │ PostgreSQL │          │ S3  │
│ (Cron Jobs)      │          │    Service       │          │            │          │     │
└────────┬─────────┘          └────────┬─────────┘          └─────┬──────┘          └──┬──┘
         │                             │                          │                    │
         │  Every Hour                 │                          │                    │
         │  cleanupExpiredUploads()    │                          │                    │
         │────────────────────────────►│                          │                    │
         │                             │                          │                    │
         │                             │  Find PENDING            │                    │
         │                             │  where expired           │                    │
         │                             │─────────────────────────►│                    │
         │                             │                          │                    │
         │                             │  List<PresignedUpload>   │                    │
         │                             │◄─────────────────────────│                    │
         │                             │                          │                    │
         │                             │  For each: delete S3     │                    │
         │                             │─────────────────────────────────────────────►│
         │                             │                          │                    │
         │                             │  Mark as EXPIRED         │                    │
         │                             │─────────────────────────►│                    │
         │                             │                          │                    │
         │  Cleaned count              │                          │                    │
         │◄────────────────────────────│                          │                    │
         │                             │                          │                    │
         │                             │                          │                    │
         │  Daily at 2 AM             │                          │                    │
         │  purgeOldRecords(3)         │                          │                    │
         │────────────────────────────►│                          │                    │
         │                             │                          │                    │
         │                             │  DELETE WHERE            │                    │
         │                             │  status=EXPIRED AND      │                    │
         │                             │  updatedAt < now-3days   │                    │
         │                             │─────────────────────────►│                    │
         │                             │                          │                    │
         │                             │  DELETE WHERE            │                    │
         │                             │  status=CANCELLED AND    │                    │
         │                             │  updatedAt < now-3days   │                    │
         │                             │─────────────────────────►│                    │
         │                             │                          │                    │
         │  Purged count               │                          │                    │
         │◄────────────────────────────│                          │                    │
         │                             │                          │                    │
```

---

## Upload Status State Machine

```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    │              PENDING                    │
                    │     (Presigned URL generated,           │
                    │      waiting for upload)                │
                    │                                         │
                    └──────────────┬──────────────────────────┘
                                   │
                    ┌──────────────┼──────────────────────────┐
                    │              │                          │
                    ▼              ▼                          ▼
    ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
    │                   │  │                   │  │                   │
    │     UPLOADED      │  │     EXPIRED       │  │    CANCELLED      │
    │  (S3 upload done, │  │  (URL expired,    │  │  (User cancelled) │
    │   not confirmed)  │  │   file deleted)   │  │                   │
    │                   │  │                   │  │                   │
    └─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘
              │                      │                      │
              ▼                      │                      │
    ┌───────────────────┐            │                      │
    │                   │            │                      │
    │    CONFIRMED      │            │                      │
    │  (Verified in S3, │            │                      │
    │   permanent)      │            │                      │
    │                   │            ▼                      ▼
    └───────────────────┘     ┌─────────────────────────────────┐
                              │                                 │
                              │     HARD DELETED (after 3 days) │
                              │     (Removed from database)     │
                              │                                 │
                              └─────────────────────────────────┘
```

---

## Database Schema

### Table: `presigned_upload_sessions`

```sql
CREATE TABLE presigned_upload_sessions (
    id                      VARCHAR(36) PRIMARY KEY,
    session_id              VARCHAR(36) NOT NULL,
    user_id                 VARCHAR(100) NOT NULL,
    s3_key                  VARCHAR(500) NOT NULL,
    bucket_name             VARCHAR(100),
    original_filename       VARCHAR(255),
    content_type            VARCHAR(100),
    expected_size           BIGINT,
    actual_size             BIGINT,
    upload_type             VARCHAR(50),
    status                  VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    presigned_url_expires_at TIMESTAMP WITH TIME ZONE,
    uploaded_at             TIMESTAMP WITH TIME ZONE,
    confirmed_at            TIMESTAMP WITH TIME ZONE,
    reference_id            VARCHAR(36),
    reference_type          VARCHAR(50),
    metadata                JSONB,
    created_at              TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_presigned_session_id ON presigned_upload_sessions(session_id);
CREATE INDEX idx_presigned_user_id ON presigned_upload_sessions(user_id);
CREATE INDEX idx_presigned_status ON presigned_upload_sessions(status);
CREATE INDEX idx_presigned_expires_at ON presigned_upload_sessions(presigned_url_expires_at);
CREATE INDEX idx_presigned_reference ON presigned_upload_sessions(reference_id, reference_type);
```

---

## API Endpoints

### 1. Initialize Presigned Upload

```http
POST /api/v1/core/mobile/success-stories/upload/presigned/init
Content-Type: application/json
Authorization: Bearer {jwt_token}

Request:
{
  "sessionId": "optional-custom-session-id",
  "files": [
    {
      "filename": "before_photo.jpg",
      "contentType": "image/jpeg",
      "fileSize": 1024000,
      "folder": "success-stories/photos",
      "metadata": {
        "angleId": "FRONT"
      }
    }
  ],
  "expirySeconds": 3600
}

Response:
{
  "success": true,
  "data": {
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "uploadType": "SUCCESS_STORY",
    "uploads": [
      {
        "uploadId": "660e8400-e29b-41d4-a716-446655440001",
        "s3Key": "success-stories/photos/user123/20241226/abc12345-before_photo.jpg",
        "uploadUrl": "https://bucket.s3.amazonaws.com/...?X-Amz-Signature=...",
        "filename": "before_photo.jpg",
        "contentType": "image/jpeg",
        "expiresAt": "2024-12-26T11:00:00Z",
        "metadata": {
          "angleId": "FRONT"
        }
      }
    ],
    "expiresAt": "2024-12-26T11:00:00Z",
    "expirySeconds": 3600,
    "totalFiles": 1
  }
}
```

### 2. Confirm Upload

```http
POST /api/v1/core/mobile/success-stories/upload/presigned/confirm
Content-Type: application/json
Authorization: Bearer {jwt_token}

Request:
{
  "sessionId": "550e8400-e29b-41d4-a716-446655440000",
  "uploadIds": [
    "660e8400-e29b-41d4-a716-446655440001"
  ],
  "referenceId": "story-id-123",
  "referenceType": "SUCCESS_STORY"
}

Response:
{
  "success": true,
  "data": {
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "results": [
      {
        "uploadId": "660e8400-e29b-41d4-a716-446655440001",
        "s3Key": "success-stories/photos/user123/20241226/abc12345-before_photo.jpg",
        "filename": "before_photo.jpg",
        "contentType": "image/jpeg",
        "status": "CONFIRMED",
        "fileSize": 1024000,
        "previewUrl": "https://bucket.s3.amazonaws.com/...?X-Amz-Signature=...",
        "metadata": {
          "angleId": "FRONT"
        }
      }
    ],
    "totalConfirmed": 1,
    "totalFailed": 0,
    "message": "All 1 upload(s) confirmed successfully"
  }
}
```

### 3. Get Session Status

```http
GET /api/v1/core/mobile/success-stories/upload/presigned/status/{sessionId}
Authorization: Bearer {jwt_token}

Response:
{
  "success": true,
  "data": {
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "uploads": [
      {
        "uploadId": "660e8400-e29b-41d4-a716-446655440001",
        "s3Key": "...",
        "filename": "before_photo.jpg",
        "status": "CONFIRMED",
        "previewUrl": "https://...",
        "fileSize": 1024000,
        "expiresAt": "2024-12-26T11:00:00Z",
        "uploadedAt": "2024-12-26T10:05:00Z"
      }
    ],
    "totalPending": 0,
    "totalUploaded": 0,
    "totalConfirmed": 1,
    "totalExpired": 0
  }
}
```

### 4. Cancel Upload

```http
DELETE /api/v1/core/mobile/success-stories/upload/presigned/cancel/{sessionId}
Authorization: Bearer {jwt_token}

Response:
{
  "success": true,
  "message": "Uploads cancelled successfully"
}
```

---

## Scheduled Jobs

| Job Name | Cron Expression | Description |
|----------|-----------------|-------------|
| `cleanupExpiredUploads` | `0 0 * * * *` (every hour) | Mark expired PENDING uploads as EXPIRED, delete from S3 |
| `purgeOldUploadRecords` | `0 0 2 * * *` (daily 2 AM) | Hard delete EXPIRED/CANCELLED records older than 3 days |

---

## Configuration Properties

```yaml
# application.yml

# S3 Configuration
aws:
  s3:
    bucket-name: akasia-uploads
    presigned-url-expiry-seconds: 3600  # 1 hour
    min-file-size: 1024  # 1 KB minimum

# Scheduler Configuration
schedulers:
  upload-cleanup:
    cron: "0 0 * * * *"  # Every hour
  upload-purge:
    cron: "0 0 2 * * *"  # Daily at 2 AM

# RabbitMQ Configuration
rabbitmq:
  image-processing:
    queue: image.processing.queue
    exchange: image.processing.exchange
    routing-key: image.processing
    dlq: image.processing.dlq
```

---

## File Structure

```
akasia365mc-api/
├── common/
│   └── src/main/java/com/akasia/common/storage/
│       ├── ObjectStorageService.java              # Interface for S3 operations
│       ├── S3ObjectStorageService.java            # S3 implementation
│       └── presigned/
│           ├── PresignedUploadService.java        # Internal service interface
│           ├── impl/
│           │   └── PresignedUploadServiceImpl.java # Internal service implementation
│           ├── api/                               # ★ API Layer (use this!)
│           │   ├── PresignedUploadApiService.java # API service interface
│           │   ├── PresignedUploadApiServiceImpl.java
│           │   ├── PresignedUploadInitRequest.java    # API DTOs
│           │   ├── PresignedUploadInitResponse.java
│           │   ├── PresignedUploadFileRequest.java
│           │   ├── PresignedUploadFileResponse.java
│           │   ├── PresignedUploadConfirmRequest.java
│           │   ├── PresignedUploadConfirmResponse.java
│           │   ├── PresignedUploadConfirmResult.java
│           │   ├── PresignedUploadStatusResponse.java
│           │   └── PresignedUploadCancelRequest.java
│           ├── dto/                               # Internal DTOs
│           │   ├── BatchPresignedUploadRequest.java
│           │   ├── BatchPresignedUploadResponse.java
│           │   └── ...
│           ├── entity/
│           │   └── PresignedUploadSession.java
│           ├── repository/
│           │   └── PresignedUploadSessionRepository.java
│           ├── config/
│           │   └── PresignedUploadAutoConfiguration.java
│           └── event/
│               ├── UploadConfirmedEvent.java
│               └── UploadEventPublisher.java
│
├── core/
│   └── src/main/java/com/akasia/core/
│       ├── scheduler/
│       │   └── ApplicationSchedulers.java         # Cleanup jobs
│       └── successstory/
│           ├── controller/
│           │   └── MobileSuccessStoryController.java  # Uses PresignedUploadApiService
│           ├── service/
│           │   ├── ImageProcessingPublisher.java
│           │   └── impl/
│           │       └── ImageProcessingPublisherImpl.java
│           ├── consumer/
│           │   └── ImageProcessingConsumer.java   # RabbitMQ consumer
│           ├── config/
│           │   └── ImageProcessingAmqpConfig.java
│           ├── event/
│           │   └── SuccessStoryUploadEventListener.java
│           └── dto/
│               └── presigned/
│                   └── ImageProcessingMessage.java  # RabbitMQ message only
│
└── docs/
    └── S3PresignedDocs.md
```

---

## Benefits of This Architecture

1. **Reduced Backend Load**: Files upload directly to S3, not through the API server
2. **Better Performance**: No proxy overhead for large file uploads
3. **Auto Cleanup**: Expired uploads are automatically cleaned from S3 and database
4. **Scalability**: Horizontal scaling without file upload bottlenecks
5. **Security**: Presigned URLs are temporary and user-specific
6. **Audit Trail**: All operations are logged via centralized audit system (see below)
7. **Async Processing**: Image processing happens in the background via RabbitMQ
8. **Reusable**: Common module can be used by any feature requiring file uploads
9. **Minimal Boilerplate**: Only 2 files needed per feature (Controller + EventListener)

---

## Adding Presigned Uploads to a New Feature

Adding presigned upload support to a new feature is now minimal. You only need:

### 1. Controller (use common API DTOs directly)

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final PresignedUploadApiService presignedUploadApiService;
    private final ProductService productService;

    @PostMapping("/upload/presigned/init")
    public ResponseEntity<ApiResponse<PresignedUploadInitResponse>> initUpload(
            @Valid @RequestBody PresignedUploadInitRequest request) {

        String userId = getCurrentUserId();

        // Optional: Use enricher for feature-specific logic
        PresignedUploadInitResponse response = presignedUploadApiService.initializeUpload(
                request,
                userId,
                (fileRequest, index) -> {
                    // Add feature-specific metadata if needed
                    if (fileRequest.getProductId() != null) {
                        Product product = productService.findById(fileRequest.getProductId());
                        fileRequest.setCategoryId(product.getCategoryId());
                        fileRequest.setFolder("products/" + product.getCategoryId());
                    }
                });

        return ResponseEntity.ok(ApiResponse.success(response));
    }

    @PostMapping("/upload/presigned/confirm")
    public ResponseEntity<ApiResponse<PresignedUploadConfirmResponse>> confirmUpload(
            @Valid @RequestBody PresignedUploadConfirmRequest request) {

        request.setReferenceType("PRODUCT");
        return ResponseEntity.ok(ApiResponse.success(
                presignedUploadApiService.confirmUploads(request, getCurrentUserId())));
    }
}
```

### 2. Event Listener (for post-processing)

```java
@Component
@RequiredArgsConstructor
public class ProductUploadEventListener {

    private final ImageProcessingPublisher imageProcessor;

    @EventListener
    public void onUploadConfirmed(UploadConfirmedEvent event) {
        if (!"PRODUCT_IMAGE".equals(event.getUploadType())) {
            return;
        }

        // Trigger image processing (resize, thumbnails, etc.)
        for (var upload : event.getConfirmedUploads()) {
            imageProcessor.publishForProcessing(
                    upload.getUploadId(),
                    upload.getS3Key(),
                    upload.getContentType());
        }
    }
}
```

### Available Optional Fields in API DTOs

The API DTOs include common optional fields for different features:

| Field | Type | Use Case |
|-------|------|----------|
| `angleId` | String | Success story photo angles |
| `bodyPartId` | String | Body part references |
| `productId` | String | Product image uploads |
| `categoryId` | String | Categorized uploads |
| `position` | Integer | Image ordering/sequence |
| `metadata` | Map | Custom key-value pairs |

---

## Audit Trail

All presigned upload operations are tracked via the centralized audit system using the `@Audit` annotation.

### How It Works

```
┌─────────────┐     ┌─────────────┐     ┌───────────┐     ┌─────────────────┐
│ Controller  │────►│ AuditAspect │────►│ RabbitMQ  │────►│ AuditMessage    │
│ @Audit      │     │ (AOP)       │     │           │     │ Listener        │
└─────────────┘     └─────────────┘     └───────────┘     └────────┬────────┘
                                                                    │
                                                                    ▼
                                                          ┌─────────────────┐
                                                          │   audit_logs    │
                                                          │   (PostgreSQL)  │
                                                          └─────────────────┘
```

### Audited Actions

| Action | Description |
|--------|-------------|
| `INIT_PRESIGNED_UPLOAD` | Generate presigned PUT URLs |
| `CONFIRM_PRESIGNED_UPLOAD` | Verify uploads are complete in S3 |
| `CANCEL_PRESIGNED_UPLOAD` | Cancel pending uploads |
| `CREATE_STORY` | Create success story with photos |
| `UPDATE_STORY` | Update existing story |
| `DELETE_STORY` | Delete story |
| `LIKE_STORY` | Like a story |
| `UNLIKE_STORY` | Unlike a story |
| `CREATE_PLACEHOLDER_ICON` | Create placeholder icon |
| `UPDATE_PLACEHOLDER_ICON` | Update placeholder icon |
| `DELETE_PLACEHOLDER_ICON` | Delete placeholder icon |

### Audit Log Fields

```sql
-- audit_logs table (utility module)
id              VARCHAR(36) PRIMARY KEY
occurred_at     TIMESTAMP WITH TIME ZONE
username        VARCHAR         -- User who performed action
action          VARCHAR         -- e.g., INIT_PRESIGNED_UPLOAD
resource        VARCHAR         -- API path
http_method     VARCHAR         -- GET, POST, PUT, DELETE
client_ip       VARCHAR         -- Client IP address
device          TEXT            -- User-Agent
before_json     TEXT            -- Request payload (sanitized)
after_json      TEXT            -- Response payload (sanitized)
event_type      VARCHAR         -- CHANGES, USER_ACCESS, SYSTEM
module          VARCHAR         -- e.g., SUCCESS_STORY
status          VARCHAR         -- SUCCESS or ERROR
error_message   TEXT            -- Error details if failed
duration_ms     BIGINT          -- Request duration
```

### Usage Example

```java
@PostMapping("/upload/presigned/init")
@Audit(menu = "SUCCESS_STORY", action = "INIT_PRESIGNED_UPLOAD")
public ResponseEntity<ApiResponse<PresignedUploadInitResponse>> initPresignedUpload(...) {
    // ...
}
```

---

## Security Considerations

1. **Presigned URL Expiry**: Default 1 hour, configurable
2. **User Validation**: Each operation validates user ownership
3. **File Validation**: Confirm checks file exists and meets minimum size
4. **Content Type Restriction**: Only allowed content types can be uploaded
5. **S3 Key Isolation**: Each user's files are in separate S3 prefixes

---

## Monitoring & Metrics

Recommended metrics to track:
- `presigned.uploads.initialized` - Count of initialized uploads
- `presigned.uploads.confirmed` - Count of confirmed uploads
- `presigned.uploads.expired` - Count of expired uploads
- `presigned.uploads.purged` - Count of purged records
- `presigned.uploads.processing.success` - Image processing success
- `presigned.uploads.processing.failed` - Image processing failures

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2024-12-26 | Initial implementation |
| 1.0.1 | 2024-12-26 | Added hard delete for expired records (3 days retention) |
| 1.0.2 | 2024-12-26 | Added @Audit annotations for audit trail integration |
| 2.0.0 | 2024-12-26 | **Major refactor**: Moved API DTOs to common module, added PresignedUploadApiService facade. Features now only need Controller + EventListener (2 files instead of ~13). Removed duplicate DTOs from core module. |
