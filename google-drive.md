# Google Drive System Design

Google drive is a file storage and synchronization service, enables users to store their data on a remote server.


### Functional and Non-functional Requirements

Functional Requirements
1. Users should be able to upload and download files/photos from any device

Non-functional Requirements
1. The system should be highly available
2. Minimum latency
3. Strong consistency


Our user in this system can upload and download files. The user uploads files from the client application/browser, and the server will store them. And user can download updated files from the server. So, let’s see how we handle upload and download of files for such a massive amount of users.

### Upload/Download Files

If we upload the file with full size, it will cost us storage and bandwidth. And also, latency will be increased to complete upload or download.  
So its better to drive each file into smaller chunks. Then we can modify only small pieces where data is changed, not the whole file. 
In case data upload failure also this strategy will help.  If a chunk is not uploaded, then only the failing chunk will be retried.

### What will happen when the client is offline?
A client component, Watcher, will observe client-side folders. 
If any change occurs by the user, it will notify the Index Controller(another client component) about the action of the user. 
It will also monitor if any change is happening on other clients(devices), which are broadcasted by the Notification server.

When the Metadata service receives an update/upload request, it needs to check with the metadata DB for consistency and then proceed with the update. After that, a notification will be sent to all subscribed devices to report the file update.


### Metadata Database:
We need a database that is responsible for keeping information about files, users, etc. 
It can be a relational database like MySQL or NoSQL like MongoDB. We need to save data like chunks, files, user information, etc. in the Database.


### Synchronization:
Now the client updates a file from a device; there needs to be a component that process updates and applies the change to other devices. It needs to sync the client’s local Database and remote Metadata DB. MetaData server can perform the job to manage metadata and synchronize the user’s files.


### Message Queue:
If we have a huge amount of users who are uploading files simultaneously, we may use a message queue between client and server.

The message queue provides temporary message storage when the destination program is busy or not connected. It provides an asynchronous communications protocol. It is a system that puts a message onto a queue and does not require an immediate response to continue processing. RabbitMQ, Apache Kafka, etc. are some of the examples of the messaging queue.


### Cloud Storage
It stores files(chunks) uploaded by the users. Clients can interact with the storage through File Processing Server to send and receive objects from it. It holds only the files; Metadata DB keeps the data of the chunk size and numbers of a file.


### File processing Workflow
Client A uploads chunk to cloud storage. Client A updates metadata and commits changes in MetadataDB using the Metadata server. The client gets confirmation, and notifications are sent to other devices of the same user. Other devices receive metadata changes and download updated chunks from cloud storage.

