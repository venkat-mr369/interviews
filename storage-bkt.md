**Google Cloud Storage** is a fully managed, highly scalable service for storing unstructured data—such as files, images, backups, and large datasets—on Google Cloud. It is designed for durability, availability, and performance, and is widely used for everything from website hosting to data analytics and machine learning workloads[1][2][3][4].

## Core Concepts

- **Object**: The fundamental unit of storage (a file of any format). Objects are immutable and can be up to 5 TB in size[1][5][4].
- **Bucket**: A container for objects. Each bucket belongs to a project and has a globally unique name. Buckets can be configured for location, storage class, and access control[1][3][4].
- **Project**: A grouping for resources (including buckets) under an organization[1][3].
- **Managed Folder**: Logical grouping within buckets, allowing for more granular access control[1].
- **Storage Classes**: Different options for cost and availability, including Standard, Nearline, Coldline, and Archive[5][3].

## Storage Hierarchy Example

| Level         | Example Name      | Description                                   |
|---------------|------------------|-----------------------------------------------|
| Organization  | exampleinc.org   | Your company’s Google Cloud organization      |
| Project       | my-app-project   | Each application or workload                  |
| Bucket        | photos, videos   | Containers for storing objects                |
| Object        | puppy.png        | Individual files stored in buckets            |
| Managed Folder| animals/         | Logical folders within a bucket               |

## Common gcloud Storage Commands

The `gcloud storage` command group is used to manage buckets and objects from the command line[6][7][8].

### 1. **Create a Bucket**
```sh
gcloud storage buckets create gs://my-bucket --location=US
```
- Replace `my-bucket` with your unique bucket name.
- `--location=US` specifies the geographic location[9].

**With more options:**
```sh
gcloud storage buckets create gs://my-bucket \
  --project=my-project \
  --default-storage-class=STANDARD \
  --location=US \
  --uniform-bucket-level-access \
  --soft-delete-duration=10d
```
- Adds project, storage class, access control, and retention settings[9].

### 2. **List Buckets**
```sh
gcloud storage ls
```
- Lists all buckets in your current project[10].

### 3. **Upload a File (Object)**
```sh
gcloud storage cp Desktop/dog.png gs://my-bucket
```
- Copies `dog.png` from your local desktop to the bucket[11][8].

### 4. **List Objects in a Bucket**
```sh
gcloud storage ls gs://my-bucket
```
- Lists all objects in `my-bucket`[8].

### 5. **Download a File (Object)**
```sh
gcloud storage cp gs://my-bucket/dog.png ./dog.png
```
- Downloads `dog.png` from the bucket to your current directory[8].

### 6. **Delete a File (Object)**
```sh
gcloud storage rm gs://my-bucket/dog.png
```
- Removes `dog.png` from the bucket[8].

### 7. **Delete a Bucket**
```sh
gcloud storage buckets delete gs://my-bucket
```
- Deletes the specified bucket (must be empty first).

## Additional Features

- **Access Control**: Set permissions at bucket or object level using IAM or ACLs[5][3].
- **Object Lifecycle Management**: Define rules for automatic deletion or transition to cheaper storage classes[3].
- **Versioning**: Enable to keep old versions of objects[3].
- **Batch Operations**: Perform actions on millions of objects at scale[2].
- **Hierarchical Namespace**: Buckets can be configured to support a logical file system structure, useful for analytics and ML workloads[1].

## Example Workflow

1. **Create a bucket:**
   ```sh
   gcloud storage buckets create gs://my-sample-bucket --location=US
   ```
2. **Upload a file:**
   ```sh
   gcloud storage cp report.pdf gs://my-sample-bucket
   ```
3. **List files in the bucket:**
   ```sh
   gcloud storage ls gs://my-sample-bucket
   ```
4. **Download a file:**
   ```sh
   gcloud storage cp gs://my-sample-bucket/report.pdf ./report.pdf
   ```
5. **Delete a file:**
   ```sh
   gcloud storage rm gs://my-sample-bucket/report.pdf
   ```
6. **Delete the bucket:**
   ```sh
   gcloud storage buckets delete gs://my-sample-bucket
   ```

**Note:** Google recommends using `gcloud storage` over the older `gsutil` for new workflows[7][8].

For a full list of commands and options, use:
```sh
gcloud storage --help
```
or refer to the official documentation[6].

[1] https://cloud.google.com/storage/docs/introduction
[2] https://cloud.google.com/storage
[3] https://k21academy.com/google-cloud/google-cloud-storage/
[4] https://support.google.com/cloud/answer/6250993
[5] https://en.wikipedia.org/wiki/Google_Cloud_Storage
[6] https://cloud.google.com/sdk/gcloud/reference/storage
[7] https://support.terra.bio/hc/en-us/articles/6453490899099-gcloud-storage-tutorial
[8] https://parallelworks.com/docs/storage/transferring-data/google-cloud-storage-buckets
[9] https://cloud.google.com/storage/docs/creating-buckets
[10] https://cloud.google.com/storage/docs/listing-buckets
[11] https://cloud.google.com/storage/docs/uploading-objects
[12] https://cloud.google.com/learn/what-is-cloud-storage
[13] https://support.google.com/googleone/answer/9312312
[14] https://console.cloud.google.com/marketplace/product/google-cloud-platform/cloud-storage
[15] https://cloud.google.com/sdk/gcloud
[16] https://www.googlecloudcommunity.com/gc/Community-Hub/syntax-for-gcloud-storage-command/td-p/792786
[17] https://cloud.google.com/sdk/docs/cheatsheet
[18] https://www.economize.cloud/blog/how-to-use-google-cloud-shell/
[19] https://blog.invgate.com/google-drive-vs-google-cloud-storage
[20] https://www.cloudthat.com/resources/blog/google-cloud-storage-options-and-overview
